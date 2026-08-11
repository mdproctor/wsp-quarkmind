---
layout: post
title: "Both Sides of the Board"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [QuarkusMind]
tags: [emulated-game, enemy-ai, pathfinding, architecture]
---

The scouting calibration was quick — ran the replay dataset, updated three string constants to match the live DRL thresholds, done. The real work was the enemy AI.

I'd been treating the emulated game asymmetrically. The friendly side: CDI beans, an intent queue, a full plugin architecture. The enemy: scattered fields in `EmulatedGame`, a `tickEnemyStrategy()` method that directly mutated lists. When brainstorming the design — reactive counters? fixed build orders? tech tree? — the answer turned out to be all three, and that observation led somewhere larger.

If both players share the same `PlayerState` — units, buildings, minerals, targets, cooldowns, completions — and both drain through the same generalized `applyIntent`, the enemy AI is just a `PlayerBehavior` implementation. The same interface the friendly agent satisfies.

```java
void applyIntent(Intent intent, PlayerState state) {
    switch (intent) {
        case TrainIntent  t -> handleTrain(t, state);
        case BuildIntent  b -> handleBuild(b, state);
        case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation(), true,  state);
        case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation(), false, state);
        case BlinkIntent  b -> executeBlink(b.unitTag(), state);
    }
}
```

Train a Zealot: deduct minerals, queue a pending completion. Applies unchanged to either player. `EmulatedGame` becomes a referee rather than a one-sided simulation.

Three things surfaced during implementation that weren't obvious going in.

The enemy's supply defaulted to zero after `PlayerState.clear()`. `handleTrain()` checks `supplyUsed + cost > supply` — with `supply=0`, every `TrainIntent` is silently rejected. No exception, no log. Production just stops. A code reviewer caught it; the tests ran too briefly to detect the missing units.

`EnemyBehavior` also pre-deducted minerals before pushing the `TrainIntent`, then `handleTrain()` deducted a second time. Two sites of truth for the same resource. Fixed by making `handleTrain()` the single point of deduction.

On the pathfinder side, we hit a subtler problem. Path smoothing used Bresenham line-of-sight — tile-centre to tile-centre. Mathematically correct for ray-casting, wrong for discrete movement. A unit at 0.5 tiles/tick follows a different intermediate-tile sequence than the Bresenham line implies. The wall enforcement fired, invalidated the path, and the unit looped indefinitely. The fix: sample every 0.4 world units along the direction vector.

```java
double step = 0.4; // strictly below movement speed (0.5)
for (double d = 0; d <= dist + step; d += step) {
    double t = Math.min(d, dist);
    if (!grid.isWalkable((int)(from.x() + nx * t), (int)(from.y() + ny * t))) return false;
}
```

The pathfinder itself turned out to be mostly done — A* already in `domain/`, already wired into `EmulatedEngine`. What was missing: RAMP tiles now cost 1.5× in the graph, waypoints get greedy string-pulled after each path computation, and the real SC2 side gets the terrain grid from `SC2BotAgent.onGameStart()` now that it's a proper CDI bean.

The emulator has both sides of the board now.
