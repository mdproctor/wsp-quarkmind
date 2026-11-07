---
layout: post
title: "When Physics Isn't the Answer"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy, simulation, starcraft2, economy, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# When Physics Isn't the Answer

*Continues the [100% accuracy series](2026-11-07-mdp02-the-gas-that-never-came-back.md).*

The gas worker walk simulation was physically correct. Workers walked to the geyser at probe speed, entered the building, harvested for the right number of ticks, walked back, deposited. Every state transition was mechanically sound. And it produced 15–33% less gas than real StarCraft II, getting worse the longer the game ran.

The per-worker efficiency ratio told the story. At 3 minutes: 0.93 — close enough. At 5 minutes: 0.85. At 7 minutes: 0.67. The simulation fell further behind with every passing minute, even though nothing in the code was wrong.

## The walk-time gap

SC2 has optimised worker pathing that no external simulation can replicate without reverse-engineering the client's movement code. Workers accelerate, take shortcuts around buildings, and sometimes appear to teleport short distances during gas runs. A physics model that walks at the nominal probe speed of 3.94 tiles/second, with honest collision geometry and tick-boundary transitions, will always be slower than what the real game achieves.

We had already calibrated a formula rate from 118 oracle replays: 54.8 gas per minute for each of the first two workers, 28.8 for the third. That formula was sitting in `SC2Data.gasIncomePerTick()`, used only for comparison reporting. It matched ground truth because it was derived from ground truth. The physical simulation was derived from first principles — and first principles don't account for the game engine's shortcuts.

![Per-worker gas efficiency: physical simulation vs formula](images/gas-physics-vs-formula.svg)

The fix was to stop simulating gas trips entirely. Workers still get assigned to gas slots — that tracking matters for rebalancing when workers die and for knowing how many are mining versus collecting gas. But the income calculation now calls the calibrated formula directly: `SC2Data.gasIncomePerTick(slot.assignedWorkers.size())`. No walk cycle. No transition ticks. Just the rate that real games actually produce.

Gas rate MAPE dropped from 30.9% to 24.8% at the five-minute checkpoint. The remaining gap is the formula's calibration accuracy — a follow-up issue, not a simulation bug.

## The hypothesis that data killed

With gas income now formula-based, I wanted to verify that geyser distance — which varies by map — no longer mattered. The original theory was sound: a geyser 10 tiles from the town hall should produce less gas than one at 7 tiles, because each round trip is longer. With walk-based simulation, distance should correlate directly with income error.

Claude ran the diagnostic across 168 replays with gas buildings at the five-minute checkpoint, bucketed by average geyser-to-townhall distance.

Pearson r = 0.037. Zero correlation.

The formula-based income doesn't care about distance — it produces the same rate regardless of map geometry. And 96% of replays had geysers between 7 and 8 tiles from the town hall anyway. Distance variance was small to begin with and irrelevant after the formula switch.

The broader point: measuring before fixing would have saved hours. If I'd run the distance correlation before building a per-map distance calibration system, I'd have seen the same 0.037 and moved on immediately.

## The expansion problem nobody teaches you

A second source of income inaccuracy had nothing to do with gas. Mineral income was 17% below ground truth at the seven-minute checkpoint — and the cause was worker distribution, not mining rates.

When a new expansion finishes in real SC2, experienced players immediately transfer workers to the new base. The simulation didn't. Every new worker went to the production building's nearest base, which was usually the main. By minute seven, the main had 20+ workers on 8 mineral patches — four workers earning nearly nothing — while the expansion had empty patches.

![Worker distribution before and after expansion transfer](images/worker-transfer-expansion.svg)

Workers 17 through 20 on a base earn roughly 10% of what worker 1 earns. Moving them to an empty expansion where they mine at the full first-worker rate is a 10x improvement per worker transferred. The real player does this instinctively. The simulation didn't do it at all.

The fix transfers up to four mineral workers from the most oversaturated base whenever a new town hall completes. Four workers, not a full rebalance — real players don't perfectly optimise, they rally a handful and move on. The cap keeps the simulation realistic rather than prescriptive.

Mineral rate MAPE at 7 minutes: 17.2% → 13.9%.

## When the premise is wrong

The last investigation this session was a spending tracker — a separate InProgress accumulator to account for resources committed to in-progress buildings and units. The hypothesis: ground truth's `mineralsUsedCurrentArmy` field includes spending for items still being built, but the emulator only records spending at completion. Adding an InProgress layer should close the gap.

I built the infrastructure, wired it in, ran the baseline. Spending MAPE went from 5% to 60%. Massively worse in every category.

The premise was wrong. SC2's `UsedCurrent` fields track completed-and-alive items only — the "Current" means "currently existing," not "current total." In-progress spending lives in separate `UsedInProgress` fields that we weren't mapping to. Adding InProgress to Current doubled every in-progress item's accounting.

The infrastructure stayed — it's tested, correct, and might serve a future need. But the wiring was reverted within the same session. Sometimes the most productive outcome of an investigation is learning that the investigation was wrong.

## What calibration actually means

Four investigations. Two improved accuracy, one disproved a hypothesis before it cost implementation time, one disproved its own premise after implementation. All four produced actionable knowledge.

The pattern that keeps emerging: physically correct simulation and calibrated-from-data simulation are not the same thing. The walk simulation was right about physics and wrong about income. The formula was right about income and says nothing about physics. For training an agent, the income accuracy is what matters — the agent doesn't care whether the gas arrived via a simulated walk or a lookup table. It cares that the economic state matches what a real game would produce.

This is the difference between a digital twin and a training environment. A digital twin aims for physical fidelity. A training environment aims for outcome fidelity. When the two diverge, calibrate to outcomes.
