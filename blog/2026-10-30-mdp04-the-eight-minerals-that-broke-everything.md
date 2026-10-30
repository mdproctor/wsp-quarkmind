---
layout: post
title: "The Eight Minerals That Broke Everything"
date: 2026-10-30
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, emulated-game, sc2, physics, economy, dual-run]
series: issue-396-dual-run-calibration
---

# The Eight Minerals That Broke Everything

The [previous entry](2026-10-30-mdp03-fix-the-first-broken-thing.md) described the dual-run calibration framework — record SC2 ground truth once, then iterate EmulatedGame calibration offline with early exit on the first divergence. The theory: run the first real baseline, see where the physics engine breaks, fix it.

The first real SC2 baseline broke at tick 1. Not tick 50, not tick 200. Tick 1. Eight extra minerals.

## Why eight minerals matter

In StarCraft II, workers start the game standing at the town hall. They walk to the nearest mineral patch, harvest, walk back, deposit. That initial walk takes real time — at least one game step where no income is earned. The first observation you receive from SC2 still shows 50 starting minerals.

EmulatedGame didn't model this. On the very first tick, twelve probes earn their full per-tick income: ~8.18 minerals. SC2 shows 50. EmulatedGame shows 58. Eight minerals is the delta, and it's present on every subsequent tick.

![Mineral divergence timeline — SC2 vs EmulatedGame before and after fix](images/mining-divergence-timeline.svg)

Eight minerals sounds small. It isn't. The first Probe costs 50 minerals. With the extra 8, EmulatedGame reaches 50 faster, queues the Probe earlier, and completes it one tick ahead of SC2. By tick 6, the unit counts disagree. By tick 10, the entire production timeline has shifted. A constant offset in the mining model cascades into a structural divergence in unit production.

The fix is one constant: `MINING_START_DELAY_TICKS = 1`. Skip mineral and gas income on the first game tick. Not a general "workers take time to walk" model — just the game-start case, calibrated directly from the ground truth.

## The convention that ate the economy

The mineral offset was the obvious bug. The second one was architectural, and I didn't see it coming.

EmulatedGame distinguishes two kinds of train intents by their building tag prefix. Tracker-driven intents — reconstructed from replay data — carry an `r-` prefix. These represent units the real player actually produced, so they skip the economy gate: no mineral check, no cost deduction, just physics simulation. The player already afforded it; we're measuring physics accuracy, not economic accuracy.

The playbook delivery handler needed the same building auto-resolution (find a Nexus when the playbook says "train Probe" without specifying which one). The path of least resistance: reuse the `r-` prefix. Tag the intent `r-playbook-probe`. Auto-resolution works. Ship it.

The side effect: every playbook train intent also skipped the economy gate. Minerals were never deducted. SC2 deducts 50 minerals the moment you click the train button. EmulatedGame's playbook path charged nothing. Probes were free.

![Tag prefix cost deduction flow — before and after fix](images/tag-prefix-cost-deduction.svg)

This is a pattern that bites in any codebase with convention-driven dispatch. A tag prefix designed to mean "trust this intent, skip economy checks" gets borrowed by a different subsystem that needs the auto-resolution half but not the economy-skip half. The convention bundles two unrelated behaviours, and the bundling is invisible until ground truth reveals it.

The fix separates the two concerns. A `trackerDriven` flag controls the economy gate. Auto-resolution triggers on either `r-` or the new `pb-` prefix. Playbook intents resolve buildings automatically and deduct costs. Tracker intents resolve buildings and trust the real player's economy.

```java
boolean autoResolved = false;
boolean trackerDriven = false;
if (building == null) {
    if (model != null && buildingTag != null
        && (buildingTag.startsWith("r-") || buildingTag.startsWith("pb-"))) {
        autoResolved = true;
        trackerDriven = buildingTag.startsWith("r-");
        // ... resolve building
    }
}

if (!trackerDriven) {
    // economy gate: check minerals, deduct cost
}
```

Six lines of intent, one convention split into two. The bundling was there from the start — it just never mattered until real data showed up.

## The snapshot that didn't match

There was a third issue, subtler than the other two. The dual-run test was comparing EmulatedGame snapshots taken *after* playbook actions to SC2 snapshots taken *before* actions. In SC2, the adapter observes state, records the snapshot, then sends commands. Effects appear in the next observation. The dual-run test was running the game tick, firing playbook actions (which deduct costs immediately), then snapshotting — making cost deductions visible one tick earlier than they'd appear in SC2.

Moving the snapshot before the playbook block aligns the two engines: observe the state, record it, then act. Effects from this tick's actions show up in the next tick's snapshot. Same semantics, same comparison point.

## What ground truth actually buys you

These bugs had been present since the playbook system was first written. The mining model always started one tick too early. The cost deduction was never wired up. The snapshot timing was always off. None of this showed up in unit tests, integration tests, or replay validation. The unit tests tested EmulatedGame against itself — internally consistent but wrong. The replay validation compared aggregate accuracy at minute-long checkpoints, too coarse to catch a tick-1 offset.

Ground truth comparison at tick granularity found all three in the first run. The dual-run framework exists because aggregate metrics hide root causes — and the first time it ran against real data, it proved the point.
