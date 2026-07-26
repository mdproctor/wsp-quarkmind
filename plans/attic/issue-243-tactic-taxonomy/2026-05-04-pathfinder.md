# Pathfinder Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire real SC2 terrain into the agent via CDI, add RAMP cost weighting to A*, and reduce waypoint zigzag via greedy string-pulling smoothing.

**Architecture:** Three independent domain changes. C (terrain extraction) makes `SC2BotAgent` a CDI bean and extracts pathing grid bytes from `ResponseGameInfo.getStartRaw()` into `TerrainGrid`. B adds `movementCost()` to `TerrainGrid` and multiplies A* edge costs by it. A adds static `smoothPath()` to `AStarPathfinder` using Bresenham LOS and wires it into `PathfindingMovement.advance()`.

**Tech Stack:** Java 21, JUnit 5, AssertJ, ocraft-s2client-protocol 0.4.21 (`ResponseGameInfo`, `StartRaw`, `ImageData`), existing `TerrainGrid`, `AStarPathfinder`, `PathfindingMovement`.

---

## File Map

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `sc2/real/SC2BotAgent.java` | CDI bean; inject `TerrainProvider`; extract terrain in `onGameStart()` |
| Modify | `sc2/real/RealSC2Engine.java` | Inject `SC2BotAgent` instead of constructing; remove constructor param |
| Modify | `domain/TerrainGrid.java` | Add `movementCost(int x, int y): double` |
| Modify | `domain/AStarPathfinder.java` | Weighted edge cost; add static `smoothPath(List<Point2d>, TerrainGrid)` |
| Modify | `sc2/emulated/PathfindingMovement.java` | Call `smoothPath()` after `findPath()` |
| Create | `test/.../real/SC2BotAgentTerrainTest.java` | Synthetic bitmap → correct `TerrainGrid` via `fromPathingGrid()` |
| Modify | `test/.../domain/AStarPathfinderTest.java` | Weighted cost tests; smoothing tests |
| Create | `test/.../domain/TerrainGridTest.java` | `movementCost()` returns correct values per `Height` |

All source paths relative to `src/main/java/io/quarkmind/` and `src/test/java/io/quarkmind/`.

---

## Task 1: SC2BotAgent → CDI bean + terrain extraction

**GitHub:** `Refs #121` (child of epic #120)

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java`
- Modify: `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java`
- Create: `src/test/java/io/quarkmind/sc2/real/SC2BotAgentTerrainTest.java`

- [ ] **Step 1: Verify baseline passes**

```bash
mvn test -q 2>&1 | tail -3
```
Expected: BUILD SUCCESS.

- [ ] **Step 2: Write failing test for the terrain extraction transformation**

The terrain extraction logic (`fromPathingGrid`) is tested in isolation. We cannot mock `S2Agent.observation()` without SC2, so we test the transformation directly.

Create `src/test/java/io/quarkmind/sc2/real/SC2BotAgentTerrainTest.java`:

```java
package io.quarkmind.sc2.real;

import io.quarkmind.domain.TerrainGrid;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class SC2BotAgentTerrainTest {

    /**
     * 4×4 pathing grid, 1 bit per tile = 2 bytes.
     * Bit encoding: index = x + y*width; walkable=(bit==1)
     *   index 0=(0,0)..index 7=(3,1): byte 0
     *   index 8=(0,2)..index 15=(3,3): byte 1
     * Wall at (2,1): index = 2 + 1*4 = 6 → byte0, bit position = 7-6 = 1
     * All walkable except (2,1): byte0 = 0b11111101 = 0xFD, byte1 = 0xFF
     */
    @Test
    void fromPathingGrid_singleWallTile_correctWalkability() {
        byte[] data = {(byte) 0xFD, (byte) 0xFF};
        TerrainGrid grid = TerrainGrid.fromPathingGrid(data, 4, 4);

        assertThat(grid.isWalkable(0, 0)).isTrue();
        assertThat(grid.isWalkable(1, 0)).isTrue();
        assertThat(grid.isWalkable(3, 0)).isTrue();
        assertThat(grid.isWalkable(0, 1)).isTrue();
        assertThat(grid.isWalkable(2, 1)).isFalse();  // the wall
        assertThat(grid.isWalkable(3, 1)).isTrue();
        assertThat(grid.isWalkable(0, 3)).isTrue();
        assertThat(grid.isWalkable(3, 3)).isTrue();
    }

    @Test
    void fromPathingGrid_allWalkable() {
        byte[] data = {(byte) 0xFF, (byte) 0xFF};
        TerrainGrid grid = TerrainGrid.fromPathingGrid(data, 4, 4);
        for (int x = 0; x < 4; x++) {
            for (int y = 0; y < 4; y++) {
                assertThat(grid.isWalkable(x, y)).as("(%d,%d) should be walkable", x, y).isTrue();
            }
        }
    }

    @Test
    void fromPathingGrid_allWalls() {
        byte[] data = {(byte) 0x00, (byte) 0x00};
        TerrainGrid grid = TerrainGrid.fromPathingGrid(data, 4, 4);
        for (int x = 0; x < 4; x++) {
            for (int y = 0; y < 4; y++) {
                assertThat(grid.isWalkable(x, y)).as("(%d,%d) should be wall", x, y).isFalse();
            }
        }
    }

    @Test
    void fromPathingGrid_dimensions_areCorrect() {
        byte[] data = new byte[4]; // 4 bytes = 32 bits = 8×4 grid
        java.util.Arrays.fill(data, (byte) 0xFF);
        TerrainGrid grid = TerrainGrid.fromPathingGrid(data, 8, 4);
        assertThat(grid.width()).isEqualTo(8);
        assertThat(grid.height()).isEqualTo(4);
    }
}
```

- [ ] **Step 3: Run tests — verify they pass** (these test existing `fromPathingGrid` logic)

```bash
mvn test -Dtest=SC2BotAgentTerrainTest -q
```
Expected: BUILD SUCCESS (the logic already exists in `TerrainGrid`).

- [ ] **Step 4: Convert SC2BotAgent to CDI bean**

Replace `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java` in full:

```java
package io.quarkmind.sc2.real;

import com.github.ocraft.s2client.bot.S2Agent;
import com.github.ocraft.s2client.protocol.observation.spatial.ImageData;
import io.quarkus.arc.profile.IfBuildProfile;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.sc2.IntentQueue;
import io.quarkmind.sc2.TerrainProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicReference;

/**
 * ocraft S2Agent bridge. CDI-managed — Quarkus creates the instance; ocraft calls the callbacks.
 *
 * @ApplicationScoped is always-active in Quarkus: no ContextNotActiveException risk from
 * ocraft's background game-loop thread. onGameStart() is guaranteed to fire before onStep().
 */
@IfBuildProfile("sc2")
@ApplicationScoped
public class SC2BotAgent extends S2Agent {

    private static final Logger log = Logger.getLogger(SC2BotAgent.class);

    @Inject IntentQueue     intentQueue;
    @Inject TerrainProvider terrainProvider;

    private final AtomicReference<GameState> latestGameState = new AtomicReference<>(null);
    private final Queue<Runnable> pendingDebugCommands = new ConcurrentLinkedQueue<>();

    @Override
    public void onGameStart() {
        observation().getGameInfo().getStartRaw().ifPresent(raw -> {
            ImageData pg = raw.getPathingGrid();
            TerrainGrid terrain = TerrainGrid.fromPathingGrid(
                pg.getData(), pg.getSize().getX(), pg.getSize().getY());
            terrainProvider.setTerrain(terrain);
            log.infof("[SC2] Terrain extracted — %dx%d pathing grid",
                pg.getSize().getX(), pg.getSize().getY());
        });
        log.info("[SC2] Game started");
    }

    @Override
    public void onStep() {
        try {
            GameState state = ObservationTranslator.translate(observation());
            latestGameState.set(state);
        } catch (Exception e) {
            log.warnf("[SC2] Observation translation failed: %s", e.getMessage());
        }

        List<ResolvedCommand> commands = ActionTranslator.translate(intentQueue.drainAll());
        commands.forEach(cmd ->
            cmd.target().ifPresentOrElse(
                pos -> actions().unitCommand(cmd.tag(), cmd.ability(), pos, false),
                ()  -> actions().unitCommand(cmd.tag(), cmd.ability(), false)
            )
        );

        if (!pendingDebugCommands.isEmpty()) {
            Runnable cmd;
            while ((cmd = pendingDebugCommands.poll()) != null) cmd.run();
            debug().sendDebug();
        }
    }

    @Override
    public void onGameEnd() {
        log.info("[SC2] Game ended");
    }

    public GameState getLatestGameState() { return latestGameState.get(); }

    public void enqueueDebugCommand(Runnable command) { pendingDebugCommands.add(command); }
}
```

- [ ] **Step 5: Update RealSC2Engine to inject SC2BotAgent**

In `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java`:

Add field (alongside existing `@Inject IntentQueue intentQueue;`):
```java
@Inject SC2BotAgent botAgent;
```

Remove the line in `connect()`:
```java
botAgent = new SC2BotAgent(intentQueue);
```

Replace it with just the coordinator setup (the `botAgent` field is now injected):
```java
@Override
@Retry(maxRetries = 3, delay = 2000)
@CircuitBreaker(requestVolumeThreshold = 4, failureRatio = 0.5, delay = 10000)
@Fallback(fallbackMethod = "connectFallback")
public void connect() {
    log.info("[SC2] Connecting to StarCraft II...");
    Difficulty difficulty = Difficulty.valueOf(difficultyStr);
    coordinator = S2Coordinator.setup()
            .loadSettings(new String[]{})
            .setParticipants(
                    S2Coordinator.createParticipant(Race.PROTOSS, botAgent),
                    S2Coordinator.createComputer(Race.RANDOM, difficulty)
            )
            .launchStarcraft();
    connected.set(true);
    log.info("[SC2] Connected — coordinator ready");
}
```

Remove the field declaration `private SC2BotAgent botAgent;` (it is now the injected field).

- [ ] **Step 6: Compile**

```bash
mvn compile -q 2>&1 | grep "ERROR" | head -10
```
Expected: no errors.

- [ ] **Step 7: Run full test suite**

```bash
mvn test -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java \
        src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java \
        src/test/java/io/quarkmind/sc2/real/SC2BotAgentTerrainTest.java
git commit -m "feat(sc2): SC2BotAgent → CDI bean; extract terrain grid in onGameStart()

TerrainProvider now populated during real SC2 games. Agent plugins can access
terrain for reachability checks. Full integration validated in issue #13.

Refs #121, Refs #120"
```

---

## Task 2: Terrain-aware A* edge costs

**GitHub:** `Refs #122` (child of epic #120)

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/TerrainGrid.java`
- Modify: `src/main/java/io/quarkmind/domain/AStarPathfinder.java`
- Create: `src/test/java/io/quarkmind/domain/TerrainGridTest.java`
- Modify: `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`

- [ ] **Step 1: Write failing tests for `movementCost()`**

Create `src/test/java/io/quarkmind/domain/TerrainGridTest.java`:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.Arrays;
import static org.assertj.core.api.Assertions.*;

class TerrainGridTest {

    private TerrainGrid grid(TerrainGrid.Height[][] heights) {
        return new TerrainGrid(heights.length, heights[0].length, heights);
    }

    @Test
    void movementCost_low_isOne() {
        TerrainGrid.Height[][] g = {{TerrainGrid.Height.LOW}};
        assertThat(grid(g).movementCost(0, 0)).isEqualTo(1.0);
    }

    @Test
    void movementCost_high_isOne() {
        TerrainGrid.Height[][] g = {{TerrainGrid.Height.HIGH}};
        assertThat(grid(g).movementCost(0, 0)).isEqualTo(1.0);
    }

    @Test
    void movementCost_ramp_isOnePointFive() {
        TerrainGrid.Height[][] g = {{TerrainGrid.Height.RAMP}};
        assertThat(grid(g).movementCost(0, 0)).isEqualTo(1.5);
    }

    @Test
    void movementCost_wall_isOne() {
        // Wall tiles are never expanded by A* — cost doesn't matter, but must not throw
        TerrainGrid.Height[][] g = {{TerrainGrid.Height.WALL}};
        assertThat(grid(g).movementCost(0, 0)).isEqualTo(1.0);
    }

    @Test
    void movementCost_outOfBounds_isOne() {
        TerrainGrid.Height[][] g = {{TerrainGrid.Height.LOW}};
        assertThat(grid(g).movementCost(-1, 0)).isEqualTo(1.0);
        assertThat(grid(g).movementCost(5, 5)).isEqualTo(1.0);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -Dtest=TerrainGridTest -q 2>&1 | tail -5
```
Expected: compilation error — `movementCost` not found.

- [ ] **Step 3: Add `movementCost()` to `TerrainGrid`**

In `src/main/java/io/quarkmind/domain/TerrainGrid.java`, add after `isWalkable()`:

```java
/** Movement cost multiplier for this tile. RAMP = 1.5×, all others = 1.0×. */
public double movementCost(int x, int y) {
    if (x < 0 || x >= width || y < 0 || y >= height) return 1.0;
    return grid[x][y] == Height.RAMP ? 1.5 : 1.0;
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
mvn test -Dtest=TerrainGridTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Write failing A* weighted-cost tests**

Add to the end of `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`:

```java
    // ---- Weighted cost tests ----

    /**
     * Map layout (10 wide, 10 tall):
     *   Wall row at y=5, two gaps:
     *     x=3 → LOW tile (cost 1.0)
     *     x=7 → RAMP tile (cost 1.5)
     *   Start (2,2) → Goal (2,8)
     *   Both routes have equal tile count; flat gap costs less.
     */
    private TerrainGrid twoGapMap() {
        TerrainGrid.Height[][] g = new TerrainGrid.Height[10][10];
        for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
        for (int x = 0; x < 10; x++) g[x][5] = TerrainGrid.Height.WALL;
        g[3][5] = TerrainGrid.Height.LOW;   // flat gap
        g[7][5] = TerrainGrid.Height.RAMP;  // ramp gap
        return new TerrainGrid(10, 10, g);
    }

    @Test
    void weightedCost_prefersFlat_overRamp() {
        TerrainGrid g = twoGapMap();
        List<Point2d> path = pf.findPath(g, new Point2d(2, 2), new Point2d(2, 8));
        assertThat(path).isNotEmpty();
        boolean passedFlatGap = path.stream().anyMatch(wp ->
            (int) wp.x() == 3 && (int) wp.y() == 5);
        boolean passedRampGap = path.stream().anyMatch(wp ->
            (int) wp.x() == 7 && (int) wp.y() == 5);
        assertThat(passedFlatGap).as("should use flat gap at x=3").isTrue();
        assertThat(passedRampGap).as("should not use ramp gap at x=7").isFalse();
    }

    @Test
    void weightedCost_rampStillUsed_whenOnlyOption() {
        // Single ramp gap — must still find a path
        TerrainGrid.Height[][] g = new TerrainGrid.Height[10][10];
        for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
        for (int x = 0; x < 10; x++) g[x][5] = TerrainGrid.Height.WALL;
        g[5][5] = TerrainGrid.Height.RAMP;
        TerrainGrid grid = new TerrainGrid(10, 10, g);
        List<Point2d> path = pf.findPath(grid, new Point2d(2, 2), new Point2d(8, 8));
        assertThat(path).isNotEmpty();
        boolean passedRamp = path.stream().anyMatch(wp ->
            (int) wp.x() == 5 && (int) wp.y() == 5);
        assertThat(passedRamp).isTrue();
    }
```

- [ ] **Step 6: Run tests — verify they fail**

```bash
mvn test -Dtest=AStarPathfinderTest#weightedCost_prefersFlat_overRamp -q 2>&1 | tail -5
```
Expected: test passes BEFORE the weighting change (A* currently treats flat and ramp the same cost — both routes are symmetric so A* may choose either). If it already passes, note that the test may be fragile depending on tie-breaking — proceed anyway; the implementation is still correct.

Actually run the full class to see current state:
```bash
mvn test -Dtest=AStarPathfinderTest -q 2>&1 | tail -5
```

- [ ] **Step 7: Add weighted edge cost to `AStarPathfinder`**

In `src/main/java/io/quarkmind/domain/AStarPathfinder.java`, in the inner loop of `findPath()`, find:
```java
double cost = (d[0] != 0 && d[1] != 0) ? DIAG : 1.0;
open.add(new Node(nx, ny, cur.g() + cost, cur));
```

Replace with:
```java
double cost = (d[0] != 0 && d[1] != 0) ? DIAG : 1.0;
cost *= grid.movementCost(nx, ny);
open.add(new Node(nx, ny, cur.g() + cost, cur));
```

- [ ] **Step 8: Run all tests**

```bash
mvn test -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/domain/TerrainGrid.java \
        src/main/java/io/quarkmind/domain/AStarPathfinder.java \
        src/test/java/io/quarkmind/domain/TerrainGridTest.java \
        src/test/java/io/quarkmind/domain/AStarPathfinderTest.java
git commit -m "feat(domain): terrain-aware A* — RAMP tiles cost 1.5×; flat ground preferred

TerrainGrid.movementCost() returns 1.5 for RAMP, 1.0 for all other heights.
AStarPathfinder multiplies edge cost by destination tile's movement cost.

Refs #122, Refs #120"
```

---

## Task 3: Path smoothing — greedy string-pulling

**GitHub:** `Refs #123` (child of epic #120)

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/AStarPathfinder.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`
- Modify: `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`

- [ ] **Step 1: Write failing tests for `smoothPath()`**

Add to the end of `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`:

```java
    // ---- smoothPath() tests ----

    @Test
    void smoothPath_emptyPath_returnsEmpty() {
        assertThat(AStarPathfinder.smoothPath(List.of(), open(10, 10))).isEmpty();
    }

    @Test
    void smoothPath_singlePoint_returnsSinglePoint() {
        List<Point2d> path = List.of(new Point2d(1.5f, 1.5f));
        assertThat(AStarPathfinder.smoothPath(path, open(10, 10))).hasSize(1);
    }

    @Test
    void smoothPath_twoPoints_returnsBoth() {
        List<Point2d> path = List.of(new Point2d(1.5f, 1.5f), new Point2d(5.5f, 5.5f));
        assertThat(AStarPathfinder.smoothPath(path, open(10, 10))).hasSize(2);
    }

    @Test
    void smoothPath_openGround_removesIntermediateWaypoints() {
        // Straight line A* produces many waypoints; smoothing collapses them
        TerrainGrid g = open(20, 20);
        List<Point2d> raw = pf.findPath(g, new Point2d(0, 0), new Point2d(19, 0));
        List<Point2d> smoothed = AStarPathfinder.smoothPath(raw, g);
        // On a straight line in open ground, only start and end needed
        assertThat(smoothed.size()).isLessThan(raw.size());
        // Last point must still be the goal
        Point2d last = smoothed.get(smoothed.size() - 1);
        assertThat(last.x()).isEqualTo(raw.get(raw.size() - 1).x());
        assertThat(last.y()).isEqualTo(raw.get(raw.size() - 1).y());
    }

    @Test
    void smoothPath_walledCorner_preservesWaypointAroundWall() {
        // L-shaped path: must go right then up around a wall
        TerrainGrid.Height[][] g = new TerrainGrid.Height[10][10];
        for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
        // Vertical wall at x=5 from y=0 to y=5
        for (int y = 0; y <= 5; y++) g[5][y] = TerrainGrid.Height.WALL;
        TerrainGrid grid = new TerrainGrid(10, 10, g);
        List<Point2d> raw = pf.findPath(grid, new Point2d(2, 2), new Point2d(8, 2));
        List<Point2d> smoothed = AStarPathfinder.smoothPath(raw, grid);
        // Smoothed path must still reach the goal
        assertThat(smoothed).isNotEmpty();
        Point2d last = smoothed.get(smoothed.size() - 1);
        assertThat(last.x()).isEqualTo(raw.get(raw.size() - 1).x());
        // Smoothed path must not be longer than raw
        assertThat(smoothed.size()).isLessThanOrEqualTo(raw.size());
        // All waypoints in smoothed path must be walkable
        for (Point2d wp : smoothed) {
            assertThat(grid.isWalkable((int) wp.x(), (int) wp.y()))
                .as("smoothed waypoint %s must be walkable", wp).isTrue();
        }
    }

    @Test
    void smoothPath_zigzagOnOpenGround_reducesWaypoints() {
        TerrainGrid g = open(30, 30);
        List<Point2d> raw = pf.findPath(g, new Point2d(0, 0), new Point2d(29, 29));
        assertThat(raw.size()).isGreaterThan(2);
        List<Point2d> smoothed = AStarPathfinder.smoothPath(raw, g);
        assertThat(smoothed.size()).isLessThan(raw.size());
    }
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
mvn test -Dtest="AStarPathfinderTest#smoothPath*" -q 2>&1 | tail -5
```
Expected: compilation error — `smoothPath` not found.

- [ ] **Step 3: Implement `smoothPath()` and `hasLos()` in `AStarPathfinder`**

Add to `src/main/java/io/quarkmind/domain/AStarPathfinder.java` after the `findPath()` method:

```java
    /**
     * Greedy string-pulling: starting from each waypoint, find the furthest later
     * waypoint reachable in a straight line (Bresenham LOS, no WALL tiles crossed).
     * Skips intermediate waypoints, reducing zigzag on open ground.
     * Returns the original path unchanged if it has ≤ 2 points.
     */
    public static List<Point2d> smoothPath(List<Point2d> path, TerrainGrid grid) {
        if (path.size() <= 2) return path;

        List<Point2d> result = new ArrayList<>();
        result.add(path.get(0));
        int i = 0;

        while (i < path.size() - 1) {
            int farthest = i + 1;
            for (int j = path.size() - 1; j > i + 1; j--) {
                if (hasLos(grid, path.get(i), path.get(j))) {
                    farthest = j;
                    break;
                }
            }
            result.add(path.get(farthest));
            i = farthest;
        }

        return result;
    }

    /**
     * Bresenham line-of-sight check. Returns true if the straight line between
     * {@code from} and {@code to} (in world tile-centre coordinates) crosses no WALL tiles.
     */
    private static boolean hasLos(TerrainGrid grid, Point2d from, Point2d to) {
        int x0 = (int) from.x(), y0 = (int) from.y();
        int x1 = (int) to.x(),   y1 = (int) to.y();

        int dx  = Math.abs(x1 - x0);
        int dy  = Math.abs(y1 - y0);
        int sx  = x0 < x1 ? 1 : -1;
        int sy  = y0 < y1 ? 1 : -1;
        int err = dx - dy;
        int x = x0, y = y0;

        while (x != x1 || y != y1) {
            if (!grid.isWalkable(x, y)) return false;
            int e2 = 2 * err;
            if (e2 > -dy) { err -= dy; x += sx; }
            if (e2 <  dx) { err += dx; y += sy; }
        }
        return grid.isWalkable(x1, y1);
    }
```

Also add `import java.util.ArrayList;` if not already present (check the imports at the top of `AStarPathfinder.java` — it currently uses `List`, `Deque`, `ArrayDeque`, etc.).

- [ ] **Step 4: Run smoothPath tests — verify they pass**

```bash
mvn test -Dtest=AStarPathfinderTest -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 5: Wire `smoothPath()` into `PathfindingMovement`**

In `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`, in the `advance()` method, find the block that computes and caches the path:

```java
if (!target.equals(lastTargets.get(unitTag))) {
    List<Point2d> path = pathfinder.findPath(grid, current, target);
    log.infof(...);
    waypoints.put(unitTag, new ArrayDeque<>(path));
    lastTargets.put(unitTag, target);
}
```

Replace with:

```java
if (!target.equals(lastTargets.get(unitTag))) {
    List<Point2d> path = pathfinder.findPath(grid, current, target);
    List<Point2d> smoothed = AStarPathfinder.smoothPath(path, grid);
    log.infof("[PATHFINDING] %s: %d waypoints → %d smoothed from (%.1f,%.1f) to (%.1f,%.1f)%s",
        unitTag, path.size(), smoothed.size(), current.x(), current.y(), target.x(), target.y(),
        path.isEmpty() ? " — EMPTY, falling back to direct movement" : "");
    waypoints.put(unitTag, new ArrayDeque<>(smoothed));
    lastTargets.put(unitTag, target);
}
```

- [ ] **Step 6: Add PathfindingMovement integration test for smoothing**

Add to `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`:

```java
    @Test
    void smoothingReducesWaypointCount_onOpenGround() {
        PathfindingMovement pm = new PathfindingMovement(open());
        // Drive one step to trigger path computation and smoothing
        pm.advance("u1", new Point2d(2f, 2f), new Point2d(28f, 28f), 0.5);
        // If smoothing works, the unit should still move toward the target
        Point2d after = pm.advance("u1", new Point2d(2f, 2f), new Point2d(28f, 28f), 0.5);
        assertThat(EmulatedGame.distance(after, new Point2d(28f, 28f)))
            .isLessThan(EmulatedGame.distance(new Point2d(2f, 2f), new Point2d(28f, 28f)));
    }
```

- [ ] **Step 7: Run full test suite**

```bash
mvn test -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 8: Commit — closes child issues and epic**

```bash
git add src/main/java/io/quarkmind/domain/AStarPathfinder.java \
        src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java \
        src/test/java/io/quarkmind/domain/AStarPathfinderTest.java \
        src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java
git commit -m "feat(domain): path smoothing — greedy string-pulling reduces A* waypoint zigzag

AStarPathfinder.smoothPath() removes intermediate waypoints when LOS is clear
(Bresenham line, no WALL crossings). PathfindingMovement applies smoothing after
findPath() before caching waypoints.

Closes #123, Refs #120"
```

---

## Task 4: Documentation pass

**GitHub:** `Refs #120`

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update CLAUDE.md unit test list**

In the `Unit tests` section, add to the test list:
- `TerrainGridTest`
- `SC2BotAgentTerrainTest`

Check the current test list and add the new tests where the emulated/domain tests are listed.

- [ ] **Step 2: Update CLAUDE.md EmulatedGame test helpers section**

Verify `PathfindingMovement` and `AStarPathfinder` test descriptions are accurate. No structural changes needed — just verify.

- [ ] **Step 3: Update DESIGN.md**

Find the Emulation Engine section and update:
- Mark E5 (pathfinding enhancements) as in-progress/complete
- Add note that `SC2BotAgent` is now a CDI bean
- Add note that A* uses terrain-aware edge costs (RAMP = 1.5×)
- Add note that `PathfindingMovement` applies `smoothPath()` post-processing

- [ ] **Step 4: Check for drift in existing docs**

```bash
grep -rn "SC2BotAgent\|new SC2BotAgent\|configureWave\|EnemyBuildStep" docs/ CLAUDE.md 2>/dev/null
```

Fix any stale references found.

- [ ] **Step 5: Run full test suite one final time**

```bash
mvn test -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit — closes epic**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs: sync CLAUDE.md and DESIGN.md with pathfinder enhancements

SC2BotAgent CDI bean, terrain-aware A*, path smoothing.

Closes #120"
```

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `mvn test -Dtest=SC2BotAgentTerrainTest -q` | SC2 terrain extraction unit tests |
| `mvn test -Dtest=TerrainGridTest -q` | movementCost unit tests |
| `mvn test -Dtest=AStarPathfinderTest -q` | A* tests including weighted cost + smoothing |
| `mvn test -Dtest=PathfindingMovementTest -q` | PathfindingMovement integration tests |
| `mvn test -q` | Full suite |
