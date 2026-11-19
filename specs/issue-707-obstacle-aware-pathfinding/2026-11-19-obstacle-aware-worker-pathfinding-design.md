# Obstacle-Aware Worker Pathfinding — Design Spec

**Issue:** #707
**Branch:** issue-707-obstacle-aware-pathfinding
**Date:** 2026-11-19
**Scope:** Worker economy paths only (mineral + gas walks). Army pathfinding deferred to #731/#732.

## Problem

The EM moves workers in straight lines (`SimulatedWorker.moveOneLoop()`). SC2 workers navigate around buildings, mineral patches, and other obstacles. Every compensation constant in the approach docking stack exists to paper over this missing physics:

| Compensation | What it absorbs | File |
|---|---|---|
| `computeApproachDocking()` | Per-patch path deviation from mineral-line geometry | MineralPatch.java:63-88 |
| `effectiveApproachDocking()` | Density-dependent outbound path delay scaling | WorkerSimulation.java:95-110 |
| `PATH_DEVIATION_SCALE` (0.4) | Walk distance scaling factor | MineralPatch.java:29 |
| `PATH_DEVIATION_BASE` (4.5) | Base path delay per patch | MineralPatch.java:30 |
| `outboundPathDelay` | Per-patch outbound delay (density-scaled) | MineralPatch.java:27 |
| `approachDockingLoops` | Per-patch total approach overhead | MineralPatch.java:26 |

SC2 evidence (#690): outbound walk is 6% longer than straight-line (routing around mineral line), return walk is 35% shorter (direct path, no obstacles between TH and patch approach side).

## Design

Three changes, applied together.

### 1. Dynamic obstacle layer on TerrainGrid

Add mutable obstacle state to `TerrainGrid`. Obstacles override walkability — a cell marked as an obstacle returns `isWalkable() = false` regardless of terrain height.

```java
// TerrainGrid — new fields and methods
private final boolean[][] obstacles;  // [x][y], default false

public void markObstacle(int x, int y) {
    if (x >= 0 && x < width && y >= 0 && y < height) obstacles[x][y] = true;
}

public void clearObstacle(int x, int y) {
    if (x >= 0 && x < width && y >= 0 && y < height) obstacles[x][y] = false;
}

// Updated isWalkable
public boolean isWalkable(int x, int y) {
    if (x < 0 || x >= width || y < 0 || y >= height) return false;
    return grid[x][y] != Height.WALL && !obstacles[x][y];
}
```

**Obstacle footprints:** Buildings and mineral patches register their footprint as blocked cells. Each obstacle occupies a rectangular footprint based on its radius:

```java
// TerrainGrid — convenience method
public void markCircularObstacle(Point2d center, float radius) {
    int minX = Math.max(0, (int)(center.x() - radius));
    int maxX = Math.min(width - 1, (int)(center.x() + radius));
    int minY = Math.max(0, (int)(center.y() - radius));
    int maxY = Math.min(height - 1, (int)(center.y() + radius));
    for (int x = minX; x <= maxX; x++) {
        for (int y = minY; y <= maxY; y++) {
            double dx = (x + 0.5) - center.x();
            double dy = (y + 0.5) - center.y();
            if (dx * dx + dy * dy <= radius * radius) {
                markObstacle(x, y);
            }
        }
    }
}
```

**Registration points:**
- **Mineral patches:** Registered when `WorkerSimulation.onTownHallCompleted()` creates patches. Mineral patch radius = `MINERAL_PATCH_RADIUS` (1.0). Each patch blocks ~4 cells (2x2 at 1 tile/unit).
- **Town halls:** Registered on `onTownHallCompleted()`. TH radius = `TOWN_HALL_RADIUS` (2.75). Blocks ~25 cells (5x5).
- **Gas buildings:** Registered on `onGasBuildingCompleted()`. Gas radius = `GAS_BUILDING_RADIUS` (1.5). Blocks ~9 cells (3x3).
- **Other buildings:** Registered via `EmulatedGame` building lifecycle. Building radii from `SC2Data.buildingRadius()`.

**Clearing:** When a building is destroyed or a mineral patch depleted, clear its footprint cells. Use a reference-counting approach: `obstacles` becomes `int[][]` instead of `boolean[][]`, incremented on mark, decremented on clear. A cell is walkable when its count reaches 0. This handles overlapping footprints (e.g., adjacent mineral patches sharing edge cells) without requiring footprint tracking per obstacle.

**WorkerSimulation access to TerrainGrid:** `WorkerSimulation` currently has no reference to `TerrainGrid`. Add it as a constructor parameter, set by `EmulatedGame` at construction.

### 2. Worker waypoint-based movement

Workers use `PathfindingMovement` for route planning while keeping their own `moveOneLoop()` physics for movement execution.

**New field on SimulatedWorker:**

```java
Deque<Point2d> waypoints;  // null = straight-line (backward compat)
```

**Modified movement in stepOneMineralLoop / stepOneGasLoop:**

When a worker's target changes (assigned to patch, returning to TH, going to gas), compute an A* path and store waypoints:

```java
// In WorkerSimulation, when setting a new target:
private void setTargetWithPath(SimulatedWorker w, Point2d target) {
    List<Point2d> path = pathfinder.findPath(terrainGrid, w.position, target);
    if (path.isEmpty()) {
        // Direct line — no obstacles between current position and target
        w.waypoints = null;
        w.targetPosition = target;
    } else {
        List<Point2d> smoothed = AStarPathfinder.smoothPath(path, terrainGrid);
        w.waypoints = new ArrayDeque<>(smoothed);
        w.targetPosition = w.waypoints.poll();  // first waypoint
    }
}
```

**Modified moveOneLoop in stepOneMineralLoop:**

```java
// After moveOneLoop returns true (arrived at targetPosition):
if (w.waypoints != null && !w.waypoints.isEmpty()) {
    w.targetPosition = w.waypoints.poll();
    // Don't transition state — still walking, just reached a waypoint
} else {
    // Arrived at final destination — proceed with mining/depositing
    w.waypoints = null;
    // ... existing state transition logic
}
```

**Modified isAtTarget:**

```java
boolean isAtFinalTarget() {
    return isAtTarget() && (waypoints == null || waypoints.isEmpty());
}
```

**Where paths are computed (all target-setting sites in WorkerSimulation):**

| Call site | Outbound target | Return target |
|---|---|---|
| `returnToPatchOrReassign()` | `miningTarget(patch, baseIndex)` | — |
| `assignToNearestPatch()` | `miningTarget(patch, baseIndex)` | — |
| Deposit → return to patch | `miningTarget(patch, baseIndex)` | — |
| Mining complete → return | `depositTarget(w)` | — |
| Gas assigned | `gasTarget(slot)` | — |
| Gas harvest → return | `gasReturnTarget(slot)` | — |

Each of these currently sets `w.targetPosition = X` directly. Replace with `setTargetWithPath(w, X)`.

**Performance:** A* runs once per route, not per loop. With 12 mineral workers cycling every ~6 ticks, that's ~4 A* calls per tick. On a 64x64 grid with ~50 obstacle cells, this is well under 1ms total.

### 3. Remove compensation stack

Delete entirely:
- `MineralPatch.computeApproachDocking()` — the per-patch formula
- `MineralPatch.outboundPathDelay` field
- `MineralPatch.PATH_DEVIATION_SCALE` and `PATH_DEVIATION_BASE` constants
- `WorkerSimulation.effectiveApproachDocking()` method
- `MineralPatch.approachDockingLoops` field (or reduce to `APPROACH_ANIMATION_LOOPS` only)
- `WorkerPhysicsConfig` properties for path deviation scale/base (if they exist)

Keep:
- `APPROACH_ANIMATION_LOOPS` (20) — models real SC2 deceleration at the patch face. This is not compensation; it's a distinct physical phase observed in SC2 cadence data.

**Migration:** `MineralPatch.approachDockingLoops` currently combines animation + path deviation. After removing path deviation, the approach phase duration is just `APPROACH_ANIMATION_LOOPS`. All references to `effectiveApproachDocking()` are replaced with the constant.

## Gas worker paths

Gas workers follow the same pattern. `stepOneGasLoop` currently uses straight-line movement. With obstacle-aware paths:
- **Outbound (TH → geyser):** A* path around buildings near the geyser. Protoss Pylons near geysers, Supply Depots near refineries — these create real SC2 pathing.
- **Return (geyser → TH):** A* path, typically direct unless buildings block.
- `GAS_APPROACH_DOCKING_LOOPS` — analogous to mineral approach docking. If gas walks become obstacle-aware, this constant may also need recalibration or removal. Leave it initially; recalibrate after measuring gas accuracy with obstacle paths.

## Interaction with existing systems

**Collision resolution (`resolveAllMineralCollisions`):** Unchanged. Collision operates on worker positions after movement. With obstacle-aware paths, workers approach patches from different angles, which may change collision patterns. The `LOW_SATURATION_DODGE_SCALE` from #656 stays — it models a real physics effect (workers at low density don't physically jostle).

**Distance-first routing (`assignToNearestPatch`):** Unchanged. Routing selects the nearest available patch by Euclidean distance. Workers then path around obstacles to reach it. The path may be longer than straight-line distance, but selection by straight-line distance is still correct — it matches SC2's routing heuristic.

**Patch lock / reroute:** Unchanged. Workers still wait at occupied patches and reroute after `REROUTE_THRESHOLD_LOOPS`.

**MULEs:** MULEs use `stepOneMuleLoop` which also calls `moveOneLoop()`. Add waypoint support for MULEs too — they're affected by the same mineral-line obstacles.

**Real SC2 maps:** `TerrainGrid.fromPathingGrid()` already constructs from SC2's pathing data at the map's native resolution (typically 200x200). Buildings and resources register as obstacles in the same way. No special handling needed — the same infrastructure scales.

## Validation

| Test | What it measures | Before | Target |
|---|---|---|---|
| `DualRunCalibrationTest` (12w) | Mineral total at tick 304 | +3.6% (1165 vs 1125) | Measure — expect temporary regression then recalibration |
| `DualRunCalibrationTest` (all playbooks) | Self-baseline (EM vs EM) | Passing | Must pass (no internal inconsistency) |
| `SC2PlaybookCalibrationTest` | All 9 playbook assertions | Passing | Assess — economy playbooks may need checkpoint adjustment |
| `WorkerCadenceComparisonTest` | Per-worker cycle variance | σ=0.92 (EM) vs σ=0.56 (SC2) | Expect σ to change — may improve or worsen depending on path variance |
| New: `ObstaclePathTest` | Outbound path > straight-line, return ≈ straight-line | — | Outbound paths are measurably longer than Euclidean distance |
| New: `TerrainGridObstacleTest` | markObstacle/clearObstacle, isWalkable with obstacles | — | Obstacle cells block A* |
| `PathfindingMovementTest` | Existing army pathfinding | Passing | No regression |
| `EmulatedGameBenchmarkTest` | Physics tick throughput | Passing | No significant performance regression |

**Expected accuracy trajectory:** Removing the compensation stack will initially worsen mineral accuracy because obstacle paths won't produce the same deviation as the tuned constants. This is expected and correct — the constants were calibrated to straight-line walks. After obstacle paths land, recalibrate `APPROACH_ANIMATION_LOOPS` and validate against dual-run baselines.

## Files changed summary

| File | Change |
|------|--------|
| `TerrainGrid.java` | Add `obstacles[][]`, `markObstacle()`, `clearObstacle()`, `markCircularObstacle()`, update `isWalkable()` |
| `SimulatedWorker.java` | Add `waypoints` field, `isAtFinalTarget()` method |
| `WorkerSimulation.java` | Add `terrainGrid` + `pathfinder` fields, `setTargetWithPath()` method, update all target-setting sites, remove `effectiveApproachDocking()` |
| `MineralPatch.java` | Remove `computeApproachDocking()`, `outboundPathDelay`, `PATH_DEVIATION_SCALE`, `PATH_DEVIATION_BASE`, simplify `approachDockingLoops` to animation-only |
| `EmulatedGame.java` | Pass `terrainGrid` to `WorkerSimulation`, register building/resource obstacles on lifecycle events |
| `WorkerPhysicsConfig.java` | Remove path deviation config properties (if present) |
| `WorkerPhysicsConfigProducer.java` | Remove corresponding CDI properties |
| `docs/sc2-physics-constants.md` | Update §3 (approach docking → obstacle paths), §15 (path deviation now physical) |
| New: `TerrainGridObstacleTest.java` | Unit tests for obstacle layer |
| New: `ObstaclePathTest.java` | Validates outbound path > straight-line, return ≈ straight-line |

## Out of scope

- Army pathfinding around buildings (#731)
- Army pathfinding around mineral patches and geysers (#732)
- Movement system unification (workers + army into single system) — follow-up
- Grid resolution increase (128x128) — only if measured path accuracy is insufficient
- Gas approach docking recalibration — measure first, recalibrate in follow-up if needed

## References

- WorkerSimulation.java:414-508 — `stepOneMineralLoop()` (current straight-line movement loop)
- WorkerSimulation.java:95-110 — `effectiveApproachDocking()` (density-dependent compensation)
- MineralPatch.java:63-88 — `computeApproachDocking()` (per-patch formula to remove)
- SimulatedWorker.java:148-186 — `moveOneLoop()` (acceleration/deceleration physics)
- TerrainGrid.java — current immutable terrain grid (domain package)
- AStarPathfinder.java — 8-directional A* with path smoothing
- PathfindingMovement.java — army waypoint movement wrapper
- EmulatedGame.java:264-297 — `enforceWall()` (building collision for army)
- docs/sc2-physics-constants.md — §3 mining constants, §15 worker pathing gap
- #690 — SC2 cadence data (outbound +6%, return -35% vs straight-line)
- #656 spec — mineral-line cycle distribution design (the compensation this replaces)
- #620 spec — distance-first worker routing (already landed)
- Protocol: worker-physics-cdi-defaults-must-match-constants
- Protocol: sc2data-spatial-constants-require-calibration
- Protocol: calibration-milestones-are-waypoints
- Memory: tick-alignment-priority — compensation constants forbidden
- Memory: feedback-simulation-over-formulas — simulate per-loop, never formulas
- Memory: forage-tick-quantization-masks-physics — sub-tick chaining pattern
