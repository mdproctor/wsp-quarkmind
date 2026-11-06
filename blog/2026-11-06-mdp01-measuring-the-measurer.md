---
layout: post
title: "Measuring the Measurer"
date: 2026-11-06
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, ground-truth, accuracy-metrics, starcraft2, simulation, measurement]
series: issue-534-emulated-game-100-accuracy
---

# Measuring the Measurer

Command Center accuracy at 15.7%. Hatchery at 64%. The emulator was supposed to be tracking buildings from tracker events — a system we'd validated at 100% for Protoss Nexus across 118 professional SC2 replays. Something was destroying Terran and Zerg base buildings faster than a 4-pool.

The instinct was obvious: the emulator is dropping expansion buildings. Maybe BuildIntents aren't firing for second and third bases. Maybe the completion callback isn't wired. Maybe the CC-to-Orbital-Command morph is consuming the building without creating the replacement. I went looking for the bug in the emulator.

It wasn't there.

## The morph event nobody handled

StarCraft II tracks building lifecycle through four tracker events: `UnitBorn` (instant appearance), `UnitInit` (construction begins), `UnitDone` (construction complete), `UnitDied` (destroyed). When a Terran player morphs a Command Center into an Orbital Command, a fifth event fires: `UnitTypeChange`. Same building tag, new type.

The emulator handles this correctly. TrackerIntentExtractor sees the `UnitTypeChange`, emits a `MorphIntent`, and EmulatedGame calls `replaceAllBuildings` — the CC becomes an OC. One building in, one building out.

The ground truth doesn't handle it at all.

`ReplaySimulatedGame.applyTrackerEvent()` dispatches on event ID. It handles `UnitBorn`, `UnitInit`, `UnitDone`, `UnitDied`, `PlayerStats`, `Upgrade`, `UnitPositions`. Seven event types. `UnitTypeChange` falls through the switch with no case.

What happens when a CC morphs to an OC in the ground truth: nothing. The CC stays a CC forever. The OC never appears. Every Terran Command Center that was ever morphed to Orbital Command — which is nearly all of them — remained counted as a CC in the ground truth data.

![The GT morph bug — UnitTypeChange had no handler, so morphed buildings kept their original type](images/gt-morph-bug.svg)

The 15.7% accuracy wasn't measuring emulator quality. It was measuring how many CCs hadn't been morphed yet by the 7-minute mark. Across 74 Terran player-runs, only 18 of 115 ground-truth "CCs" were actually still Command Centers. The other 97 were Orbital Commands that the ground truth was miscounting.

## The diagnostic that proved it

A single aggregate table made the root cause undeniable:

| Building | 1-min | 3-min | 5-min | 7-min |
|----------|-------|-------|-------|-------|
| CC (before) | GT=74, EM=74 | GT=94, EM=42 | GT=113, EM=22 | GT=115, EM=18 |
| CC (after) | GT=74, EM=74 | GT=41, EM=42 | GT=22, EM=22 | GT=18, EM=18 |
| Nexus | GT=61, EM=61 | GT=72, EM=72 | GT=80, EM=80 | GT=68, EM=68 |

Nexus — the Protoss town hall that never morphs — was 100% at every checkpoint, both before and after the fix. That's the control. CC and Hatchery, the two town halls with morph targets (Orbital Command, Lair), were the only buildings affected. The fix was one switch case.

![Building accuracy before and after the GT fix — every morphable building jumps to 100%](images/accuracy-before-after-535.svg)

## The spending anomaly hiding in plain sight

With buildings fixed, I turned to the economy fields. Technology spending at 1-minute: the emulator reported 25,100 minerals. Ground truth reported 0.

That's not a rounding error. It's a category error. SC2's `PlayerStatsEvent` reports spending in three buckets — `Current`, `InProgress`, and `Lost`. `mineralsUsedCurrentTechnology` tracks minerals invested in *completed* technology buildings. At 1-minute, most technology buildings are still under construction. Their costs live in the `InProgress` bucket until construction finishes, then migrate to `Current`.

The emulator was recording spending at order time. The ground truth reports it at completion time. Every building ordered but not yet finished inflated the emulator's spending relative to the ground truth. The fix: move `recordBuildSpending` from the intent handler into the `PendingCompletion` callback. Resources are still deducted immediately — the minerals are gone from the player's bank the moment they click Build. But the spending *classification* defers to completion, matching SC2's semantics.

Technology spending MAPE at 3-minutes dropped from 46% to 16.8%. The 1-minute anomaly vanished entirely.

One more: starting buildings. Every player begins with a town hall — Nexus, Command Center, or Hatchery. These arrive as `BuildingBornIntent` (instant, no construction). The handler added the building, registered it with mineral patches, tracked its supply. It never recorded the spending. Across 236 player runs, that's roughly 82,000 minerals of economy spending that the emulator never reported.

Economy spending MAPE at 1-minute dropped from 27% to 1.6%.

## The broader lesson

Three bugs. One in the ground truth, two in the emulator's reporting. None in the emulator's physics. Every one was a measurement problem — the simulation was producing correct state, but the numbers extracted from that state were wrong.

This is the calibration equivalent of the observer effect. The act of measuring the simulation introduced errors that had nothing to do with the simulation itself. The ground truth overcounted. The spending tracker recorded at the wrong lifecycle phase. The starting buildings were invisible to the accounting system. Each error was small enough to dismiss, large enough to compound, and structurally undetectable from the aggregate accuracy number alone.

The fix in every case was the same: trace the data path from source to measurement. Not "what is the number?" but "how does this number get produced?" The CC accuracy wasn't wrong because the emulator dropped buildings. It was wrong because the ground truth's event dispatcher had a gap. The spending wasn't wrong because the emulator miscalculated costs. It was wrong because the emulator reported costs at a different lifecycle phase than the ground truth expected.

For anyone building a simulation-vs-reference calibration pipeline — in game AI, in robotics, in any domain where you compare a model against recorded data — the lesson is concrete: validate the reference. Ground truth is code too. It has bugs. When the model disagrees with the reference, the model is the suspect, but the reference is not exempt.
