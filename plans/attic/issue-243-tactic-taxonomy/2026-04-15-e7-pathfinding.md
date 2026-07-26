# E7: Pathfinding — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add terrain-aware pathfinding to the SC2 emulation engine: a 64×64 `WalkabilityGrid` with a natural chokepoint, A* path computation, a `MovementStrategy` abstraction that keeps all existing tests stable, and a visualizer terrain overlay.

**Architecture:** `WalkabilityGrid` and `AStarPathfinder` live in `domain/` for reuse by both the emulated engine and the real SC2 engine. A `MovementStrategy` interface in `sc2/emulated/` has two implementations: `DirectMovement` (default, wraps `stepToward`, preserves all existing tests) and `PathfindingMovement` (A* based, opt-in per test). `EmulatedGame` delegates movement to the strategy. A QA endpoint serves wall data; the visualizer draws a static terrain layer once at startup.

**Tech Stack:** Pure Java (no new dependencies), JUnit 5, AssertJ, RestAssured, Quarkus JAX-RS, PixiJS 8.

---

## File Map

| Action | Path | Purpose |
|---|---|---|
| Create | `src/main/java/io/quarkmind/domain/WalkabilityGrid.java` | 64×64 boolean grid |
| Create | `src/main/java/io/quarkmind/domain/AStarPathfinder.java` | Stateless A* algorithm |
| Create | `src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java` | Movement interface |
| Create | `src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java` | Wraps stepToward |
| Create | `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java` | A* waypoint following |
| Create | `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java` | GET /qa/emulated/terrain |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Delegate to MovementStrategy |
| Modify | `src/main/resources/META-INF/resources/visualizer.js` | Static terrain layer |
| Create | `src/test/java/io/quarkmind/domain/WalkabilityGridTest.java` | Grid data tests |
| Create | `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java` | Algorithm tests |
| Create | `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java` | Movement strategy tests |
| Create | `src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java` | REST endpoint tests |
| Modify | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | 3 new integration tests |

---

## Task 1: GitHub Epic and Issues

- [ ] **Step 1: Create milestone and issues**

```bash
cd /Users/mdproctor/claude/quarkmind

MILESTONE=$(gh api repos/mdproctor/quarkmind/milestones \
  -f title="E7: Pathfinding" -f state=open --jq '.number')
echo "Milestone: $MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-1: WalkabilityGrid in domain/ — 64×64 grid, emulatedMap(), fromPathingGrid()" \
  --body "Pure Java class. boolean[][] grid. isWalkable(x,y) returning false for out-of-bounds. Static factories: emulatedMap() with wall at y=18 except gap x=11-13; fromPathingGrid() for real SC2." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-2: AStarPathfinder in domain/ — 8-directional A*, tile centres" \
  --body "Stateless. findPath(grid, from, to) returns List<Point2d> tile centres. Cardinal cost=1, diagonal=sqrt(2), Euclidean heuristic. Snaps unwalkable endpoints to nearest walkable." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-3: MovementStrategy interface + DirectMovement + PathfindingMovement" \
  --body "MovementStrategy: advance(tag, current, target, speed), clearUnit(tag), reset(). DirectMovement: wraps stepToward. PathfindingMovement: A* with per-unit waypoint Deques, path recomputed on target change." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-4: EmulatedGame — movementStrategy field, delegation, clearUnit, reset" \
  --body "Add MovementStrategy field defaulting to DirectMovement. Delegate moveFriendlyUnits/moveEnemyUnits to strategy.advance(). Call clearUnit on unit death and staging transfer. Call reset() in EmulatedGame.reset(). Add setMovementStrategy() test helper." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-5: EmulatedTerrainResource — GET /qa/emulated/terrain" \
  --body "QA endpoint returning width, height, and sparse wall list as int[][]. @UnlessBuildProfile(prod). No injection needed — calls WalkabilityGrid.emulatedMap() directly." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-6: Visualizer — static terrain layer from /qa/emulated/terrain" \
  --body "loadTerrain() fetches once at startup. Draws dark rectangles for wall tiles using PIXI.Graphics. Inserted between background and resource layers." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E7-7: Tests — WalkabilityGridTest, AStarPathfinderTest, PathfindingMovementTest, EmulatedTerrainIT, EmulatedGameTest additions" \
  --body "Full TDD coverage: grid data, A* routing around walls, waypoint following, REST endpoint, EmulatedGame integration with pathfinding enabled." \
  --milestone "$MILESTONE"
```

- [ ] **Step 2: Record all 7 issue numbers** — you'll use them in commit messages as `Refs #N`.

---

## Task 2: WalkabilityGrid (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/domain/WalkabilityGridTest.java`
- Create: `src/main/java/io/quarkmind/domain/WalkabilityGrid.java`

- [ ] **Step 1: Write the failing test class**

```java
// src/test/java/io/quarkmind/domain/WalkabilityGridTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class WalkabilityGridTest {

    @Test void emulatedMapNexusTileIsWalkable() {
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(8, 8)).isTrue();
    }
    @Test void emulatedMapStagingTileIsWalkable() {
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(26, 26)).isTrue();
    }
    @Test void emulatedMapWallTileIsBlocked() {
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(20, 18)).isFalse();
    }
    @Test void emulatedMapChokeGapIsWalkable() {
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(12, 18)).isTrue();
    }
    @Test void emulatedMapChokeEdgesAreBlocked() {
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(10, 18)).isFalse();
        assertThat(WalkabilityGrid.emulatedMap().isWalkable(14, 18)).isFalse();
    }
    @Test void outOfBoundsReturnsFalse() {
        WalkabilityGrid g = WalkabilityGrid.emulatedMap();
        assertThat(g.isWalkable(-1,  0)).isFalse();
        assertThat(g.isWalkable(64,  0)).isFalse();
        assertThat(g.isWalkable( 0, 64)).isFalse();
    }
    @Test void fromPathingGrid_bit1IsWalkable() {
        // 0xFF: all 8 bits set = 8 walkable tiles
        WalkabilityGrid g = WalkabilityGrid.fromPathingGrid(new byte[]{(byte)0xFF}, 8, 1);
        assertThat(g.isWalkable(0, 0)).isTrue();
        assertThat(g.isWalkable(7, 0)).isTrue();
    }
    @Test void fromPathingGrid_bit0IsBlocked() {
        WalkabilityGrid g = WalkabilityGrid.fromPathingGrid(new byte[]{(byte)0x00}, 8, 1);
        assertThat(g.isWalkable(0, 0)).isFalse();
        assertThat(g.isWalkable(7, 0)).isFalse();
    }
    @Test void widthAndHeightCorrect() {
        WalkabilityGrid g = WalkabilityGrid.emulatedMap();
        assertThat(g.width()).isEqualTo(64);
        assertThat(g.height()).isEqualTo(64);
    }
}
```

- [ ] **Step 2: Run — confirm failure**
```bash
mvn test -Dtest=WalkabilityGridTest -q 2>&1 | tail -5
```
Expected: compilation failure.

- [ ] **Step 3: Implement WalkabilityGrid**

```java
// src/main/java/io/quarkmind/domain/WalkabilityGrid.java
package io.quarkmind.domain;

import java.util.Arrays;

public final class WalkabilityGrid {

    private final boolean[][] walkable;  // [x][y], true = can walk
    private final int width;
    private final int height;

    public WalkabilityGrid(int width, int height, boolean[][] walkable) {
        this.width    = width;
        this.height   = height;
        this.walkable = walkable;
    }

    public boolean isWalkable(int x, int y) {
        if (x < 0 || x >= width || y < 0 || y >= height) return false;
        return walkable[x][y];
    }

    public int width()  { return width; }
    public int height() { return height; }

    /**
     * 64×64 emulated map: all walkable except a horizontal wall at y=18
     * spanning x=0..63, with a 3-tile chokepoint gap at x=11,12,13.
     * The straight-line path from staging (26,26) to nexus (8,8) crosses
     * y=18 at x≈18, so all units must detour 5+ tiles to reach the gap.
     */
    public static WalkabilityGrid emulatedMap() {
        boolean[][] grid = new boolean[64][64];
        for (boolean[] col : grid) Arrays.fill(col, true);
        for (int x = 0; x < 64; x++) {
            if (x < 11 || x > 13) grid[x][18] = false;
        }
        return new WalkabilityGrid(64, 64, grid);
    }

    /**
     * Constructs a WalkabilityGrid from ocraft's PathingGrid bitmap.
     * Bit encoding: index = x + y*width; bit = (data[index/8] >> (7 - index%8)) & 1
     * 1 = walkable, 0 = wall. Called by ObservationTranslator for real SC2 (future phase).
     */
    public static WalkabilityGrid fromPathingGrid(byte[] data, int width, int height) {
        boolean[][] grid = new boolean[width][height];
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                int index = x + y * width;
                grid[x][y] = ((data[index / 8] >> (7 - index % 8)) & 1) == 1;
            }
        }
        return new WalkabilityGrid(width, height, grid);
    }
}
```

- [ ] **Step 4: Run — confirm all 9 tests pass**
```bash
mvn test -Dtest=WalkabilityGridTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run full suite — no regressions**
```bash
mvn test -q
```

- [ ] **Step 6: Commit**
```bash
git add src/main/java/io/quarkmind/domain/WalkabilityGrid.java \
        src/test/java/io/quarkmind/domain/WalkabilityGridTest.java
git commit -m "feat(e7): WalkabilityGrid — 64×64 emulated map with chokepoint at y=18

Refs #<E7-1 issue number>"
```

---

## Task 3: AStarPathfinder (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`
- Create: `src/main/java/io/quarkmind/domain/AStarPathfinder.java`

- [ ] **Step 1: Write the failing test class**

```java
// src/test/java/io/quarkmind/domain/AStarPathfinderTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.Arrays;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class AStarPathfinderTest {

    private final AStarPathfinder pf = new AStarPathfinder();

    /** Fully walkable width×height grid. */
    private WalkabilityGrid open(int w, int h) {
        boolean[][] g = new boolean[w][h];
        for (boolean[] col : g) Arrays.fill(col, true);
        return new WalkabilityGrid(w, h, g);
    }

    /** Grid with horizontal wall at wallY (full width) except gap at gapX. */
    private WalkabilityGrid withWall(int w, int h, int wallY, int gapX) {
        boolean[][] g = new boolean[w][h];
        for (boolean[] col : g) Arrays.fill(col, true);
        for (int x = 0; x < w; x++) {
            if (x != gapX) g[x][wallY] = false;
        }
        return new WalkabilityGrid(w, h, g);
    }

    @Test void pathOnOpenMap_isNonEmpty() {
        List<Point2d> path = pf.findPath(open(10, 10), new Point2d(0, 0), new Point2d(9, 9));
        assertThat(path).isNotEmpty();
    }

    @Test void pathOnOpenMap_allWaypointsWalkable() {
        WalkabilityGrid g = open(10, 10);
        List<Point2d> path = pf.findPath(g, new Point2d(0, 0), new Point2d(9, 9));
        for (Point2d wp : path) {
            assertThat(g.isWalkable((int) wp.x(), (int) wp.y()))
                .as("waypoint %s should be walkable", wp).isTrue();
        }
    }

    @Test void pathOnOpenMap_endsNearGoal() {
        List<Point2d> path = pf.findPath(open(10, 10), new Point2d(0, 0), new Point2d(9, 9));
        Point2d last = path.get(path.size() - 1);
        // Last waypoint is the centre of the goal tile: (9.5, 9.5)
        assertThat(last.x()).isBetween(9f, 10f);
        assertThat(last.y()).isBetween(9f, 10f);
    }

    @Test void pathAroundWall_doesNotCrossWall() {
        // 10×10, wall at y=5 full width except gap at x=5
        WalkabilityGrid g = withWall(10, 10, 5, 5);
        List<Point2d> path = pf.findPath(g, new Point2d(2, 2), new Point2d(7, 8));
        for (Point2d wp : path) {
            int tx = (int) wp.x();
            int ty = (int) wp.y();
            if (ty == 5) {
                assertThat(tx).as("must cross wall only at gap (x=5)").isEqualTo(5);
            }
        }
    }

    @Test void pathAroundWall_reachesGoal() {
        WalkabilityGrid g = withWall(10, 10, 5, 5);
        List<Point2d> path = pf.findPath(g, new Point2d(2, 2), new Point2d(7, 8));
        assertThat(path).isNotEmpty();
        Point2d last = path.get(path.size() - 1);
        assertThat((int) last.x()).isEqualTo(7);
        assertThat((int) last.y()).isEqualTo(8);
    }

    @Test void sameStartAndGoal_returnsEmpty() {
        assertThat(pf.findPath(open(10, 10), new Point2d(5, 5), new Point2d(5, 5))).isEmpty();
    }

    @Test void unreachableGoal_returnsEmpty() {
        // Fully-walled column at x=5 — left side can't reach right side
        boolean[][] g = new boolean[10][10];
        for (boolean[] col : g) Arrays.fill(col, true);
        for (int y = 0; y < 10; y++) g[5][y] = false;
        WalkabilityGrid blocked = new WalkabilityGrid(10, 10, g);
        List<Point2d> path = pf.findPath(blocked, new Point2d(2, 5), new Point2d(8, 5));
        assertThat(path).isEmpty();
    }

    @Test void pathOnEmulatedMap_nexusToStaging_isNonEmpty() {
        WalkabilityGrid g = WalkabilityGrid.emulatedMap();
        List<Point2d> path = pf.findPath(g, new Point2d(8, 8), new Point2d(26, 26));
        assertThat(path).isNotEmpty();
    }

    @Test void pathOnEmulatedMap_passesNearChokepoint() {
        // Path from nexus (8,8) to staging (26,26) must cross y=18 only at x=11,12,13
        WalkabilityGrid g = WalkabilityGrid.emulatedMap();
        List<Point2d> path = pf.findPath(g, new Point2d(8, 8), new Point2d(26, 26));
        for (Point2d wp : path) {
            int tx = (int) wp.x();
            int ty = (int) wp.y();
            if (ty == 18) {
                assertThat(tx).as("must cross wall only through gap x=[11,13]").isBetween(11, 13);
            }
        }
    }

    @Test void pathOnEmulatedMap_stagingToNexus() {
        // Reverse direction also routes through chokepoint
        WalkabilityGrid g = WalkabilityGrid.emulatedMap();
        List<Point2d> path = pf.findPath(g, new Point2d(26, 26), new Point2d(8, 8));
        assertThat(path).isNotEmpty();
        for (Point2d wp : path) {
            int tx = (int) wp.x();
            int ty = (int) wp.y();
            if (ty == 18) {
                assertThat(tx).isBetween(11, 13);
            }
        }
    }
}
```

- [ ] **Step 2: Run — confirm failure**
```bash
mvn test -Dtest=AStarPathfinderTest -q 2>&1 | tail -5
```
Expected: compilation failure.

- [ ] **Step 3: Implement AStarPathfinder**

```java
// src/main/java/io/quarkmind/domain/AStarPathfinder.java
package io.quarkmind.domain;

import java.util.*;

/**
 * Stateless 8-directional A* pathfinder for the SC2 emulation engine.
 * Returns tile-centre world coordinates (Point2d at x+0.5, y+0.5).
 * Used by both the emulated engine (WalkabilityGrid.emulatedMap()) and
 * the real SC2 engine (WalkabilityGrid.fromPathingGrid()) — no code duplication.
 */
public final class AStarPathfinder {

    private static final int[][] DIRS = {
        {1, 0}, {-1, 0}, {0, 1}, {0, -1},   // cardinal
        {1, 1}, {1,-1}, {-1, 1}, {-1,-1}     // diagonal
    };
    private static final double DIAG = Math.sqrt(2);

    /**
     * Find a path from {@code from} to {@code to}.
     * Returns tile-centre waypoints in order, excluding the start tile.
     * Returns empty if start==goal tile or goal is unreachable.
     */
    public List<Point2d> findPath(WalkabilityGrid grid, Point2d from, Point2d to) {
        int[] start = nearestWalkable(grid, (int) from.x(), (int) from.y());
        int[] goal  = nearestWalkable(grid, (int) to.x(),   (int) to.y());
        final int sx = start[0], sy = start[1];
        final int gx = goal[0],  gy = goal[1];

        if (sx == gx && sy == gy) return List.of();

        record Node(int x, int y, double g, Node parent) {}

        PriorityQueue<Node> open = new PriorityQueue<>(
            Comparator.comparingDouble(n ->
                n.g() + Math.sqrt((n.x() - gx) * (double)(n.x() - gx)
                                + (n.y() - gy) * (double)(n.y() - gy))));

        boolean[][] closed = new boolean[grid.width()][grid.height()];
        open.add(new Node(sx, sy, 0, null));

        while (!open.isEmpty()) {
            Node cur = open.poll();
            if (closed[cur.x()][cur.y()]) continue;
            closed[cur.x()][cur.y()] = true;

            if (cur.x() == gx && cur.y() == gy) {
                Deque<Point2d> path = new ArrayDeque<>();
                for (Node n = cur; n.parent() != null; n = n.parent()) {
                    path.addFirst(new Point2d(n.x() + 0.5f, n.y() + 0.5f));
                }
                return new ArrayList<>(path);
            }

            for (int[] d : DIRS) {
                int nx = cur.x() + d[0];
                int ny = cur.y() + d[1];
                if (!grid.isWalkable(nx, ny) || closed[nx][ny]) continue;
                double cost = (d[0] != 0 && d[1] != 0) ? DIAG : 1.0;
                open.add(new Node(nx, ny, cur.g() + cost, cur));
            }
        }
        return List.of();
    }

    /** If tile (x,y) is walkable, return it. Otherwise spiral outward to find nearest walkable tile. */
    private static int[] nearestWalkable(WalkabilityGrid grid, int x, int y) {
        if (grid.isWalkable(x, y)) return new int[]{x, y};
        for (int r = 1; r <= Math.max(grid.width(), grid.height()); r++) {
            for (int dx = -r; dx <= r; dx++) {
                for (int dy = -r; dy <= r; dy++) {
                    if (Math.abs(dx) == r || Math.abs(dy) == r) {
                        if (grid.isWalkable(x + dx, y + dy))
                            return new int[]{x + dx, y + dy};
                    }
                }
            }
        }
        return new int[]{x, y};
    }
}
```

- [ ] **Step 4: Run — confirm all 10 tests pass**
```bash
mvn test -Dtest=AStarPathfinderTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run full suite — no regressions**
```bash
mvn test -q
```

- [ ] **Step 6: Commit**
```bash
git add src/main/java/io/quarkmind/domain/AStarPathfinder.java \
        src/test/java/io/quarkmind/domain/AStarPathfinderTest.java
git commit -m "feat(e7): AStarPathfinder — 8-directional A* with tile-centre waypoints

Refs #<E7-2 issue number>"
```

---

## Task 4: MovementStrategy + DirectMovement + PathfindingMovement (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`
- Create: `src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java`
- Create: `src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java`
- Create: `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`

- [ ] **Step 1: Write the failing test class**

```java
// src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.Arrays;
import static org.assertj.core.api.Assertions.assertThat;

class PathfindingMovementTest {

    /** Fully walkable 32×32 grid — no obstacles. */
    private WalkabilityGrid open() {
        boolean[][] g = new boolean[32][32];
        for (boolean[] col : g) Arrays.fill(col, true);
        return new WalkabilityGrid(32, 32, g);
    }

    @Test void unitMovesTowardFirstWaypoint() {
        PathfindingMovement pm = new PathfindingMovement(open());
        Point2d start  = new Point2d(2f, 2f);
        Point2d target = new Point2d(20f, 20f);

        Point2d after = pm.advance("u1", start, target, 0.5);

        double before = EmulatedGame.distance(start, target);
        double after2 = EmulatedGame.distance(after, target);
        assertThat(after2).isLessThan(before);
    }

    @Test void unitArrivalAfterManyTicks() {
        PathfindingMovement pm = new PathfindingMovement(open());
        Point2d target = new Point2d(10f, 10f);
        Point2d pos = new Point2d(2f, 2f);
        for (int i = 0; i < 100; i++) {
            pos = pm.advance("u1", pos, target, 0.5);
            if (EmulatedGame.distance(pos, target) < 1.0) break;
        }
        assertThat(EmulatedGame.distance(pos, target)).isLessThan(1.0);
    }

    @Test void targetChangeClearsOldPath() {
        PathfindingMovement pm = new PathfindingMovement(open());
        Point2d target1 = new Point2d(20f, 2f);
        Point2d target2 = new Point2d(2f, 20f);
        Point2d pos = new Point2d(2f, 2f);

        // Move toward target1 a few steps
        for (int i = 0; i < 4; i++) pos = pm.advance("u1", pos, target1, 0.5);

        // Switch target — next advance should move toward target2
        double distBefore = EmulatedGame.distance(pos, target2);
        Point2d next = pm.advance("u1", pos, target2, 0.5);
        assertThat(EmulatedGame.distance(next, target2)).isLessThanOrEqualTo(distBefore);
    }

    @Test void clearUnitAllowsCleanRestart() {
        PathfindingMovement pm = new PathfindingMovement(open());
        Point2d target = new Point2d(20f, 20f);
        Point2d pos = new Point2d(2f, 2f);

        for (int i = 0; i < 5; i++) pos = pm.advance("u1", pos, target, 0.5);
        pm.clearUnit("u1");

        // After clear, fresh start recomputes path correctly
        Point2d fresh = new Point2d(5f, 5f);
        Point2d after = pm.advance("u1", fresh, target, 0.5);
        assertThat(EmulatedGame.distance(after, target))
            .isLessThan(EmulatedGame.distance(fresh, target));
    }

    @Test void unreachableTargetFallsBackToStepToward() {
        // Vertical wall at x=5 — target on right side is unreachable from left
        boolean[][] g = new boolean[10][10];
        for (boolean[] col : g) Arrays.fill(col, true);
        for (int y = 0; y < 10; y++) g[5][y] = false;
        PathfindingMovement pm = new PathfindingMovement(new WalkabilityGrid(10, 10, g));

        Point2d start  = new Point2d(2f, 5f);
        Point2d target = new Point2d(8f, 5f);
        // Must not throw — falls back to stepToward
        Point2d result = pm.advance("u1", start, target, 0.5);
        assertThat(result).isNotNull();
    }

    @Test void resetClearsAllUnitState() {
        PathfindingMovement pm = new PathfindingMovement(open());
        pm.advance("u1", new Point2d(2f, 2f), new Point2d(20f, 20f), 0.5);
        pm.advance("u2", new Point2d(3f, 3f), new Point2d(18f, 18f), 0.5);

        pm.reset();

        // After reset, both units get fresh paths — no exception
        Point2d a = pm.advance("u1", new Point2d(1f, 1f), new Point2d(10f, 10f), 0.5);
        Point2d b = pm.advance("u2", new Point2d(2f, 2f), new Point2d(12f, 12f), 0.5);
        assertThat(a).isNotNull();
        assertThat(b).isNotNull();
    }
}
```

- [ ] **Step 2: Run — confirm failure**
```bash
mvn test -Dtest=PathfindingMovementTest -q 2>&1 | tail -5
```
Expected: compilation failure.

- [ ] **Step 3: Create MovementStrategy interface**

```java
// src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;

public interface MovementStrategy {

    /** Advance one tick: return new world position for a unit moving toward target. */
    Point2d advance(String unitTag, Point2d current, Point2d target, double speed);

    /** Called when a unit is permanently removed (dead or transferred to staging). */
    default void clearUnit(String unitTag) {}

    /** Called from EmulatedGame.reset() — clear ALL per-unit state. */
    default void reset() {}
}
```

- [ ] **Step 4: Create DirectMovement**

```java
// src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;

/** Default movement: straight-line interpolation. Preserves all pre-E7 test behaviour. */
public class DirectMovement implements MovementStrategy {

    @Override
    public Point2d advance(String unitTag, Point2d current, Point2d target, double speed) {
        return EmulatedGame.stepToward(current, target, speed);
    }
    // clearUnit and reset are no-ops (inherited defaults)
}
```

- [ ] **Step 5: Create PathfindingMovement**

```java
// src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.AStarPathfinder;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.WalkabilityGrid;

import java.util.*;

/** A* based movement: computes and follows waypoint queues per unit. */
public class PathfindingMovement implements MovementStrategy {

    private final WalkabilityGrid grid;
    private final AStarPathfinder pathfinder = new AStarPathfinder();
    private final Map<String, Deque<Point2d>> waypoints   = new HashMap<>();
    private final Map<String, Point2d>        lastTargets = new HashMap<>();

    public PathfindingMovement(WalkabilityGrid grid) { this.grid = grid; }

    @Override
    public Point2d advance(String unitTag, Point2d current, Point2d target, double speed) {
        // Recompute path when target changes
        if (!target.equals(lastTargets.get(unitTag))) {
            List<Point2d> path = pathfinder.findPath(grid, current, target);
            waypoints.put(unitTag, new ArrayDeque<>(path));
            lastTargets.put(unitTag, target);
        }

        Deque<Point2d> queue = waypoints.get(unitTag);
        if (queue == null || queue.isEmpty()) {
            // No path (unreachable or arrived) — fall back to direct movement
            return EmulatedGame.stepToward(current, target, speed);
        }

        Point2d next   = queue.peek();
        Point2d newPos = EmulatedGame.stepToward(current, next, speed);

        // Advance to next waypoint when current one is reached (within 0.1 tiles)
        if (EmulatedGame.distance(newPos, next) < 0.1) {
            queue.poll();
        }
        return newPos;
    }

    @Override
    public void clearUnit(String unitTag) {
        waypoints.remove(unitTag);
        lastTargets.remove(unitTag);
    }

    @Override
    public void reset() {
        waypoints.clear();
        lastTargets.clear();
    }
}
```

- [ ] **Step 6: Run — confirm all 6 tests pass**
```bash
mvn test -Dtest=PathfindingMovementTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 7: Run full suite**
```bash
mvn test -q
```

- [ ] **Step 8: Commit**
```bash
git add src/main/java/io/quarkmind/sc2/emulated/MovementStrategy.java \
        src/main/java/io/quarkmind/sc2/emulated/DirectMovement.java \
        src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java \
        src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java
git commit -m "feat(e7): MovementStrategy + DirectMovement + PathfindingMovement

Refs #<E7-3 issue number>"
```

---

## Task 5: EmulatedGame Integration (TDD)

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Write 3 failing tests**

Add at the end of `EmulatedGameTest` (before closing `}`):

```java
// ---- E7: pathfinding integration ----

@Test
void withPathfinding_unitEventuallyReachesTargetAcrossWall() {
    game.setMovementStrategy(new PathfindingMovement(WalkabilityGrid.emulatedMap()));
    // From nexus side (8,8) to staging side (12,22) — must cross wall at y=18 via chokepoint
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(8, 8));
    game.applyIntent(new MoveIntent(tag, new Point2d(12, 22)));

    // ~30 tiles at 0.5/tick = ~60 ticks; use 120 for safety
    for (int i = 0; i < 120; i++) game.tick();

    Unit unit = game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals(tag)).findFirst().orElseThrow();
    assertThat(EmulatedGame.distance(unit.position(), new Point2d(12, 22))).isLessThan(2.0);
}

@Test
void withPathfinding_unitDoesNotCrossWallOutsideChokepoint() {
    game.setMovementStrategy(new PathfindingMovement(WalkabilityGrid.emulatedMap()));
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(8, 8));
    game.applyIntent(new MoveIntent(tag, new Point2d(12, 22)));

    // Run ticks and check that whenever unit is at tileY=18, tileX is in gap [11,13]
    for (int i = 0; i < 80; i++) {
        game.tick();
        game.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(tag)).findFirst().ifPresent(unit -> {
                int tileX = (int) unit.position().x();
                int tileY = (int) unit.position().y();
                if (tileY == 18) {
                    assertThat(tileX)
                        .as("unit at y=18 must be in gap x=[11,13], was x=%d", tileX)
                        .isBetween(11, 13);
                }
            });
    }
}

@Test
void directMovementDefaultIsUnchanged() {
    // No setMovementStrategy call — defaults to DirectMovement (straight line)
    game.applyIntent(new MoveIntent("probe-0", new Point2d(15, 9)));
    Point2d before = game.snapshot().myUnits().get(0).position();
    game.tick();
    Point2d after = game.snapshot().myUnits().get(0).position();
    // Moved toward x=15 (right), y unchanged
    assertThat(after.x()).isGreaterThan(before.x());
    assertThat(after.y()).isCloseTo(before.y(), org.assertj.core.data.Offset.offset(0.1f));
}
```

- [ ] **Step 2: Run — confirm 3 new tests fail**
```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -5
```
Expected: 3 failures (setMovementStrategy doesn't exist yet).

- [ ] **Step 3: Update EmulatedGame**

**3a — Add field after the existing `damageCalculator` field:**
```java
private MovementStrategy movementStrategy = new DirectMovement();
```

**3b — Replace `moveFriendlyUnits()` body:**
```java
private void moveFriendlyUnits() {
    myUnits.replaceAll(u -> {
        Point2d target = unitTargets.get(u.tag());
        if (target == null) return u;
        Point2d newPos = movementStrategy.advance(u.tag(), u.position(), target, unitSpeed);
        if (distance(newPos, target) < 0.2) unitTargets.remove(u.tag());
        return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                        u.shields(), u.maxShields());
    });
}
```

**3c — Replace `moveEnemyUnits()` body:**
```java
private void moveEnemyUnits() {
    enemyUnits.replaceAll(u -> {
        Point2d target = enemyTargets.getOrDefault(u.tag(), NEXUS_POS);
        Point2d newPos = movementStrategy.advance(u.tag(), u.position(), target, unitSpeed);
        return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                        u.shields(), u.maxShields());
    });
}
```

**3d — Add `movementStrategy.clearUnit(u.tag())` in `resolveCombat()` dead-unit blocks.**

Find the friendly dead-unit block:
```java
myUnits.removeIf(u -> {
    if (u.health() <= 0) {
        unitTargets.remove(u.tag());
        attackingUnits.remove(u.tag());
        unitCooldowns.remove(u.tag());
        return true;
    }
    return false;
});
```
Add `movementStrategy.clearUnit(u.tag());` after `unitCooldowns.remove(u.tag());`.

Find the enemy dead-unit block:
```java
enemyUnits.removeIf(u -> {
    if (u.health() <= 0) {
        enemyTargets.remove(u.tag());
        enemyCooldowns.remove(u.tag());
        retreatingUnits.remove(u.tag());
        return true;
    }
    return false;
});
```
Add `movementStrategy.clearUnit(u.tag());` after `retreatingUnits.remove(u.tag());`.

**3e — Add `movementStrategy.clearUnit(u.tag())` in `tickEnemyRetreat()` staging transfer.**

Find the staging transfer block in `tickEnemyRetreat()`:
```java
retreatingUnits.remove(u.tag());
enemyTargets.remove(u.tag());
enemyStagingArea.add(u);
```
Add `movementStrategy.clearUnit(u.tag());` after `enemyTargets.remove(u.tag());`.

**3f — Add `movementStrategy.reset()` in `reset()`.**

After `retreatingUnits.clear();` add:
```java
movementStrategy.reset();
```

**3g — Add test helper at the bottom of EmulatedGame:**
```java
/** Swap movement strategy — used by pathfinding tests. */
void setMovementStrategy(MovementStrategy s) { this.movementStrategy = s; }
```

- [ ] **Step 4: Run EmulatedGameTest — all tests pass**
```bash
mvn test -Dtest=EmulatedGameTest -q
```
Expected: BUILD SUCCESS. All existing tests still pass (DirectMovement is the default).

- [ ] **Step 5: Run full suite**
```bash
mvn test -q
```

- [ ] **Step 6: Commit**
```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e7): EmulatedGame — MovementStrategy delegation, clearUnit, reset

Closes #<E7-4 issue number>"
```

---

## Task 6: EmulatedTerrainResource + Integration Test (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java`
- Create: `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java`

- [ ] **Step 1: Write the failing integration test**

```java
// src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class EmulatedTerrainIT {

    @Test
    void terrainEndpointReturnsExpectedDimensions() {
        given().when().get("/qa/emulated/terrain")
            .then()
            .statusCode(200)
            .body("width",  equalTo(64))
            .body("height", equalTo(64));
    }

    @Test
    void terrainEndpointHasCorrectWallCount() {
        // Wall at y=18 spans 64 tiles minus 3-tile gap = 61 walls
        given().when().get("/qa/emulated/terrain")
            .then()
            .statusCode(200)
            .body("walls.size()", equalTo(61));
    }

    @Test
    void terrainEndpointIncludesWallTile() {
        // [20, 18] is a wall tile (outside the gap)
        given().when().get("/qa/emulated/terrain")
            .then()
            .statusCode(200)
            .body("walls.find { it == [20, 18] }", notNullValue());
    }

    @Test
    void terrainEndpointExcludesGapTile() {
        // [12, 18] is in the chokepoint gap — walkable, must NOT appear in walls
        given().when().get("/qa/emulated/terrain")
            .then()
            .statusCode(200)
            .body("walls.find { it == [12, 18] }", nullValue());
    }
}
```

- [ ] **Step 2: Run — confirm failure**
```bash
mvn test -Dtest=EmulatedTerrainIT -q 2>&1 | tail -5
```
Expected: FAIL — 404 (endpoint doesn't exist yet).

- [ ] **Step 3: Implement EmulatedTerrainResource**

```java
// src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java
package io.quarkmind.qa;

import io.quarkmind.domain.WalkabilityGrid;
import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import java.util.ArrayList;
import java.util.List;

/**
 * Returns the emulated map's terrain as a sparse wall list.
 * Fetched once by the visualizer at startup; never changes during a game.
 */
@UnlessBuildProfile("prod")
@Path("/qa/emulated/terrain")
@Produces(MediaType.APPLICATION_JSON)
public class EmulatedTerrainResource {

    @GET
    public TerrainResponse getTerrain() {
        WalkabilityGrid grid = WalkabilityGrid.emulatedMap();
        List<int[]> walls = new ArrayList<>();
        for (int x = 0; x < grid.width(); x++) {
            for (int y = 0; y < grid.height(); y++) {
                if (!grid.isWalkable(x, y)) walls.add(new int[]{x, y});
            }
        }
        return new TerrainResponse(grid.width(), grid.height(), walls);
    }

    public record TerrainResponse(int width, int height, List<int[]> walls) {}
}
```

- [ ] **Step 4: Run EmulatedTerrainIT — confirm all 4 tests pass**
```bash
mvn test -Dtest=EmulatedTerrainIT -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run full suite**
```bash
mvn test -q
```

- [ ] **Step 6: Commit**
```bash
git add src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java \
        src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java
git commit -m "feat(e7): EmulatedTerrainResource — GET /qa/emulated/terrain wall list

Closes #<E7-5 issue number>"
```

---

## Task 7: Visualizer Terrain Layer

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

- [ ] **Step 1: Add `loadTerrain()` function**

In `visualizer.js`, add this function after `drawGrid()` and before `makeUnitSprite()`:

```javascript
/**
 * Fetch terrain once at startup and draw static wall tiles.
 * Silently no-ops if the endpoint is unavailable (non-emulated profile).
 * Walls render as dark filled rectangles beneath the grid.
 */
async function loadTerrain(container) {
    try {
        const resp = await fetch('/qa/emulated/terrain');
        if (!resp.ok) return;
        const data = await resp.json();
        const g = new PIXI.Graphics();
        data.walls.forEach(([wx, wy]) => {
            // Convert game tile (wx, wy) to canvas rectangle.
            // tile(wx, wy) gives canvas position of tile bottom-left (Y-axis flipped).
            const canvasX = wx * SCALE;
            const canvasY = (VIEWPORT_H - wy - 1) * SCALE;
            g.rect(canvasX, canvasY, SCALE, SCALE).fill({ color: 0x333333, alpha: 0.85 });
        });
        container.addChild(g);
    } catch (e) {
        console.warn('Could not load terrain:', e);
    }
}
```

- [ ] **Step 2: Add terrain container to the layer stack in `init()`**

Find this block in `init()`:
```javascript
const background = new PIXI.Container();
const resource   = new PIXI.Container();
const building   = new PIXI.Container();
const unit       = new PIXI.Container();
const enemy      = new PIXI.Container();
const staging    = new PIXI.Container();
const hud        = new PIXI.Container();
app.stage.addChild(background, resource, building, unit, enemy, staging, hud);
```

Replace with:
```javascript
const background = new PIXI.Container();
const terrain    = new PIXI.Container();  // static wall overlay, loaded once
const resource   = new PIXI.Container();
const building   = new PIXI.Container();
const unit       = new PIXI.Container();
const enemy      = new PIXI.Container();
const staging    = new PIXI.Container();
const hud        = new PIXI.Container();
app.stage.addChild(background, terrain, resource, building, unit, enemy, staging, hud);
```

- [ ] **Step 3: Call `loadTerrain()` in `init()` after `loadAssets()`**

Find:
```javascript
await loadAssets();
connect();
initConfigPanel();
```

Replace with:
```javascript
await loadAssets();
await loadTerrain(terrain);
connect();
initConfigPanel();
```

- [ ] **Step 4: Verify visualizer compiles and terrain layer renders**

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

Open `http://localhost:8080/visualizer.html`. Confirm:
- Dark wall tiles appear at `y=18` across the map
- Grid lines are visible on top of the wall tiles
- Gap at `x=11-13, y=18` is open (no dark tile)
- Unit sprites are not obscured

Kill the server with Ctrl+C when verified.

- [ ] **Step 5: Commit**
```bash
git add src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(e7): visualizer — static terrain layer drawn from /qa/emulated/terrain

Closes #<E7-6 issue number>
Closes #<E7-7 issue number>"
```

---

## Self-Check Before Calling Done

```bash
mvn test
```

Verify:
- `WalkabilityGridTest`: 9 tests pass
- `AStarPathfinderTest`: 10 tests pass
- `PathfindingMovementTest`: 6 tests pass
- `EmulatedGameTest`: all existing + 3 new E7 tests pass
- `EmulatedTerrainIT`: 4 tests pass
- All other test classes: unchanged, still passing

Confirm all 7 GitHub issues are linked and close the E7 milestone.
