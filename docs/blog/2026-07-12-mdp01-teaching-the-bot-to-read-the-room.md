---
layout: post
title: "Teaching the Bot to Read the Room"
date: 2026-07-12
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [scouting, drools, pattern-classification, emulated-game]
---

## Teaching the Bot to Read the Room

The scouting layer could see things — Marines here, Roaches there, no expansion yet. What it couldn't do was synthesise those observations into a judgment. Seeing six Marines at the three-minute mark and seeing no expansion are two separate facts. Knowing that combination means a rush is a different kind of intelligence.

We built a confidence-weighted pattern classifier that runs alongside the existing build-order detection rules. Same Drools CEP infrastructure, same `ScoutingSessionManager` event buffers, but a second `RuleUnit` firing after the first. Each rule emits an `EvidenceMarker` — a weighted signal for a specific archetype. "Six Marines before four minutes" is 0.6 weight toward `TERRAN_MARINE_RUSH`. "No expansion with Marines present" adds another 0.3. The Java layer aggregates using `1 - ∏(1 - weight_i)` — the probability-of-at-least-one formula — so two independent 0.5 signals produce 0.75 confidence, not 1.0.

The classifier covers ten archetypes across all three SC2 races. Eighteen DRL rules. The design review caught something I'd missed: `gameTimeMin` can't be a Drools global because `eval()` + `accumulate()` in the same rule doesn't compile in the current Drools version. The fix — making it a single-element `DataStore<Double>` — is the same workaround already used in `StrategyRuleUnit` for posture and timing. Consistency with the existing pattern, but I wouldn't have caught the bug until runtime.

Confidence is monotonic — it only goes up. Seeing Marines doesn't become less meaningful over time. This is a deliberate MVP trade-off: if the enemy pivots, the old classification persists until the new one accumulates past it. Decay and counter-indication are tracked in a follow-up issue, but for early warning — which is the primary value — monotonic maximises sensitivity.

The strategy layer consumes the assessment at salience 215, slotting between Nexus Under Attack (220) and Timing Attack (210). A rush classification with 0.7+ confidence produces DEFEND. The old build-order detection (`TERRAN_3RAX`, `ZERG_ROACH_RUSH`, `PROTOSS_4GATE`) stays as a calibration benchmark until the replay validation confirms the new classifier matches or exceeds it.

While we were in `EmulatedGame.handleTrain()`, we also fixed a Zergling batch cost bug that the design review surfaced: the method deducted 25 minerals per `TrainIntent` but spawned two Zerglings. The actual SC2 cost is 50 for the pair. The same reviewer caught that the enemy training path had a hardcoded spawn count of 1 — after the cost fix, enemies would pay 50 minerals but only get one Zergling.

The interesting question is what the classifier enables downstream. The LLM advisory team now receives structured context — `TERRAN_MARINE_RUSH (confidence 0.82)` — instead of raw unit counts. That's the difference between "there are Marines" and "you're about to be rushed." The commentator can narrate intent, not just observation.
