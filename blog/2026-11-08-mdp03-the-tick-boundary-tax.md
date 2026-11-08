---
layout: post
title: "The Tick Boundary Tax"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulated-game, economy, gas-income, tick-quantisation, sc2-physics, simulation, time-model]
series: issue-534-emulated-game-100-accuracy
---

# The Tick Boundary Tax

*Continues the [100% accuracy series](2026-11-08-mdp02-the-workers-who-werent-there.md).*

A gas worker in StarCraft II enters a refinery, stands inside for exactly 1.415 seconds, exits carrying 4 vespene, walks home, deposits, and repeats. At the game's internal clock rate of 22.4 loops per second, the harvest takes 31.696 game loops. Not 32. Not "about 32." 31.696. The community reverse-engineered this years ago — Liquipedia documents it, and you can cross-check it against the published income rate: 169.61 gas per minute from a saturated geyser equals 60 divided by 1.415 multiplied by 4. The arithmetic is tight.

QuarkMind's simulation runs in ticks of 22 game loops. Each tick, the worker's harvest counter decremented by 22. A 32-loop harvest needed two ticks — but two ticks is 44 loops. The worker sat inside the refinery for 12 loops longer than it should, every single trip. A 38% time overhead on the harvest phase, compounding across three workers running continuous cycles for the duration of the game.

The previous entry in this series tore out the formula that was compensating for this — a cargo reduction from 4.0 to 3.3 that matched aggregate data while contradicting the documented game mechanic. The formula is gone. The quantisation remains.

## Where the time disappears

The gas worker cycle has three phase transitions where time is lost to tick boundaries:

![Gas worker tick boundary diagram](images/gas-worker-tick-boundaries.svg)

**Entering the refinery.** A worker walking toward the geyser arrives partway through a tick. In the old simulation, the remaining time in that tick was wasted — the worker stood at the doorstep until the next tick started, then began harvesting. In real SC2, the worker enters immediately and starts the harvest counter from the moment of arrival.

**Exiting the refinery.** When the harvest counter hits zero mid-tick, the worker has overflow time — loops left in the current tick after the harvest completed. That time should translate into proportional movement toward the town hall.

**Arriving at the town hall.** The worker deposits instantly and turns around. The remaining movement budget in that tick should carry the worker back toward the geyser. The old simulation consumed the entire tick on the deposit, starting the return trip fresh next tick.

## The first fix, and why it was wrong

The obvious approach: carry-forward. When a phase completes mid-tick, compute the remaining fraction and apply it to the next phase. At geyser entry, subtract the leftover loops from the harvest counter. At exit, give proportional movement. At base arrival, start moving back immediately.

We built it. It worked — unit tests showed gas income jump from 2.4 to 3.0 gas per tick for three workers at a 5-tile distance. The maths was correct. Every transition was accounted for.

Then I looked at what we'd built.

The harvest was still a countdown: `cycleLoopsRemaining -= LOOPS_PER_TICK`. The carry-forward was patching the entry and exit of that countdown to compensate for its fundamental limitation — that it can only subtract in whole-tick chunks. We'd added fractional adjustments on top of an integer-step model. The adjustments were correct. The model underneath was the problem.

The rest of the codebase already knew this. `TrainIntent`, `BuildIntent`, `MorphIntent` — every other timed phase in the simulation stores an absolute loop timestamp: the exact game loop when the phase ends. No countdown. No tick-sized decrements. No carry-forward needed. The harvest counter was the only holdout still using the decrement pattern, and we were layering corrections on top of it instead of replacing it.

## Absolute loop timing

The fix is one line of concept:

```java
w.phaseEndLoop = arrivalLoop + 31.696;
```

The worker enters the geyser at whatever fractional loop it arrives. The harvest ends at exactly that loop plus 31.696. Each tick checks whether the tick's loop window passes the end point:

```java
if (tickEndLoop >= w.phaseEndLoop) {
    double overflowLoops = tickEndLoop - w.phaseEndLoop;
    double overflowFraction = overflowLoops / SC2Data.LOOPS_PER_TICK;
    // exit with proportional movement
    w.moveToward(workerDistancePerTick * overflowFraction);
}
```

No carry-forward. No (int) casts. No special-case logic for entry, exit, or base arrival. The arrival loop is computed from the fraction of movement budget used to reach the geyser. The exit overflow is exact arithmetic — `tickEndLoop - phaseEndLoop`. The proportional movement falls out naturally.

The same change applies to mineral mining. `tryStartMining` now sets `phaseEndLoop = currentLoop + MINING_DURATION_LOOPS` instead of a countdown. Every timed worker phase uses the same model — absolute timestamps, fractional precision, no tick-boundary awareness.

## What the calibration reveals

With absolute timing and the published 31.696-loop harvest, the simulation produces about 90% of ground-truth gas income. The remaining 10% gap isn't in the harvest timing — it's in the movement model. Real SC2 workers have acceleration curves, priority pathing near resource structures, and collision avoidance. The simulation uses constant-speed linear movement. Fixing that is future work, and now clearly identified as the next target.

The gas income improvement at the seven-minute checkpoint: vespeneCollectionRate MAPE dropped from 22.8% to 19.6%. The structural floor for this field sits around 12-15%, so the remaining gap is closing.

## Three models, one lesson

This session tried three approaches to the same problem:

**Countdown with carry-forward.** Correct results, wrong abstraction. The countdown forces tick-aligned steps; the carry-forward patches the boundaries. Every phase transition needs its own carry-forward logic. Adding a new timed phase means remembering to add carry-forward too.

**Countdown with full-tick exit bonus.** The version before carry-forward. The exit gave a full tick of movement regardless of overflow — an over-compensation that accidentally masked the entry problem. The calibration matched ground truth at 0.956 ratio, but for the wrong reasons.

**Absolute loop timestamps.** One concept, no patches. The arrival loop is exact. The end loop is `arrival + duration`. The overflow is `tickEnd - phaseEnd`. Every transition is intrinsically correct because the model doesn't have tick boundaries to compensate for. The rest of the codebase already used this pattern.

The lesson isn't about gas harvesting. It's about recognising when a fix is compensating for a deeper abstraction problem. The carry-forward worked. It passed every test. It improved accuracy. And it was the wrong solution — a correction layered on a model that shouldn't need correcting. The right model doesn't need carry-forward because it doesn't have the quantisation in the first place.

The same instinct that led us to tear out the cargo formula — "a calibrated approximation that hides the real mechanism" — applies to carry-forward too. If the fix requires special-case logic at every transition point, the problem isn't the transitions. It's the model between them.
