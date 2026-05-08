# E8: Terrain Height Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `WalkabilityGrid` with `TerrainGrid` (HIGH/LOW/RAMP/WALL), wire a 25% miss chance for ranged attacks from low to high ground, and add topographic height shading to the visualizer.

**Architecture:** `TerrainGrid` replaces `WalkabilityGrid` everywhere — same `isWalkable()` contract, adds `heightAt()`. Miss check is a guard in `resolveCombat()` before `DamageCalculator` is called. Visualizer fetches the extended terrain API and draws a static shaded layer.

**Tech Stack:** Java 21, JUnit 5, AssertJ, RestAssured, PixiJS 8, Playwright (E2E)

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `src/main/java/io/quarkmind/domain/WalkabilityGrid.java` | **Delete** | Replaced by TerrainGrid |
| `src/main/java/io/quarkmind/domain/TerrainGrid.java` | **Create** | Height enum + grid, emulatedMap(), fromPathingGrid() |
| `src/main/java/io/quarkmind/domain/AStarPathfinder.java` | **Modify** | WalkabilityGrid → TerrainGrid parameter types |
| `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java` | **Modify** | WalkabilityGrid → TerrainGrid field/constructor |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | **Modify** | terrainGrid field, setTerrainGrid(), random field, setRandomForTesting(), missesHighGround(), resolveCombat() miss guard |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java` | **Modify** | joinGame() uses TerrainGrid.emulatedMap() |
| `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java` | **Modify** | TerrainResponse adds highGround + ramps, uses TerrainGrid |
| `src/main/resources/META-INF/resources/visualizer.js` | **Modify** | loadTerrain() draws HIGH/RAMP shading |
| `src/test/java/io/quarkmind/domain/WalkabilityGridTest.java` | **Delete** | Superseded by TerrainGridTest |
| `src/test/java/io/quarkmind/domain/TerrainGridTest.java` | **Create** | Unit tests for TerrainGrid |
| `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java` | **Modify** | Helper methods: boolean[][] → Height[][] |
| `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java` | **Modify** | Helper methods: boolean[][] → Height[][] |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | **Modify** | E7 tests: WalkabilityGrid → TerrainGrid; add 6 E8 miss-chance tests |
| `src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java` | **Modify** | Add highGround + ramps assertions |
| `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` | **Modify** | Add terrain shading pixel-sampling test |
| `CLAUDE.md` | **Modify** | Update test helper: setWalkabilityGrid → setTerrainGrid |

---

## Task 0: Create GitHub issue for E8

**Files:** none (GitHub only)

- [ ] **Step 1: Create the GitHub issue**

```bash
gh issue create \
  --title "E8: Terrain height — TerrainGrid, miss chance, visualizer shading" \
  --body "$(cat <<'EOF'
## What

Replace binary WalkabilityGrid with TerrainGrid (HIGH/LOW/RAMP/WALL).
Wire 25% miss chance for ranged attacks from low to high ground.
Add topographic height shading to the visualizer.

## Spec

docs/superpowers/specs/2026-04-16-e8-terrain-height-design.md

## Map layout

- y > 18: HIGH ground (enemy staging side)
- y < 18: LOW ground (nexus/defender side)
- y = 18, x=11–13: RAMP
- y = 18, x=0–10 and x=14–63: WALL

## Out of scope

- Vision / fog of war (dedicated epic)
- Ramp directionality (not SC2 canon)
EOF
)"
```

- [ ] **Step 2: Note the issue number**

The command prints a URL like `https://github.com/mdproctor/quarkmind/issues/42`.
Use that number (e.g. `42`) in all `Refs #N` trailers in subsequent commits.

---

## Task 1: TerrainGrid — TDD

**Files:**
- Create: `src/main/java/io/quarkmind/domain/TerrainGrid.java`
- Create: `src/test/java/io/quarkmind/domain/TerrainGridTest.java`

- [ ] **Step 1: Write the failing tests**

Create `src/test/java/io/quarkmind/domain/TerrainGridTest.java`:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TerrainGridTest {

    // ---- isWalkable — same contract as WalkabilityGrid ----

    @Test void emulatedMapNexusTileIsWalkable() {
        assertThat(TerrainGrid.emulatedMap().isWalkable(8, 8)).isTrue();
    }
    @Test void emulatedMapStagingTileIsWalkable() {
        assertThat(TerrainGrid.emulatedMap().isWalkable(26, 26)).isTrue();
    }
    @Test void emulatedMapWallTileIsBlocked() {
        assertThat(TerrainGrid.emulatedMap().isWalkable(20, 18)).isFalse();
    }
    @Test void emulatedMapChokeGapIsWalkable() {
        assertThat(TerrainGrid.emulatedMap().isWalkable(12, 18)).isTrue();
    }
    @Test void emulatedMapChokeEdgesAreBlocked() {
        assertThat(TerrainGrid.emulatedMap().isWalkable(10, 18)).isFalse();
        assertThat(TerrainGrid.emulatedMap().isWalkable(14, 18)).isFalse();
    }
    @Test void outOfBoundsIsNotWalkable() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.isWalkable(-1,  0)).isFalse();
        assertThat(g.isWalkable(64,  0)).isFalse();
        assertThat(g.isWalkable( 0, 64)).isFalse();
    }

    // ---- heightAt ----

    @Test void emulatedMapHighGroundCorrect() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.heightAt(0, 19)).isEqualTo(TerrainGrid.Height.HIGH);
        assertThat(g.heightAt(26, 26)).isEqualTo(TerrainGrid.Height.HIGH);
    }
    @Test void emulatedMapLowGroundCorrect() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.heightAt(8, 8)).isEqualTo(TerrainGrid.Height.LOW);
        assertThat(g.heightAt(0, 17)).isEqualTo(TerrainGrid.Height.LOW);
    }
    @Test void emulatedMapRampCorrect() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.heightAt(11, 18)).isEqualTo(TerrainGrid.Height.RAMP);
        assertThat(g.heightAt(12, 18)).isEqualTo(TerrainGrid.Height.RAMP);
        assertThat(g.heightAt(13, 18)).isEqualTo(TerrainGrid.Height.RAMP);
    }
    @Test void emulatedMapWallsCorrect() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.heightAt(10, 18)).isEqualTo(TerrainGrid.Height.WALL);
        assertThat(g.heightAt(14, 18)).isEqualTo(TerrainGrid.Height.WALL);
        assertThat(g.heightAt( 0, 18)).isEqualTo(TerrainGrid.Height.WALL);
    }
    @Test void isWalkableMatchesHeight() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.isWalkable( 0, 19)).isTrue();   // HIGH → walkable
        assertThat(g.isWalkable( 8,  8)).isTrue();   // LOW  → walkable
        assertThat(g.isWalkable(12, 18)).isTrue();   // RAMP → walkable
        assertThat(g.isWalkable(20, 18)).isFalse();  // WALL → blocked
    }
    @Test void outOfBoundsHeightIsWall() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.heightAt(-1,  0)).isEqualTo(TerrainGrid.Height.WALL);
        assertThat(g.heightAt(64,  0)).isEqualTo(TerrainGrid.Height.WALL);
    }

    // ---- dimensions ----

    @Test void widthAndHeightCorrect() {
        TerrainGrid g = TerrainGrid.emulatedMap();
        assertThat(g.width()).isEqualTo(64);
        assertThat(g.height()).isEqualTo(64);
    }

    // ---- fromPathingGrid ----

    @Test void fromPathingGrid_walkableTileIsLow() {
        TerrainGrid g = TerrainGrid.fromPathingGrid(new byte[]{(byte) 0xFF}, 8, 1);
        assertThat(g.heightAt(0, 0)).isEqualTo(TerrainGrid.Height.LOW);
        assertThat(g.isWalkable(0, 0)).isTrue();
    }
    @Test void fromPathingGrid_nonWalkableTileIsWall() {
        TerrainGrid g = TerrainGrid.fromPathingGrid(new byte[]{(byte) 0x00}, 8, 1);
        assertThat(g.heightAt(0, 0)).isEqualTo(TerrainGrid.Height.WALL);
        assertThat(g.isWalkable(0, 0)).isFalse();
    }
    @Test void fromPathingGrid_mixedBitsDecodeCorrectly() {
        // 0xB2 = 1011_0010: bit7=1(walk), bit6=0(wall), bit0=0(wall)
        TerrainGrid g = TerrainGrid.fromPathingGrid(new byte[]{(byte) 0xB2}, 8, 1);
        assertThat(g.isWalkable(0, 0)).isTrue();   // bit 7 = 1
        assertThat(g.isWalkable(1, 0)).isFalse();  // bit 6 = 0
        assertThat(g.isWalkable(7, 0)).isFalse();  // bit 0 = 0
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure (TerrainGrid doesn't exist yet)**

```bash
mvn test -Dtest=TerrainGridTest -q 2>&1 | head -20
```

Expected: compilation error mentioning `TerrainGrid`.

- [ ] **Step 3: Implement TerrainGrid**

Create `src/main/java/io/quarkmind/domain/TerrainGrid.java`:

```java
package io.quarkmind.domain;

import java.util.Arrays;

public final class TerrainGrid {

    public enum Height { HIGH, LOW, RAMP, WALL }

    private final Height[][] grid;  // [x][y]
    private final int width;
    private final int height;

    public TerrainGrid(int width, int height, Height[][] grid) {
        this.width  = width;
        this.height = height;
        this.grid   = new Height[width][height];
        for (int x = 0; x < width; x++) {
            this.grid[x] = Arrays.copyOf(grid[x], height);
        }
    }

    public Height heightAt(int x, int y) {
        if (x < 0 || x >= width || y < 0 || y >= height) return Height.WALL;
        return grid[x][y];
    }

    public boolean isWalkable(int x, int y) {
        return heightAt(x, y) != Height.WALL;
    }

    public int width()  { return width; }
    public int height() { return height; }

    /**
     * 64×64 emulated map.
     * y > 18: HIGH ground (enemy staging side, visually top of screen)
     * y < 18: LOW ground (nexus/defender side, visually bottom)
     * y = 18, x = 11–13: RAMP (chokepoint gap)
     * y = 18, x = 0–10 and x = 14–63: WALL
     */
    public static TerrainGrid emulatedMap() {
        Height[][] grid = new Height[64][64];
        for (Height[] col : grid) Arrays.fill(col, Height.LOW);
        for (int x = 0; x < 64; x++) {
            for (int y = 19; y < 64; y++) {
                grid[x][y] = Height.HIGH;
            }
        }
        for (int x = 0; x < 64; x++) {
            grid[x][18] = (x >= 11 && x <= 13) ? Height.RAMP : Height.WALL;
        }
        return new TerrainGrid(64, 64, grid);
    }

    /**
     * Constructs a TerrainGrid from ocraft's PathingGrid bitmap.
     * All walkable tiles return LOW; height data integration deferred to real SC2 phase.
     * Bit encoding: index = x + y*width; bit = (data[index/8] >> (7 - index%8)) & 1
     */
    public static TerrainGrid fromPathingGrid(byte[] data, int width, int height) {
        Height[][] grid = new Height[width][height];
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                int index = x + y * width;
                boolean walkable = ((data[index / 8] >> (7 - index % 8)) & 1) == 1;
                grid[x][y] = walkable ? Height.LOW : Height.WALL;
            }
        }
        return new TerrainGrid(width, height, grid);
    }
}
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
mvn test -Dtest=TerrainGridTest -q
```

Expected: `BUILD SUCCESS`, 16 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/domain/TerrainGrid.java \
        src/test/java/io/quarkmind/domain/TerrainGridTest.java
git commit -m "$(cat <<'EOF'
feat(e8): TerrainGrid — HIGH/LOW/RAMP/WALL height model

Replaces WalkabilityGrid (deleted in next task). Maintains isWalkable()
contract; adds heightAt() for height-aware mechanics. emulatedMap() now
encodes y>18 as HIGH, y=18 gap as RAMP, y<18 as LOW.

Refs #N
EOF
)"
```

---

## Task 2: Migrate WalkabilityGrid → TerrainGrid everywhere

**Files:**
- Delete: `src/main/java/io/quarkmind/domain/WalkabilityGrid.java`
- Modify: `src/main/java/io/quarkmind/domain/AStarPathfinder.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (field + setter only)
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java`
- Modify: `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java` (import only — response shape in Task 5)
- Delete: `src/test/java/io/quarkmind/domain/WalkabilityGridTest.java`
- Modify: `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Update AStarPathfinder.java — change WalkabilityGrid to TerrainGrid**

In `src/main/java/io/quarkmind/domain/AStarPathfinder.java`, make two changes:

Change `findPath` signature (line 18):
```java
// BEFORE:
public List<Point2d> findPath(WalkabilityGrid grid, Point2d from, Point2d to) {

// AFTER:
public List<Point2d> findPath(TerrainGrid grid, Point2d from, Point2d to) {
```

Change `nearestWalkable` signature (line 63):
```java
// BEFORE:
private static int[] nearestWalkable(WalkabilityGrid grid, int x, int y) {

// AFTER:
private static int[] nearestWalkable(TerrainGrid grid, int x, int y) {
```

No import needed — both classes are in `io.quarkmind.domain`.

- [ ] **Step 2: Update PathfindingMovement.java**

In `src/main/java/io/quarkmind/sc2/emulated/PathfindingMovement.java`:

```java
// BEFORE (lines 5, 15, 20):
import io.quarkmind.domain.WalkabilityGrid;
...
private final WalkabilityGrid grid;
...
public PathfindingMovement(WalkabilityGrid grid) { this.grid = grid; }

// AFTER:
import io.quarkmind.domain.TerrainGrid;
...
private final TerrainGrid grid;
...
public PathfindingMovement(TerrainGrid grid) { this.grid = grid; }
```

- [ ] **Step 3: Update EmulatedGame.java — field and setter only**

In `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`:

```java
// BEFORE (line 57):
private WalkabilityGrid walkabilityGrid = null;

// AFTER:
private TerrainGrid terrainGrid = null;
```

```java
// BEFORE (line 557):
void setWalkabilityGrid(WalkabilityGrid g)   { this.walkabilityGrid  = g; }

// AFTER:
void setTerrainGrid(TerrainGrid g) { this.terrainGrid = g; }
```

Also update `enforceWall()` to use `terrainGrid` instead of `walkabilityGrid`:
```java
// BEFORE (lines 120-130):
private Point2d enforceWall(String unitTag, Point2d proposed, Point2d current) {
    if (walkabilityGrid == null) return proposed;
    int tx = (int) proposed.x();
    int ty = (int) proposed.y();
    if (!walkabilityGrid.isWalkable(tx, ty)) {
        log.warnf("[PHYSICS] Wall collision blocked %s at (%.2f,%.2f) tile(%d,%d) — invalidating path",
            unitTag, proposed.x(), proposed.y(), tx, ty);
        movementStrategy.invalidatePath(unitTag);
        return current;
    }
    return proposed;
}

// AFTER:
private Point2d enforceWall(String unitTag, Point2d proposed, Point2d current) {
    if (terrainGrid == null) return proposed;
    int tx = (int) proposed.x();
    int ty = (int) proposed.y();
    if (!terrainGrid.isWalkable(tx, ty)) {
        log.warnf("[PHYSICS] Wall collision blocked %s at (%.2f,%.2f) tile(%d,%d) — invalidating path",
            unitTag, proposed.x(), proposed.y(), tx, ty);
        movementStrategy.invalidatePath(unitTag);
        return current;
    }
    return proposed;
}
```

Remove the `import io.quarkmind.domain.WalkabilityGrid;` line (if present as explicit import — the file uses `import io.quarkmind.domain.*;` so no change needed).

- [ ] **Step 4: Update EmulatedEngine.java**

In `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java`:

```java
// BEFORE (lines 8, 55, 56, 57):
import io.quarkmind.domain.WalkabilityGrid;
...
WalkabilityGrid grid = WalkabilityGrid.emulatedMap();
game.setMovementStrategy(new PathfindingMovement(grid));
game.setWalkabilityGrid(grid);

// AFTER:
import io.quarkmind.domain.TerrainGrid;
...
TerrainGrid grid = TerrainGrid.emulatedMap();
game.setMovementStrategy(new PathfindingMovement(grid));
game.setTerrainGrid(grid);
```

- [ ] **Step 5: Update EmulatedTerrainResource.java — import only**

In `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java`:

```java
// BEFORE (line 3):
import io.quarkmind.domain.WalkabilityGrid;

// AFTER:
import io.quarkmind.domain.TerrainGrid;
```

Also update the `getTerrain()` body (line 23):
```java
// BEFORE:
WalkabilityGrid grid = WalkabilityGrid.emulatedMap();

// AFTER:
TerrainGrid grid = TerrainGrid.emulatedMap();
```

Leave the response record unchanged for now — full response shape is done in Task 5.

- [ ] **Step 6: Delete WalkabilityGrid.java**

```bash
git rm src/main/java/io/quarkmind/domain/WalkabilityGrid.java
```

- [ ] **Step 7: Delete WalkabilityGridTest.java**

```bash
git rm src/test/java/io/quarkmind/domain/WalkabilityGridTest.java
```

- [ ] **Step 8: Update AStarPathfinderTest.java — rewrite helper methods**

In `src/test/java/io/quarkmind/domain/AStarPathfinderTest.java`, replace the two helper methods and all `WalkabilityGrid` references:

```java
// BEFORE — line 4 (add Arrays import if missing):
import java.util.Arrays;

// BEFORE — lines 12-25 (two helpers):
private WalkabilityGrid open(int w, int h) {
    boolean[][] g = new boolean[w][h];
    for (boolean[] col : g) Arrays.fill(col, true);
    return new WalkabilityGrid(w, h, g);
}

private WalkabilityGrid withWall(int w, int h, int wallY, int gapX) {
    boolean[][] g = new boolean[w][h];
    for (boolean[] col : g) Arrays.fill(col, true);
    for (int x = 0; x < w; x++) {
        if (x != gapX) g[x][wallY] = false;
    }
    return new WalkabilityGrid(w, h, g);
}

// AFTER:
private TerrainGrid open(int w, int h) {
    TerrainGrid.Height[][] g = new TerrainGrid.Height[w][h];
    for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
    return new TerrainGrid(w, h, g);
}

private TerrainGrid withWall(int w, int h, int wallY, int gapX) {
    TerrainGrid.Height[][] g = new TerrainGrid.Height[w][h];
    for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
    for (int x = 0; x < w; x++) {
        if (x != gapX) g[x][wallY] = TerrainGrid.Height.WALL;
    }
    return new TerrainGrid(w, h, g);
}
```

Replace the inline `WalkabilityGrid` construction in `unreachableGoal_returnsEmpty` (lines 74-78):
```java
// BEFORE:
boolean[][] g = new boolean[10][10];
for (boolean[] col : g) Arrays.fill(col, true);
for (int y = 0; y < 10; y++) g[5][y] = false;
WalkabilityGrid blocked = new WalkabilityGrid(10, 10, g);

// AFTER:
TerrainGrid.Height[][] g = new TerrainGrid.Height[10][10];
for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
for (int y = 0; y < 10; y++) g[5][y] = TerrainGrid.Height.WALL;
TerrainGrid blocked = new TerrainGrid(10, 10, g);
```

Replace the 3 `WalkabilityGrid.emulatedMap()` calls in the emulatedMap tests (lines 82, 88, 100):
```java
// BEFORE:
WalkabilityGrid g = WalkabilityGrid.emulatedMap();

// AFTER (all three occurrences):
TerrainGrid g = TerrainGrid.emulatedMap();
```

Update the two test methods that have `WalkabilityGrid g = open(10, 10)` (lines 33, 49, 61):
```java
// BEFORE:
WalkabilityGrid g = open(10, 10);   // line 33
WalkabilityGrid g = withWall(10, 10, 5, 5);  // lines 49, 61

// AFTER:
TerrainGrid g = open(10, 10);
TerrainGrid g = withWall(10, 10, 5, 5);
```

- [ ] **Step 9: Update PathfindingMovementTest.java**

In `src/test/java/io/quarkmind/sc2/emulated/PathfindingMovementTest.java`:

```java
// BEFORE — helper (lines 10-14):
private WalkabilityGrid open() {
    boolean[][] g = new boolean[32][32];
    for (boolean[] col : g) Arrays.fill(col, true);
    return new WalkabilityGrid(32, 32, g);
}

// AFTER:
private TerrainGrid open() {
    TerrainGrid.Height[][] g = new TerrainGrid.Height[32][32];
    for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
    return new TerrainGrid(32, 32, g);
}
```

Replace the inline grid construction in `unreachableTargetFallsBackToStepToward` (lines 60-63):
```java
// BEFORE:
boolean[][] g = new boolean[10][10];
for (boolean[] col : g) Arrays.fill(col, true);
for (int y = 0; y < 10; y++) g[5][y] = false; // vertical wall — right side unreachable
PathfindingMovement pm = new PathfindingMovement(new WalkabilityGrid(10, 10, g));

// AFTER:
TerrainGrid.Height[][] g = new TerrainGrid.Height[10][10];
for (TerrainGrid.Height[] col : g) Arrays.fill(col, TerrainGrid.Height.LOW);
for (int y = 0; y < 10; y++) g[5][y] = TerrainGrid.Height.WALL;
PathfindingMovement pm = new PathfindingMovement(new TerrainGrid(10, 10, g));
```

- [ ] **Step 10: Update EmulatedGameTest.java — four E7 references**

In `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`:

Line 820:
```java
// BEFORE:
game.setMovementStrategy(new PathfindingMovement(WalkabilityGrid.emulatedMap()));
// AFTER:
game.setMovementStrategy(new PathfindingMovement(TerrainGrid.emulatedMap()));
```

Line 833:
```java
// BEFORE:
game.setMovementStrategy(new PathfindingMovement(WalkabilityGrid.emulatedMap()));
// AFTER:
game.setMovementStrategy(new PathfindingMovement(TerrainGrid.emulatedMap()));
```

Line 858:
```java
// BEFORE:
game.setWalkabilityGrid(WalkabilityGrid.emulatedMap());
// AFTER:
game.setTerrainGrid(TerrainGrid.emulatedMap());
```

Line 878:
```java
// BEFORE:
game.setMovementStrategy(new PathfindingMovement(WalkabilityGrid.emulatedMap()));
// AFTER:
game.setMovementStrategy(new PathfindingMovement(TerrainGrid.emulatedMap()));
```

- [ ] **Step 11: Build and run all tests**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`. All previously passing tests still pass. Count should match pre-E8 total (368).

- [ ] **Step 12: Commit**

```bash
git add -u
git commit -m "$(cat <<'EOF'
refactor(e8): migrate WalkabilityGrid → TerrainGrid across all call sites

Deletes WalkabilityGrid and WalkabilityGridTest. All 6 production files and
4 test files updated. isWalkable() contract unchanged — A* and wall physics
continue to work correctly. TerrainGrid.emulatedMap() encodes height levels.

Refs #N
EOF
)"
```

---

## Task 3: EmulatedGame — miss chance (TDD)

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Write six failing miss-chance tests**

Add the following section at the end of `EmulatedGameTest.java` (after the existing E7 tests):

```java
// ---- E8: high-ground miss chance ----

@Test
void rangedAttackLowToHighMissesWhenRngSaysNo() {
    // Stalker (friendly, range=5) on LOW (y=14), enemy Zealot on HIGH (y=19).
    // Distance = 5 tiles = attack range → attack fires.
    // Always-miss RNG (nextDouble()=0.0 < 0.25) → every ranged low→high attack misses.
    game.setTerrainGrid(TerrainGrid.emulatedMap());
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 0.0; } // always < 0.25 → miss
    });
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(5, 14));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(5, 19));
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(5, 19)));

    game.tick(); // Stalker fires but misses — no damage to Zealot

    Unit zealot = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    assertThat(zealot.shields()).isEqualTo(SC2Data.maxShields(UnitType.ZEALOT)); // 50 — untouched
}

@Test
void rangedAttackLowToHighHitsWhenRngSaysYes() {
    // Same positions, never-miss RNG (nextDouble()=1.0 ≥ 0.25) → attack lands.
    game.setTerrainGrid(TerrainGrid.emulatedMap());
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 1.0; } // never < 0.25 → hit
    });
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(5, 14));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(5, 19));
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(5, 19)));

    game.tick();

    Unit zealot = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    // Stalker vs Zealot: 13 base + 0 bonus (Zealot is LIGHT not ARMORED) - 1 armour = 12
    assertThat(zealot.shields()).isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - 12); // 38
}

@Test
void meleeAttackLowToHighNeverMisses() {
    // Zealot (friendly, range=0.5 ≤ 1.0 → melee) should never invoke the miss check.
    // Custom 2-tile TerrainGrid: tile(0,0)=LOW, tile(1,0)=HIGH.
    // Zealot at (0.6, 0) → floor → tile(0,0)=LOW.
    // Enemy Marine at (1.1, 0) → floor → tile(1,0)=HIGH.
    // Distance = 0.5 tiles ≤ Zealot range 0.5 → attack fires.
    // Always-miss RNG → melee must still deal full damage (range check skips the miss roll).
    TerrainGrid.Height[][] heights = new TerrainGrid.Height[2][1];
    heights[0][0] = TerrainGrid.Height.LOW;
    heights[1][0] = TerrainGrid.Height.HIGH;
    game.setTerrainGrid(new TerrainGrid(2, 1, heights));
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 0.0; } // always-miss — must be ignored for melee
    });
    String zealotTag = game.spawnFriendlyForTesting(UnitType.ZEALOT, new Point2d(0.6f, 0));
    game.spawnEnemyForTesting(UnitType.MARINE, new Point2d(1.1f, 0));
    game.applyIntent(new AttackIntent(zealotTag, new Point2d(1.1f, 0)));

    game.tick();

    Unit marine = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.MARINE)
        .findFirst().orElseThrow();
    // Zealot vs Marine: 8 base + 0 bonus - 0 armour = 8. Marine max shields = 0, HP 45 → 37.
    assertThat(marine.health()).isEqualTo(SC2Data.maxHealth(UnitType.MARINE) - 8); // 37
}

@Test
void rangedAttackEqualHeightNeverMisses() {
    // Both attacker and target on LOW — no miss check regardless of RNG.
    // Stalker (LOW, y=9) vs enemy Zealot (LOW, y=14). Distance = 5 = range.
    game.setTerrainGrid(TerrainGrid.emulatedMap());
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 0.0; } // would always miss if check fires
    });
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(5, 9));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(5, 14));
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(5, 14)));

    game.tick();

    Unit zealot = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    assertThat(zealot.shields()).isLessThan(SC2Data.maxShields(UnitType.ZEALOT)); // took damage
}

@Test
void rangedAttackHighToLowNeverMisses() {
    // Enemy Stalker on HIGH (y=19) attacks friendly on LOW (y=14). No miss — shooting downhill.
    // Distance from (5,19) to (5,14) = 5 ≤ Stalker range 5 → attacks fire.
    game.setTerrainGrid(TerrainGrid.emulatedMap());
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 0.0; } // would miss if check fires
    });
    game.spawnEnemyForTesting(UnitType.STALKER, new Point2d(5, 19));
    game.spawnFriendlyForTesting(UnitType.PROBE, new Point2d(5, 14));

    game.tick(); // enemy Stalker auto-attacks nearest friendly — no miss check for high→low

    // The newly spawned probe (not probe-0 at y=9) should take damage.
    // probe-0 is at (9,9) — distance to enemy (5,19) ≈ 10.8 > 5. Out of range.
    // test-unit-N is at (5,14) — distance 5 ≤ range 5. Takes damage.
    boolean anyProbeDamaged = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.PROBE)
        .anyMatch(u -> u.shields() < SC2Data.maxShields(UnitType.PROBE));
    assertThat(anyProbeDamaged).isTrue();
}

@Test
void rampAttackerDoesNotTriggerMissChance() {
    // Stalker (friendly) on RAMP tile — RAMP ≠ LOW → no miss check even with always-miss RNG.
    // RAMP tiles: x=11-13, y=18. Position (12.5, 18.5) → floor → tile(12,18) = RAMP.
    // Enemy Zealot at (12.5, 19.5) → tile(12,19) = HIGH. Distance ≈ 1 ≤ Stalker range 5.
    game.setTerrainGrid(TerrainGrid.emulatedMap());
    game.setRandomForTesting(new java.util.Random() {
        @Override public double nextDouble() { return 0.0; } // always-miss if check fires
    });
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(12.5f, 18.5f));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(12.5f, 19.5f));
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(12.5f, 19.5f)));

    game.tick();

    Unit zealot = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    assertThat(zealot.shields()).isLessThan(SC2Data.maxShields(UnitType.ZEALOT)); // took damage
}
```

- [ ] **Step 2: Run the failing tests**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | grep -E "FAIL|ERROR|setTerrainGrid|setRandomForTesting" | head -20
```

Expected: compile errors for `setTerrainGrid`, `setRandomForTesting`.

- [ ] **Step 3: Add terrain field, random field, helpers, and miss guard to EmulatedGame.java**

**3a — Add `random` field** (near the `walkabilityGrid` field, around line 57):

```java
// Replace:
private WalkabilityGrid walkabilityGrid = null;

// With:
private TerrainGrid terrainGrid = null;
private java.util.Random random = new java.util.Random(42L);
```

**3b — Add `setRandomForTesting` helper** (near the end of the test helpers section):

```java
/** Injects a predictable Random for miss-chance tests. */
void setRandomForTesting(java.util.Random r) { this.random = r; }
```

**3c — Add `missesHighGround` private method** (near `resolveCombat`, e.g. after `nearestInRange`):

```java
/**
 * Returns true if this attack should miss due to low-ground-to-high-ground penalty.
 * Condition: attacker on LOW, target on HIGH, attack is ranged (range > 1.0), and RNG says miss.
 * Returns false (no miss) when terrainGrid is null or the height condition is not met.
 */
private boolean missesHighGround(Point2d attackerPos, Point2d targetPos, UnitType attackerType) {
    if (terrainGrid == null) return false;
    if (SC2Data.attackRange(attackerType) <= 1.0f) return false; // melee — never penalised
    TerrainGrid.Height ah = terrainGrid.heightAt((int) attackerPos.x(), (int) attackerPos.y());
    TerrainGrid.Height th = terrainGrid.heightAt((int) targetPos.x(),   (int) targetPos.y());
    if (ah != TerrainGrid.Height.LOW || th != TerrainGrid.Height.HIGH) return false;
    return random.nextDouble() < 0.25;
}
```

**3d — Wire miss guard in `resolveCombat()`**

In the friendly-attacks loop, wrap the `computeEffective` call:

```java
// BEFORE (inside the for (Unit attacker : myUnits) loop):
nearestInRange(attacker.position(), enemyUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(),
            damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        firedFriendly.add(attacker.tag());
    });

// AFTER:
nearestInRange(attacker.position(), enemyUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        if (!missesHighGround(attacker.position(), target.position(), attacker.type())) {
            pending.merge(target.tag(),
                damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        }
        firedFriendly.add(attacker.tag()); // cooldown resets even on miss (attack fires, projectile misses)
    });
```

In the enemy-attacks loop:

```java
// BEFORE (inside the for (Unit attacker : enemyUnits) loop):
nearestInRange(attacker.position(), myUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(),
            damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        firedEnemy.add(attacker.tag());
    });

// AFTER:
nearestInRange(attacker.position(), myUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        if (!missesHighGround(attacker.position(), target.position(), attacker.type())) {
            pending.merge(target.tag(),
                damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        }
        firedEnemy.add(attacker.tag());
    });
```

**3e — Update setter** (replace the old `setWalkabilityGrid` line at the end of the file):

```java
// Already done in Task 2. Verify the line now reads:
void setTerrainGrid(TerrainGrid g) { this.terrainGrid = g; }
```

- [ ] **Step 4: Run all six new tests**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`. All 6 new tests pass, all existing tests still pass.

- [ ] **Step 5: Run full suite as sanity check**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "$(cat <<'EOF'
feat(e8): 25% miss chance for ranged attacks from low to high ground

missesHighGround() guard in resolveCombat(): attacker on LOW, target on HIGH,
range > 1.0 → 25% miss via seeded Random. Melee (range ≤ 1.0), equal height,
high→low, and ramp positions are never penalised. terrainGrid null = no effect.

Refs #N
EOF
)"
```

---

## Task 4: EmulatedEngine — wire TerrainGrid

**Files:**
- Already modified in Task 2.

- [ ] **Step 1: Verify EmulatedEngine compiles and integration tests pass**

The changes to `EmulatedEngine.joinGame()` were made in Task 2. Run the integration tests now to confirm the engine boots correctly with `TerrainGrid`:

```bash
mvn test -Dtest=EmulatedTerrainIT -q
```

Expected: `BUILD SUCCESS`, 4 tests pass (the pre-existing tests from before E8).

- [ ] **Step 2: Run the QA endpoints test**

```bash
mvn test -Dtest=QaEndpointsTest -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit (no code changes — this task is a verification gate only)**

No commit needed — `EmulatedEngine` was already committed in Task 2.

---

## Task 5: Terrain REST API — extend response (TDD)

**Files:**
- Modify: `src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java`
- Modify: `src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java`

- [ ] **Step 1: Write failing integration tests**

Add four new tests to `src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java`:

```java
@Test
void terrainEndpointReturnsHighGroundArray() {
    // y=19..63 = 45 rows × 64 cols = 2880 HIGH tiles
    given().when().get("/qa/emulated/terrain")
        .then()
        .statusCode(200)
        .body("highGround",        notNullValue())
        .body("highGround.size()", equalTo(45 * 64));
}

@Test
void terrainEndpointReturnsRampArray() {
    // x=11,12,13 at y=18 = 3 ramp tiles
    given().when().get("/qa/emulated/terrain")
        .then()
        .statusCode(200)
        .body("ramps",        notNullValue())
        .body("ramps.size()", equalTo(3));
}

@Test
void terrainEndpointHighGroundIncludesStagingTile() {
    // Staging area (26,26) is HIGH ground
    given().when().get("/qa/emulated/terrain")
        .then()
        .statusCode(200)
        .body("highGround.find { it == [26, 26] }", notNullValue());
}

@Test
void terrainEndpointRampIncludesChokeGap() {
    // x=11, y=18 is the leftmost ramp tile
    given().when().get("/qa/emulated/terrain")
        .then()
        .statusCode(200)
        .body("ramps.find { it == [11, 18] }", notNullValue());
}
```

- [ ] **Step 2: Run to confirm they fail**

```bash
mvn test -Dtest=EmulatedTerrainIT -q 2>&1 | grep -E "FAIL|AssertionError" | head -10
```

Expected: failures because `highGround` and `ramps` are null (not in response yet).

- [ ] **Step 3: Extend EmulatedTerrainResource**

Replace the entire `EmulatedTerrainResource.java`:

```java
package io.quarkmind.qa;

import io.quarkmind.domain.TerrainGrid;
import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import java.util.ArrayList;
import java.util.List;

/**
 * Returns the emulated map's terrain as sparse typed tile lists.
 * Fetched once by the visualizer at startup; terrain never changes during a game.
 * LOW tiles are the default — not included in any list.
 */
@UnlessBuildProfile("prod")
@Path("/qa/emulated/terrain")
@Produces(MediaType.APPLICATION_JSON)
public class EmulatedTerrainResource {

    @GET
    public TerrainResponse getTerrain() {
        TerrainGrid grid = TerrainGrid.emulatedMap();
        List<int[]> walls      = new ArrayList<>();
        List<int[]> highGround = new ArrayList<>();
        List<int[]> ramps      = new ArrayList<>();
        for (int x = 0; x < grid.width(); x++) {
            for (int y = 0; y < grid.height(); y++) {
                switch (grid.heightAt(x, y)) {
                    case WALL -> walls.add(new int[]{x, y});
                    case HIGH -> highGround.add(new int[]{x, y});
                    case RAMP -> ramps.add(new int[]{x, y});
                    case LOW  -> {} // default — omitted
                }
            }
        }
        return new TerrainResponse(grid.width(), grid.height(), walls, highGround, ramps);
    }

    public record TerrainResponse(
        int width,
        int height,
        List<int[]> walls,
        List<int[]> highGround,
        List<int[]> ramps
    ) {}
}
```

- [ ] **Step 4: Run all terrain integration tests**

```bash
mvn test -Dtest=EmulatedTerrainIT -q
```

Expected: `BUILD SUCCESS`, all 8 tests pass (4 pre-existing + 4 new).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/qa/EmulatedTerrainResource.java \
        src/test/java/io/quarkmind/qa/EmulatedTerrainIT.java
git commit -m "$(cat <<'EOF'
feat(e8): terrain API — add highGround and ramps arrays to /qa/emulated/terrain

TerrainResponse gains two sparse tile lists. LOW tiles omitted (default).
2880 HIGH tiles (y=19..63) and 3 RAMP tiles (x=11-13, y=18) in payload.

Refs #N
EOF
)"
```

---

## Task 6: Visualizer — height shading + E2E test

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Add Playwright E2E test for HIGH tile shading**

Add this test to `VisualizerRenderTest.java`. Add it after the existing tests, before the closing brace:

```java
@Test
void highGroundTileRendersWithBrownShading() {
    assumeTrue(playwrightAvailable, "Chromium not installed — skipping Playwright test");

    try (Playwright playwright = Playwright.create()) {
        Browser browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions().setHeadless(true));
        Page page = browser.newPage();
        page.navigate(visualizerUrl.toString());

        // Wait for WebSocket to connect (terrain is drawn before WS connects — safe to sample after)
        page.waitForFunction("window.__test && window.__test.wsConnected()");

        // HIGH ground tile (5, 20): canvasX = 5*20 = 100, canvasY = (32-20-1)*20 = 220
        // Sample center of tile: (110, 230)
        // Background: rgb(26, 26, 46). HIGH tile blended: ~rgb(88, 70, 32).
        // Assert red channel is distinctly higher than background (26).
        int canvasX = 5 * SCALE;
        int canvasY = (VIEWPORT_H - 20 - 1) * SCALE;
        int sampleX = canvasX + SCALE / 2;
        int sampleY = canvasY + SCALE / 2;

        byte[] screenshot = page.screenshot();
        BufferedImage img = ImageIO.read(new ByteArrayInputStream(screenshot));
        Color pixel = new Color(img.getRGB(sampleX, sampleY), true);

        // High ground (brown/tan) has red channel ≈ 88, background red = 26.
        // Assert > 50 to confirm the tile is shaded, not bare background.
        assertThat(pixel.getRed())
            .as("HIGH ground tile at canvas (%d,%d) should be brownish, not background (red=26)",
                sampleX, sampleY)
            .isGreaterThan(50);

        browser.close();
    }
}
```

- [ ] **Step 2: Run the E2E test to confirm it fails (terrain JS not updated yet)**

```bash
mvn test -Dtest=VisualizerRenderTest#highGroundTileRendersWithBrownShading -q 2>&1 | tail -5
```

Expected: test fails — pixel is still background colour (red=26) because `loadTerrain()` doesn't draw HIGH tiles yet. (If Chromium is not installed, test is skipped — proceed to Step 3 regardless.)

- [ ] **Step 3: Update loadTerrain() in visualizer.js**

Replace the entire `loadTerrain` function in `visualizer.js`:

```javascript
/**
 * Fetch terrain once at startup and draw static height-shaded tiles.
 * Colour scheme (topographic): HIGH=tan/brown, RAMP=mid-brown, WALL=dark-grey.
 * LOW tiles are the default canvas background — no fill drawn.
 * Silently no-ops if the endpoint is unavailable (non-emulated profile).
 */
async function loadTerrain(container) {
    try {
        const resp = await fetch('/qa/emulated/terrain');
        if (!resp.ok) return;
        const data = await resp.json();
        const g = new PIXI.Graphics();

        // HIGH ground — warm tan/brown
        (data.highGround ?? []).forEach(([wx, wy]) => {
            const canvasX = wx * SCALE;
            const canvasY = (VIEWPORT_H - wy - 1) * SCALE;
            g.rect(canvasX, canvasY, SCALE, SCALE).fill({ color: 0x8B6914, alpha: 0.55 });
        });

        // RAMP — mid brown (transitional slope)
        (data.ramps ?? []).forEach(([wx, wy]) => {
            const canvasX = wx * SCALE;
            const canvasY = (VIEWPORT_H - wy - 1) * SCALE;
            g.rect(canvasX, canvasY, SCALE, SCALE).fill({ color: 0x7A6040, alpha: 0.40 });
        });

        // WALL — dark grey (unchanged from E7)
        data.walls.forEach(([wx, wy]) => {
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

- [ ] **Step 4: Run the E2E test again — expect pass**

```bash
mvn test -Dtest=VisualizerRenderTest#highGroundTileRendersWithBrownShading -q
```

Expected: `BUILD SUCCESS` (or `SKIPPED` if Chromium not installed).

- [ ] **Step 5: Run full Playwright suite**

```bash
mvn test -Dtest=VisualizerRenderTest -q
```

Expected: all existing Playwright tests still pass.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "$(cat <<'EOF'
feat(e8): visualizer — topographic height shading for HIGH/RAMP/WALL tiles

loadTerrain() draws HIGH ground tan/brown (0x8B6914 α0.55), RAMP mid-brown
(0x7A6040 α0.40), WALL unchanged. LOW is bare canvas background. Playwright
E2E test samples HIGH tile pixel and asserts brownish shading.

Refs #N
EOF
)"
```

---

## Task 7: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the EmulatedGame test helper list**

In `CLAUDE.md`, find the `**EmulatedGame test helpers**` section. Update the `setWalkabilityGrid` entry:

```markdown
<!-- BEFORE -->
- `setWalkabilityGrid(WalkabilityGrid)` — activate the physics wall constraint for tests that verify wall enforcement (default null = no wall checking)

<!-- AFTER -->
- `setTerrainGrid(TerrainGrid)` — activate terrain for tests that verify wall enforcement or miss-chance mechanics (default null = no terrain effects)
```

Also add `setRandomForTesting(Random)` to the list:

```markdown
- `setRandomForTesting(Random)` — inject a predictable Random for high-ground miss-chance tests (always-miss: return 0.0; always-hit: return 1.0)
```

- [ ] **Step 2: Run full test suite one last time**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`. Test count should be pre-E8 total + 6 EmulatedGame + 6 TerrainGrid + 4 EmulatedTerrainIT + 1 Playwright = 368 + 17 = 385 (or 384 if Playwright is skipped).

- [ ] **Step 3: Close the GitHub issue and commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs(e8): update CLAUDE.md — setTerrainGrid, setRandomForTesting helpers

Closes #N
EOF
)"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] TerrainGrid data model (Task 1)
- [x] Map layout — HIGH/LOW/RAMP/WALL encoding in emulatedMap() (Task 1)
- [x] AStarPathfinder and PathfindingMovement migration (Task 2)
- [x] EmulatedGame field + miss guard in resolveCombat() (Task 3)
- [x] EmulatedEngine wired with TerrainGrid (Task 2 + Task 4)
- [x] Terrain REST API — highGround + ramps (Task 5)
- [x] Visualizer height shading (Task 6)
- [x] Unit tests — TerrainGrid (Task 1), miss-chance (Task 3)
- [x] Integration tests — EmulatedTerrainIT (Task 5)
- [x] E2E Playwright test — pixel sampling (Task 6)
- [x] CLAUDE.md updated (Task 7)
- [x] Vision explicitly deferred — no tasks for it
- [x] Ramp directionality explicitly deferred — no tasks for it

**Placeholder scan:** No TBD, TODO, or vague steps.

**Type consistency:** `TerrainGrid` and `TerrainGrid.Height` used consistently. `setTerrainGrid` / `setRandomForTesting` defined in Task 3, used in Task 3 tests. `missesHighGround()` defined and called in same task. `TerrainResponse` record updated in Task 5 only.
