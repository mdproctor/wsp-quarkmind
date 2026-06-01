---
layout: post
title: "Opening the Race Plugin Seam: Splitting PlayerState"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [playerstate, refactor, plugin-seam, racemodel, physicsstate]
---

Issue #164 was a small one by description: `PlayerState` is package-private and `RaceModel` takes it as a parameter, so external race implementations (required by #74) can't implement the interface. Make `PlayerState` public.

The obvious fix is one access modifier. I didn't want the obvious fix.

`PlayerState` was doing three things: holding game state (units, buildings, minerals), holding EmulatedGame's physics machinery (movement targets, cooldowns, production queues, pending completions), and holding the enemy retreat buffer (`stagingArea`). Making that class public with the physics fields exposed would have handed external plugin authors a type polluted with EmulatedGame internals. That's a stable API in the same way a leaky pipe is a plumbing fixture.

The design conversation settled on a three-way split. `PlayerState` keeps only game state — the fields that make sense to a race model plugin. A new `PhysicsState` (package-private) takes the seven maps that EmulatedGame uses internally: unit targets, cooldowns, blink cooldowns, building queues, training timers, completion callbacks. The `stagingArea` — the enemy retreat buffer — moves to `EnemyBehavior`, where it belonged semantically all along.

That last one was obvious in hindsight. The staging area is wave management state. It has nothing to do with the player's game state or EmulatedGame's physics. It was in `PlayerState` because early `PlayerState` was a grab bag.

The public API on `PlayerState` is split along one clear line: public methods are semantic operations meaningful to a race plugin (`addUnit`, `setMinerals`, `addSupply`), package-private methods are bulk structural operations for the physics engine (`replaceAllUnits`, `removeUnitsWhere`). The `removeUnitsWhere` bulk op returns the removed units as a `List<Unit>` — that's not obvious but necessary. The dead-unit cleanup in `resolveCombat` needs to remove from three physics maps for each dead unit. Without the returned list, you'd need a separate pre-collect pass. With it, the cleanup reads naturally.

The spec review cycle caught a real bug before any code was written. The `resolveCombat` code example in the spec had `u.hp() <= 0` — there is no `.hp()` accessor on `Unit`. The correct method is `.health()`. Claude flagged it during the second review pass. One of those things that would have compiled fine and then failed subtly at runtime if the spec example had been copied blindly.

We migrated the three race model implementations to the new public API alongside the refactor, not after. This was deliberate: the only way to know the public API surface is complete is to use it. Terran's MULE calldown, Zerg's larva cycle and EGG lifecycle, Protoss's Pylon supply — all exercised through the typed methods. If anything had been missing, the migration would have caught it.

841 tests pass. `RaceModel`, `ProductionResult`, and `RaceModelFactory` are public. External implementations can now implement `RaceModel` against a clean, stable `PlayerState`. The installation mechanism — how `EmulatedGame` accepts an external model — stays package-private until #74 designs it properly. That's tracked in #165.
