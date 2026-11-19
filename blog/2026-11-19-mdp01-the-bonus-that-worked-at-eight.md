---
layout: post
title: "The Bonus That Worked at Eight Workers"
date: 2026-11-19
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, calibration, worker-simulation, saturation]
---

# The Bonus That Worked at Eight Workers

The emulated game's worker mining was 3% too fast at 8 workers. We fixed it by adding a 4.5-loop approach bonus — extra time at the mineral patch when saturation is low (workers ≤ patches). The 8-worker delta dropped from +3% to +0.4%. The 12-worker baseline held at -0.3%.

Then we recorded SC2 at 4 workers.

## The instrument-first principle

The 8-worker calibration worked with tick-level comparisons: match the aggregate mineral count at tick 304, adjust the constant until the delta is under 1%. But aggregate numbers can't tell you which phase of the mining cycle is wrong. Is the walk too fast? The mining animation too short? The deposit too long?

We added per-loop phase instrumentation to `WorkerSimulation` — a listener callback that fires at every state transition: walking to patch, mining, walking to base, depositing. Each event records the game loop. A diagnostic test captures the full cycle breakdown for every worker across 300 ticks.

The 8-worker breakdown showed two distinct clusters. Close patches (1.3 units from the town hall edge) cycle at 110 loops — 5.04 ticks. Far patches (2.3 units) cycle at 120 loops — 5.44 ticks. SC2's cadence data shows close at 4.8 ticks and far at 5.6 ticks. The EM is slightly too slow on close patches and slightly too fast on far patches. The per-patch approach docking formula partially normalises these, but the constant-per-distance correction can't get both groups right simultaneously.

This is useful but not actionable for the 8-worker case — the aggregate accuracy is good enough. The real value of per-phase instrumentation is what it reveals at other saturation levels.

## What 4 workers exposed

SC2 at 4 workers on 8 patches: 1245 minerals at tick 304. The emulated game: 1125. That's -9.6%.

The approach bonus adds 4.5 loops to every mining cycle when workers ≤ patches. At 8 workers on 8 patches, that's the right threshold — the bonus activates and slows mining by exactly the amount needed to match SC2. At 4 workers on 8 patches, the bonus also activates — same 4.5 loops per cycle. But with half the workers, each worker completes proportionally more cycles per MULE lifetime. The fixed overhead compounds: 4.5 extra loops across 60+ cycles per worker adds up to hundreds of loops of unnecessary delay.

![Mineral accuracy by worker count](images/2026-11-19-saturation-boundary.svg)

The pattern is clear. Below the bonus threshold (4w, 6w), the EM under-produces. Above it (10w, 12w), accuracy is within 2%. Right at it (8w), the bonus was calibrated.

6 workers: -5.0%. 10 workers: -1.7%. The bonus is a step function applied to a continuous gradient. SC2's mining gets faster per worker as saturation drops — there's less contention, shorter waits, more direct paths. The EM's flat bonus treats all low-saturation levels the same. At 8 it's right. At 4 it's too much.

## The fix is proportional, not threshold

The bonus needs to scale with saturation. At 4 workers on 8 patches (50% saturation), the bonus should be roughly half. At 7 workers on 8 patches (87.5%), nearly full. At 8 workers (100%), full — matching the current calibration.

A simple linear scale — `bonus × (workers / patches)` — would give 4w a bonus of 2.25 loops instead of 4.5. Whether that's enough to close the -9.6% gap is a calibration question for the next recording session. But the shape of the fix is clear: a constant that works at one point on the curve can't work at all points on the curve. The same lesson as flat-rate MULE income — a number calibrated at one condition becomes a compensation constant that hides the mechanism.

The 8-worker accuracy didn't lie. It was correct at the point where it was measured. The instrument showed us the cycle breakdown. The boundary recordings showed us where the constant breaks down. Without both, the -9.6% at 4 workers would eventually surface as a mystery delta in a multi-base economy where workers transfer between bases at varying saturation levels. Better to find it in a controlled playbook than debug it in a full game.
