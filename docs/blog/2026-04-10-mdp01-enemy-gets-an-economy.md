---
layout: post
title: "The Enemy Gets a Brief"
date: 2026-04-10
---
# The Enemy Gets a Brief

**Date:** 2026-04-10
**Type:** phase-update

---

E3 left us with a working combat loop: units fight, shields absorb, units die. The enemy Zealots still marched from spawn to nexus on a script — at frame 200, regardless of what the player bot was doing. That was fine for testing combat mechanics. It's not fine for testing actual AI.

E4 is about giving the enemy a real job. Not a scripted wave, but an economy: workers mining minerals, production buildings going up, Marines rolling out, and an attack decision that fires when conditions are right. The kind of opponent the bot's scouting and tactics can actually respond to.

I ran the post-E3 benchmark before touching anything:

```
Phase               mean     p95      max
Total gameTick()      0ms      1ms      1ms
```

Identical to the pre-E2 baseline. E3's two-pass combat resolution added zero measurable latency. E4 can add an enemy economic loop without any historical pressure to keep it thin.

![SC2 Marine — the enemy's combat output](/quarkmind/blog/assets/sc2-marine.jpg)

## The design question: how real?

The first question was how realistic the enemy economy should be. I brought Claude in to brainstorm the design. The obvious divide was between a simplified income loop — minerals tick up on a timer, units pop out periodically — and a full simulation where the enemy has visible workers, buildings, and a real economic pipeline.

Claude proposed a middle option: run the full economic loop internally, but keep enemy buildings out of `GameState` until a future scouting phase needed to see them. A sensible hedge. I pushed back. The `GameState` domain change — adding `List<Building> enemyBuildings` — cascades through about nine callsites, but the compiler catches every one. Given we'd write tests anyway, there was no good reason to phase it. We committed to the full version: enemy workers and buildings visible in `GameState` from day one.

![SC2 Command Center — the enemy's starting building](/quarkmind/blog/assets/sc2-command-center.jpg)

The race choice was Terran. SC2Data already has Marine and Marauder damage/range data from E3, and Terran's supply model maps cleanly onto what Pylon already does for the player side. It's not a hard constraint — `EnemyRace` is an enum designed to swap in Zerg or Protoss mirror later.

## Three attack strategies, not one

The other interesting decision was attack triggering. I'd expected to pick one model: fire when army size hits a threshold, or fire on a timer. Claude offered three variants and I asked why not all three. There's no complexity argument against it — the implementation is one sealed interface with three records:

```java
sealed interface AttackStrategy
    permits AttackStrategy.Threshold, AttackStrategy.Timer, AttackStrategy.Hybrid {

    record Threshold(int minArmy)               implements AttackStrategy {}
    record Timer(int everyNTicks)               implements AttackStrategy {}
    record Hybrid(int minArmy, int everyNTicks) implements AttackStrategy {}

    default boolean shouldAttack(int armySize, long gameFrame) {
        return switch (this) {
            case Threshold(int min)         -> armySize >= min;
            case Timer(int every)           -> gameFrame > 0 && gameFrame % every == 0;
            case Hybrid(int min, int every) -> armySize >= min && gameFrame > 0 && gameFrame % every == 0;
        };
    }
}
```

All three strategies in one pattern match. Tests for each variant are pure logic — pass in army size and frame, check the boolean.

## Eleven tasks, commit-by-commit

We ended up with eleven implementation tasks, all TDD. Domain scaffolding goes first — adding `SCV`, the Terran building types, and `GameState.enemyBuildings` — because the compiler will catch every callsite automatically. Then `AttackStrategy`, `EnemyRace`, and `EnemyAI` as a new class that holds references to `EmulatedGame`'s shared lists and writes to them each tick.

The most involved piece is `EmulatedGame` itself: removing `EnemyWave` and `configureWave()`, wiring in `configureEnemyAI()`, and fixing a bug carried over from E3 — a unit given a `MoveIntent` after an `AttackIntent` doesn't stop attacking. The `attackingUnits` set was never cleared by a move command. That gets fixed in the same PR.

Eleven tasks standing by.
