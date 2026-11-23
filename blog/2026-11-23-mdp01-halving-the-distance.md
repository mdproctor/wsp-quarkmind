---
layout: post
title: "Halving the Distance"
date: 2026-11-23
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, economy, accuracy, calibration, sliding-window, rate-algorithm]
series: issue-534-emulated-game-100-accuracy
---

# Halving the Distance

*Continues the [100% accuracy series](2026-11-08-mdp04-the-physics-that-was-always-there.md).*

The first standalone economy baseline was 178% MAPE on mineral balance at the 7-minute mark. The emulator was going negative — producing negative mineral totals while SC2 showed thousands in the bank. Gas balance was worse: 147%. The rate metrics looked more reasonable — mineral rate at 29.5%, gas rate at 55.6% — but those numbers hid a deeper problem. The rate algorithm was wrong in a way that happened to produce plausible averages.

That was five months ago. The current numbers:

| Field | Original | Current | Delta |
|-------|----------|---------|-------|
| mineralsCurrent | 178.4% | 84.9% | -93.5pp |
| vespeneCurrent | 147.3% | 61.1% | -86.2pp |
| vespeneCollectionRate | 55.6% | 18.7% | -36.9pp |
| mineralsCollectionRate | 29.5% | 38.2% | +8.7pp |

Three of four fields improved by double-digit percentage points. Balance errors halved. Gas rate cut by two-thirds. The mineral rate went up — and that's the interesting part.

## What moved the numbers

The balance improvements came from getting the physics right. Mining animation duration was wrong by 3×. Workers passed through each other instead of colliding. Gas workers entered the geyser simultaneously instead of queuing. Walk distances used radii for rectangular buildings. Each fix was small, specific, and independently testable. None involved tuning a constant to minimise error — they all replaced an approximation with the actual SC2 mechanism.

The gas rate improvement came from decomposing the harvest cycle into its real phases: 32 loops inside the geyser, 11.5 loops of docking animation, serial access with one worker at a time. Combined with per-worker walk physics and proportional docking absorption (workers that wait at the entrance skip the docking they've already performed by standing there), the EM now tracks gas income within 2-3% of SC2 in controlled dual-run baselines and within 19% across 118 tournament replays.

## 240 loops

The mineral rate regression has a specific cause. SC2 quantises mineral rates at exactly 28 minerals/min — a suspiciously round number. It falls out of the arithmetic: 5 minerals per cargo × 1344 loops per minute ÷ 240 = 28. SC2 counts deliveries in a 240-loop sliding window.

The emulator was using an exponential moving average. EMAs smooth delivery timing jitter — which is exactly what you want when your delivery timing is wrong. The EMA's α=0.25 was compensating for the EM's overly uniform worker cycles, producing rate numbers that looked reasonable despite the underlying physics being off. At 7 minutes, EMA gave 38.6% mineral rate MAPE. The correct sliding window gives 38.2%. Not much difference.

But at 1 minute, the story changes. EMA: 54.2%. Sliding window: 21.6%. The window is 32 percentage points better in early game because there's no lag — the first delivery registers immediately in the window instead of being damped by the EMA's exponential decay. The EMA was hiding how good the early-game physics actually were.

This is the right tradeoff. The sliding window gives worse or equal numbers at late game and dramatically better numbers early. More importantly, it matches what SC2 actually does. An algorithm that works by compensating for broken physics is a trap — the moment you fix the physics, the compensation becomes noise.

## The variance wall

The mineral rate MAPE is now 38.2% against a structural floor of about 10-12%. That 26-point gap is the next target. And unlike the previous gaps, it's not from a wrong constant or a missing mechanism. It's from cycle uniformity.

SC2 worker mining cycles have a standard deviation of 3+ ticks. Workers collide with each other during walks, dynamically route to whichever patch happens to be idle, and encounter path stochasticity from the flow-field pathfinder. The emulator's cycles have σ≈0.5 — they're too regular. Workers go to the same patch every trip, walk straight lines between obstacles, and rarely encounter each other during transit.

This matters because the rate metric counts deliveries in a fixed 240-loop window. When all workers deliver at nearly the same cadence, their deliveries cluster into the same windows — or miss the same windows — as a group. SC2's spread-out deliveries produce steadier rates. The EM's synchronised deliveries produce rates that are correct on average but volatile at any given snapshot.

The fix isn't a constant or a formula. It's adding the missing variance sources: dynamic patch selection on return, worker-worker collision during transit walks, and potentially path stochasticity. Each one independently testable, each one measurable against the SC2 per-worker baseline from the instrumented observations.

The balance fields will follow. mineralsCurrent MAPE is roughly the rate error × elapsed time — a 2.2× amplification factor at 7 minutes. If mineral rate drops from 38% to 20%, mineral balance should drop from 85% to roughly 45%, near the structural floor where spending timing errors dominate.

Five months from going negative to half the distance. The next half is about variance, not constants.
