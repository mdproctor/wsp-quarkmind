---
layout: post
title: "Two Fixes, Finally Together"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [QuarkMind]
---

I started the session expecting to begin work on train-timing precision. Within two
minutes it was clear the work was already done, closed, and merged. My notes from
the last session were wrong about the state of things.

The real situation: the saturation mining epic had four commits sitting on a branch
that had never made it to main. The train-timing fixes from the session after that
had landed on main just fine — nine commits, no problem — but the saturation model
was quietly stranded. Design journal. Protocol entry. Spec. The implementation
itself. All four needed a rebase onto main before they could do anything.

The rebase surfaced why both branches had touched `SC2Data.java`. The train-timing
work added `LOOPS_PER_TICK`, `GAME_LOOPS_PER_SECOND`, and `trainTimeInLoops` so
sub-tick train completion could be computed correctly. The saturation model had been
written before those constants existed, so the tier rates used hardcoded literals:

```java
50.0 / 60.0 * 22.0 / 22.4,   // first worker per patch
25.0 / 60.0 * 22.0 / 22.4,   // second worker per patch
 5.0 / 60.0 * 22.0 / 22.4,   // third worker per patch
```

The conflict resolution kept both sets of additions and replaced the literals with
the constants that now existed:

```java
50.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,
25.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,
 5.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,
```

If the tick constants change, there is now one place to update instead of three.

The other conflicts were lighter. `ReplayValidationHarness.java` had the probe count
multiplier: the old flat-rate model needed `countProbes(gtBefore) * SC2Data.LOOPS_PER_TICK`
because the per-probe rate was calibrated per game loop, not per outer tick. The
saturation model handles the per-tick rate internally, so the multiplication goes
away. `ReplayValidationTest.java` had two independent sets of doc changes — one from
each branch — which merged into a single description of the combined state.

672 tests, no failures.

With both fixes on main together, the replay validation numbers tell the complete
story. The sub-tick timing fix moved `firstUnitDivergenceTick` from 36 to 86,
correcting the systematic 1-tick-early completions when a train command arrived
late in a tick window. The saturation model dropped `maxMineralDelta` from 11,564
to around 850 over three minutes, replacing the flat per-probe rate with a three-tier
model that reflects how SC2 mineral collection actually works.

Neither fix achieves the original target of zero divergence. The remaining gap is
building costs — we inject buildings into the harness for free, so the emulated
bank drifts above ground truth regardless of how accurate the mining model is. A
separate problem.
