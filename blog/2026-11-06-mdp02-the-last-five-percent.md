---
layout: post
title: "The Last Five Percent"
date: 2026-11-06
entry_type: article
subtype: diary
projects: [casehubio/quarkmind]
tags: [calibration, accuracy, simulation, starcraft2, event-mapping, digital-twin]
series: issue-534-emulated-game-100-accuracy
---

# The Last Five Percent

Building accuracy was at 94.7%. Unit accuracy at 98.0%. Upgrades already at 100%. The emulator tracked every Protoss Gateway, every Zerg Spawning Pool, every Terran Barracks. Across 118 professional SC2 replays and 236 player runs, the physics engine was producing the right entities — it just wasn't producing *all* of them.

The gap was 5.3% for buildings and 2.0% for units. Both are now 0%.

Getting there required five distinct fixes, each addressing a different failure mode. None were physics bugs. None were algorithm errors. Every one was a gap in the event-to-handler mapping — a place where the real game produced an event that the simulator didn't know how to process.

## The pattern: lookup tables as maintenance surfaces

StarCraft II's replay format communicates game state through tracker events. A building starts construction (`UnitInit`), finishes (`UnitDone`), changes type (`UnitTypeChange`), and eventually dies (`UnitDied`). The emulator maps each event to a handler: `UnitInit` becomes a `BuildIntent`, `UnitTypeChange` becomes a `MorphIntent`, and so on.

The mapping lives in a set of lookup tables: `MORPH_BUILDING_TARGETS` maps SC2 type names to `BuildingType` enum values, `MORPH_UNIT_TARGETS` does the same for units, and `BUILDING_NAMES` determines which tracker entities are recognised as buildings at all.

These tables are correct for every entry they contain. The problem is the entries they *don't* contain.

![SC2 tracker events flow through handlers — five gaps found, each a missing entry or lifecycle edge case](images/event-handler-gaps.svg)

## Fix 1: Ghost completions

When a building is destroyed mid-construction, `handleBuildingDied` removes it from state. But the `PendingCompletion` callback — scheduled when construction started — was never cancelled. The callback fires at the expected completion tick, records spending for a building that no longer exists, and silently inflates the technology spending total.

The fix adds an `entityTag` field to `PendingCompletion`. When a building dies, all pending completions for that tag are cancelled before the building is removed. This eliminates a class of ghost callbacks that would have grown worse as games ran longer and more buildings were destroyed.

## Fix 2: Add-on swaps

Terran players routinely swap add-ons between production buildings. Lift a Factory off its Reactor, land a Starport on it — the Reactor's type changes from `FactoryReactor` to `StarportReactor`. SC2 emits a `UnitTypeChange` event for this, and `TrackerIntentExtractor` correctly generates a `MorphIntent`.

But `MORPH_BUILDING_TARGETS` only contained five entries: Lair, Hive, Greater Spire, Orbital Command, and Planetary Fortress. It had no entries for add-on variants — six type names that the handler silently rejected. The building kept its original type. Factory count: wrong. Starport count: wrong.

The fix adds an `ADDON_SWAP_TARGETS` map for all six add-on variants and handles them as instant type swaps — no construction time, no pending completion, the building just changes type. Factory, Starport, and Barracks all jumped to 100%.

## Fix 3: Morph completion collision

This was the subtlest bug. When a Command Center morphs into an Orbital Command, the building tag gets two `UnitDone` events: one when the CC finishes construction, and one when the OC morph completes. The pre-pass that collects completion loops stored them in a flat map — last write wins. The CC construction got the OC's completion loop.

The consequence: the CC's `PendingCompletion` was pinned to the *morph's* completion time, not the *construction's* completion time. When the `MorphIntent` for the OC arrived, the CC was still marked incomplete. The morph was rejected. The building stayed a CC forever. Orbital Command accuracy at 3-minutes: 84.9%. Lair accuracy at 5-minutes: 22.2%.

The fix changes the pre-pass from `Map<Long, Integer>` to `Map<Long, Deque<Integer>>`. Each `UnitInit` consumer polls the next entry from the queue, getting its own correct completion loop. Combined with making tracker-driven morphs instant type swaps (matching how the ground truth handles them), both OC and Lair jumped to 100%.

## Fix 4: Creep Tumor burrow

When a Queen places a Creep Tumor, SC2 creates a `CreepTumorQueen` building. It then burrows, becoming `CreepTumorBurrowed`. The `UnitTypeChange` event fires, `TrackerIntentExtractor` generates a `MorphIntent` — and `MORPH_BUILDING_TARGETS` doesn't contain `"CreepTumorBurrowed"`. The building stays typed as `CreepTumorQueen` instead of becoming `CreepTumor`. The baseline counts `CREEP_TUMOR` buildings. None of the Queen-placed tumors register.

One line: `"CreepTumorBurrowed", BuildingType.CREEP_TUMOR`. Creep Tumor accuracy went from 0-57% to 100%. **Buildings hit 100% at every checkpoint.**

## Fix 5: Mode switches and a name casing bug

Siege Tanks siege. Warp Prisms phase. Liberators enter AG mode. Each mode switch is a `UnitTypeChange` event that the emulator filtered out — the handler only generated `MorphIntent`s for permanent morphs (Roach→Ravager, Overlord→Overseer), gated by `SC2Data.morphSourceType() != null`. Mode switches returned null and were silently dropped.

Alongside this, a name casing bug: the `UnitTypeChange` handler used `newType.name()` — the Java enum name `"RAVAGER"` — but `MORPH_UNIT_TARGETS` used the SC2 camelCase name `"Ravager"`. The lookup failed for every permanent morph generated from a type-change event.

Two changes: remove the morph-source filter so mode switches generate intents, and use the SC2 event name instead of the Java enum name. Add a `MODE_SWITCH_TARGETS` map for instant unit type swaps that preserve HP, shields, and cooldowns. **Units hit 100% at every checkpoint.**

![Accuracy progression across five fixes — buildings and units both reach 100%](images/accuracy-climb-to-100.svg)

## Why this matters beyond StarCraft

QuarkMind is a living lab for the CaseHub agentic harness — a system where plugin agents coordinate via a shared blackboard to make decisions under time pressure. The SC2 domain is the proving ground, but the pattern is the product. The emulator's accuracy directly determines whether the harness can train, evaluate, and improve its agents without access to a running game.

At 100% entity accuracy, the emulator produces the same buildings, units, and upgrades that a real SC2 replay records. An agent trained against emulated games sees the same decision surface it would see in a real match. That's the threshold where offline training becomes trustworthy.

But the path to 100% reveals something more general about simulation fidelity. The five bugs share a structural signature: every one is a gap in a finite lookup table or a handler that doesn't account for a lifecycle edge case. The physics engine was correct from the start. The economy model produces income within 2% of ground truth. Supply tracking is accurate to 1%. The gaps were all in the *mapping layer* — the code that translates between the source system's event vocabulary and the simulator's internal model.

This is the pattern that bites every digital twin project. You build the core simulation, validate it against reference data, get to 95%, and declare it good enough. The last 5% is where you discover that the source system has more event types than you enumerated, more lifecycle transitions than you handled, and more edge cases than your lookup tables anticipated. Each missing entry is individually trivial — one line of code, one map entry, one enum value. Collectively, they're the difference between a simulation that's good for demos and one that's good for training.

The fix in every case is the same: per-type breakdowns, not aggregate accuracy. An aggregate number of 94.7% hides the fact that Factory accuracy is 86% and Creep Tumor accuracy is 0%. The per-type breakdown makes the gap visible. The gap tells you which handler to read. The handler tells you which lookup table is incomplete.

Enumerate, don't estimate. The source system's event vocabulary is finite. Your handler coverage either matches it or it doesn't. The last five percent is just the entries you haven't added yet.
