---
layout: post
title: "The Engine Fights Back"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, sc2, java, testing]
excerpt: "Phase 6 requires the emulated engine to behave like real SC2 — fixing independent building queues, supply reservation timing, building type validation, and an enemy AI that generated fake building tags."
---

Phase 6 requires that the emulated engine behave like real SC2. Not approximately — divergence from real replays shows up as systematic error, not noise. Two things that clearly didn't match were training queues and combat.

In real SC2, buildings train units independently. Two Gateways produce Zealots simultaneously; supply is consumed when you queue the unit, not when it finishes; a Nexus cannot train a Stalker. None of this was enforced. The emulator had a single flat `pendingCompletions` list — every train intent added to it regardless of which building issued it, supply was reserved at completion time, and building tags were ignored entirely.

The enemy AI was worse. `EnemyBehavior.tickProduction()` generated invented building tags like `"enemy-nexus-1042"` that had no counterpart in `enemy.buildings`. You can't validate building types against fake tags.

We fixed the full picture. `PlayerState` gained two maps: `buildingQueues` and `buildingTrainingUntil`. Each building now has its own training slot, capped at five total — one active, four waiting, matching SC2. Supply is reserved at queue time. `handleTrain` validates the building tag against `state.buildings` and checks that the building type matches `SC2Data.trainedBy(unitType)`. `EnemyBehavior` now looks up the real building from `enemy.buildings` using that same mapping.

The interface cleanup fell out of code review. `Intent.unitTag()` was declared on the sealed interface, but running `ide_find_references` on it returned zero usages through the interface type. Every call site pattern-matched to a concrete type first. The method was dead on the interface. We removed it and dropped the `@Override` bridge in `TrainIntent` that had been keeping the contract alive.

Auto-engage was architecturally simpler, but the discovery was interesting. `moveEnemyUnits()` already had a stop-to-fight range check: if a friendly entered an enemy unit's weapon range, the enemy stopped moving and `resolveCombat()` was supposed to handle the attack. But `resolveCombat()` gated on `attackingUnits.contains(tag)` — a set populated only by explicit `AttackIntent`. Units stopped but didn't fire unless they'd been part of a wave launch.

The two mechanisms were contradicting each other. We removed both: the `attackingUnits` gate from `resolveCombat` and the stop-to-fight block from `moveEnemyUnits`. Any unit with an enemy in weapon range fires each tick. Units advance and fire simultaneously. That's what SC2 does.

Eleven tests needed adjustment — combat tests where enemy units previously held position now see them advancing while engaging. The adjustments were mostly position changes to isolate specific interactions. One precision failure was worth noting: a test asserting `zealot.shields() == maxShields - damagePerAttack(PROBE)` expected 45, got 46. Zealot has 1 armour. The emulator applies armour uniformly to total damage before splitting between shields and HP — shields aren't armour-exempt. The correct assertion uses `max(0, rawDamage - armour)`.

Phase 5 has one item left: the mineral saturation model. The engine can now queue training correctly, validate building types, and engage without being commanded to. Replay validation is within reach.
