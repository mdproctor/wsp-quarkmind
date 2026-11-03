---
layout: post
title: "Let the Workers Work"
date: 2026-11-03
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulation, simulation, emergent-behaviour, architecture, starcraft2]
series: issue-500-worker-simulation
---

# Let the Workers Work

In an [earlier entry](2026-11-03-mdp01-when-the-formula-is-the-bug.md), I wrote about how QuarkMind's income model was architecturally broken — two income paths disagreeing about where workers were, with aggregate formulas hiding the disagreement. The formula wasn't wrong. The abstraction was.

This entry is about the fix. Not the bug fix — that was a two-line patch. The *architectural* fix: replacing the entire formula-driven income model with individual worker simulation, so that income emerges from what each worker is actually doing.

## Why formulas lie

Here's the formula that computed mineral income for the last six months:

```java
public static double mineralIncomePerTick(int probeCount) {
    double income = 0;
    for (int tier = 0; tier < MINERAL_TIER_RATES_PER_TICK.length; tier++) {
        int probesInTier = Math.min(Math.max(probeCount - tier * 8, 0), 8);
        income += probesInTier * MINERAL_TIER_RATES_PER_TICK[tier];
    }
    return income;
}
```

Three tiers of 8 workers each. First 8 workers earn full rate. Next 8 earn half. Next 8 earn a tenth. Calibrated from 118 oracle replays. It's a reasonable mathematical model of SC2 mineral saturation.

The problem isn't the math. The problem is that the formula is the entire model. Feed it a number, get income back. No way to ask *why* that number. No way to detect when the number is wrong. No way to validate the formula's assumptions without building a second, independent model to check it against.

The formula encodes three specific claims about how mining works:

1. Saturation happens at discrete tier boundaries (8, 16, 24 workers)
2. All workers within a tier earn identically
3. Income depends only on worker count, not on worker position, distance, or state

All three are wrong. In SC2, saturation is a smooth curve driven by patch competition. Workers near close patches earn more than workers at far patches. And a worker walking back to the town hall earns zero for those ticks — the formula has no concept of "in transit."

## Simulation as a statement about what matters

The replacement is called `WorkerSimulation`. Each worker is a tiny agent with three things: a position, a velocity, and a current task.

![Worker state machine — walk to patch, mine, return with cargo, deposit](images/worker-state-machine.svg)

That's the entire model. Each tick, every worker advances one step. If it's walking, it moves toward its target. If it's mining, it counts down. If it arrives at the town hall with cargo, it deposits — and that deposit event is the income.

There's no `mineralIncomePerTick` function computing income. Income is a side effect of workers completing trips. Mineral saturation isn't encoded — it emerges from workers competing for 8 patches. When 16 workers share 8 patches, each patch has a queue of 2. The second worker waits. That wait time is the saturation penalty. No formula. Physics.

![Saturation emerges from patch competition — formula produces stepped tiers, simulation produces a smooth curve](images/saturation-emergence.svg)

The same principle holds for gas. Each gas building has an interior slot and a wait queue. 3 workers rotate: one enters the building, harvests for ~29 game loops, exits, walks to town hall, deposits. While one is inside, the others wait or walk. The 3rd-worker penalty that the formula approximated with a lower per-worker rate now falls out of the rotation geometry.

## What this makes possible

The immediate payoff is accuracy. The replay validation harness can now sync *both* mineral and gas workers from ground truth — something that was architecturally impossible before, because `WorkerAssignment` was a counting structure with no concept of individual worker state. The harness says "these 3 workers are near this gas building" and the simulation assigns them directly. No formula interprets the count. No pool tracking fails to grow because CmdEvents were missed.

But the deeper value is falsifiability.

The old formula was a black box. Feed it 12, get income. If the income was wrong, you had two possible explanations: the input count was wrong, or the formula rates were miscalibrated. Both require external validation — run it against replays, compare, adjust. The formula couldn't help you find its own bugs.

Now the formula still exists, but as a validator:

```java
FormulaComparison compareWithFormula(TickIncome simulated) {
    double formulaMineral = 0;
    for (int count : mineralWorkersPerBase()) {
        formulaMineral += SC2Data.mineralIncomePerTick(count);
    }
    // ...
    return new FormulaComparison(simulated, formula);
}
```

Run the simulation. Also compute the formula result. When they diverge, something is wrong — and the divergence tells you where. If the formula says 15 minerals/tick but the simulation says 12, either workers are spending too long walking (simulation bug) or the formula's tier rates are miscalibrated (formula bug). The error has diagnostic structure. Neither model is privileged; each validates the other.

## The general principle

This isn't specific to game AI. The pattern shows up everywhere a formula approximates a physical process:

- **Financial models** approximate market dynamics with aggregate functions (Black-Scholes, DCF). Agent-based models simulate individual actors and let market behaviour emerge. When the two disagree, you learn something about the assumptions built into the formula.

- **Network simulators** can model throughput with formulas (Little's Law, queueing theory) or simulate individual packet routing. The formula tells you the steady-state average. The simulation tells you about bursts, congestion, and edge cases the formula assumes away.

- **Population models** approximate growth with logistic curves. Agent-based models simulate individual births, deaths, and resource competition. The logistic curve is a reasonable first approximation — until local resource depletion or spatial clustering makes the aggregate assumption break.

The lesson isn't that formulas are wrong. It's that a formula operating alone is unfalsifiable. It computes what it computes. You can calibrate it against data, but calibration assumes the model structure is correct and only the parameters need tuning. When the structure is wrong — when saturation isn't actually a discrete 3-tier function, when the gas income curve isn't actually symmetric across workers — no amount of parameter tuning will fix it.

A simulation running alongside the formula gives you structural validation. The formula makes claims about the shape of the system. The simulation makes different claims. Where they agree, you're probably right. Where they disagree, you've found a question worth answering.

The formula was never the bug. Trusting it alone was.
