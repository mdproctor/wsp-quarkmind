---
title: "Does the Cascade See the Army?"
date: 2026-09-11
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [scouting, posture, cascade, calibration, replay]
---

# Does the Cascade See the Army?

With real enemy movement landed (#298) and posture persistence locked in (#300), the question was whether TacticalPosture — the military posture classification that feeds strategy, dominance, and commentary — actually responds to army proximity. The SpatialCalibrationTest showed PostTrn=1.0 across all 59 replays: exactly one posture transition per game. That sounds like nothing is happening. But the metric was measuring the wrong thing.

PostTrn tracks *economic* posture: ALL_IN, MACRO, UNKNOWN. It's a simplified heuristic in the test — check unit buffer, check expansion buffer, classify. The real TacticalPosture cascade runs deeper: ScoutingSessionManager detects army near base, the DRL fires a timing alert, MomentDetectionTask produces BATTLE_STARTED and NEXUS_UNDER_ATTACK moments, and GamePhaseSummariser classifies those moments into military postures like DEFENSIVE_HOLD and MID_SKIRMISH.

None of that was being measured. So I extended the test to simulate the cascade, wiring GamePhaseSummariser directly into the replay loop with a SummarisationRunner and the same 672-frame window policy the real pipeline uses. When the army buffer transitions from empty to non-empty — the same condition that makes the DRL "Timing Attack Incoming" rule fire — the test generates the BATTLE_STARTED and NEXUS_UNDER_ATTACK moments the DRL would produce. Deterministic simulation, no CDI required.

The results: 6 out of 59 replays produce DEFENSIVE_HOLD. The cascade works. Two PvT games, three PvZ, one PvP — all replays where the enemy army reaches the player's base early enough for the summarisation window to flush.

That last qualifier matters. Three replays have 225 army-near-base events but show zero TacticalPosture transitions. The window policy explains it: the SummarisationRunner needs 672 frames (30 seconds) or 5 events before it flushes a batch. The timing cascade produces only 2 events per transition — BATTLE_STARTED plus NEXUS_UNDER_ATTACK. If the army arrives with fewer than 672 frames remaining in the test's 900-tick ceiling, the window never flushes. In the real pipeline, which runs continuously, this doesn't happen. It's a test ceiling artefact, not a cascade gap.

The dataset doesn't contain true all-in games, so EARLY_AGGRESSION never fires — that requires an ALL_IN economic posture plus combat, and every replay in both the AI Arena and IEM10 sets has expansions. MID_SKIRMISH needs multiple combat events in one window, which would require the battle FSM to contribute BATTLE_ENDED alongside the timing-alert-driven BATTLE_STARTED. The calibration test doesn't simulate the battle FSM — that would need own-army value tracking — so MID_SKIRMISH remains unverified. A follow-up with rush-specific replays or the full @QuarkusTest pipeline would cover both gaps.

The cascade sees the army. That's what this set out to answer, and the answer is yes.
