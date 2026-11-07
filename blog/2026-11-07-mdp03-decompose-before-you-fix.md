---
layout: post
title: "Decompose Before You Fix"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy, simulation, starcraft2, economy, methodology, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# Decompose Before You Fix

*Continues the [100% accuracy series](2026-11-07-mdp02-the-gas-that-never-came-back.md).*

The emulator had too much money. At the five-minute mark, across 168 player runs against oracle replays, the simulated mineral balance was consistently higher than ground truth. The natural hypothesis: spending is incomplete. The emulator tracks what players build and train, deducting costs as units and buildings complete. Miss a few deductions, and unspent minerals accumulate.

It's a reasonable hypothesis. It was wrong.

## The equation nobody checks

Mineral balance is a residual. It's what's left after subtracting cumulative spending from cumulative income:

```
balance = gross_income - total_spending
```

When the balance is wrong, the error could live in either term. Or both. Aggregate balance error hides the source completely. A balance that's 400 minerals too high could mean we over-collected by 400, or under-spent by 400, or over-collected by 800 and over-spent by 400. You can't tell without decomposing it.

Both terms are computable from the existing `PlayerEconomyStats` record. Ground truth provides `mineralsCurrent` (the balance) and `mineralsUsedCurrentArmy`, `mineralsUsedCurrentEconomy`, `mineralsUsedCurrentTechnology` (the spending). Gross income is the balance plus total spending. No new instrumentation needed — just arithmetic on data that was already there.

![Income vs spending decomposition — 5-minute checkpoint](images/income-spending-decomposition.svg)

The decomposition said: income error averages 396 minerals. Spending error averages 209. Income is the primary driver — nearly twice the magnitude of spending.

## The direction tells a different story

Mean error magnitudes are useful, but the direction matters more. Of 168 replays at the five-minute checkpoint:

- **Income:** 91 over-collected, 76 under-collected. Nearly balanced. The mining model produces the right amount of minerals *on average*.
- **Spending:** 63 over-spent, 97 under-spent. Biased. The emulator consistently deducts less than the real player spends.

The spending bias is real — 58% of replays under-spend. But because the income error is larger in magnitude and nearly balanced, income variance dominates the per-replay accuracy metric. Fixing spending alone would shift the mean but not reduce the variance that makes per-replay MAPE high.

## The fix that broke things

I tried the obvious income fix: increase the mining dwell time from 1 tick to 2 ticks. In real StarCraft, a worker spends about three ticks at a mineral patch during the mining animation. Our simulation uses one tick — faster than reality, but calibrated to produce the right average throughput with multi-occupant patch sharing.

Doubling the dwell should slow collection by roughly 20%, pulling income closer to ground truth for the replays where we over-collect.

It over-corrected catastrophically. The income direction swung from 91/76 (balanced) to 29/139 (massive under-collection). Zerg MAPE doubled from 48% to 98%. The model that was slightly too generous became drastically too stingy — because 1 tick was already close to the right average, and 2 ticks is far past it.

The tick discretisation is the root constraint. At 22 game loops per tick, the simulation can only represent mining durations of 1 tick, 2 ticks, 3 ticks — nothing in between. The real SC2 mining animation lasts 2.8 ticks. Representing it accurately would require sub-tick timing, which is a different kind of change than a parameter adjustment.

## Per-race tells the rest

![Per-race income vs spending error — minerals at 5-minute](images/per-race-error-decomposition.svg)

Protoss income error (477) is 2.4 times its spending error (197). Zerg income error (410) is 1.4 times spending (293). Terran is the cleanest at 2.2 times (326 vs 150). Income dominates universally, but the ratio varies by race — Zerg's spending is relatively worse, probably because Zerg economic mechanics (Inject Larva, Hatchery-as-production) create more spending opportunities that the tracker events don't fully capture.

The Protoss story is especially stark. Protoss mineral MAPE degrades from 100% at five minutes to 141% at seven — it gets *worse* over time while Terran and Zerg stay flat. Chronoboost, warp-in timing, and Protoss tech-tree depth are likely compounding factors, but the decomposition shows the root is income-side, not spending-side.

## The method matters more than the fix

This decomposition took ten lines of arithmetic on existing data. No new simulation code. No new tracker events. The `PlayerEconomyStats` record already carried both halves of the equation — nobody had thought to split them.

The lesson generalises beyond game simulation. Any time a residual metric is wrong — account balance, inventory level, resource pool — the error lives in either the inflow or the outflow. Checking the aggregate tells you something is wrong. Checking the components tells you *what*. And the direction of each component tells you whether a global parameter change will help or hurt.

I spent half a day building toward a spending fix that would have addressed the secondary driver. The decomposition took twenty minutes and redirected the entire investigation.
