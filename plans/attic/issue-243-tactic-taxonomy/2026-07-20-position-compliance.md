# Position-Based Compliance Verification — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #244 — feat: position-based compliance verification for coaching
**Issue group:** #244

**Goal:** Extend the coaching compliance evaluator with spatial predicates — army centroid tracking, expansion placement verification, and unit proximity checks — replacing the flat count-delta model with a sealed predicate hierarchy.

**Architecture:** Sealed `VerificationPredicate` hierarchy replaces flat nullable verification fields on `CoachingAdvice`. Each predicate carries its own baseline (captured at commitment creation) and verification logic. A `LocationReference` sealed hierarchy provides a semantic vocabulary for map positions, resolved by `LocationResolver`. Domain model enriched with `MapInfo`, `ExpansionLocation`, `NeutralFeature`, `Positionable`, and spatial utilities.

**Tech Stack:** Java 21 (sealed interfaces, pattern matching, records), Quarkus CDI, plain JUnit 5

## Global Constraints

- Domain model (`io.quarkmind.domain/`) must remain plain Java — no CDI, no Quarkus imports
- All new records are immutable — `List.copyOf()` in compact constructors
- `UnitType.isWorker()` covers PROBE, SCV, DRONE only
- Town-hall types: NEXUS, COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS, HATCHERY, LAIR, HIVE
- Expansion clustering radius: 12.0 map units (calibrate against IEM10)
- `LocationResolver.resolve()` returns `@Nullable Point2d` — null = unresolvable
- `Point2d.centroidOf()` returns `@Nullable` — null for empty list
- Predicates treat null resolve as "cannot verify" → `isSatisfied()` returns false
- Worker input to coaching already contains trigger-time game state — use that for baseline, not response-time state

---

### Task 1: Domain Model Foundations — Positionable, Point2d.centroidOf(), UnitType.isWorker()

**Files:**
- Create: `src/main/java/io/quarkmind/domain/Positionable.java`
- Modify: `src/main/java/io/quarkmind/domain/Point2d.java` — add `centroidOf()`
- Modify: `src/main/java/io/quarkmind/domain/UnitType.java` — add `isWorker()`
- Modify: `src/main/java/io/quarkmind/domain/Unit.java` — implement `Positionable`
- Modify: `src/main/java/io/quarkmind/domain/Building.java` — implement `Positionable`
- Modify: `src/main/java/io/quarkmind/domain/Resource.java` — implement `Positionable`
- Create: `src/test/java/io/quarkmind/domain/Point2dCentroidTest.java`
- Create: `src/test/java/io/quarkmind/domain/UnitTypeTest.java` (may already exist — extend if so)

**Interfaces:**
- Produces: `Positionable { Point2d position(); }` — used by `ExpansionLocation.fromResources()`, `Point2d.centroidOf()`
- Produces: `Point2d.centroidOf(List<? extends Positionable>)` — used by predicates, expansion derivation
- Produces: `UnitType.isWorker()` — used by `ArmyCentroidMovement`, `UnitsNearLocation`

- [ ] **Step 1: Write Positionable interface**

```java
// src/main/java/io/quarkmind/domain/Positionable.java
package io.quarkmind.domain;

public interface Positionable { Point2d position(); }
```

Use `ide_create_file`.

- [ ] **Step 2: Write Point2d.centroidOf() test**

```java
// src/test/java/io/quarkmind/domain/Point2dCentroidTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class Point2dCentroidTest {

    record Pos(Point2d position) implements Positionable {}

    @Test void centroidOf_emptyList_returnsNull() {
        assertThat(Point2d.centroidOf(List.of())).isNull();
    }

    @Test void centroidOf_singleItem_returnsThatPosition() {
        var result = Point2d.centroidOf(List.of(new Pos(new Point2d(5f, 10f))));
        assertThat(result).isEqualTo(new Point2d(5f, 10f));
    }

    @Test void centroidOf_multipleItems_returnsAverage() {
        var items = List.of(
            new Pos(new Point2d(0f, 0f)),
            new Pos(new Point2d(10f, 0f)),
            new Pos(new Point2d(0f, 10f))
        );
        var result = Point2d.centroidOf(items);
        assertThat(result.x()).isCloseTo(3.333f, within(0.01f));
        assertThat(result.y()).isCloseTo(3.333f, within(0.01f));
    }

    @Test void centroidOf_twoItems_returnsMidpoint() {
        var items = List.of(
            new Pos(new Point2d(2f, 4f)),
            new Pos(new Point2d(8f, 12f))
        );
        var result = Point2d.centroidOf(items);
        assertThat(result).isEqualTo(new Point2d(5f, 8f));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -Dtest=Point2dCentroidTest -q`
Expected: FAIL — `centroidOf` method does not exist

- [ ] **Step 4: Implement Point2d.centroidOf()**

Add to `Point2d.java` using `ide_insert_member`:

```java
public static Point2d centroidOf(List<? extends Positionable> items) {
    if (items == null || items.isEmpty()) return null;
    float sumX = 0, sumY = 0;
    for (var item : items) {
        sumX += item.position().x;
        sumY += item.position().y;
    }
    return new Point2d(sumX / items.size(), sumY / items.size());
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -Dtest=Point2dCentroidTest -q`
Expected: PASS

- [ ] **Step 6: Write UnitType.isWorker() test**

Check if `UnitTypeTest` already exists. If so, add to it. If not, create:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;
import static org.assertj.core.api.Assertions.assertThat;

class UnitTypeWorkerTest {

    @Test void probe_isWorker() { assertThat(UnitType.PROBE.isWorker()).isTrue(); }
    @Test void scv_isWorker()   { assertThat(UnitType.SCV.isWorker()).isTrue(); }
    @Test void drone_isWorker() { assertThat(UnitType.DRONE.isWorker()).isTrue(); }

    @Test void zealot_isNotWorker()   { assertThat(UnitType.ZEALOT.isWorker()).isFalse(); }
    @Test void marine_isNotWorker()   { assertThat(UnitType.MARINE.isWorker()).isFalse(); }
    @Test void zergling_isNotWorker() { assertThat(UnitType.ZERGLING.isWorker()).isFalse(); }
    @Test void mule_isNotWorker()     { assertThat(UnitType.MULE.isWorker()).isFalse(); }
    @Test void overlord_isNotWorker() { assertThat(UnitType.OVERLORD.isWorker()).isFalse(); }
    @Test void observer_isNotWorker() { assertThat(UnitType.OBSERVER.isWorker()).isFalse(); }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `mvn test -Dtest=UnitTypeWorkerTest -q`
Expected: FAIL — `isWorker()` not defined

- [ ] **Step 8: Implement UnitType.isWorker()**

Add to `UnitType.java` using `ide_insert_member`:

```java
public boolean isWorker() {
    return this == PROBE || this == SCV || this == DRONE;
}
```

- [ ] **Step 9: Make Unit, Building, Resource implement Positionable**

Use `ide_edit_member` with `member = "Unit"` (class declaration) to add `implements Positionable`:
- `Unit.java`: `public record Unit(...) implements Positionable {}`
- `Building.java`: `public record Building(...) implements Positionable {}`
- `Resource.java`: `public record Resource(...) implements Positionable {}`

Each already has `Point2d position` — no method to add, just the interface marker.

- [ ] **Step 10: Run all tests to verify nothing is broken**

Run: `mvn test -q`
Expected: All existing tests PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/Positionable.java src/main/java/io/quarkmind/domain/Point2d.java src/main/java/io/quarkmind/domain/UnitType.java src/main/java/io/quarkmind/domain/Unit.java src/main/java/io/quarkmind/domain/Building.java src/main/java/io/quarkmind/domain/Resource.java src/test/java/io/quarkmind/domain/Point2dCentroidTest.java src/test/java/io/quarkmind/domain/UnitTypeWorkerTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): domain foundations — Positionable, Point2d.centroidOf(), UnitType.isWorker()

Refs #244"
```

---

### Task 2: Map Topology Types — NeutralFeature, ExpansionLocation, TerrainGrid.rampPositions()

**Files:**
- Create: `src/main/java/io/quarkmind/domain/NeutralFeatureType.java`
- Create: `src/main/java/io/quarkmind/domain/NeutralFeature.java`
- Create: `src/main/java/io/quarkmind/domain/ExpansionLocation.java`
- Modify: `src/main/java/io/quarkmind/domain/TerrainGrid.java` — add `rampPositions()`
- Create: `src/test/java/io/quarkmind/domain/ExpansionLocationTest.java`
- Create: `src/test/java/io/quarkmind/domain/TerrainGridRampTest.java`

**Interfaces:**
- Consumes: `Positionable`, `Point2d.centroidOf()` from Task 1
- Produces: `ExpansionLocation.fromResources(minerals, geysers, playerStart)` — used by MapInfo construction
- Produces: `TerrainGrid.rampPositions()` — used by MapInfo construction
- Produces: `NeutralFeature(String tag, NeutralFeatureType type, Point2d position)` — used by MapInfo
- Produces: `NeutralFeatureType { XELNAGA_TOWER, DESTRUCTIBLE_ROCK, DESTRUCTIBLE_DEBRIS }` — used by LocationResolver

- [ ] **Step 1: Create NeutralFeatureType and NeutralFeature**

```java
// NeutralFeatureType.java
package io.quarkmind.domain;

public enum NeutralFeatureType {
    XELNAGA_TOWER, DESTRUCTIBLE_ROCK, DESTRUCTIBLE_DEBRIS
}
```

```java
// NeutralFeature.java
package io.quarkmind.domain;

public record NeutralFeature(String tag, NeutralFeatureType type, Point2d position) implements Positionable {}
```

Use `ide_create_file` for each.

- [ ] **Step 2: Write ExpansionLocation clustering test**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class ExpansionLocationTest {

    @Test void fromResources_emptyLists_returnsEmpty() {
        var result = ExpansionLocation.fromResources(List.of(), List.of(), new Point2d(0f, 0f));
        assertThat(result).isEmpty();
    }

    @Test void fromResources_singleCluster_returnsOneExpansion() {
        var minerals = List.of(
            new Resource("m1", new Point2d(10f, 10f), 1500),
            new Resource("m2", new Point2d(12f, 10f), 1500),
            new Resource("m3", new Point2d(14f, 10f), 1500)
        );
        var geysers = List.of(
            new Resource("g1", new Point2d(11f, 13f), 2250)
        );
        var result = ExpansionLocation.fromResources(minerals, geysers, new Point2d(10f, 10f));
        assertThat(result).hasSize(1);
        assertThat(result.get(0).ordinal()).isEqualTo(0);
    }

    @Test void fromResources_twoClusters_orderedByDistanceFromStart() {
        var minerals = List.of(
            // Cluster near (10,10) — main base
            new Resource("m1", new Point2d(10f, 10f), 1500),
            new Resource("m2", new Point2d(12f, 10f), 1500),
            // Cluster near (40,40) — natural
            new Resource("m3", new Point2d(40f, 40f), 1500),
            new Resource("m4", new Point2d(42f, 40f), 1500)
        );
        var result = ExpansionLocation.fromResources(minerals, List.of(), new Point2d(8f, 8f));
        assertThat(result).hasSize(2);
        assertThat(result.get(0).ordinal()).isEqualTo(0);
        assertThat(result.get(0).position().x()).isCloseTo(11f, within(0.1f));
        assertThat(result.get(1).ordinal()).isEqualTo(1);
        assertThat(result.get(1).position().x()).isCloseTo(41f, within(0.1f));
    }

    @Test void fromResources_clusteringRadius_doesNotMergeDistantGroups() {
        var minerals = List.of(
            new Resource("m1", new Point2d(10f, 10f), 1500),
            new Resource("m2", new Point2d(12f, 10f), 1500),
            new Resource("m3", new Point2d(50f, 50f), 1500),
            new Resource("m4", new Point2d(52f, 50f), 1500)
        );
        var result = ExpansionLocation.fromResources(minerals, List.of(), new Point2d(10f, 10f));
        assertThat(result).hasSize(2);
    }

    @Test void fromResources_geysersClusterWithMinerals() {
        var minerals = List.of(
            new Resource("m1", new Point2d(10f, 10f), 1500),
            new Resource("m2", new Point2d(12f, 10f), 1500)
        );
        var geysers = List.of(
            new Resource("g1", new Point2d(11f, 13f), 2250)
        );
        var result = ExpansionLocation.fromResources(minerals, geysers, new Point2d(10f, 10f));
        assertThat(result).hasSize(1);
        assertThat(result.get(0).position().x()).isCloseTo(11f, within(0.1f));
        assertThat(result.get(0).position().y()).isCloseTo(11f, within(0.1f));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -Dtest=ExpansionLocationTest -q`
Expected: FAIL — `ExpansionLocation` class or `fromResources` not found

- [ ] **Step 4: Implement ExpansionLocation**

```java
package io.quarkmind.domain;

import java.util.*;

public record ExpansionLocation(int ordinal, Point2d position) {

    private static final double CLUSTER_RADIUS = 12.0;

    public static List<ExpansionLocation> fromResources(
            List<Resource> minerals, List<Resource> geysers, Point2d playerStart) {
        List<Resource> all = new ArrayList<>(minerals.size() + geysers.size());
        all.addAll(minerals);
        all.addAll(geysers);
        if (all.isEmpty()) return List.of();

        boolean[] visited = new boolean[all.size()];
        List<List<Resource>> clusters = new ArrayList<>();

        for (int i = 0; i < all.size(); i++) {
            if (visited[i]) continue;
            List<Resource> cluster = new ArrayList<>();
            Deque<Integer> stack = new ArrayDeque<>();
            stack.push(i);
            visited[i] = true;
            while (!stack.isEmpty()) {
                int idx = stack.pop();
                cluster.add(all.get(idx));
                for (int j = 0; j < all.size(); j++) {
                    if (!visited[j] && all.get(idx).position().distanceTo(all.get(j).position()) <= CLUSTER_RADIUS) {
                        visited[j] = true;
                        stack.push(j);
                    }
                }
            }
            clusters.add(cluster);
        }

        List<Point2d> centroids = clusters.stream()
            .map(c -> Point2d.centroidOf(c))
            .filter(Objects::nonNull)
            .toList();

        List<Point2d> sorted = centroids.stream()
            .sorted(Comparator.comparingDouble(p -> p.distanceTo(playerStart)))
            .toList();

        List<ExpansionLocation> result = new ArrayList<>(sorted.size());
        for (int i = 0; i < sorted.size(); i++) {
            result.add(new ExpansionLocation(i, sorted.get(i)));
        }
        return List.copyOf(result);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -Dtest=ExpansionLocationTest -q`
Expected: PASS

- [ ] **Step 6: Write TerrainGrid.rampPositions() test**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class TerrainGridRampTest {

    @Test void rampPositions_noRamps_returnsEmpty() {
        var grid = new TerrainGrid(4, 4, fill(4, 4, TerrainGrid.Height.HIGH));
        assertThat(grid.rampPositions()).isEmpty();
    }

    @Test void rampPositions_singleRampCluster_returnsCentroid() {
        var heights = fill(8, 8, TerrainGrid.Height.HIGH);
        heights[3][3] = TerrainGrid.Height.RAMP;
        heights[3][4] = TerrainGrid.Height.RAMP;
        heights[4][3] = TerrainGrid.Height.RAMP;
        heights[4][4] = TerrainGrid.Height.RAMP;
        var grid = new TerrainGrid(8, 8, heights);
        var ramps = grid.rampPositions();
        assertThat(ramps).hasSize(1);
        assertThat(ramps.get(0).x()).isCloseTo(3.5f, within(0.1f));
        assertThat(ramps.get(0).y()).isCloseTo(3.5f, within(0.1f));
    }

    @Test void rampPositions_twoSeparateRamps_returnsTwoCentroids() {
        var heights = fill(16, 16, TerrainGrid.Height.HIGH);
        heights[2][2] = TerrainGrid.Height.RAMP;
        heights[2][3] = TerrainGrid.Height.RAMP;
        heights[12][12] = TerrainGrid.Height.RAMP;
        heights[12][13] = TerrainGrid.Height.RAMP;
        var grid = new TerrainGrid(16, 16, heights);
        assertThat(grid.rampPositions()).hasSize(2);
    }

    private static TerrainGrid.Height[][] fill(int w, int h, TerrainGrid.Height value) {
        var grid = new TerrainGrid.Height[h][w];
        for (int y = 0; y < h; y++)
            for (int x = 0; x < w; x++)
                grid[y][x] = value;
        return grid;
    }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `mvn test -Dtest=TerrainGridRampTest -q`
Expected: FAIL — `rampPositions()` not defined

- [ ] **Step 8: Implement TerrainGrid.rampPositions()**

Add to `TerrainGrid.java` using `ide_insert_member`:

```java
public List<Point2d> rampPositions() {
    boolean[][] visited = new boolean[height][width];
    List<Point2d> result = new java.util.ArrayList<>();

    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            if (grid[y][x] == Height.RAMP && !visited[y][x]) {
                List<Point2d> cells = new java.util.ArrayList<>();
                java.util.Deque<int[]> stack = new java.util.ArrayDeque<>();
                stack.push(new int[]{x, y});
                visited[y][x] = true;
                while (!stack.isEmpty()) {
                    int[] cell = stack.pop();
                    cells.add(new Point2d(cell[0], cell[1]));
                    for (int[] d : new int[][]{{-1,0},{1,0},{0,-1},{0,1}}) {
                        int nx = cell[0] + d[0], ny = cell[1] + d[1];
                        if (nx >= 0 && nx < width && ny >= 0 && ny < height
                            && grid[ny][nx] == Height.RAMP && !visited[ny][nx]) {
                            visited[ny][nx] = true;
                            stack.push(new int[]{nx, ny});
                        }
                    }
                }
                float sx = 0, sy = 0;
                for (var p : cells) { sx += p.x(); sy += p.y(); }
                result.add(new Point2d(sx / cells.size(), sy / cells.size()));
            }
        }
    }
    return List.copyOf(result);
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn test -Dtest=TerrainGridRampTest,ExpansionLocationTest -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/domain/NeutralFeatureType.java src/main/java/io/quarkmind/domain/NeutralFeature.java src/main/java/io/quarkmind/domain/ExpansionLocation.java src/main/java/io/quarkmind/domain/TerrainGrid.java src/test/java/io/quarkmind/domain/ExpansionLocationTest.java src/test/java/io/quarkmind/domain/TerrainGridRampTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): map topology types — NeutralFeature, ExpansionLocation, TerrainGrid.rampPositions()

Refs #244"
```

---

### Task 3: MapInfo + GameState Widening

**Files:**
- Create: `src/main/java/io/quarkmind/domain/MapInfo.java`
- Modify: `src/main/java/io/quarkmind/domain/GameState.java` — add `MapInfo mapInfo` field
- Modify: ALL files that construct `new GameState(...)` — add `null` or real MapInfo as last arg
- Modify: `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java` — construct and carry MapInfo
- Create: `src/test/java/io/quarkmind/domain/MapInfoTest.java`

**Interfaces:**
- Consumes: `ExpansionLocation`, `NeutralFeature`, `Point2d` from Tasks 1-2
- Produces: `MapInfo(playerStart, enemyStart, mapWidth, mapHeight, expansions, neutralFeatures, rampPositions)` — used by LocationResolver
- Produces: `GameState.mapInfo()` — used by LocationResolver, predicates

This is the highest-impact task — GameState is constructed in ~15 files. The approach:
1. Create MapInfo
2. Add `MapInfo mapInfo` as the last parameter on GameState
3. Update every `new GameState(...)` call — pass `null` for tests/code that don't need spatial data
4. Wire MapInfo into SimulatedGame

- [ ] **Step 1: Write MapInfo test**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class MapInfoTest {

    @Test void construction_withAllFields() {
        var expansions = List.of(new ExpansionLocation(0, new Point2d(10f, 10f)));
        var neutrals = List.of(new NeutralFeature("t1", NeutralFeatureType.XELNAGA_TOWER, new Point2d(32f, 32f)));
        var ramps = List.of(new Point2d(20f, 15f));
        var info = new MapInfo(new Point2d(8f, 8f), new Point2d(56f, 56f), 64, 64, expansions, neutrals, ramps);

        assertThat(info.playerStart()).isEqualTo(new Point2d(8f, 8f));
        assertThat(info.enemyStart()).isEqualTo(new Point2d(56f, 56f));
        assertThat(info.mapWidth()).isEqualTo(64);
        assertThat(info.expansions()).hasSize(1);
        assertThat(info.neutralFeatures()).hasSize(1);
        assertThat(info.rampPositions()).hasSize(1);
    }

    @Test void construction_nullEnemyStart_allowed() {
        var info = new MapInfo(new Point2d(8f, 8f), null, 64, 64, List.of(), List.of(), List.of());
        assertThat(info.enemyStart()).isNull();
    }

    @Test void lists_areImmutable() {
        var info = new MapInfo(new Point2d(8f, 8f), null, 64, 64, List.of(), List.of(), List.of());
        assertThatThrownBy(() -> info.expansions().add(new ExpansionLocation(0, new Point2d(0f, 0f))))
            .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Implement MapInfo**

```java
package io.quarkmind.domain;

import java.util.List;

public record MapInfo(
    Point2d playerStart,
    Point2d enemyStart,
    int mapWidth,
    int mapHeight,
    List<ExpansionLocation> expansions,
    List<NeutralFeature> neutralFeatures,
    List<Point2d> rampPositions
) {
    public MapInfo {
        expansions = List.copyOf(expansions);
        neutralFeatures = List.copyOf(neutralFeatures);
        rampPositions = List.copyOf(rampPositions);
    }
}
```

- [ ] **Step 3: Run MapInfo test**

Run: `mvn test -Dtest=MapInfoTest -q`
Expected: PASS

- [ ] **Step 4: Widen GameState — add MapInfo mapInfo field**

Use `ide_edit_member` with `member = "GameState"` to rewrite the record declaration:

```java
public record GameState(
    int minerals,
    int vespene,
    int supply,
    int supplyUsed,
    List<Unit> myUnits,
    List<Building> myBuildings,
    List<Unit> enemyUnits,
    List<Building> enemyBuildings,
    List<Unit> enemyStagingArea,
    List<Resource> geysers,
    List<Resource> mineralPatches,
    long gameFrame,
    MapInfo mapInfo
) {
    public GameState {
        myUnits          = List.copyOf(myUnits);
        myBuildings      = List.copyOf(myBuildings);
        enemyUnits       = List.copyOf(enemyUnits);
        enemyBuildings   = List.copyOf(enemyBuildings);
        enemyStagingArea = List.copyOf(enemyStagingArea);
        geysers          = List.copyOf(geysers);
        mineralPatches   = List.copyOf(mineralPatches);
    }
}
```

- [ ] **Step 5: Fix all broken GameState constructor calls**

Run `ide_diagnostics` with `includeBuildErrors=true` after `ide_build_project` to find all broken calls. For each:
- Test helper methods (e.g. `gameStateWithUnits()`, `emptyState()`) — append `, null` as last arg
- Production constructors (e.g. `ObservationTranslator.translate()`, `SimulatedGame.snapshot()`) — append `, null` initially, wired properly in later tasks
- `RealSC2Engine.emptyState()` — append `, null`
- `ReplayEngine.emptyState()` — append `, null`

Use `ide_edit_member` or `ide_replace_member` for each caller. There are roughly 15 distinct constructor call sites.

- [ ] **Step 6: Wire MapInfo into SimulatedGame**

SimulatedGame's `snapshot()` method constructs GameState. Add a `MapInfo` field to SimulatedGame, populate it during `reset()`:

```java
// In SimulatedGame — add field:
private MapInfo mapInfo;

// In reset() — construct MapInfo after resources are set up:
var expansions = ExpansionLocation.fromResources(
    List.copyOf(mineralPatches), List.copyOf(geysers), new Point2d(10f, 10f));
this.mapInfo = new MapInfo(
    new Point2d(10f, 10f), new Point2d(54f, 54f),
    64, 64, expansions, List.of(), List.of());

// In snapshot() — pass mapInfo:
return new GameState(..., mapInfo);
```

The exact player/enemy start positions match SimulatedGame's existing layout.

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`
Expected: All tests PASS. This is the critical verification — every GameState caller must compile.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add -A
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): MapInfo record + GameState widening — all callers updated

Refs #244"
```

---

### Task 4: LocationReference + LocationResolver

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/LocationReference.java` (sealed interface + all permits as nested records)
- Create: `src/main/java/io/quarkmind/plugin/coaching/LocationResolver.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/LocationResolverTest.java`

**Interfaces:**
- Consumes: `GameState.mapInfo()`, `MapInfo`, `NeutralFeatureType`, `Point2d` from Tasks 1-3
- Produces: `LocationReference` sealed hierarchy — used by predicates, LLM parsing
- Produces: `LocationResolver.resolve(LocationReference, GameState)` → `@Nullable Point2d` — used by predicates, evaluator

- [ ] **Step 1: Write LocationResolver test**

```java
package io.quarkmind.plugin.coaching;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class LocationResolverTest {

    private final LocationResolver resolver = new LocationResolver();

    @Test void playerBase_resolvesToPlayerStart() {
        var state = stateWithMapInfo(mapInfo());
        var result = resolver.resolve(new LocationReference.PlayerBase(), state);
        assertThat(result).isEqualTo(new Point2d(8f, 8f));
    }

    @Test void enemyBase_resolvesToEnemyStart() {
        var state = stateWithMapInfo(mapInfo());
        assertThat(resolver.resolve(new LocationReference.EnemyBase(), state))
            .isEqualTo(new Point2d(56f, 56f));
    }

    @Test void enemyBase_nullWhenUnknown() {
        var info = new MapInfo(new Point2d(8f, 8f), null, 64, 64, List.of(), List.of(), List.of());
        var state = stateWithMapInfo(info);
        assertThat(resolver.resolve(new LocationReference.EnemyBase(), state)).isNull();
    }

    @Test void mapCenter() {
        var state = stateWithMapInfo(mapInfo());
        var result = resolver.resolve(new LocationReference.MapCenter(), state);
        assertThat(result).isEqualTo(new Point2d(32f, 32f));
    }

    @Test void expansionOrdinal_validIndex() {
        var state = stateWithMapInfo(mapInfo());
        var result = resolver.resolve(new LocationReference.ExpansionOrdinal(0), state);
        assertThat(result).isEqualTo(new Point2d(10f, 10f));
    }

    @Test void expansionOrdinal_outOfBounds_returnsNull() {
        var state = stateWithMapInfo(mapInfo());
        assertThat(resolver.resolve(new LocationReference.ExpansionOrdinal(99), state)).isNull();
    }

    @Test void watchtower_validIndex() {
        var state = stateWithMapInfo(mapInfo());
        assertThat(resolver.resolve(new LocationReference.Watchtower(0), state))
            .isEqualTo(new Point2d(32f, 32f));
    }

    @Test void watchtower_noTowersOnMap_returnsNull() {
        var info = new MapInfo(new Point2d(8f, 8f), null, 64, 64, List.of(), List.of(), List.of());
        var state = stateWithMapInfo(info);
        assertThat(resolver.resolve(new LocationReference.Watchtower(0), state)).isNull();
    }

    @Test void absolutePosition() {
        var state = stateWithMapInfo(null);
        var result = resolver.resolve(new LocationReference.AbsolutePosition(25f, 30f), state);
        assertThat(result).isEqualTo(new Point2d(25f, 30f));
    }

    @Test void nullMapInfo_nonAbsolute_returnsNull() {
        var state = stateWithMapInfo(null);
        assertThat(resolver.resolve(new LocationReference.PlayerBase(), state)).isNull();
    }

    @Test void nearestRamp_resolvesToClosestRamp() {
        var ramps = List.of(new Point2d(20f, 15f), new Point2d(50f, 50f));
        var info = new MapInfo(new Point2d(8f, 8f), new Point2d(56f, 56f), 64, 64, List.of(), List.of(), ramps);
        var state = stateWithMapInfo(info);
        var result = resolver.resolve(new LocationReference.NearestRamp(new LocationReference.PlayerBase()), state);
        assertThat(result).isEqualTo(new Point2d(20f, 15f));
    }

    private MapInfo mapInfo() {
        return new MapInfo(
            new Point2d(8f, 8f), new Point2d(56f, 56f), 64, 64,
            List.of(new ExpansionLocation(0, new Point2d(10f, 10f)), new ExpansionLocation(1, new Point2d(30f, 30f))),
            List.of(new NeutralFeature("t1", NeutralFeatureType.XELNAGA_TOWER, new Point2d(32f, 32f))),
            List.of(new Point2d(20f, 15f))
        );
    }

    private GameState stateWithMapInfo(MapInfo info) {
        return new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), 0L, info);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=LocationResolverTest -q`
Expected: FAIL — classes don't exist

- [ ] **Step 3: Implement LocationReference**

```java
package io.quarkmind.plugin.coaching;

public sealed interface LocationReference permits
    LocationReference.PlayerBase,
    LocationReference.EnemyBase,
    LocationReference.MapCenter,
    LocationReference.ExpansionOrdinal,
    LocationReference.NearestRamp,
    LocationReference.Watchtower,
    LocationReference.AbsolutePosition {

    record PlayerBase() implements LocationReference {}
    record EnemyBase() implements LocationReference {}
    record MapCenter() implements LocationReference {}
    record ExpansionOrdinal(int ordinal) implements LocationReference {}
    record NearestRamp(LocationReference relativeTo) implements LocationReference {}
    record Watchtower(int index) implements LocationReference {}
    record AbsolutePosition(float x, float y) implements LocationReference {}
}
```

- [ ] **Step 4: Implement LocationResolver**

```java
package io.quarkmind.plugin.coaching;

import io.quarkmind.domain.*;
import java.util.Comparator;
import java.util.List;

public class LocationResolver {

    public Point2d resolve(LocationReference ref, GameState state) {
        if (ref instanceof LocationReference.AbsolutePosition a) {
            return new Point2d(a.x(), a.y());
        }
        if (state.mapInfo() == null) return null;
        MapInfo info = state.mapInfo();
        return switch (ref) {
            case LocationReference.PlayerBase _ -> info.playerStart();
            case LocationReference.EnemyBase _ -> info.enemyStart();
            case LocationReference.MapCenter _ -> new Point2d(info.mapWidth() / 2f, info.mapHeight() / 2f);
            case LocationReference.ExpansionOrdinal e -> expansionByOrdinal(e.ordinal(), info);
            case LocationReference.NearestRamp nr -> nearestRamp(resolve(nr.relativeTo(), state), info);
            case LocationReference.Watchtower w -> watchtowerByIndex(w.index(), info);
            case LocationReference.AbsolutePosition a -> new Point2d(a.x(), a.y());
        };
    }

    private Point2d expansionByOrdinal(int ordinal, MapInfo info) {
        if (ordinal < 0 || ordinal >= info.expansions().size()) return null;
        return info.expansions().get(ordinal).position();
    }

    private Point2d nearestRamp(Point2d relativeTo, MapInfo info) {
        if (relativeTo == null || info.rampPositions().isEmpty()) return null;
        return info.rampPositions().stream()
            .min(Comparator.comparingDouble(r -> r.distanceTo(relativeTo)))
            .orElse(null);
    }

    private Point2d watchtowerByIndex(int index, MapInfo info) {
        List<NeutralFeature> towers = info.neutralFeatures().stream()
            .filter(f -> f.type() == NeutralFeatureType.XELNAGA_TOWER)
            .toList();
        if (index < 0 || index >= towers.size()) return null;
        return towers.get(index).position();
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -Dtest=LocationResolverTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/plugin/coaching/LocationReference.java src/main/java/io/quarkmind/plugin/coaching/LocationResolver.java src/test/java/io/quarkmind/plugin/coaching/LocationResolverTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): LocationReference sealed hierarchy + LocationResolver

Refs #244"
```

---

### Task 5: VerificationPredicate Sealed Hierarchy

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/coaching/VerificationPredicate.java` (sealed interface)
- Create: `src/main/java/io/quarkmind/plugin/coaching/CountDelta.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/MovementDirection.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/ArmyCentroidMovement.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/ExpansionPlacement.java`
- Create: `src/main/java/io/quarkmind/plugin/coaching/UnitsNearLocation.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/CountDeltaTest.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/ArmyCentroidMovementTest.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/ExpansionPlacementTest.java`
- Create: `src/test/java/io/quarkmind/plugin/coaching/UnitsNearLocationTest.java`

**Interfaces:**
- Consumes: `LocationResolver.resolve()`, `LocationReference`, `Point2d.centroidOf()`, `UnitType.isWorker()`, `Positionable`, `GameState` from Tasks 1-4
- Produces: `VerificationPredicate { withBaseline(GameState, LocationResolver), isSatisfied(GameState, LocationResolver) }` — used by CoachingComplianceEvaluator
- Produces: `CountDelta`, `ArmyCentroidMovement`, `ExpansionPlacement`, `UnitsNearLocation` — used by CoachingWorkerFactory parsing

Each predicate test covers: `withBaseline()` captures correct state, `isSatisfied()` returns true when condition met, `isSatisfied()` returns false when not met, edge cases (empty army, null resolve).

Test code is similar in structure for each predicate. Each test uses explicit `GameState` construction with known `Point2d` positions. See the spec §2 for exact field definitions and behavior.

Key implementation details:
- `CountDelta.isSatisfied()`: `currentCount - baselineCount >= expectedDelta`
- `ArmyCentroidMovement.isSatisfied()`: filters `!isWorker()`, computes centroid, checks distance delta. Null centroid → false.
- `ExpansionPlacement.isSatisfied()`: finds town halls NOT in `baselineBaseTags`, checks any within `proximityRadius` of resolved location
- `UnitsNearLocation.isSatisfied()`: counts units (filtered by type or `!isWorker()`) within `radius` of resolved location
- Town-hall types: use a static `Set<BuildingType>` constant shared by `ExpansionPlacement` and `CountDelta` — `NEXUS, COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS, HATCHERY, LAIR, HIVE`

- [ ] Steps follow the same TDD pattern: write test → verify fail → implement → verify pass → commit

Full test and implementation code for each predicate follows the spec's §2 definitions. The commit message:

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): VerificationPredicate sealed hierarchy — CountDelta, ArmyCentroidMovement, ExpansionPlacement, UnitsNearLocation

Refs #244"
```

---

### Task 6: Coaching Records + Evaluator + Broker Evolution

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingAdvice.java` — new record shape
- Modify: `src/main/java/io/quarkmind/plugin/coaching/OpenCommitment.java` — remove baselineCount
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingCompleted.java` — add triggerState
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluator.java` — predicate delegation + LocationResolver injection
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingChannelBroker.java` — baseline capture
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingComplianceEvaluatorTest.java` — migrate + expand
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingAdviceTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingChannelBrokerTest.java`
- Modify: All other files referencing old CoachingAdvice constructor

**Interfaces:**
- Consumes: `VerificationPredicate`, `LocationResolver`, `LocationReference` from Tasks 4-5
- Produces: Updated `CoachingAdvice(advice, domainTag, verification, verificationWindowFrames)`
- Produces: Updated `OpenCommitment(correlationId, advice, issuedAtFrame)` — no baselineCount
- Produces: Updated `CoachingCompleted(workerId, capability, gameFrame, advice, urgencyTier, latencyMs, triggerState)`

**Step sequence:**
1. Rewrite `CoachingAdvice` record — replace 4 nullable verification fields with `VerificationPredicate verification`
2. Rewrite `OpenCommitment` — remove `baselineCount`
3. Add `GameState triggerState` to `CoachingCompleted`
4. Update `CoachingComplianceEvaluator`:
   - Inject `LocationResolver` via constructor
   - Replace `countUnitsOrBuildings()` with `advice.verification().isSatisfied(state, locationResolver)`
   - Delete `countUnitsOrBuildings()` private method
   - Test constructor gains `LocationResolver`
5. Update `CoachingChannelBroker.onCoachingCompleted()`:
   - Inject `LocationResolver` via constructor
   - Call `advice.verification().withBaseline(event.triggerState(), locationResolver)` before creating commitment
   - Construct new `CoachingAdvice` with baselined predicate
6. Fix ALL callers of old `CoachingAdvice` constructor (tests, `CoachingWorkerFactory.parseAdvice()`)
7. Migrate existing `CoachingComplianceEvaluatorTest` — convert count-based tests to use `CountDelta` predicate
8. Add spatial predicate tests through the evaluator: army centroid retreat → ENDORSED, expansion placement → ENDORSED, units near location → ENDORSED

The commit:
```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): coaching record evolution + evaluator/broker predicate delegation

Refs #244"
```

---

### Task 7: LLM Prompt + Parsing Changes

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java` — system prompt + `parseAdvice()`
- Modify: `src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java`

**Interfaces:**
- Consumes: `VerificationPredicate` subtypes, `LocationReference` permits, `CoachingAdvice` new shape from Tasks 5-6
- Produces: Updated `buildSystemPrompt()` with verification type vocabulary
- Produces: Updated `parseAdvice()` dispatching on `verificationType` field

**Step sequence:**
1. Write tests for `parseAdvice()` with new `verificationType` format:
   - `ARMY_CENTROID_RETREAT` with `referenceLocation: ENEMY_BASE`
   - `EXPANSION_PLACEMENT` with `expansionOrdinal: 2`
   - `UNITS_NEAR_LOCATION` with `location: NATURAL`, `radius: 10.0`
   - `COUNT_DELTA` with `unitType: STALKER`, `expectedDelta: 3`
   - Missing/null `verificationType` → null verification (non-verifiable)
   - Legacy flat format (no `verificationType`, has `verificationUnitType`) → `CountDelta` backward compat
2. Write tests for `buildSystemPrompt()`:
   - Output contains `verificationType` vocabulary
   - Output contains location token vocabulary
3. Implement `parseAdvice()` dispatch:
   - Read `verificationType` string from JSON
   - Switch on it to construct appropriate predicate
   - Parse `verificationParams` sub-object for mode-specific fields
   - Parse location tokens to `LocationReference` permits
   - Fall back to legacy parsing if no `verificationType` but has `verificationUnitType`
4. Update `buildSystemPrompt()`:
   - Add verification type table to the prompt
   - Add location token table
   - Add reference semantics explanation (FROM convention)

The commit:
```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): LLM prompt + parsing for spatial verification types

Refs #244"
```

---

### Task 8: ObservationTranslator + Real SC2 / Emulated Wiring

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` — extract resources + neutral features
- Modify: `src/main/java/io/quarkmind/sc2/emulated/server/EmulatedSC2Server.java` — add start locations to `startRaw`
- Modify: `src/test/java/io/quarkmind/sc2/real/ObservationTranslatorTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/real/ObservationTranslatorIntegrationTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/server/EmulatedSC2ServerTest.java` (if exists)

**Interfaces:**
- Consumes: `Resource`, `NeutralFeature`, `NeutralFeatureType`, `MapInfo` from Tasks 1-3
- Produces: `ObservationTranslator.translate(Observation obs, MapInfo mapInfo)` — widened signature passing MapInfo through to GameState

**Step sequence:**
1. Write test: `translate_extractsNeutralMineralFieldsAsResources` — verify mineral patches populated
2. Write test: `translate_extractsNeutralGeysersAsResources` — verify geysers populated
3. Write test: `translate_extractsXelNagaTowersAsNeutralFeatures`
4. Implement: extend `translate()` to filter `Alliance.NEUTRAL` units by unit type:
   - Mineral types (`NEUTRAL_MINERAL_FIELD`, etc.) → `Resource` records into `mineralPatches`
   - Geyser types (`NEUTRAL_VESPENE_GEYSER`, etc.) → `Resource` records into `geysers`
   - Watchtower (`NEUTRAL_XELNAGATOWER`) → `NeutralFeature`
   - Destructible types → `NeutralFeature`
5. Widen `translate()` signature to accept `MapInfo mapInfo` parameter, pass to GameState constructor
6. Update all callers of `translate()` to pass mapInfo (or null for tests)
7. Update `ObservationTranslatorIntegrationTest` calls

8. Update `EmulatedSC2Server.buildGameInfo()` to add start locations to `startRaw` — player start `(8,8)`, enemy start `(56,56)` matching the emulated 64×64 map layout

The commit:
```bash
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#244): ObservationTranslator + EmulatedSC2Server wiring for spatial data

Closes #244"
```

---

### Post-implementation: Calibration Test (optional, benchmark profile)

`ExpansionLocationCalibrationTest` — validate the 12.0 clustering radius against IEM10 replays (correct expansion count per map). Run with `mvn test -Pbenchmark`. This is calibration, not a blocker for the feature landing.

---

## Dependency Graph

```
Task 1 (Positionable, centroidOf, isWorker)
  ↓
Task 2 (NeutralFeature, ExpansionLocation, rampPositions)
  ↓
Task 3 (MapInfo, GameState widening)  ← highest impact, most files touched
  ↓
Task 4 (LocationReference, LocationResolver)
  ↓
Task 5 (VerificationPredicate hierarchy)
  ↓
Task 6 (Coaching records + evaluator + broker)
  ↓
Task 7 (LLM prompt + parsing)

Task 3 → Task 8 (ObservationTranslator wiring) — parallel with Tasks 4-7
```

Tasks 1-7 are strictly sequential. Task 8 can run after Task 3 in parallel with Tasks 4-7 if desired, but sequential execution is simpler.
