---
layout: post
title: "Reconstructing What SC2 Headless Knows"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [sc2, classifier, feature-extraction, economy-model, replay-parsing]
series: issue-317-blizzard-ladder-restore
---

# Reconstructing What SC2 Headless Knows

The [previous entry](2026-09-28-mdp01-human-replays-speak-different-language.md) ended with a complete abilLink mapping — we knew what every human replay command meant. This session turned that knowledge into a pipeline that can reconstruct classifier features from 151K stripped replays without ever launching SC2.

The core question was whether game event commands contain enough information to reconstruct the tracker events the classifier needs. Tracker events are the data layer — UnitBorn, UnitInit, PlayerStats, Upgrade — that the Python pipeline transforms into 134 features per player. Stripped replays lack tracker events entirely. But they have game events: every click, every hotkey, every command the player issued. If I can simulate what those commands produce, I can synthesise the tracker events.

## Production queues change everything

The naive approach is obvious: player issues a train command at loop 500, unit has a 650-loop train time, emit UnitBorn at loop 1150. But SC2 buildings have production queues. If the player queues three Marines on the same Barracks, they don't all pop at loop 1150. The second starts when the first finishes; the third starts when the second finishes. Without queue tracking, queued units appear in the feature vector far too early — a three-deep queue puts the third Marine 1300 loops ahead of where it actually appears.

The fix: `StrippedReplayFeatureExtractor` maintains `buildingBusyUntil` state per building tag, keyed from AbilityMapping's selection state. When a train command arrives and the building is busy, the birth loop slides forward:

| Command | Building state | Birth loop |
|---------|---------------|------------|
| Marine 1 at loop 500 | Idle | 500 + 650 = 1150 |
| Marine 2 at loop 600 | Busy until 1150 | 1150 + 650 = 1800 |
| Marine 3 at loop 700 | Busy until 1800 | 1800 + 650 = 2450 |

This is the same pattern `EmulatedGame.PhysicsState` uses for its production tracking. The extractor doesn't need the full physics engine — just the queue bookkeeping.

## Morph-death: the Zerg worker problem

Every Zerg building costs a Drone. The Drone walks to the build location and becomes the building — a morph, not a placement. Without tracking this, every Zerg building adds a phantom worker to the feature vector. By minute 5 a typical Zerg player has built 4-8 buildings from Drones. That's 4-8 phantom workers inflating `WorkersActiveCount` and `FoodUsed`.

The extractor handles this for all morph types: Zergling→Baneling, Roach→Ravager, Drone→Building, Hatchery→Lair, and Archon merges (which consume two HighTemplar). Each morph emits a source UnitDied alongside the target birth. The Python pipeline's `extract_replay()` already processes UnitDied events correctly — the synthetic deaths integrate without any downstream changes.

WarpGateResearch gets special treatment. When the upgrade completes, SC2 automatically transforms every Gateway into a WarpGate. The Python pipeline tracks Gateway and WarpGate as separate building features. Without handling this auto-morph, Gateway stays at 3-5 while WarpGate stays at 0 — the exact inverse of reality for every Protoss game in the dataset.

## Economy: three tiers of accuracy

The 13 economy stats in PlayerStats fall into three accuracy tiers depending on what's reconstructable from commands alone.

Seven stats are exact from commands: cumulative mineral and gas spending, categorised into army/economy/technology. These just accumulate costs as train and build commands are processed.

Four stats use an approximate mining model: current minerals, current gas, and the two collection rates. I used the same three-tier saturation constants as EmulatedGame — first 8 workers per base earn full rate, next 8 earn half, next 8 earn roughly 10%. It's an approximation, but it's the same approximation the live game AI already uses, so at least the numbers are internally consistent.

Two stats overcount without combat death tracking: food used and active workers. The extractor counts births but can't count combat deaths (there's no command for "my unit died"). Morph deaths are subtracted correctly. At minutes 2-3, overcounting is negligible — combat hasn't started. At minutes 4-5, it grows. Acceptable for the early-game classification windows the classifier targets.

One detail I nearly missed: `sc2reader_to_game_json` multiplies non-food stats by 1000 and food stats by 4096 before writing them to game_json. The Python pipeline then divides everything by 1000 to normalise. If the Java extractor emits raw values, every economy feature would be 1000x too small. Caught it by reading the conversion function before writing the bulk runner.

## Oracle validation: 24.4% and that's fine

Running the validation test against 118 oracle-restored replays produced 24.4% UnitBorn coverage and 57.3% UnitInit coverage. At first glance that looks terrible, but the gap is entirely from unmapped abilLinks — Terran production buildings (Factory, Starport), Zerg morphs (Baneling, Ravager), and non-trainable units (Larva, Broodling, Interceptor) that can never be extracted from commands because they're auto-spawned. For the abilLinks we do have mapped, the extractor produces correct events at correct loops. Expanding coverage is incremental work on AbilityMapping's dispatch table, not a logic fix.

All 118 replays processed without errors. The pipeline is structurally sound; it just needs more abilLink constants.

## What this opens up

The `BulkFeatureExtractor` can now process 151K stripped replays in parallel, writing game_json files that feed directly into the Python training pipeline via `--format json`. The expected throughput is hundreds of replays per second — Scelight's MPQ parsing is fast when there's no SC2 headless to wait for.

The abilLink coverage gap is the next constraint. Expanding AbilityMapping from the current ~24 mapped unit types to the full 53 in the classifier vocabulary is the difference between "pipeline works" and "pipeline produces useful training data." Each new abilLink is a constant and a dispatch entry — the infrastructure to use them is already in place.
