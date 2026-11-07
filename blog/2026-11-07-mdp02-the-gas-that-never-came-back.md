---
layout: post
title: "The Gas That Never Came Back"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy, simulation, starcraft2, economy, discrete-time, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# The Gas That Never Came Back

*Continues the [100% accuracy series](2026-11-06-mdp02-the-last-five-percent.md).*

By the seven-minute checkpoint, the emulator had 11,000 vespene. The real player had 52,000. Same replay, same buildings, same workers trained. The gas was entering the system — and then quietly bleeding out.

This wasn't a rate calibration problem. The mineral income model had been calibrated against 118 oracle replays and matched ground truth to within 15%. Gas used the same formula structure, the same three-tier saturation constants, the same per-tick accumulation. And yet gas income was 48% below ground truth by the time a typical professional game reaches its mid-game.

Two bugs. One hiding in code that worked exactly as designed. The other hiding in the physics of discrete time.

## The method that nobody called

`WorkerSimulation` has a method called `rebalanceGasWorkers()`. It does exactly what the name says: scans each gas slot for vacancies, checks if the base has enough mineral workers to spare, and transfers one across. It handles the state transitions — removes the worker from its mineral patch, assigns it to the gas slot, updates the task to `HARVESTING_GAS`, sets the target position.

Claude found it during a codebase exploration. Fourteen lines of correct, well-structured code. Written, tested in isolation, and never wired into anything.

The consequence plays out across every professional replay we validate against. A Zealot rush kills two workers at the natural expansion. One of them was mining gas. The gas slot drops from three workers to two. In real StarCraft, the player immediately pulls a mineral worker to replace it — it's one of the first things any competitive player learns. In the emulator, that slot stays at two workers for the rest of the game. Multiply by every worker death across every expansion across a 10-minute game, and the gas income deficit compounds to 78%.

![Gas worker death — before and after rebalanceGasWorkers()](images/gas-worker-lifecycle.svg)

The fix is one line in `onWorkerDied()`:

```java
if (wasGasWorker) {
    rebalanceGasWorkers();
}
```

When a gas worker dies, immediately attempt to replace it from the mineral pool. The existing `MIN_MINERAL_WORKERS_BEFORE_GAS` threshold (8 workers) prevents mineral starvation — if the base doesn't have enough mineral workers, the gas slot stays undersaturated, which is also what a real player would do.

The rebalance alone improved vespeneCollectionRate MAPE at seven minutes from 48.8% to 44.0%. Meaningful, but the aggregate gas income was still only 40% of ground truth. Something else was eating the remaining 60%.

## The tick that did nothing

The worker simulation operates on discrete ticks. Each tick is 22 game loops — roughly one second of real-time play. A gas worker's trip cycle looks like this: walk to the geyser, enter the building, harvest for two ticks, exit, walk back to the town hall, deposit, walk to the geyser again.

I traced the cycle tick by tick for a geyser five tiles from the town hall. Six ticks per round trip. Four gas per trip. That's 0.667 gas per tick per worker.

The formula model — calibrated from the same 118 oracle replays — says a three-worker geyser should produce 2.27 gas per tick. Three workers at 0.667 gives 2.0. Twelve percent below the formula, just from the trip cycle.

The deficit is structural. When the harvest completes on tick 4, the worker exits the geyser, switches its task to `RETURNING_GAS`, and sets a target position. Then the method returns zero. No movement happens. The worker stands outside the geyser for a full tick doing nothing before it starts walking home on tick 5.

![Gas worker trip cycle — before and after eliminating the wasted transition tick](images/gas-trip-cycle.svg)

The fix: when the harvest completes, immediately start walking in the same tick.

```java
if (w.cycleTicksRemaining <= 0) {
    w.cargo = SimulatedWorker.GAS_CARGO;
    slot.exitBuilding(w);
    w.task           = WorkerTask.RETURNING_GAS;
    w.targetPosition = bases.get(w.baseIndex).position();
    // Start walking immediately
    w.moveToward(workerDistancePerTick);
    // ...
}
```

Trip drops from six ticks to five. Gas per worker per tick goes from 0.667 to 0.8. Three workers produce 2.4 gas per tick — closer to the formula's 2.27 than the old 2.0.

Combined with the worker replacement fix, the vespene results across 236 player runs:

| Checkpoint | Before | After |
|---|---|---|
| 3-min | 46.0% | 39.6% |
| 5-min | 63.0% | 59.0% |
| 7-min | 101.3% | 77.3% |

The aggregate EM/GT ratio at seven minutes went from 0.22 to 1.01. The emulator now holds roughly the same total gas as the real player. The remaining 77% MAPE is per-replay variance — individual replays diverge because gas geysers sit at different distances from the town hall on different maps, and worker death rates vary per game.

## Why discrete ticks eat your lunch

The wasted-tick bug has a structural lesson. In continuous-time simulation, state transitions are instantaneous — a worker finishes harvesting and starts walking in the same infinitesimal moment. In discrete-time simulation, every state transition has to land on a tick boundary. If your tick handler processes states sequentially — harvest, then check for walking, then move — a transition between two states wastes the remainder of the tick.

The pattern shows up everywhere in game simulation. A unit finishes building and can't attack until the next tick. A research completes and the upgrade doesn't apply until the next frame. A worker deposits minerals and can't start walking to the next patch until the next cycle. Each wasted transition is individually small — one tick out of five or six. But they compound across every entity on every trip for the duration of the game.

The fix is also structural: when a state transition fires, immediately execute the first step of the new state within the current tick. Don't return and wait for the next tick to pick it up. This is the difference between a simulation that processes state transitions as boundaries (pause at each one) and one that processes them as events (flow through).

I only applied this fix to gas workers, not mineral workers. Mineral mining is already overcollecting — speeding up mineral trips would widen the gap. The asymmetry is deliberate: gas was 48% below ground truth and needed speeding up; minerals were 17% above and don't. Once per-replay mineral diagnostics identify the overcollection source, the same transition fix might be applied to minerals alongside a compensating constant change.

## What the dead code teaches

The `rebalanceGasWorkers()` method was correct, tested, and dead. It existed because someone — probably during initial development — knew gas workers needed replacement. They wrote the logic, gave it a clean interface, and then either forgot to wire it up or removed the call during a refactor.

Dead code that was written for the right reason is a specific kind of technical debt. It's not a TODO comment or an unfinished feature. It's a completed implementation that silently stopped participating in the system. The tests pass because no test checks for the *absence* of a gas worker replacement. The unit test validates that `rebalanceGasWorkers()` correctly transfers workers when called — it just never checks that it *gets* called.

This is the testing equivalent of checking that the fire extinguisher works but never checking that it's mounted on the wall.

The broader point applies to any simulation that models resource flows: income and spending are the high-profile accuracy targets, but the steady-state maintenance — replacing dead workers, rebalancing between collection types, handling the small lifecycle events — is where simulation accuracy actually lives. A model with perfect rate constants and broken maintenance bleeds accuracy silently, one dead worker at a time.
