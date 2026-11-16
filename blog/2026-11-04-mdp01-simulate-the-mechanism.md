---
layout: post
title: "Simulate the Mechanism, Not the Output"
date: 2026-11-04
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [simulation-design, gas-harvesting, emulation, first-principles, starcraft2]
series: issue-503-gas-harvesting-calibration
---

# Simulate the Mechanism, Not the Output

QuarkMind's gas harvesting model had a queue. One worker enters the gas building, harvests for a fixed duration, exits with 4 gas, walks back to the command centre, deposits, walks back. While it's inside, the other two workers wait at the entrance. When the first worker exits, the next enters. First-in, first-out. Orderly.

The model felt right. Mineral patches work this way — one worker mines at a time per patch, others wait. Gas buildings seemed analogous. A building is a resource. Workers access it one at a time. Queue discipline.

SC2 doesn't do this. All three workers enter the gas building simultaneously. Each harvests independently on its own timer. No queue, no mutual exclusion, no waiting. The building is more like a room than a turnstile.

## Why the queue felt correct

The mineral model had to solve a real contention problem. Eight mineral patches per base, each with a walk-mine-walk-deposit cycle. Put 8 workers on 8 patches and each gets exclusive access. Put 16 and they're waiting for slots. Saturation emerges from physical contention — not from a diminishing-returns formula, but from workers actually competing for a limited resource.

I carried this mental model to gas. A gas building holds 2250 vespene. Workers extract from it. Surely they take turns, like mineral patches. The analogy felt so natural I never questioned it. The code implemented a `GasSlot` with an `insideWorker` pointer and a `waitQueue`. Clean abstraction, wrong physics.

## What one-at-a-time costs

![Queue vs concurrent gas harvesting — worker timelines over 20 ticks](images/queue-vs-concurrent-gas.svg)

The diagram makes the cost visible. With a 2-tick harvest time and 2-tick walk each way, a single worker completes a full cycle in 6 ticks. With the queue model, worker 2 waits 2 ticks before entering (while worker 1 finishes), and worker 3 waits 4 ticks. Over 20 ticks: 6 deposits with the queue, 9 with concurrent entry. That's 50% more gas income from the same three workers, same building, same cycle time — the only difference is whether the building door admits one worker or three.

In a unit test with a gas building 5 units from the base, the effect is small enough to miss. Workers cycle fast, the queue drains quickly, and the aggregate numbers look plausible. Against 118 real replay games over five minutes of gameplay, the 35% vespene collection rate error accumulates visibly. The emulator consistently under-produced gas — not because the formula was wrong or the constants were uncalibrated, but because workers were standing in a queue that doesn't exist in the real game.

## The broader principle

This is the third time the same lesson has surfaced in QuarkMind's emulator work:

**Train times** looked like a formula problem — `seconds × 22.4` to get game loops. The formula was an approximation. SC2 stores integer game loops that don't follow the community formula. Calibrating from replay ground truth replaced 22 values and recovered 100% unit accuracy.

**Mineral income** looked like a saturation curve — `mineralIncomePerTick(N)` with diminishing returns baked into the function shape. Replacing it with per-worker trip simulation, where each worker walks to a patch, mines, walks back, and deposits, produced the saturation curve as an emergent property of patch competition. The formula became a validator, not a calculator.

**Gas income** looked like a queue — one worker inside at a time, throughput limited by harvest duration. Replacing the queue with concurrent entry, where each worker independently enters, harvests, and exits, produced the correct gas rate without any formula adjustment.

The pattern: **every time we modelled the output instead of the mechanism, the model was wrong in ways we couldn't detect from the output alone.** The formula matched within tolerance. The curve had the right shape. The queue produced gas. But the mechanism was wrong, and the error compounded over time.

Simulating the mechanism means modelling what actually happens — workers entering a building, standing at a mineral patch, walking between two points. The numbers fall out. When the numbers are wrong, the mechanism is wrong, and fixing the mechanism fixes the numbers. When the numbers come from a formula, "wrong numbers" could mean wrong formula, wrong inputs, wrong constants, or wrong architecture — and the formula can't tell you which.

## Calibration is measurement, not curve fitting

The gas model's harvest duration — how many game loops a worker spends inside the gas building — is currently set to 29 loops, a community-sourced value. It might be right. It might not. The correct value isn't derivable from the gas income formula or from the community wiki. It's a physical constant of SC2's game engine, and the only way to know it is to measure it against replay ground truth.

We wrote a calibration sweep that runs the emulator against 118 oracle replays for each candidate harvest duration, measuring vespene collection rate error at each. The value that minimises the error is the answer. Not a formula. Not an approximation. A measurement.

That sweep can't run right now — the replay validation baseline has regressed from a dependency update, breaking accuracy across all categories. Once that's resolved, the calibration is a single test run. The gas model's concurrent entry mechanism is already correct. The harvest duration is the last parameter to pin down, and the tool to pin it down is ready.

## Why this matters for game AI

An AI agent reasoning about resource income needs to predict how much gas it will have in 30 seconds. If the income model uses a formula, the agent is reasoning about a mathematical abstraction. If the model simulates individual workers, the agent can reason about mechanism — "if I pull one worker off gas to build a pylon, gas income drops by exactly one worker's contribution, which depends on the walk distance at this specific base." The agent's decisions become grounded in the same physics the real game uses.

Formula-driven income is a black box. Simulation-driven income is transparent to the agent's planner. The agent doesn't need to learn the formula's quirks — it just needs to understand that workers harvest gas, and fewer workers means less gas. The mechanism IS the model.
