---
layout: post
title: "Convergence Is Not Correctness"
date: 2026-08-12
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [cbr, scouting, game-ai]
---

The goal was simple: at game close, compare what the scouting system predicted early against what actually happened, and store the result as a CBR feature. Future games could then query: "in past games where our early scouting was reliable, which strategy worked best?"

The original issue proposed `posture_correct` and `scouting_accuracy` as feature names — implying the bot would know whether its scouting was *right*. During the design phase, that framing fell apart.

The problem is ground truth. At game close, the scouting system has a final `PatternAssessment` — its best guess after observing the full game. But that assessment is itself an estimate, produced by the same classifier that made the early prediction. Comparing the classifier's first answer against its last answer measures *consistency*, not accuracy. A scouting system that consistently misidentifies macro play as harass would score perfectly.

True correctness would require a composition-derived reverse-classifier — looking at the enemy's actual units and buildings and inferring what strategy they represent. That's a different feature entirely, and one that doesn't exist yet.

So the feature was renamed to reflect what it actually measures: `scouting_convergence` (0.0–1.0) and `assessment_stable` (boolean). The signal is still useful for CBR retrieval — games where the early read was stable are games where the strategy choice was well-informed, regardless of whether the read was *correct* in an absolute sense.

The scoring model went through its own evolution. The first design was phase-aware: if the early assessment was EARLY-phase and the final was MID-phase, don't penalise — that's normal game progression. Review caught two problems: the rule had no race guard (a Terran rush and a Zerg rush share the RUSH category, so cross-race mismatches scored 0.5), and it only covered EARLY→MID transitions, not MID→LATE. The fix was simpler than the original: same race AND same `ArchetypeCategory` → 0.5, regardless of phase. Phase transitions are handled naturally because category match doesn't require phase match.

The subtler bug was the pivot-collapse problem. `STRATEGY_ROUTED_ARCHETYPE` — the archetype used for strategy selection — gets overwritten when the bot pivots. After a pivot, the routed archetype *is* the latest assessment, so comparing them produces 1.0 by construction. The comparison is meaningless in exactly the games where scouting changed its mind — the most interesting cases for learning. The fix: a new `STRATEGY_INITIAL_ARCHETYPE` CaseFile key, written once on first routing and never overwritten. The convergence score compares this initial read against the final assessment.

The implementation is three pieces: a pure-function `ScoutingConvergenceEvaluator` (category-aware tiered scoring with a race guard), two CaseFile keys (write-once initial archetype, unconditional per-tick final assessment), and the enrichment wiring in `SC2CbrRetentionObserver`. The features are stored in CBR cases but not yet wired into retrieval queries — query integration is a follow-up when the feature has enough data to be meaningful.

The convergence-not-correctness distinction matters beyond naming. It's a constraint on how the feature should be interpreted downstream. A CBR query filtered by `assessment_stable = true` is asking "give me games where scouting was internally consistent" — not "give me games where scouting was right." If a composition-derived classifier arrives later, it could produce a true `scouting_accuracy` feature that complements convergence rather than replacing it. Both signals are useful: convergence tells you about the classifier's confidence trajectory, correctness tells you about its calibration.
