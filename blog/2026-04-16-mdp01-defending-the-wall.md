---
layout: post
title: "Defending the Wall"
date: 2026-04-16
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, pathfinding, debugging]
---

Something went through the wall. I said it was the probe — consistently,
across multiple restarts. Claude kept checking enemy pathfinding.

The wall was supposed to be fixed. E7 wired `PathfindingMovement`, the
diagnostic log confirmed it was active, the unit test showed enemies
respecting the gap at x=11–13. And yet.

We added file logging to `application.properties` to get the output to
disk — `%emulated.quarkus.log.file.enable=true`, path to `/tmp/quarkmind-emulated.log`
— so we could `grep` it without copy-pasting hundreds of lines from a
live terminal. That surfaced the culprit immediately:

```
[SCOUTING] Scout probe unit-200 dispatched toward Point2d[x=224.0, y=224.0]
```

`unit-200` was a friendly probe, not an enemy. `DroolsScoutingTask`
dispatches scouts using real SC2 map coordinates. The emulated map is
64×64. The target was (224, 224).

The bug was in `AStarPathfinder.nearestWalkable`. The spiral search for a
valid tile is capped at `Math.max(grid.width, grid.height) = 64`. From
(224, 224), you need radius 161 to reach any tile within bounds. The cap
hits at 64; the function returns the original point unchanged. A* finds no
path. `PathfindingMovement` falls back silently to `stepToward` — which
is `DirectMovement`, straight through walls.

The fix is one line before the search:

```java
x = Math.max(0, Math.min(grid.width()  - 1, x));
y = Math.max(0, Math.min(grid.height() - 1, y));
```

(224, 224) becomes (63, 63) — immediately walkable. A* finds a path
through the gap and the scout navigates correctly to the far corner of the
map.

## One rule isn't enough

Fixing the pathfinder handles this case. It doesn't handle future movement
strategies, path computation edge cases, or targets we haven't tested yet.
So we added a physics constraint directly to `EmulatedGame` — independent
of whatever movement strategy is wired:

```java
private Point2d enforceWall(String unitTag, Point2d proposed, Point2d current) {
    if (walkabilityGrid == null) return proposed;
    if (!walkabilityGrid.isWalkable((int) proposed.x(), (int) proposed.y())) {
        movementStrategy.invalidatePath(unitTag);
        return current;
    }
    return proposed;
}
```

Every movement call passes through this. If the proposed position is a
wall tile, the unit holds its position and the path is invalidated so
A* recomputes on the next tick. The pathfinder handles routing; the
physics layer handles correctness. Each is independently testable —
`DirectMovement` plus the wall grid is enough to verify the constraint
fires, no pathfinding needed.

The scout now detours around the wall and comes through the gap. I can
watch it from the visualizer.

368 tests. The wall holds.
