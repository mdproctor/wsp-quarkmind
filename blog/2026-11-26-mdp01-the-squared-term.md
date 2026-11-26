---
layout: post
title: "The Squared Term"
date: 2026-11-26
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, economy, acceleration, physics, profiler]
series: issue-534-emulated-game-100-accuracy
---

# The Squared Term

*Continues the [100% accuracy series](2026-11-23-mdp02-the-assumption-that-wasnt.md).*

The emulated game's mining cycle was 100 loops. SC2's was 110. Ten loops doesn't sound like much — until you compound it over 304 ticks across 12 workers. That's a 12.8% mineral overshoot, enough to train two extra probes and cascade into a completely different game state.

We'd already measured the cycle phases directly from SC2 — per-loop position and cargo tracking at every game loop. The mining phase matched perfectly: 20 loops approach animation, 22 loops extraction, 7 loops post-pickup departure. All confirmed with zero variance. The gap was entirely in the walks.

## The velocity profile that broke the assumption

I went back to the profiler data — 8,800 observations at 8 workers, every game loop, position and cargo. Instead of looking at cycle timing, I computed per-loop velocity: the distance each worker moved between consecutive game loops.

The speed increase was perfectly linear. 0.216 units/second per loop. Every loop, every worker, no exceptions. I could compute the acceleration directly:

```
0.216 u/s per loop × 22.4 loops/second = 4.84 u/s²
```

The emulator was using 3.5 u/s².

For months, every session that touched worker physics assumed 3.5 was correct. The SC2 data editor stores acceleration at 2.5 for Normal game speed. The community documentation says "multiply by 1.4 for Faster speed." 2.5 × 1.4 = 3.5. It's in the physics doc. It was calibrated in issue #578 and restored after a brief excursion to 10.0 during #634.

It was wrong.

## The dimensional analysis

Acceleration is distance per time *squared*. When game speed scales time by 1.4×, velocity (distance per time) scales by 1.4. But acceleration scales by 1.4² = 1.96. The difference between ×1.4 and ×1.96 is 40%.

```
Correct:  2.5 × 1.96 = 4.9 u/s²
Measured: 4.84 u/s² (σ=0.059, n=2,239 observations)
Used:     3.5 u/s²
```

2,239 acceleration observations and 1,851 deceleration observations, all consistent to within σ=0.06. The deceleration was 4.96 — essentially the same as the theoretical 4.9. The velocity profile was trapezoidal, not triangular: 18 loops of acceleration, 2–3 loops at max speed (3.934 u/s), then 12 loops of deceleration.

The profiler also revealed that SC2 workers arrive at about 1.3 u/s — they don't decelerate to zero. The effective arrival threshold is v²/(2a) = 0.17 units, not the 0.5 we'd been using.

## Three fixes from one measurement

The velocity profile broke three things open at once.

**Acceleration:** 3.5 → 4.9 u/s². Direct measurement, not a formula.

**Intermediate waypoint handling:** The profiler showed constant acceleration through the entire walk. No deceleration at intermediate A* waypoints, no stop-and-restart. SC2's flow-field pathfinder flies workers through turns. The emulator was decelerating at every waypoint, then jumping to max speed — a stop-restart pattern that happened to be fast but physically wrong.

**Arrival threshold:** 0.5 → 0.17 units. Workers stop at 1.3 u/s, not zero. The old 0.5u threshold was skipping 12 loops of deceleration per walk — over-compensating for the too-slow acceleration.

After all three: the emulated mining cycle hit 110 loops. Walk to patch 29.4 (SC2: 30). Walk to base 31.2 (SC2: 31). Mining phase 49.0 (SC2: 49). The cycle matched.

## What 0.0% looks like

We re-recorded the 8-worker baseline from SC2. Fresh game, real TorchesAIE terrain, real mineral patch positions. SC2 produced 2,325 minerals in 304 ticks. The emulator produced 2,325.

The 12-worker baseline needed two more fixes — the reroute threshold was equal to the docking overhead (workers rerouting at the exact moment a patch became free), and the collision dodge scale needed recalibration for the higher acceleration. After those: 12w at +1.8%.

The saturation curve from 6 to 12 workers per patch sits within ±4% across the board. That's from first-principles physics — measured acceleration, measured arrival speed, measured walk distances — not from sweeping constants until the numbers looked right.

## The contention wall

The pinned test — 12 workers ramping to 36 — still shows -26%. The fixed-worker rates are correct; the gap accumulates during the ramp-up at 2–3 workers per patch.

We profiled walk distances at five saturations. The finding was clear: SC2's collision avoidance adds walk *distance*, not speed reduction. Acceleration is 4.84 u/s² at every saturation level. But walk-to-patch distance explodes from 2.92 units at 1 w/p to 4.43 units at 3 w/p — a 52% increase. Walk-to-base stays constant. The flow-field routes workers around occupied patches on approach only.

The emulator's lateral displacement model is the wrong mechanism. It pushes workers sideways instead of routing them on longer paths. We tried dynamic A* obstacles from active miners, but the terrain grid's 1-tile resolution can't express 0.20-unit path extensions. A lateral detour waypoint made things worse — the bent path costs more time than the straight one it replaced.

The fix is flow-field or sub-tile pathfinding. That's a different scale of work, tracked in #769. The profiler data gives the design input — approach-only, direction-dependent, nonlinear with saturation. The mechanism matters as much as the magnitude.

## The scaling rule

The physics doc's scaling rule said "multiply by 1.4." For velocity and turn rate, that's correct — they're per-second quantities. For acceleration, it's wrong. Per-second-squared quantities scale by the square of the speed factor.

It's the kind of error that hides in plain sight. The number 3.5 had been in the codebase since issue #578. It was "calibrated" and "restored" multiple times. Other constants — deposit duration, approach animation, collision dodge — were swept and re-swept to compensate for the walks being too slow. Each compensation made the system harder to reason about without fixing the root cause.

One direct measurement — 2,239 velocity observations, 0.216 u/s per loop — replaced months of parameter sweeping.
