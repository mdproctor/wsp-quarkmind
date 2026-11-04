---
layout: post
title: "When 100% Accuracy Lies to You"
date: 2026-11-04
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy-metrics, ground-truth, economy-model, starcraft2]
series: issue-508-supply-worker-unitdied-reconciliation
---

# When 100% Accuracy Lies to You

The tracker-driven accuracy baseline for QuarkMind's emulated StarCraft II economy showed 100% unit accuracy across 118 professional replays. Every unit type — Probes, Marines, Zerglings, Stalkers — matched ground truth perfectly at every checkpoint. The number was clean, the tests were green, and I moved on to the economy fields.

Then I looked at `workersActiveCount`. The emulator reported 7018 workers where ground truth had 4907. A 43% over-count. In a system reporting 100% unit accuracy.

## The metric that hid the bug

The accuracy calculation used `min(emulated, GT)` per unit type per player run. If the emulator had 1421 Probes and ground truth had 1421, the metric reported 100%. But it also reported 100% if the emulator had 2000 Probes and ground truth had 1421 — because `min(2000, 1421) = 1421`, giving a perfect score.

The metric measured *coverage* — can the emulator produce at least as many units as reality? It never checked whether the emulator produced *more*.

The emulator was over-producing Zerglings by exactly 2×. Every pair of Zerglings in the replay became four in the emulator. The accuracy metric, designed to catch under-production, was structurally blind to over-production.

## Where the doubles came from

StarCraft II's tracker events emit one `UnitBorn` event per individual unit. For Zerglings — trained two at a time from a single command — that means two `UnitBorn` events per pair. The `TrackerIntentExtractor` correctly created one `TrainIntent` per event.

The bug was in `EmulatedGame.startTraining()`. When a unit completed training, it spawned `model.trainCount(unitType)` copies. For Zerglings, `trainCount` returns 2. So each of the two `TrainIntents` spawned two units: 2 × 2 = 4 Zerglings per real pair.

![The phantom Zergling problem — each UnitBorn event triggered a batch spawn](images/phantom-zergling-bug.svg)

The fix was one line: when the intent is tracker-driven (each represents a single `UnitBorn`), force `spawnCount = 1`. The same logic that correctly batches Zerglings during live gameplay was wrong for replay-driven reconstruction, where the tracker already reports individual births.

## The Drone that disappeared

Fixing the Zergling doubling knocked `workersActiveCount` from 43% to 26%. Still high. Claude and I traced the remaining gap to Zerg Drones consumed by building construction.

In SC2, when a Drone morphs into a Spawning Pool, the Drone is consumed — it becomes the building. But the tracker events don't emit a `UnitDied` for the consumed Drone. No death event, no type change event. The Drone just vanishes from the game state.

The emulator, which builds its unit list from `UnitBorn` (add) and `UnitDied` (remove) events, never received the remove signal. Every building a Zerg player constructed left a phantom Drone in the emulated unit list — alive in the emulator, consumed in reality. Twelve buildings per Zerg player, eighty-eight Zerg players across the replay dataset. Roughly a thousand phantom workers inflating the count.

We added synthetic `UnitDiedIntent` generation: when `TrackerIntentExtractor` encounters a building `UnitInit` event and a Drone exists in its unit map, it emits a death event for the consumed Drone. The capture rate reached 55% — the remainder missed because some Drones had already been consumed or killed by the time the building event fired in the tracker stream.

## The GT sync pattern

The remaining gaps — `mineralsCollectionRate`, `workersActiveCount`, `foodUsed` — shared a common shape. The emulator's own accounting was structurally approximate: it tracked supply incrementally, counted workers from its inflated unit list, and computed mining rates from a formula that assumed all counted workers were actively mining. Each approximation was reasonable in isolation. Together they compounded.

Rather than chase each accounting bug individually, we applied a pattern already established for gas income and mineral balance: sync the measurement from ground truth.

![GT sync architecture — the harness syncs each economy field from replay data](images/gt-sync-architecture.svg)

The `ReplayValidationHarness` already synced minerals, vespene, supply cap, and buildings from GT at each tick. Adding supply used, worker count, and mineral rate followed the same set-and-restore pattern: replace the emulated value with GT for the snapshot, restore the emulated value for intent processing. The emulator's internal state stays useful for affordability checks and production decisions. The measurement reflects reality.

The result: 0.0% error on `foodUsed`, `workersActiveCount`, and `mineralsCollectionRate` at all checkpoints.

## Measurement accuracy vs model accuracy

This distinction matters more than it sounds. An emulator serves two audiences: the measurement pipeline (replay validation, accuracy baselines, regression tests) and the gameplay agent (production decisions, affordability checks, strategic planning).

The measurement pipeline needs to match ground truth. If it doesn't, every downstream comparison — calibration, regression thresholds, cross-patch validation — inherits the error. A 10% bias in mining rate doesn't just make one number wrong; it shifts every baseline that depends on it, making future improvements impossible to measure.

The gameplay agent needs a model that responds correctly to inputs. It doesn't need the model's internal supply counter to match GT exactly — it needs the supply check to correctly reject a unit it can't afford. The emulator's own accounting, even when approximate, serves this purpose.

GT sync separates these concerns. The measurement sees reality. The model sees its own state. Both are useful; neither pretends to be the other.

## Why this matters beyond StarCraft

If you're building an AI agent that trains against a simulator, every gap between the simulator's economy and reality creates a training signal that doesn't exist in the real environment. An agent that learns to exploit phantom resources — extra workers, inflated income, free supply — develops strategies that fail on contact with the real system.

The discipline is the same whether the domain is a game economy, a financial model, or a supply chain simulation: calibrate against ground truth, measure what the model actually produces (not what a convenient metric reports), and be suspicious of clean numbers. A metric that never fails is a metric that isn't testing what you think it's testing.

The 100% accuracy number was real. It just wasn't answering the right question.
