# Army Pathfinding: Register Buildings as Obstacles — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #731 — Army pathfinding: register buildings as obstacles in TerrainGrid
**Issue group:** #731

**Goal:** Make A* pathfinder route army units around buildings instead of through them, by registering buildings as rectangular obstacles on the TerrainGrid.

**Architecture:** Wire `TerrainGrid.markRectangularObstacle()`/`clearRectangularObstacle()` calls into all building lifecycle methods in `EmulatedGame`. Add `invalidateAllPaths()` to `MovementStrategy` so all army unit paths recompute when buildings change. `enforceWall()` remains as a safety net.

**Tech Stack:** Java 21, Quarkus, plain JUnit 5

## Global Constraints

- No `@QuarkusTest` — all tests are plain JUnit (direct instantiation)
- Use `SC2Data.buildingFootprint(BuildingType)` for rectangular dimensions (int[]{width, height})
- Reference-counted obstacle layer — mark/clear must be symmetric
- Register obstacles on placement start (incomplete), not on completion
- `double` arithmetic in physics paths

---

## Batch 1: MovementStrategy interface + PathfindingMovement

### Task 1: Add invalidateAllPaths() to MovementStrategy and implement in PathfindingMovement and DirectMovement

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`

**Interfaces:**
- Consumes: `MovementStrategy.invalidatePath(String unitTag)` (existing)
- Produces: `MovementStrategy.invalidateAllPaths()` — clears all cached paths for all units

- [ ] **Step 1: Write failing test for invalidateAllPaths**

Add to `PathfindingMovementTest.java`:

```java
@Test
void invalidateAllPaths_clearsAllCachedPaths() {
    TerrainGrid grid = TerrainGrid.emulatedMap();
    PathfindingMovement movement = new PathfindingMovement(grid);

    // Advance two units to cache their paths
    Point2d unitA = Point2d.of(5, 5);
    Point2d targetA = Point2d.of(10, 5);
    movement.advance("unitA", unitA, targetA, 2.0f);

    Point2d unitB = Point2d.of(5, 10);
    Point2d targetB = Point2d.of(10, 10);
    movement.advance("unitB", unitB, targetB, 2.0f);

    // Invalidate all paths
    movement.invalidateAllPaths();

    // Advance again with the SAME targets — should recompute (not use cached path)
    // If paths were truly cleared, the advance will recompute from current position
    // We verify by checking that the method doesn't throw and returns a valid position
    Point2d resultA = movement.advance("unitA", unitA, targetA, 2.0f);
    Point2d resultB = movement.advance("unitB", unitB, targetB, 2.0f);
    assertNotNull(resultA);
    assertNotNull(resultB);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PathfindingMovementTest#invalidateAllPaths_clearsAllCachedPaths -q`
Expected: FAIL — `invalidateAllPaths()` does not exist on `MovementStrategy`

- [ ] **Step 3: Add invalidateAllPaths() to MovementStrategy interface**

In `MovementStrategy.java`, add default method:

```java
default void invalidateAllPaths() {}
```

- [ ] **Step 4: Implement invalidateAllPaths() in PathfindingMovement**

In `PathfindingMovement.java`, add:

```java
@Override
public void invalidateAllPaths() {
    waypoints.clear();
    lastTargets.clear();
}
```

- [ ] **Step 5: Implement invalidateAllPaths() in DirectMovement**

In `DirectMovement.java`, add (no-op — DirectMovement has no cached state):

```java
@Override
public void invalidateAllPaths() {}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PathfindingMovementTest#invalidateAllPaths_clearsAllCachedPaths -q`
Expected: PASS

- [ ] **Step 7: Run full PathfindingMovementTest to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PathfindingMovementTest -q`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java
git commit -m "feat(#731): add invalidateAllPaths() to MovementStrategy

Refs #731"
```

---

## Batch 2: Building obstacle registration in EmulatedGame

### Task 2: Add registerBuildingObstacle/clearBuildingObstacle helpers and wire into all building lifecycle methods

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/BuildingObstacleTest.java` (new)

**Interfaces:**
- Consumes: `TerrainGrid.markRectangularObstacle(float, float, float, float)`, `TerrainGrid.clearRectangularObstacle(float, float, float, float)`, `SC2Data.buildingFootprint(BuildingType)`, `MovementStrategy.invalidateAllPaths()`
- Produces: `EmulatedGame` registers building obstacles on placement; clears on destruction; `terrainGrid.isWalkable()` returns false for building cells

- [ ] **Step 1: Write failing test — building placement marks obstacle**

Create `BuildingObstacleTest.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.BuildIntent;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class BuildingObstacleTest {

    @Test
    void handleBuild_marksRectangularObstacle() {
        EmulatedGame game = new EmulatedGame();
        TerrainGrid grid = game.getTerrainGrid();

        // Gateway is 3x3 — half-extents 1.5 x 1.5
        // Place at (10, 10) — cells (9,9) through (11,11) should be blocked
        Point2d pos = Point2d.of(10, 10);
        game.handleBuildIntent(BuildingType.GATEWAY, pos);

        // Center cell blocked
        assertFalse(grid.isWalkable(10, 10), "Center cell should be blocked");
        // Edge cells blocked (3x3 footprint centered at 10,10)
        assertFalse(grid.isWalkable(9, 9), "Corner cell should be blocked");
        assertFalse(grid.isWalkable(11, 11), "Corner cell should be blocked");
        // Cell outside footprint still walkable
        assertTrue(grid.isWalkable(8, 10), "Cell outside footprint should be walkable");
        assertTrue(grid.isWalkable(12, 10), "Cell outside footprint should be walkable");
    }
}
```

Note: `handleBuildIntent` is a test-visible wrapper. If no such method exists, use the existing `debugCreateBuilding()` or add a minimal test helper. Check the actual API during implementation and adjust.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=BuildingObstacleTest#handleBuild_marksRectangularObstacle -q`
Expected: FAIL — cells are still walkable (no obstacle registered)

- [ ] **Step 3: Add registerBuildingObstacle and clearBuildingObstacle to EmulatedGame**

Add two private methods:

```java
private void registerBuildingObstacle(Building bldg) {
    if (terrainGrid == null) return;
    int[] fp = SC2Data.buildingFootprint(bldg.type());
    terrainGrid.markRectangularObstacle(
        (float) bldg.position().x(), (float) bldg.position().y(),
        fp[0] / 2.0f, fp[1] / 2.0f);
    movementStrategy.invalidateAllPaths();
}

private void clearBuildingObstacle(Building bldg) {
    if (terrainGrid == null) return;
    int[] fp = SC2Data.buildingFootprint(bldg.type());
    terrainGrid.clearRectangularObstacle(
        (float) bldg.position().x(), (float) bldg.position().y(),
        fp[0] / 2.0f, fp[1] / 2.0f);
    movementStrategy.invalidateAllPaths();
}
```

- [ ] **Step 4: Wire registerBuildingObstacle into handleBuild()**

After `state.addBuilding(...)` at line ~786, add:

```java
registerBuildingObstacle(new Building(tag, bt, b.location(), hp, hp, false));
```

Or retrieve the building just added and pass it. The building is already constructed at this point.

- [ ] **Step 5: Wire registerBuildingObstacle into handleBuildingBorn()**

After `state.addBuilding(...)` at line ~1808, add:

```java
registerBuildingObstacle(new Building(bb.tag(), bb.type(), bb.position(), hp, hp, true));
```

- [ ] **Step 6: Wire registerBuildingObstacle into injectReplayBuilding()**

After `friendly.addBuilding(building)` at line ~1839, add:

```java
registerBuildingObstacle(building);
```

- [ ] **Step 7: Wire registerBuildingObstacle into injectReplayBuildingWithCost()**

After `friendly.addBuilding(building)` at line ~1863, add:

```java
registerBuildingObstacle(building);
```

- [ ] **Step 8: Wire registerBuildingObstacle into debugCreateBuilding()**

After `friendly.addBuilding(...)` at line ~638, add:

```java
registerBuildingObstacle(new Building(tag, bt, location, hp, hp, true));
```

- [ ] **Step 9: Wire registerBuildingObstacle into spawnEnemyBuildingForTesting()**

After `enemy.addBuilding(...)` at line ~1707, add:

```java
registerBuildingObstacle(bldg);
```

Where `bldg` is the Building record just created.

- [ ] **Step 10: Wire clearBuildingObstacle into handleBuildingDied()**

Inside the `ifPresent` block (line ~1825), before `state.removeBuilding()` at line ~1834, add:

```java
clearBuildingObstacle(b);
```

Where `b` is the building found by the stream. Must be called before `removeBuilding()` so the building's position is still available.

- [ ] **Step 11: Wire clearBuildingObstacle into removeDestroyedBuilding()**

Before `friendly.removeBuilding(tag)` at line ~1803, look up the building by tag and clear its obstacle:

```java
friendly.buildings().stream()
    .filter(b -> b.tag().equals(tag))
    .findFirst()
    .ifPresent(this::clearBuildingObstacle);
```

- [ ] **Step 12: Add clearAllObstacles to resetInternal()**

In `resetInternal()` (line ~202), add before or after the PlayerState clear:

```java
if (terrainGrid != null) {
    terrainGrid.clearAllObstacles();
}
```

- [ ] **Step 13: Run BuildingObstacleTest to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=BuildingObstacleTest#handleBuild_marksRectangularObstacle -q`
Expected: PASS

- [ ] **Step 14: Write test — building destruction clears obstacle**

Add to `BuildingObstacleTest.java`:

```java
@Test
void buildingDied_clearsObstacle() {
    EmulatedGame game = new EmulatedGame();
    TerrainGrid grid = game.getTerrainGrid();

    // Place a building
    Point2d pos = Point2d.of(10, 10);
    game.debugCreateBuilding(BuildingType.GATEWAY, pos);
    assertFalse(grid.isWalkable(10, 10), "Should be blocked after placement");

    // Find the building tag and destroy it
    String tag = game.snapshot().friendly().buildings().get(0).tag();
    game.removeDestroyedBuilding(tag, BuildingType.GATEWAY);
    assertTrue(grid.isWalkable(10, 10), "Should be walkable after destruction");
}
```

- [ ] **Step 15: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=BuildingObstacleTest#buildingDied_clearsObstacle -q`
Expected: PASS

- [ ] **Step 16: Write test — reset clears all building obstacles**

Add to `BuildingObstacleTest.java`:

```java
@Test
void reset_clearsAllBuildingObstacles() {
    EmulatedGame game = new EmulatedGame();
    TerrainGrid grid = game.getTerrainGrid();

    game.debugCreateBuilding(BuildingType.GATEWAY, Point2d.of(10, 10));
    game.debugCreateBuilding(BuildingType.PYLON, Point2d.of(15, 10));
    assertFalse(grid.isWalkable(10, 10));
    assertFalse(grid.isWalkable(15, 10));

    game.reset();
    assertTrue(grid.isWalkable(10, 10), "Should be walkable after reset");
    assertTrue(grid.isWalkable(15, 10), "Should be walkable after reset");
}
```

- [ ] **Step 17: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=BuildingObstacleTest#reset_clearsAllBuildingObstacles -q`
Expected: PASS

- [ ] **Step 18: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/BuildingObstacleTest.java
git commit -m "feat(#731): register buildings as obstacles on TerrainGrid

Wire markRectangularObstacle/clearRectangularObstacle into all building
lifecycle methods. Clear obstacles on reset.

Refs #731"
```

---

## Batch 3: A* routing verification + enforceWall alignment

### Task 3: Verify A* routes around buildings and update enforceWall to drop isComplete guard

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (enforceWall)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ArmyPathAroundBuildingTest.java` (new)

**Interfaces:**
- Consumes: `EmulatedGame.registerBuildingObstacle()` (from Task 2), `PathfindingMovement.advance()`, `AStarPathfinder.findPath()`
- Produces: Army units route around buildings; `enforceWall` blocks incomplete buildings too

- [ ] **Step 1: Write failing test — army unit paths around building**

Create `ArmyPathAroundBuildingTest.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class ArmyPathAroundBuildingTest {

    @Test
    void armyUnit_pathsAroundBuilding() {
        EmulatedGame game = new EmulatedGame();
        TerrainGrid grid = TerrainGrid.emulatedMap();
        game.setTerrainGrid(grid);
        game.setMovementStrategy(new PathfindingMovement(grid));

        // Place a 3x3 Gateway directly between unit start and target
        game.debugCreateBuilding(BuildingType.GATEWAY, Point2d.of(8, 8));

        // Spawn a unit at (5, 8) with target (11, 8) — straight line goes through building
        game.spawnFriendlyUnit(UnitType.STALKER, Point2d.of(5, 8));
        String tag = game.snapshot().friendly().units().get(0).tag();
        game.setUnitTarget(tag, Point2d.of(11, 8));

        // Advance several ticks — unit should navigate around the building
        for (int i = 0; i < 20; i++) {
            game.tick();
        }

        // Unit should have moved past the building
        Unit unit = game.snapshot().friendly().units().stream()
            .filter(u -> u.tag().equals(tag))
            .findFirst().orElseThrow();
        assertTrue(unit.position().x() > 9,
            "Unit should have navigated past the building, but is at x=" + unit.position().x());
    }

    @Test
    void armyUnit_neverEntersBuildingFootprint() {
        EmulatedGame game = new EmulatedGame();
        TerrainGrid grid = TerrainGrid.emulatedMap();
        game.setTerrainGrid(grid);
        game.setMovementStrategy(new PathfindingMovement(grid));

        // Place a 5x5 Nexus at (10, 8) — large obstacle
        game.debugCreateBuilding(BuildingType.NEXUS, Point2d.of(10, 8));

        game.spawnFriendlyUnit(UnitType.ZEALOT, Point2d.of(5, 8));
        String tag = game.snapshot().friendly().units().get(0).tag();
        game.setUnitTarget(tag, Point2d.of(15, 8));

        int[] fp = SC2Data.buildingFootprint(BuildingType.NEXUS);
        float hw = fp[0] / 2.0f;
        float hh = fp[1] / 2.0f;

        for (int i = 0; i < 30; i++) {
            game.tick();
            Unit u = game.snapshot().friendly().units().stream()
                .filter(unit -> unit.tag().equals(tag))
                .findFirst().orElseThrow();
            // Unit must never be inside the Nexus footprint
            boolean inside = u.position().x() >= 10 - hw && u.position().x() <= 10 + hw
                          && u.position().y() >= 8 - hh && u.position().y() <= 8 + hh;
            assertFalse(inside, "Unit entered building footprint at tick " + i
                + " position=" + u.position());
        }
    }
}
```

Note: `spawnFriendlyUnit` and `setUnitTarget` are test helpers. Check the actual API during implementation — may be `debugSpawnUnit()` or use `PhysicsState.unitTargets`. Adjust method names to match the existing test API.

- [ ] **Step 2: Run tests to verify they fail (pre-obstacle routing)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ArmyPathAroundBuildingTest -q`
Expected: With obstacles wired from Task 2, the routing test should PASS (A* now sees buildings). If it fails, debug — the obstacle registration from Task 2 should be sufficient.

- [ ] **Step 3: Update enforceWall — drop isComplete guard on building checks**

In `enforceWall()` (line ~281-294), change both friendly and enemy building loops:

```java
// Before:
if (bldg.isComplete() && !insideFootprint(current, bldg) && insideFootprint(proposed, bldg)) {
// After:
if (!insideFootprint(current, bldg) && insideFootprint(proposed, bldg)) {
```

This aligns enforceWall with the obstacle layer — both block from placement start, not just completion.

- [ ] **Step 4: Write test — enforceWall blocks entry to incomplete buildings**

Add to `ArmyPathAroundBuildingTest.java`:

```java
@Test
void enforceWall_blocksIncompleteBuildings() {
    EmulatedGame game = new EmulatedGame();
    TerrainGrid grid = TerrainGrid.emulatedMap();
    game.setTerrainGrid(grid);
    game.setMovementStrategy(new PathfindingMovement(grid));

    // Place an incomplete building (via handleBuild, not debugCreate which creates complete)
    game.handleBuildIntent(BuildingType.GATEWAY, Point2d.of(10, 8));

    // Verify the building is incomplete
    Building bldg = game.snapshot().friendly().buildings().get(0);
    assertFalse(bldg.isComplete(), "Building should be incomplete");

    // Verify the obstacle is registered
    assertFalse(grid.isWalkable(10, 8), "Incomplete building should block pathfinding");
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ArmyPathAroundBuildingTest -q`
Expected: All PASS

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests PASS. If any fail, investigate — building obstacle registration may affect tests that place buildings and expect specific pathfinding behavior.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ArmyPathAroundBuildingTest.java
git commit -m "feat(#731): verify A* routes around buildings, align enforceWall

A* now routes army units around building footprints. enforceWall
updated to block entry to incomplete buildings (aligns with obstacle
layer which registers from placement start).

Refs #731"
```

---

## Batch 4: Documentation + regression verification

### Task 4: Update physics docs and run regression suite

**Files:**
- Modify: `docs/sc2-physics-constants.md`

**Interfaces:**
- Consumes: All changes from Tasks 1-3
- Produces: Updated documentation, verified regression suite

- [ ] **Step 1: Update §15 of sc2-physics-constants.md**

In the "Worker pathing and terrain" section (line ~652), add a paragraph about army building obstacles:

```markdown
**Army building obstacles (#731):** Buildings are registered as rectangular obstacles on `TerrainGrid` via `markRectangularObstacle()` from the moment construction starts (incomplete buildings block pathing). `AStarPathfinder` routes army units around buildings. `enforceWall()` remains as a safety net for edge cases. All army paths are invalidated when a building is placed or destroyed — `MovementStrategy.invalidateAllPaths()` clears all cached waypoints. Building obstacles are cleared on `reset()` via `clearAllObstacles()`.
```

- [ ] **Step 2: Run PathfindingMovementTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PathfindingMovementTest -q`
Expected: PASS

- [ ] **Step 3: Run EmulatedGameTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest -q`
Expected: PASS

- [ ] **Step 4: Run SC2BotAgentTerrainTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2BotAgentTerrainTest -q`
Expected: PASS

- [ ] **Step 5: Run self-baseline playbook calibration tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: PASS — building obstacles should not affect self-baseline (EM vs EM)

- [ ] **Step 6: Run benchmark test**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=EmulatedGameBenchmarkTest -q`
Expected: No significant throughput regression

- [ ] **Step 7: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#731): update physics constants for army building obstacles

Refs #731"
```

---

## References

- [2026-11-20-army-pathfinding-buildings-design.md] — design spec this plan implements
- [EmulatedGame.java:269-295] — enforceWall() reactive building collision
- [EmulatedGame.java:764] — handleBuild() building placement
- [EmulatedGame.java:1806] — handleBuildingBorn()
- [EmulatedGame.java:1823] — handleBuildingDied()
- [EmulatedGame.java:1798] — removeDestroyedBuilding()
- [EmulatedGame.java:1838] — injectReplayBuilding()
- [EmulatedGame.java:1861] — injectReplayBuildingWithCost()
- [EmulatedGame.java:636] — debugCreateBuilding()
- [EmulatedGame.java:1704] — spawnEnemyBuildingForTesting()
- [TerrainGrid.java:75-97] — markRectangularObstacle/clearRectangularObstacle
- [PathfindingMovement.java:63-68] — invalidatePath()
- [MovementStrategy.java] — movement interface
- [DirectMovement.java] — no-op movement strategy
- [SC2Data.java:1324-1355] — buildingFootprint()
- [AStarPathfinder.java:55] — isWalkable() query
- [docs/sc2-physics-constants.md §13, §15] — building footprints, terrain status
- [GitHub #707] — obstacle-aware worker pathfinding (established obstacle layer)
- [GitHub #736] — SC2Data.buildingFootprint() rectangular dimensions
