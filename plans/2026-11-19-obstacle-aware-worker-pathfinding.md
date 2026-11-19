# Obstacle-Aware Worker Pathfinding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #707 — EM lacks obstacle-aware pathfinding for buildings and resources
**Issue group:** #707

**Goal:** Replace straight-line worker movement with A*-pathfound obstacle-aware movement, then remove the entire approach docking compensation stack so that path deviation emerges from physics.

**Architecture:** Add a dynamic obstacle layer to `TerrainGrid` (reference-counted `int[][]`). Register buildings, mineral patches, town halls, and gas buildings as obstacle footprints at lifecycle events. Workers compute A* waypoint paths once per route via `setTargetWithPath()`, then follow waypoints using existing `moveOneLoop()` acceleration physics. Remove `computeApproachDocking()`, `effectiveApproachDocking()`, `PATH_DEVIATION_*`, and `outboundPathDelay`. Keep `APPROACH_ANIMATION_LOOPS` (20).

**Tech Stack:** Java 21, plain JUnit (no `@QuarkusTest`), existing `AStarPathfinder`, `TerrainGrid` (domain package)

## Global Constraints

- Compensation constants are forbidden — path deviation must emerge from physics
- Simulate per-loop (22 loops/tick), never formulas or analytical shortcuts
- `TerrainGrid` is in `io.quarkmind.domain` — plain Java, no framework deps
- Protocol: `worker-physics-cdi-defaults-must-match-constants` — CDI defaults must match `DEFAULT_*` constants
- Protocol: `calibration-milestones-are-waypoints` — reach, assess, replan after measuring
- All tests are plain JUnit unless CDI is strictly required

---

## Batch 1: Dynamic obstacle layer on TerrainGrid

### Task 1: Add obstacle support to TerrainGrid

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/TerrainGrid.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/domain/TerrainGridObstacleTest.java`

**Interfaces:**
- Produces: `TerrainGrid.markObstacle(int x, int y)`, `TerrainGrid.clearObstacle(int x, int y)`, `TerrainGrid.markCircularObstacle(float centerX, float centerY, float radius)`, `TerrainGrid.clearCircularObstacle(float centerX, float centerY, float radius)`, `TerrainGrid.clearAllObstacles()`

- [ ] **Step 1: Write failing tests for obstacle marking**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TerrainGridObstacleTest {

    private TerrainGrid grid16() {
        TerrainGrid.Height[][] g = new TerrainGrid.Height[16][16];
        for (var col : g) java.util.Arrays.fill(col, TerrainGrid.Height.LOW);
        return new TerrainGrid(16, 16, g);
    }

    @Test
    void markObstacleMakesCellUnwalkable() {
        var grid = grid16();
        assertTrue(grid.isWalkable(5, 5));
        grid.markObstacle(5, 5);
        assertFalse(grid.isWalkable(5, 5));
    }

    @Test
    void clearObstacleRestoresWalkability() {
        var grid = grid16();
        grid.markObstacle(5, 5);
        grid.clearObstacle(5, 5);
        assertTrue(grid.isWalkable(5, 5));
    }

    @Test
    void overlappingObstaclesRequireBothClears() {
        var grid = grid16();
        grid.markObstacle(5, 5);
        grid.markObstacle(5, 5);
        grid.clearObstacle(5, 5);
        assertFalse(grid.isWalkable(5, 5), "one mark still active");
        grid.clearObstacle(5, 5);
        assertTrue(grid.isWalkable(5, 5), "both cleared");
    }

    @Test
    void wallCellsRemainUnwalkableAfterObstacleClear() {
        TerrainGrid.Height[][] g = new TerrainGrid.Height[16][16];
        for (var col : g) java.util.Arrays.fill(col, TerrainGrid.Height.LOW);
        g[5][5] = TerrainGrid.Height.WALL;
        var grid = new TerrainGrid(16, 16, g);
        assertFalse(grid.isWalkable(5, 5));
        grid.markObstacle(5, 5);
        grid.clearObstacle(5, 5);
        assertFalse(grid.isWalkable(5, 5), "wall is still a wall");
    }

    @Test
    void outOfBoundsMarkIsIgnored() {
        var grid = grid16();
        assertDoesNotThrow(() -> grid.markObstacle(-1, 5));
        assertDoesNotThrow(() -> grid.markObstacle(99, 5));
        assertDoesNotThrow(() -> grid.clearObstacle(-1, 5));
    }

    @Test
    void clearAllObstaclesResetsEverything() {
        var grid = grid16();
        grid.markObstacle(3, 3);
        grid.markObstacle(7, 7);
        grid.clearAllObstacles();
        assertTrue(grid.isWalkable(3, 3));
        assertTrue(grid.isWalkable(7, 7));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerrainGridObstacleTest -q`
Expected: compilation failures — `markObstacle`, `clearObstacle`, `clearAllObstacles` don't exist yet

- [ ] **Step 3: Implement obstacle layer on TerrainGrid**

Add to `TerrainGrid.java`:

```java
private final int[][] obstacles;  // [x][y], reference-counted

// In constructor — after grid copy:
this.obstacles = new int[width][height];

// New methods:
public void markObstacle(int x, int y) {
    if (x >= 0 && x < width && y >= 0 && y < height) obstacles[x][y]++;
}

public void clearObstacle(int x, int y) {
    if (x >= 0 && x < width && y >= 0 && y < height && obstacles[x][y] > 0) obstacles[x][y]--;
}

public void markCircularObstacle(float centerX, float centerY, float radius) {
    int minX = Math.max(0, (int)(centerX - radius));
    int maxX = Math.min(width - 1, (int)(centerX + radius));
    int minY = Math.max(0, (int)(centerY - radius));
    int maxY = Math.min(height - 1, (int)(centerY + radius));
    for (int x = minX; x <= maxX; x++) {
        for (int y = minY; y <= maxY; y++) {
            double dx = (x + 0.5) - centerX;
            double dy = (y + 0.5) - centerY;
            if (dx * dx + dy * dy <= (double) radius * radius) {
                markObstacle(x, y);
            }
        }
    }
}

public void clearCircularObstacle(float centerX, float centerY, float radius) {
    int minX = Math.max(0, (int)(centerX - radius));
    int maxX = Math.min(width - 1, (int)(centerX + radius));
    int minY = Math.max(0, (int)(centerY - radius));
    int maxY = Math.min(height - 1, (int)(centerY + radius));
    for (int x = minX; x <= maxX; x++) {
        for (int y = minY; y <= maxY; y++) {
            double dx = (x + 0.5) - centerX;
            double dy = (y + 0.5) - centerY;
            if (dx * dx + dy * dy <= (double) radius * radius) {
                clearObstacle(x, y);
            }
        }
    }
}

public void clearAllObstacles() {
    for (int[] col : obstacles) java.util.Arrays.fill(col, 0);
}
```

Update `isWalkable`:
```java
public boolean isWalkable(int x, int y) {
    if (x < 0 || x >= width || y < 0 || y >= height) return false;
    return grid[x][y] != Height.WALL && obstacles[x][y] == 0;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerrainGridObstacleTest -q`
Expected: all 6 tests PASS

- [ ] **Step 5: Add circular obstacle tests**

Add to `TerrainGridObstacleTest.java`:

```java
@Test
void markCircularObstacleBlocksCellsWithinRadius() {
    var grid = grid16();
    grid.markCircularObstacle(8.0f, 8.0f, 1.0f);
    assertFalse(grid.isWalkable(8, 8), "center");
    assertFalse(grid.isWalkable(7, 8), "left");
    assertFalse(grid.isWalkable(8, 7), "below");
    assertTrue(grid.isWalkable(6, 6), "outside radius");
}

@Test
void clearCircularObstacleRestoresCells() {
    var grid = grid16();
    grid.markCircularObstacle(8.0f, 8.0f, 1.0f);
    grid.clearCircularObstacle(8.0f, 8.0f, 1.0f);
    assertTrue(grid.isWalkable(8, 8));
    assertTrue(grid.isWalkable(7, 8));
}

@Test
void aStarRoutesAroundObstacle() {
    var grid = grid16();
    grid.markCircularObstacle(8.0f, 4.0f, 2.0f);
    var pathfinder = new AStarPathfinder();
    var path = pathfinder.findPath(grid, new Point2d(4, 4), new Point2d(12, 4));
    assertFalse(path.isEmpty(), "path should exist around obstacle");
    for (var wp : path) {
        assertTrue(grid.isWalkable((int) wp.x(), (int) wp.y()),
            "waypoint at " + wp + " must be walkable");
    }
}

@Test
void aStarPathAroundObstacleIsLongerThanDirect() {
    var grid = grid16();
    var pathfinder = new AStarPathfinder();
    var directPath = pathfinder.findPath(grid, new Point2d(4, 4), new Point2d(12, 4));
    grid.markCircularObstacle(8.0f, 4.0f, 2.0f);
    var detourPath = pathfinder.findPath(grid, new Point2d(4, 4), new Point2d(12, 4));
    double directDist = pathLength(directPath);
    double detourDist = pathLength(detourPath);
    assertTrue(detourDist > directDist, "detour should be longer: " + detourDist + " vs " + directDist);
}

private static double pathLength(java.util.List<Point2d> path) {
    double len = 0;
    for (int i = 1; i < path.size(); i++) {
        len += path.get(i - 1).distanceTo(path.get(i));
    }
    return len;
}
```

- [ ] **Step 6: Run all TerrainGrid tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerrainGridObstacleTest,TerrainGridTest,TerrainGridRampTest -q`
Expected: all tests PASS (new + existing — no regressions)

- [ ] **Step 7: Run AStarPathfinder tests for regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AStarPathfinderTest -q`
Expected: PASS — obstacle changes to `isWalkable()` don't affect tests that don't mark obstacles

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/TerrainGrid.java \
       quarkmind-sc2/src/test/java/io/quarkmind/domain/TerrainGridObstacleTest.java
git commit -m "feat(#707): add dynamic obstacle layer to TerrainGrid

Reference-counted int[][] obstacles. markObstacle/clearObstacle for
individual cells, markCircularObstacle/clearCircularObstacle for
building/resource footprints. isWalkable() checks both terrain and
obstacles. Refs #707"
```

---

## Batch 2: Worker waypoint movement

### Task 2: Add waypoint support to SimulatedWorker

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Consumes: none (SimulatedWorker is self-contained)
- Produces: `SimulatedWorker.waypoints` field (`Deque<Point2d>`), `SimulatedWorker.isAtFinalTarget()` method, `SimulatedWorker.setWaypoints(Deque<Point2d>)` method

- [ ] **Step 1: Write failing test for waypoint following**

Add to `WorkerSimulationTest.java`:

```java
@Test
void workerFollowsWaypointsToFinalTarget() {
    var w = new SimulatedWorker("w1", new Point2d(0, 0), 0);
    w.task = WorkerTask.MINING_MINERALS;
    var wp1 = new Point2d(3, 0);
    var wp2 = new Point2d(3, 3);
    w.setWaypoints(new ArrayDeque<>(List.of(wp1, wp2)));
    // targetPosition should be first waypoint
    assertEquals(wp1, w.targetPosition);
    // Move until at first waypoint
    for (int i = 0; i < 200; i++) {
        if (w.isAtTarget()) break;
        w.moveOneLoop();
    }
    assertTrue(w.isAtTarget(), "should reach first waypoint");
    assertFalse(w.isAtFinalTarget(), "not at final target yet");
    // Advance to next waypoint
    w.advanceWaypoint();
    assertEquals(wp2, w.targetPosition);
    for (int i = 0; i < 200; i++) {
        if (w.isAtTarget()) break;
        w.moveOneLoop();
    }
    assertTrue(w.isAtFinalTarget(), "should be at final target");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerFollowsWaypointsToFinalTarget -q`
Expected: compilation failure — `setWaypoints`, `isAtFinalTarget`, `advanceWaypoint` don't exist

- [ ] **Step 3: Implement waypoint support on SimulatedWorker**

Add to `SimulatedWorker.java`:

```java
import java.util.ArrayDeque;
import java.util.Deque;

// New field (after existing fields):
Deque<Point2d> waypoints;

void setWaypoints(Deque<Point2d> waypoints) {
    this.waypoints = waypoints;
    if (waypoints != null && !waypoints.isEmpty()) {
        this.targetPosition = waypoints.poll();
    }
}

boolean isAtFinalTarget() {
    return isAtTarget() && (waypoints == null || waypoints.isEmpty());
}

void advanceWaypoint() {
    if (waypoints != null && !waypoints.isEmpty()) {
        this.targetPosition = waypoints.poll();
        this.currentSpeed = 0;
    }
}

void clearWaypoints() {
    this.waypoints = null;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerFollowsWaypointsToFinalTarget -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#707): add waypoint support to SimulatedWorker

Deque<Point2d> waypoints field, setWaypoints() sets first waypoint as
targetPosition, advanceWaypoint() pops next, isAtFinalTarget() checks
both arrival and empty queue. Refs #707"
```

### Task 3: Wire WorkerSimulation to pathfinding

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ObstaclePathTest.java`

**Interfaces:**
- Consumes: `TerrainGrid.markCircularObstacle()` (Task 1), `SimulatedWorker.setWaypoints()` / `advanceWaypoint()` / `isAtFinalTarget()` (Task 2), `AStarPathfinder.findPath()`, `AStarPathfinder.smoothPath()`
- Produces: `WorkerSimulation.setTargetWithPath(SimulatedWorker, Point2d)` method, obstacle registration at lifecycle events

- [ ] **Step 1: Write failing test for obstacle-aware worker paths**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.AStarPathfinder;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.TerrainGrid;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ObstaclePathTest {

    @Test
    void outboundPathAroundMineralLineIsLongerThanStraightLine() {
        var grid = TerrainGrid.emulatedMap();
        var pathfinder = new AStarPathfinder();

        var thPos = new Point2d(10, 10);
        var patchPos = new Point2d(15, 13);

        // Place mineral patches as obstacles (simulating a mineral line arc)
        for (int i = 0; i < 8; i++) {
            double angle = Math.PI * (0.2 + 0.6 * i / 7);
            float px = thPos.x() + (float)(6.7 * Math.cos(angle));
            float py = thPos.y() + (float)(6.7 * Math.sin(angle));
            grid.markCircularObstacle(px, py, WorkerSimulation.MINERAL_PATCH_RADIUS);
        }

        var obstaclePath = pathfinder.findPath(grid, thPos, patchPos);
        grid.clearAllObstacles();
        var directPath = pathfinder.findPath(grid, thPos, patchPos);

        double obstacleDist = pathLength(obstaclePath);
        double directDist = pathLength(directPath);

        assertTrue(obstacleDist > directDist,
            "obstacle path should be longer: " + obstacleDist + " vs " + directDist);
    }

    @Test
    void returnPathFromPatchToBaseIsApproximatelyDirect() {
        var grid = TerrainGrid.emulatedMap();
        var pathfinder = new AStarPathfinder();

        var thPos = new Point2d(10, 10);

        // Place mineral patches as obstacles
        Point2d edgePatchPos = null;
        for (int i = 0; i < 8; i++) {
            double angle = Math.PI * (0.2 + 0.6 * i / 7);
            float px = thPos.x() + (float)(6.7 * Math.cos(angle));
            float py = thPos.y() + (float)(6.7 * Math.sin(angle));
            grid.markCircularObstacle(px, py, WorkerSimulation.MINERAL_PATCH_RADIUS);
            if (i == 0) edgePatchPos = new Point2d(px, py);
        }

        // Return path from edge patch — should be nearly direct
        // (patches are behind the worker, not between worker and TH)
        var returnPath = pathfinder.findPath(grid, edgePatchPos, thPos);
        double returnDist = pathLength(returnPath);
        double straightLine = edgePatchPos.distanceTo(thPos);

        // Return path should be within 30% of straight-line
        assertTrue(returnDist < straightLine * 1.3,
            "return path should be approximately direct: " + returnDist + " vs " + straightLine);
    }

    private static double pathLength(java.util.List<Point2d> path) {
        double len = 0;
        for (int i = 1; i < path.size(); i++) {
            len += path.get(i - 1).distanceTo(path.get(i));
        }
        return len;
    }
}
```

- [ ] **Step 2: Run tests to verify they pass (pure pathfinding, no WorkerSimulation wiring yet)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ObstaclePathTest -q`
Expected: PASS — these test A* + obstacles directly, validating the core hypothesis

- [ ] **Step 3: Add TerrainGrid and AStarPathfinder fields to WorkerSimulation**

Add fields and constructor parameter to `WorkerSimulation`:

```java
// New fields:
private final TerrainGrid terrainGrid;
private final AStarPathfinder pathfinder;

// Updated constructor:
WorkerSimulation(TerrainGrid terrainGrid) {
    double speed = SC2Data.unitSpeed(UnitType.PROBE);
    this.workerDistancePerTick = speed / SC2Data.GAME_LOOPS_PER_SECOND * SC2Data.LOOPS_PER_TICK;
    this.terrainGrid = terrainGrid;
    this.pathfinder = (terrainGrid != null) ? new AStarPathfinder() : null;
}

// Keep zero-arg constructor for backward compatibility during migration:
WorkerSimulation() {
    this(null);
}
```

- [ ] **Step 4: Add setTargetWithPath method to WorkerSimulation**

```java
private void setTargetWithPath(SimulatedWorker w, Point2d target) {
    if (pathfinder == null || terrainGrid == null) {
        w.targetPosition = target;
        w.clearWaypoints();
        return;
    }
    List<Point2d> path = pathfinder.findPath(terrainGrid, w.position, target);
    if (path.isEmpty()) {
        w.targetPosition = target;
        w.clearWaypoints();
    } else {
        List<Point2d> smoothed = AStarPathfinder.smoothPath(path, terrainGrid);
        w.setWaypoints(new ArrayDeque<>(smoothed));
    }
}
```

- [ ] **Step 5: Register mineral patch obstacles in onTownHallCompleted**

In `onTownHallCompleted(String tag, Point2d position)` — after patches are created, register them and the town hall as obstacles:

```java
// After patches are created and added to the base:
if (terrainGrid != null) {
    terrainGrid.markCircularObstacle(position.x(), position.y(), TOWN_HALL_RADIUS);
    for (MineralPatch p : patches) {
        terrainGrid.markCircularObstacle(p.position.x(), p.position.y(), MINERAL_PATCH_RADIUS);
    }
}
```

In `onTownHallCompleted(String tag, Point2d position, List<Resource> mineralResources)` — same pattern, register resource positions as obstacles:

```java
if (terrainGrid != null) {
    terrainGrid.markCircularObstacle(position.x(), position.y(), TOWN_HALL_RADIUS);
    for (MineralPatch p : patches) {
        terrainGrid.markCircularObstacle(p.position.x(), p.position.y(), MINERAL_PATCH_RADIUS);
    }
}
```

- [ ] **Step 6: Register gas building obstacles in onGasBuildingCompleted**

In `onGasBuildingCompleted(String tag, Point2d position)`:

```java
if (terrainGrid != null) {
    terrainGrid.markCircularObstacle(position.x(), position.y(), GAS_BUILDING_RADIUS);
}
```

- [ ] **Step 7: Update EmulatedGame to pass TerrainGrid to WorkerSimulation**

In `EmulatedGame` — where `WorkerSimulation` is constructed, pass the terrain grid:

```java
this.workerSimulation = new WorkerSimulation(terrainGrid);
```

Also register building obstacles when buildings are placed via `EmulatedGame`:

```java
// In the building placement path (applyBuildIntent or similar):
if (terrainGrid != null) {
    terrainGrid.markCircularObstacle(position.x(), position.y(),
        SC2Data.buildingRadius(buildingType));
}
```

- [ ] **Step 8: Replace all direct targetPosition assignments with setTargetWithPath**

In `WorkerSimulation`, find every site that sets `w.targetPosition = X` for mineral or gas workers and replace with `setTargetWithPath(w, X)`. Key sites:

1. `returnToPatchOrReassign()` — `w.targetPosition = miningTarget(...)` → `setTargetWithPath(w, miningTarget(...))`
2. `assignToNearestPatch()` — `w.targetPosition = miningTarget(...)` → `setTargetWithPath(w, miningTarget(...))`
3. `stepOneMineralLoop()` deposit → return: `w.targetPosition = depositTarget(w)` → `setTargetWithPath(w, depositTarget(w))`
4. `stepOneMineralLoop()` mining done → return: `w.targetPosition = depositTarget(w)` → (already covered by RETURNING_MINERALS transition)
5. `stepOneGasLoop()` assigned → geyser: `w.targetPosition = gasTarget(slot)` → `setTargetWithPath(w, gasTarget(slot))`
6. `stepOneGasLoop()` harvest done → return: `w.targetPosition = gasReturnTarget(slot)` → `setTargetWithPath(w, gasReturnTarget(slot))`
7. `onWorkerReturnFromScout()` — `setTargetWithPath(w, miningTarget(...))`
8. `stepOneMineralLoop()` reroute better patch: `w.targetPosition = miningTarget(better, ...)` → `setTargetWithPath(w, miningTarget(better, ...))`

- [ ] **Step 9: Update stepOneMineralLoop to handle waypoints**

In `stepOneMineralLoop`, after `w.moveOneLoop()` returns, check if the worker reached a waypoint but not the final target:

```java
// Replace the existing:
//   if (w.isAtTarget()) { tryStartMining(w, currentLoop); }
// With:
if (w.isAtTarget()) {
    if (!w.isAtFinalTarget()) {
        w.advanceWaypoint();
    } else {
        tryStartMining(w, currentLoop);
    }
}
```

Similarly for the RETURNING_MINERALS block:

```java
// After moveOneLoop returns true (arrived):
if (w.isAtFinalTarget()) {
    // ... existing deposit logic
} else {
    w.advanceWaypoint();
}
```

And the DEPOSITING_MINERALS → return to patch transition — already handled by `returnToPatchOrReassign()` which now calls `setTargetWithPath()`.

- [ ] **Step 10: Update stepOneGasLoop for waypoints**

Same pattern as mineral — after `moveOneLoop()` returns, check `isAtFinalTarget()` before transitioning state.

- [ ] **Step 11: Update MULE movement for waypoints**

In `stepOneMuleLoop`, apply same waypoint handling — MULEs walk to patches through the same mineral line.

- [ ] **Step 12: Run all worker tests for regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest,ObstaclePathTest -q`
Expected: PASS

- [ ] **Step 13: Run full SC2 module tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — many tests use the zero-arg `WorkerSimulation()` constructor which passes `null` terrainGrid, preserving straight-line behavior for existing tests

- [ ] **Step 14: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ObstaclePathTest.java
git commit -m "feat(#707): wire workers to A* pathfinding via obstacle grid

WorkerSimulation accepts TerrainGrid, computes A* waypoint paths via
setTargetWithPath(). All target-setting sites updated. Workers follow
waypoints using existing moveOneLoop() acceleration physics. Mineral
patches, town halls, and gas buildings registered as obstacles at
lifecycle events. Null terrainGrid preserves straight-line behavior
for existing tests. Refs #707"
```

---

## Batch 3: Remove compensation stack and validate

### Task 4: Remove approach docking compensation

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` (if path deviation properties exist)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java` (if CDI properties exist)

**Interfaces:**
- Consumes: Waypoint movement from Task 2/3 (paths are now obstacle-aware)
- Produces: Simplified `MineralPatch` with no compensation fields

- [ ] **Step 1: Remove compensation from MineralPatch**

Delete from `MineralPatch.java`:
- `PATH_DEVIATION_SCALE` field (line 29)
- `PATH_DEVIATION_BASE` field (line 30)
- `outboundPathDelay` field (line 27)
- The body of `computeApproachDocking()` — replace with animation-only:

```java
static void computeApproachDocking(List<MineralPatch> patches, Point2d basePos) {
    double townHallRadius = WorkerSimulation.TOWN_HALL_RADIUS;
    double patchRadius    = WorkerSimulation.MINERAL_PATCH_RADIUS;
    for (MineralPatch p : patches) {
        p.distanceToBase = p.position.distanceTo(basePos);
        p.approachDockingLoops = SimulatedWorker.APPROACH_ANIMATION_LOOPS;
    }
}
```

- [ ] **Step 2: Remove effectiveApproachDocking from WorkerSimulation**

Delete `effectiveApproachDocking()` method (lines 95-110). Replace all calls with direct use of `SimulatedWorker.APPROACH_ANIMATION_LOOPS`:

In `tryStartMining()` — where `effectiveApproachDocking(w.assignedPatch, w.baseIndex)` is called, replace with `SimulatedWorker.APPROACH_ANIMATION_LOOPS`.

- [ ] **Step 3: Remove path deviation config from WorkerPhysicsConfig (if present)**

Check `WorkerPhysicsConfig.java` for any `pathDeviationScale`, `pathDeviationBase`, or `outboundPathDelay` config properties. Remove them and their corresponding CDI properties in `WorkerPhysicsConfigProducer.java`. Update `SimulatedWorker.configure()` and `resetToDefaults()` if they reference these.

- [ ] **Step 4: Remove path deviation reset from resetToDefaults**

In `WorkerSimulation.resetToDefaults()` and `MineralPatch` — remove any reset of `PATH_DEVIATION_SCALE` or `PATH_DEVIATION_BASE`.

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — tests using zero-arg `WorkerSimulation()` still work (no terrainGrid = straight-line + animation-only approach). Tests using `EmulatedGame` will use obstacle paths + animation-only approach.

- [ ] **Step 6: Run playbook calibration tests to assess accuracy impact**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest -q`
Expected: May see checkpoint assertion changes — this is expected. Record results for recalibration.

- [ ] **Step 7: Run dual-run self-baseline tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: Self-baselines (EM vs EM) must PASS. SC2 comparison baselines may shift — record new values.

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java
git commit -m "refactor(#707): remove approach docking compensation stack

Delete computeApproachDocking() path deviation formula, effectiveApproachDocking()
density scaling, PATH_DEVIATION_SCALE, PATH_DEVIATION_BASE, outboundPathDelay.
Approach phase is now APPROACH_ANIMATION_LOOPS (20) only — walk deviation
emerges from obstacle-aware A* paths. Refs #707"
```

### Task 5: Update documentation

**Files:**
- Modify: `docs/sc2-physics-constants.md`

- [ ] **Step 1: Update §3 (mining constants)**

Replace approach docking compensation description with obstacle-aware path description. Document that `approachDockingLoops` is now `APPROACH_ANIMATION_LOOPS` only.

- [ ] **Step 2: Update §15 (worker pathing gap)**

Mark the pathing gap as resolved: workers now use A* pathfinding around mineral patches, buildings, and gas buildings. Document the obstacle registration lifecycle.

- [ ] **Step 3: Add section on obstacle grid**

Document the dynamic obstacle layer: reference-counted `int[][]`, circular footprint registration, clearing on destruction.

- [ ] **Step 4: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#707): update physics constants for obstacle-aware pathfinding

§3 approach docking simplified to animation-only. §15 worker pathing
gap resolved — A* around obstacles. New obstacle grid section. Refs #707"
```

---

## References

- [2026-11-19-obstacle-aware-worker-pathfinding-design.md] — design spec this plan implements
- [decisions.md] — D1-D5 design decisions
- [TerrainGrid.java] — terrain grid with Height enum, isWalkable(), emulatedMap()
- [AStarPathfinder.java] — 8-directional A*, path smoothing, nearestWalkable
- [SimulatedWorker.java] — moveOneLoop() acceleration physics, worker state
- [MineralPatch.java:63-88] — computeApproachDocking() compensation (to remove)
- [WorkerSimulation.java:95-110] — effectiveApproachDocking() density scaling (to remove)
- [WorkerSimulation.java:414-508] — stepOneMineralLoop() (waypoint integration target)
- [PathfindingMovement.java] — existing army waypoint wrapper (reference implementation)
- [EmulatedGame.java:264-297] — enforceWall() building collision (existing building radius data)
- Protocol: worker-physics-cdi-defaults-must-match-constants
- Protocol: sc2data-spatial-constants-require-calibration
- Protocol: calibration-milestones-are-waypoints
- GitHub #707, #731, #732
