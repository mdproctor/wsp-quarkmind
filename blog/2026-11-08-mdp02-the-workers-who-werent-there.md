---
layout: post
title: "The Workers Who Weren't There"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulated-game, economy, mineral-income, simulation-vs-formula, sc2-physics]
series: issue-534-emulated-game-100-accuracy
---

# The Workers Who Weren't There

*Continues the [100% accuracy series](2026-11-08-mdp01-every-fix-reveals-the-next-assumption.md).*

QuarkMind's mineral income is physically simulated. Each worker is an independent agent. It walks to a mineral patch, occupies a slot, mines for a calibrated duration, picks up cargo, walks back to the town hall, deposits, and returns for the next trip. The simulation captures real map geometry — close patches earn more than far patches, oversaturated bases penalise throughput, and expansion worker transfers happen at the right moment. Every worker mines every tick.

That last sentence is the problem.

## The gap that shouldn't exist

mineralsCurrent MAPE at the seven-minute checkpoint was 81%. The emulator was over-earning. Consistently, across all three races, the simulated workers brought home more minerals than real players accumulated in the same number of game loops. Per-worker mineral income ran 5-10% above ground truth across 118 replays.

The simulation wasn't wrong about the physics. It was wrong about the players.

A StarCraft player with 16 workers on minerals doesn't have 16 workers mining. At any given moment, two or three of them are doing something else — a Probe scouting the enemy base, an SCV walking to a construction site, workers in transit between bases, a worker sent to hold a tower and forgotten. The simulation sees 16 workers and mines with all 16. The real game has 13 actually at the patches.

## The formula we built

The first attempt was a per-race mining efficiency factor — a scalar on mineral income representing the fraction of workers actively mining. Swept 16 candidate values against 118 replays. Found optimal values: Protoss 0.92, Zerg 0.95, Terran 1.0. Shipped it. MAPE dropped. The calibration infrastructure worked. The numbers improved.

Then I looked at what we'd built.

A blanket scalar that reduces all Protoss mineral income by 8%. It doesn't know which workers are scouting. It doesn't know when they left or when they'll return. It takes aggregate replay data and fits a constant to it — the same thing we did with gas tier rates, with the mineral saturation formula, with every approximation this series has torn out. A formula fitted to aggregate data that hides the real mechanism.

The gas income blog was still open in the browser. "A calibrated approximation can match aggregate data while being wrong about the underlying mechanism." We'd written that two hours earlier. Then we built exactly the thing the blog warned against.

## The gas formula too

Same session, same mistake. Gas income was physically simulated — workers walk to the geyser, enter, harvest for 32 game loops, exit with 4 gas, walk home, deposit, repeat. But tick quantisation meant the 32-loop harvest rounded to 2 ticks (44 loops). Workers sat inside the geyser 38% longer than real SC2.

The first fix: reduce cargo from 4.0 to 3.3 to compensate. Calibrated from replay data. MAPE improved. Shipped it.

But SC2 delivers 4 gas per trip. That's not a simulation parameter — it's a game mechanic measured by the community and documented on Liquipedia. Changing it to 3.3 was tuning a constant to match aggregate data instead of fixing the actual quantisation.

## Removing everything

Both formulas were reverted. Then all formula infrastructure was removed — 260 lines of code that existed solely to compute income from tier-rate arrays and worker-count estimates:

- `mineralIncomePerTick`, `gasIncomePerTick`, the tier rate arrays, `estimateGasWorkersFromRate` — all gone from SC2Data
- `compareWithFormula`, `formulaIncome` — gone from WorkerSimulation
- `syncGasRateFromGt` — gone from EmulatedGame. This one was especially insidious: it overrode the simulation's physical gas income with a formula estimate every tick. The simulation was computing gas correctly, and the harness was replacing the result with a formula.

The formulas weren't just wrong. They were a standing invitation to calibrate them instead of fixing the physics. As long as they existed, the path of least resistance was to sweep values, fit constants, and move on. Removing them makes the only path forward the right one.

## What's actually wrong

With formulas gone, the honest baseline is clear:

| Field | 7-min MAPE | Root cause |
|-------|-----------|-----------|
| mineralsCurrent | 81.4% | All workers mine — no scouting or transit modelled |
| mineralsCollectionRate | 14.9% | Same + default patch distances ignore map geometry |
| vespeneCollectionRate | 22.8% | 32-loop harvest rounds to 2 ticks (44 loops) |
| vespeneCurrent | 54.2% | Cumulative from rate errors |

Each of these has a physics fix:

**Gas quantisation:** implement sub-tick time carry-forward. When the harvest completes mid-tick, the remaining time applies to the next phase — walking home. The worker exits after 32 loops of harvest time, not 44. Cargo stays at 4.0. The game mechanic is the game mechanic.

**Mineral worker activities:** model scouting and transit from tracker events. When a CmdEvent moves a worker away from the mineral line, pull it off mining. When it returns, reassign it. The simulation already handles construction pauses for Terran — that's why Terran needed no efficiency factor. The same approach works for scouting: track the activity, not the aggregate.

**Patch distances:** use actual map geometry instead of default distances. This is the same principle as everything else — measure the real thing instead of assuming a convenient average.

## The pattern, one more time

This series keeps finding the same lesson. Formulas absorb errors into their constants. The fit looks good because the mechanism errors average out across replays. The formula matches the aggregate while being wrong about every individual case. And as long as the formula exists, the temptation is to recalibrate it — adding one more constant, fitting one more coefficient — instead of fixing the physics underneath.

The mineral efficiency factor was the clearest case yet. We knew the principle. We'd just blogged about it. We built the formula anyway, because it was faster. Then we looked at what we'd done and removed everything — not just the new factor, but every formula that had been sitting in the codebase from the beginning.

The simulation is now the single income authority. There is no formula path. The only way to improve accuracy is to improve the physics.
