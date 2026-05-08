---
layout: post
title: "Armour, Retreat, and Walls"
date: 2026-04-15
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, damage-model, pathfinding, tdd, subagent-development]
---

The emulator has been getting more honest about SC2.

Three phases shipped today: armour and damage types (E5), enemy retreat (E6), and terrain-aware pathfinding (E7). Each was a response to the same problem — the simulation was too flat. Every hit did raw damage. Enemies fought to the death regardless of odds. Units walked through walls. None of that is SC2.

## Armour changes the math

E5 started with a data problem. Five unit types had wrong HP values — Immortals defaulting to 100 HP instead of 200, Marines at 100 instead of 45. The defaults had been accepted and passing tests for weeks because nothing asserted their exact values.

We fixed the data, then built `DamageCalculator` in `sc2/emulated/`: stateless, three inputs, one output. Takes attacker type, target unit, applies `max(1, baseDamage + bonusVsAttributes - armour)`, then applies the Immortal's Hardened Shield cap if shields are above zero.

The armour introduction immediately broke two E4 tests. `firstAttackFiresImmediately` and `cooldownExpiresAndAttackFiresAgain` had hardcoded Probe-vs-Zealot assertions: `shields == 50 - 5 = 45`. With Zealot armour now 1, effective damage is 4, so the correct value is 46. The fix: assert derived values, not magic numbers.

```java
int effective = SC2Data.damagePerAttack(UnitType.PROBE) - SC2Data.armour(UnitType.ZEALOT);
assertThat(zealot.shields()).isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - effective);
```

That assertion survives any future stat change.

## The retreat threshold trap

E6 added configurable retreat triggers — per-unit health threshold, army-wide depletion percentage. Units that retreat move back to staging with whatever HP they have, then rejoin the next wave.

Getting transfer-to-staging right required an arrival check. The obvious threshold: `distance >= 0.5`. The `stepToward` function snaps units to their exact target when `dist <= speed` (0.5). The tick before snapping, the distance is mathematically exactly 0.5 — sitting right on the floating-point boundary. The check was non-deterministic. We changed it to `>= 0.1`. After snapping, distance is 0.0; before snapping, it's always above 0.5. No ambiguity.

## Chokepoints, finally

E7 is the biggest piece. A `WalkabilityGrid` — 64×64 boolean tiles — lives in `domain/` so the real SC2 engine can reuse it when ocraft's `PathingGrid` bitmap is eventually wired in. The emulated map has one terrain feature: a horizontal wall at y=18, gap at x=11–13. The straight-line path from enemy staging to the nexus crosses y=18 at x=18 — five tiles clear of the gap. Everyone detours now.

`AStarPathfinder` is stateless, 8-directional, Euclidean heuristic. I'd have missed the 2D array defensive copy — `Arrays.copyOf` on a `boolean[][]` copies references, not rows — but the code quality reviewer caught it. We fixed it to copy each column independently.

The `MovementStrategy` interface was the key design call. `DirectMovement` — the default — wraps `stepToward` identically to before. Every existing test passes without modification. `PathfindingMovement` maintains waypoint queues and recomputes on target change. Pathfinding tests opt in explicitly.

Terrain height — high ground, ramps, the 25% miss chance — is real SC2 and worth building. I logged it in `IDEAS.md` and kept it out of E7. The `WalkabilityGrid` API is designed to extend cleanly; adding height later won't require touching `AStarPathfinder`.

The visualizer fetches wall data once from `/qa/emulated/terrain` and draws dark tiles at startup.

363 tests. The map has terrain.
