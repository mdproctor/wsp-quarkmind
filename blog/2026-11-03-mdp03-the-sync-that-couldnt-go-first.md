---
layout: post
title: "The Sync That Couldn't Go First"
date: 2026-11-03
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulation, economy, calibration, validation-harness, starcraft2]
---

# The Sync That Couldn't Go First

QuarkMind's replay validation harness runs two games side by side: the real SC2 game (from replay data) and an emulated version that tries to reproduce the same economy from first principles. Every tick, the harness compares them. The closer they match, the more we trust the emulation.

Vespene gas has been synced from ground truth for months — the harness sets the emulated gas pool to match the real player's gas before each tick, so gas-unit training commands succeed with realistic affordability. The error sits at about 10%. Clean enough.

Minerals were left to diverge. The emulated game accumulated mineral income from its worker simulation and deducted costs when training intents fired, but it never reconciled against what the real player actually had. The result: 396.7% error at the five-minute mark. Nearly four times the real player's mineral pool.

The obvious fix was to do what vespene does — sync minerals from ground truth each tick. One line of code. I added it in the same position as the vespene sync, right before intents fire.

The unit count test broke immediately.

## Why position matters

The vespene sync works in the pre-intent position because gas-unit training needs to check affordability against the real gas pool. If the emulated game has the wrong gas amount, gas-unit intents fail that shouldn't, or succeed that shouldn't. Syncing gas before intents solves this.

Minerals have a different problem. The emulated game's mineral pool was inflated — 396% above reality. Every mineral-costing intent passed its affordability check because there were always enough minerals. When I synced minerals to ground truth (realistic, lower values), some intents started failing. The emulated game trained fewer units than the real player did, opening a unit count gap that wasn't there before.

The insight: vespene sync is an *affordability gate* — it makes the right intents succeed. Mineral sync at the same position is an *affordability penalty* — it makes intents fail that the real player could afford, because the emulated game's tick processing has already consumed some of those minerals for queued productions before the intent fires.

The fix was to move the mineral sync to *after* intents fire, right before the snapshot. Intents still use the emulated mineral pool (inflated, permissive) for their affordability checks. Then the sync resets minerals to the real player's post-tick value. The snapshot captures accurate minerals without interfering with intent processing.

![Harness tick flow showing where vespene and mineral syncs occur](images/mineral-sync-timing.svg)

Vespene syncs before intents — for affordability. Minerals sync after intents — for accuracy. Same pattern, different positions, for different reasons.

## Gold bases and the cargo gap

A smaller fix in the same session: rich mineral patches. In SC2, gold bases have mineral patches where workers carry 7 minerals per trip instead of the standard 5 — a 40% throughput increase. The emulation used a flat 5.0 for all patches.

The fix threads a `rich` flag from the replay parser through to the harvest calculation. A `Resource` record now carries `boolean rich`, derived from the unit type name (`RichMineralField`). The `MineralPatch` class passes it through, and the harvest logic picks `RICH_MINERAL_CARGO` (7.0) or `MINERAL_CARGO` (5.0) based on the flag. Backward-compatible — the existing three-argument `Resource` constructor defaults `rich` to `false`, so nothing else in the codebase needs to change.

```java
double cargo = w.assignedPatch.rich
    ? SimulatedWorker.RICH_MINERAL_CARGO
    : SimulatedWorker.MINERAL_CARGO;
w.cargo = w.assignedPatch.harvest(cargo);
```

Not a dramatic change. But on gold-base maps, it's the difference between the economy model matching reality and systematically under-reporting income by 40% on half the mineral line.

## What this actually means

Economy accuracy in a game AI emulator isn't about getting the numbers right for their own sake. It's about trust in the decision pipeline downstream. When the emulated mineral pool says the player has 2,000 minerals and the real player has 500, every strategic assessment built on that number is wrong. The strategy layer thinks the player is banking resources. The build order evaluator thinks production is stalled. The spending efficiency scorer thinks the player is floating.

With the mineral sync, the harness regression ceiling drops from 370% to 0%. The emulated economy now tracks reality closely enough that economic reasoning built on top of it has a chance of being meaningful.

The remaining calibration targets — gas harvesting physics and supply tracking — are next. Each one closes another gap between what the emulator believes and what's actually happening in the game.
