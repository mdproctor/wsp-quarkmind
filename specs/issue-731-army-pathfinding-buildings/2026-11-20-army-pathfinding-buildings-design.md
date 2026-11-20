# Army Pathfinding: Register Buildings as Obstacles — Design Spec

**Issue:** #731
**Branch:** issue-731-army-pathfinding-buildings
**Date:** 2026-11-20
**Scope:** Building obstacle registration for army A* pathfinding. Worker pathfinding is unaffected (uses its own obstacle layer via WorkerSimulation).

## Problem

A* pathfinder (`AStarPathfinder`) only knows about terrain WALL tiles and the obstacle layer on `TerrainGrid`. Buildings are not registered as obstacles. Army units plan paths straight through building footprints, hit `enforceWall()` rejection on each tick, and get stuck in compute-reject-recompute loops.

The infrastructure to fix this already exists (#707):
- `TerrainGrid.markRectangularObstacle(centerX, centerY, halfW, halfH)` with reference-counted `int[][]`
- `AStarPathfinder.findPath()` queries `grid.isWalkable()` which checks both terrain and obstacles
- `PathfindingMovement.invalidatePath(unitTag)` clears cached waypoints and forces recomputation

The gap: no building lifecycle code calls these methods.

## Design

### 1. Register buildings as rectangular obstacles on placement

Add a private helper to `EmulatedGame`:

```java
private void registerBuildingObstacle(Building bldg) {
    if (terrainGrid == null) return;
    int[] fp = SC2Data.buildingFootprint(bldg.type());
    terrainGrid.markRectangularObstacle(
        (float) bldg.position().x(), (float) bldg.position().y(),
        fp[0] / 2.0f, fp[1] / 2.0f);
    invalidateAllArmyPaths();
}

private void clearBuildingObstacle(Building bldg) {
    if (terrainGrid == null) return;
    int[] fp = SC2Data.buildingFootprint(bldg.type());
    terrainGrid.clearRectangularObstacle(
        (float) bldg.position().x(), (float) bldg.position().y(),
        fp[0] / 2.0f, fp[1] / 2.0f);
    invalidateAllArmyPaths();
}
```

### 2. Wire into building lifecycle

Call `registerBuildingObstacle` at every point a building enters the game state, and `clearBuildingObstacle` at every point a building leaves:

| Method | Action | When |
|--------|--------|------|
| `handleBuild()` | `registerBuildingObstacle` | After adding incomplete building to PlayerState |
| `handleBuildingBorn()` | `registerBuildingObstacle` | After adding complete building to PlayerState |
| `injectReplayBuilding()` | `registerBuildingObstacle` | After adding building |
| `injectReplayBuildingWithCost()` | `registerBuildingObstacle` | After adding building |
| `debugCreateBuilding()` | `registerBuildingObstacle` | After adding building |
| `spawnEnemyBuildingForTesting()` | `registerBuildingObstacle` | After adding enemy building (bypasses handleBuildingBorn) |
| `handleBuildingDied()` | `clearBuildingObstacle` | Look up building by tag from PlayerState before `removeBuilding()` — `BuildingDiedIntent` has type but no position |
| `removeDestroyedBuilding()` | `clearBuildingObstacle` | Look up building by tag from PlayerState before `removeBuilding()` |

Buildings register on placement start (incomplete), not on completion. SC2 blocks pathing from the moment construction begins.

### 3. Invalidate all army paths on building change

Add `invalidateAllPaths()` to the `MovementStrategy` interface:

```java
public interface MovementStrategy {
    Point2d advance(String unitTag, Point2d current, Point2d target, float speed);
    void clearUnit(String unitTag);
    void invalidatePath(String unitTag);
    void invalidateAllPaths();  // new
    void reset();
}
```

`PathfindingMovement.invalidateAllPaths()` clears all entries in `waypoints` and `lastTargets` maps. `DirectMovement.invalidateAllPaths()` is a no-op.

`EmulatedGame.invalidateAllArmyPaths()`:

```java
private void invalidateAllArmyPaths() {
    movementStrategy.invalidateAllPaths();
}
```

Called by `registerBuildingObstacle` and `clearBuildingObstacle`. On the next tick, `PathfindingMovement.advance()` recomputes A* paths for all units with targets — these paths will now route around the new/removed obstacle.

### 4. Update enforceWall for incomplete buildings

Current `enforceWall()` only checks `bldg.isComplete()`. Since we now register obstacles on placement (incomplete), the terrain `isWalkable` check on line 1 of `enforceWall` already blocks entry via the obstacle layer. The `isComplete()` guard on the building iteration becomes a secondary safety net.

Update the building iteration to also block entry for incomplete buildings:

```java
// Before:
if (bldg.isComplete() && !insideFootprint(current, bldg) && insideFootprint(proposed, bldg)) {
// After:
if (!insideFootprint(current, bldg) && insideFootprint(proposed, bldg)) {
```

This aligns `enforceWall` with the obstacle layer — both block from placement start.

### 5. Handle reset/clear lifecycle

`EmulatedGame.reset()` (line 166) does NOT currently call `clearAllObstacles()`. Add `terrainGrid.clearAllObstacles()` to `resetInternal()` (before building lists are cleared). After reset, the building lists are empty and no obstacle state needs clearing.

Workers re-register mineral patch obstacles via `WorkerSimulation.onTownHallCompleted()` during scenario setup after reset — this happens naturally when the scenario creates town halls. Building obstacles will also be re-registered naturally as the scenario's `handleBuild()` / `debugCreateBuilding()` calls fire.

### 6. Enemy buildings

`handleBuildingBorn` and `handleBuildingDied` are parameterised by `PlayerState` — they handle both friendly and enemy buildings. Wiring `registerBuildingObstacle`/`clearBuildingObstacle` inside these methods automatically covers enemy buildings.

The only gap is `spawnEnemyBuildingForTesting()` (line 1704) which calls `enemy.addBuilding()` directly — this needs a `registerBuildingObstacle` call too (see table above).

## Interaction with existing systems

**Worker pathfinding (WorkerSimulation):** Unaffected. Workers use their own obstacle registrations (mineral patches at radius 0.3) via `WorkerSimulation`. Building obstacles on the shared `TerrainGrid` will also affect worker A* paths — this is correct. Workers should route around buildings too. The reference-counted obstacle layer handles overlapping registrations from both worker and army systems.

**enforceWall safety net:** Remains as a secondary check. With buildings registered as obstacles, A* will not plan paths through buildings, so enforceWall building collision should trigger rarely (only for edge cases like units spawned inside a building footprint).

**Collision model:** Unchanged. Army unit-to-unit collision is not affected.

**EmulatedSC2Server:** Uses `EmulatedEngine` which wires `PathfindingMovement`. Building obstacles are registered via the same `EmulatedGame` lifecycle — no additional wiring needed.

**Replay validation (ReplayValidationHarness):** Uses `injectReplayBuilding()` and `removeDestroyedBuilding()`. Both are wired (see table above). Replayed army pathfinding will now route around buildings.

## Validation

| Test | What it measures | Expected |
|------|------------------|----------|
| New: `BuildingObstacleTest` | markRectangularObstacle called on building placement, cleared on destruction; isWalkable returns false for building tiles | Pass |
| New: `ArmyPathAroundBuildingTest` | A* routes around a building placed between unit and target | Path avoids building footprint |
| `PathfindingMovementTest` | Existing army pathfinding (terrain walls) | No regression |
| `EmulatedGameTest` | Full game lifecycle | No regression |
| `SC2BotAgentTerrainTest` | Terrain-aware army movement | No regression |
| `EmulatedGameBenchmarkTest` | Tick throughput | No significant regression |
| Self-baseline playbook tests | Economy accuracy | No regression — building obstacles don't affect worker economy paths (mineral patches already registered) |

**New test: BuildingObstacleTest (plain JUnit):**
1. Create EmulatedGame with TerrainGrid
2. Place a building via handleBuild()
3. Assert terrainGrid.isWalkable() returns false for cells within the building footprint
4. Destroy the building
5. Assert cells are walkable again

**New test: ArmyPathAroundBuildingTest (plain JUnit):**
1. Create EmulatedGame with PathfindingMovement
2. Place a 3x3 building between a unit and its target
3. Set unit target, advance multiple ticks
4. Assert the unit reaches its target (path routes around building)
5. Assert the path waypoints do not enter the building footprint

## Files changed summary

| File | Change |
|------|--------|
| `EmulatedGame.java` | Add `registerBuildingObstacle()`, `clearBuildingObstacle()`, `invalidateAllArmyPaths()`; wire into all building lifecycle methods; update `enforceWall` to drop `isComplete()` guard |
| `MovementStrategy.java` | Add `invalidateAllPaths()` method |
| `PathfindingMovement.java` | Implement `invalidateAllPaths()` — clear all waypoints and lastTargets |
| `DirectMovement.java` | Implement `invalidateAllPaths()` — no-op |
| New: `BuildingObstacleTest.java` | Unit test: obstacle registration/clearing on building lifecycle |
| New: `ArmyPathAroundBuildingTest.java` | Unit test: A* routes around buildings |
| `docs/sc2-physics-constants.md` | Update §15 to note buildings are now registered as obstacles for army pathfinding |

## Out of scope

- Army pathfinding around mineral patches and geysers (#732)
- Worker pathfinding changes (already handled by #707)
- Performance optimisation of repath (intersection-filtered invalidation) — not needed at current unit counts

## References

- #707 spec — obstacle-aware worker pathfinding design (established the obstacle layer infrastructure)
- #736 — SC2Data.buildingFootprint() rectangular tile dimensions
- #733 — enforceWall conversion from circular to rectangular
- EmulatedGame.java:269-295 — enforceWall() (current reactive building collision)
- EmulatedGame.java:764 — handleBuild() (building placement lifecycle)
- EmulatedGame.java:1806 — handleBuildingBorn() (building born lifecycle)
- EmulatedGame.java:1823 — handleBuildingDied() (building destruction lifecycle)
- TerrainGrid.java:75-97 — markRectangularObstacle/clearRectangularObstacle
- PathfindingMovement.java:63-68 — invalidatePath() (per-unit path invalidation)
- AStarPathfinder.java:55 — isWalkable() query in A* neighbor expansion
- SC2Data.java:1324-1355 — buildingFootprint() rectangular lookup
- docs/sc2-physics-constants.md §13 — building footprint dimensions
- docs/sc2-physics-constants.md §15 — worker pathing and terrain status
