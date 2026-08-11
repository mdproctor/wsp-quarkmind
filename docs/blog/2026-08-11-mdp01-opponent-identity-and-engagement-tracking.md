---
layout: post
title: "Opponent identity and the battle detection problem"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [cbr, game-ai, engagement-tracking, opponent-identity]
---

The CBR layer knows who the opponent is in emulated mode — the enemy strategy name goes straight into the case. In mock mode, a config property stamps "mock-opponent" on every game. But in real SC2 mode? "unknown". The opponent identity was a static `@ConfigProperty` set at CDI boot, and the protocol data that actually identifies the opponent arrives at runtime when the game starts.

The fix was to stop treating opponent identity as configuration. It's lifecycle data — it arrives with the game, not with the application. We enriched the `GameStarted` CDI event with the opponent's race, player type, difficulty, and player ID. `GameStateTranslator` now observes that event and stores the computed hash in an `AtomicReference` — thread-safe, per-game, no new beans needed. AI opponents get `SHA-256(RACE:DIFFICULTY)`, which gives CBR stable case partitioning across games against the same AI. Human opponents get a hashed player ID for privacy.

The more interesting design problem was engagement tracking. QuarkMind already fires `BATTLE_STARTED` moments, and the CBR layer counts them. But when we looked at what `BATTLE_STARTED` actually detects, it became clear the name is misleading. The Drools rule fires when `TimingAlert.incoming == true` — meaning scouting detected an incoming timing attack. It does not fire for player-initiated attacks, skirmishes, harassment drops, or any defensive engagement that scouting didn't classify as a timing push. Building per-engagement outcome tracking on top of it would create a systematically biased training set — CBR would only learn from a subset of battles.

The solution is army value delta with hysteresis. Track own army resource value each tick. When it drops sharply (more than 15%), open a battle window. When the value stabilises, start a quiescence timer. If stability holds for 224 frames (~10 seconds at normal speed), emit `BATTLE_ENDED` with the engagement outcome. If fighting resumes during quiescence, the timer resets and the battle continues. This detects all engagements regardless of who initiated — the scouting layer doesn't gate it.

The FSM sits in `MomentDetectionTask` alongside the existing Drools moment rules. `BATTLE_STARTED` continues to fire from `TimingAlert` for advisory and coaching triggers — it's still useful as "incoming attack detected by scouting." But engagement tracking uses the army value delta channel independently. Two different signals, two different purposes, coexisting in the same component.

At battle boundaries, we snapshot both armies and compute the trade: own units lost, enemy units lost, total resource value lost on each side. A typed `EngagementOutcome` record carries the result — unit trade ratio, WON/LOST/EVEN with a 20% margin threshold. The CBR retention observer extracts these from `BATTLE_ENDED` moments at game end and writes aggregate features: engagements won, engagements lost, overall unit trade ratio.

One thing that surfaced during this work: the CDI audit from the parent repo had flagged five classes in quarkmind as having "unnecessary CDI" because they had zero `@Inject` fields. Four of them are `@Named` strategy beans consumed via `Instance<>` by `DroolsTacticsTask`. The audit checked whether each class injects anything, but missed whether it's injected by others. Removing `@ApplicationScoped` would have silently broken CDI-based strategy selection. The methodology gap is subtle — "zero injections" sounds definitive until you remember CDI serves two directions.

The engagement tracking needs calibration against replay data. The 15% threshold and 224-frame quiescence window are reasonable starting points, but real SC2 games have engagement patterns that vary wildly by matchup and game phase. A calibration test against the replay corpus is the obvious next step — same pattern as the scouting calibration that already runs against IEM10 replays.
