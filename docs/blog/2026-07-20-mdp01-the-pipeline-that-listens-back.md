---
layout: post
title: "QuarkMind — The Pipeline That Listens Back"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkMind]
tags: [coaching, compliance, cdi, qhorus, trust]
series: issue-230-coach-mode
---

*Part of a series on [#230 — Coach mode](https://github.com/casehubio/quarkmind/issues/230). Previous: [When the Coach Needs a Playbook](2026-07-19-mdp01-when-the-coach-needs-a-playbook.md).*

The design spec said "third LLM pipeline alongside advisory and commentary." That phrase hides a lot of wiring. Advisory dispatches STATUS messages — fire and forget, nobody checks whether the advice was any good. Commentary is the same — a narrator doesn't care whether you listened. Coach mode needed something different: it tells you what to do, then watches to see if you did it.

## What we believed going in: the advisory pattern would carry us

I expected coaching to be advisory with a different system prompt and a COMMAND message type instead of STATUS. Same trigger builder, same worker factory shape, same channel broker. The differences — urgency tiers with preemption, per-game personality selection, structured JSON output parsing — felt like parameterisation, not new architecture.

That turned out to be roughly right for seven of the ten components. The trigger builder, session selector, worker factory, and channel broker all followed the existing patterns closely enough that the code reads like it belongs in the same family. The domain model (`CoachingDomain`, `CoachingUrgencyTier`, `CoachingAdvice`) slotted in without friction.

## Where the coaching pipeline parts company

The compliance evaluator is the piece that has no precedent in advisory or commentary. It ticks every frame, iterates open commitments, and checks whether the player actually built the three Stalkers the coach told them to build. Verification is implicit — we never ask the player "did you do this?" We count units and buildings in the `GameState` and compare against a baseline captured when the advice was issued.

Three outcomes: ENDORSED (the player complied within the verification window), CHALLENGED (the window expired and they didn't), NEUTRAL (the advice wasn't measurably verifiable — "improve your macro" has no unit count to check). Each outcome feeds into the trust system via `CoachingEffectivenessTrustRecorder`, eventually influencing which coach personality gets selected for future games.

The `CoachingAdvice` record carries its own verification predicate — a `UnitType` or `BuildingType`, a count delta, and a window in game frames. The compact canonical form clamps the window floor to 200 frames and enforces mutual exclusion between unit and building verification. That design came straight from the spec; what the spec didn't anticipate was how cleanly the game-tick integration would divide. Coach mode gates off milestones, deferred advisory evaluation, and advisory triggers. It gates on compliance evaluation and coaching triggers. Commentary runs in both modes — a narrator doesn't care who's playing.

## The CDI proxy that ate the shared state

The integration test for end-to-end compliance — put a commitment in the broker's map, create a `GameState` with enough Stalkers, tick the evaluator, assert the commitment is gone — failed silently. The map was empty from the evaluator's perspective.

The `CoachingComplianceEvaluator` stores `broker.commitments()` in its `@Inject` constructor. Both beans are `@ApplicationScoped`, so CDI proxies them via subclassing. The proxy runs the parent's field initialiser, creating its own `ConcurrentHashMap`. When the evaluator's constructor calls `broker.commitments()`, the proxy delegates to the real bean — but the reference stored at construction time can diverge from what the proxy delegates to at runtime.

The fix is straightforward: don't cache the reference. Read `broker.commitments()` at evaluation time, or construct the evaluator manually in tests using the broker's actual map. The broader lesson is that `@ApplicationScoped` beans sharing mutable state via constructor injection are fragile in Quarkus ArC — the proxy's field initialiser creates a shadow copy that looks identical until you mutate one side and read the other.

## Where this leaves us

Six action plugins now gate on `game.mode != "coach"`. Scouting, moment detection, and summarisation run regardless — they're observation, not action. The coaching pipeline fires on the same `GameMomentType` events as advisory, but maps them to urgency tiers with preemption (crisis overrides lower-tier cooldowns) rather than to advisory roles.

The trust routing policy for coaching uses different quality floors — `coaching-effectiveness` and `response-latency` instead of `recommendation-quality` and `game-outcome`. A coach is judged on whether its advice was followed, not on whether the game was won. That's a meaningful distinction that the trust system handles naturally through its per-capability policy architecture.
