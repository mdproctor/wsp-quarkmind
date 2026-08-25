# Cascade Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #213 — IEM10 replay validation & accuracy benchmarking
**Issue group:** #213

**Goal:** Wire ONNX strategy classifier models into the three-tier cascade,
validate accuracy against 59 replay games, and establish latency benchmarks.

**Architecture:** Rewrite `StrategyFeatureExtractor` to produce 269×10 temporal
+ 6 map features matching the neocortex training pipeline. Update
`CascadingPatternClassifier` to route to per-race ONNX models (vs_terran,
vs_zerg, vs_protoss). Validate accuracy in three modes (Drools-only, ONNX-only,
cascade) against IEM10 + AI Arena replays at minutes 1–5.

**Tech Stack:** Java 21, Quarkus, ONNX Runtime (via neocortex inference-runtime),
Drools, Micrometer, JUnit 5

## Global Constraints

- Domain model (`io.quarkmind.domain`) must remain plain Java — no CDI, no Quarkus imports
- All new test classes use `@Tag("benchmark")` and run via `mvn test -Pbenchmark`
- ONNX model artifacts copied from neocortex test resources, not bundled in production jar
- Feature vector layout must match `sc2egset_extractor.py` exactly — the ONNX models are trained on that layout
- Normalization stats exported from `norm_stats.npz` as JSON resource file
- `cascadingClassifier.reset()` between every game in validation tests

---

## Batch 1: Data Foundation

### Task 1: Extend GameState with economy stats and upgrade tracking

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/PlayerEconomyStats.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/GameState.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/GameStateTest.java`

**Interfaces:**
- Produces: `PlayerEconomyStats` record with 13 economy fields (mineralsCurrent, vespeneCurrent, mineralsCollectionRate, vespeneCollectionRate, foodMade, foodUsed, workersActiveCount, mineralsUsedCurrentArmy, mineralsUsedCurrentEconomy, mineralsUsedCurrentTechnology, vespeneUsedCurrentArmy, vespeneUsedCurrentEconomy, vespeneUsedCurrentTechnology)
- Produces: `GameState` gains `playerEconomy()`, `enemyEconomy()`, `playerUpgrades()`, `enemyUpgrades()` fields

- [ ] **Step 1: Write failing test for PlayerEconomyStats**

```java
// GameStateTest.java — add test
@Test
void economyStats_carriedOnGameState() {
    var eco = new PlayerEconomyStats(400, 200, 1200, 400, 46, 38, 16,
            2000, 1500, 500, 800, 400, 200);
    var gs = new GameState(400, 200, 46, 38,
            List.of(), List.of(), List.of(), List.of(), List.of(),
            List.of(), List.of(), 5000, null,
            eco, PlayerEconomyStats.EMPTY, Set.of(), Set.of());
    assertThat(gs.playerEconomy().mineralsCollectionRate()).isEqualTo(1200);
    assertThat(gs.playerEconomy().workersActiveCount()).isEqualTo(16);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameStateTest#economyStats_carriedOnGameState -q`
Expected: FAIL — `PlayerEconomyStats` class not found

- [ ] **Step 3: Create PlayerEconomyStats record**

```java
// PlayerEconomyStats.java
package io.quarkmind.domain;

public record PlayerEconomyStats(
    int mineralsCurrent,
    int vespeneCurrent,
    int mineralsCollectionRate,
    int vespeneCollectionRate,
    int foodMade,
    int foodUsed,
    int workersActiveCount,
    int mineralsUsedCurrentArmy,
    int mineralsUsedCurrentEconomy,
    int mineralsUsedCurrentTechnology,
    int vespeneUsedCurrentArmy,
    int vespeneUsedCurrentEconomy,
    int vespeneUsedCurrentTechnology
) {
    public static final PlayerEconomyStats EMPTY = new PlayerEconomyStats(
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0);

    public float[] toFeatureVector() {
        return new float[] {
            mineralsCurrent / 1000.0f,
            vespeneCurrent / 1000.0f,
            mineralsCollectionRate / 1000.0f,
            vespeneCollectionRate / 1000.0f,
            foodMade / 1000.0f,
            foodUsed / 1000.0f,
            workersActiveCount / 1000.0f,
            mineralsUsedCurrentArmy / 1000.0f,
            mineralsUsedCurrentEconomy / 1000.0f,
            mineralsUsedCurrentTechnology / 1000.0f,
            vespeneUsedCurrentArmy / 1000.0f,
            vespeneUsedCurrentEconomy / 1000.0f,
            vespeneUsedCurrentTechnology / 1000.0f
        };
    }
}
```

- [ ] **Step 4: Extend GameState record with economy + upgrade fields**

Add four new fields to the `GameState` record:

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
    MapInfo mapInfo,
    PlayerEconomyStats playerEconomy,    // NEW
    PlayerEconomyStats enemyEconomy,     // NEW
    Set<String> playerUpgrades,          // NEW
    Set<String> enemyUpgrades            // NEW
) { ... }
```

Update the compact constructor to copy the sets. Update ALL existing callers of the GameState constructor to pass `PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY, Set.of(), Set.of()` for the new fields. Use `ide_find_references` on the GameState constructor to find all call sites.

- [ ] **Step 5: Add economy/upgrade storage to SimulatedGame base class**

Add protected fields to `SimulatedGame`:
- `PlayerEconomyStats playerEconomy = PlayerEconomyStats.EMPTY`
- `PlayerEconomyStats enemyEconomy = PlayerEconomyStats.EMPTY`
- `Set<String> playerUpgrades = new HashSet<>()`
- `Set<String> enemyUpgrades = new HashSet<>()`

Add protected setters. Update `snapshot()` to include these in the `GameState`
constructor call. Reset in the existing `reset()` method.

- [ ] **Step 6: Wire economy stats into IEM10JsonSimulatedGame**

In `IEM10JsonSimulatedGame.applyPlayerStats()`, extract the additional 9 economy
fields from the JSON PlayerStats tracker event (they exist in the data but are
currently discarded). Call the base class setters to store them.

Add `UpgradeEvent` processing: when a tracker event with `_event == "UpgradeEvent"`
appears, add the upgrade name to the appropriate player's upgrade set.

Also add `UpgradeEvent` tracker event processing: when an `UpgradeEvent` appears
in the JSON, add the upgrade name to a `Set<String>` tracked per player.

- [ ] **Step 6: Wire economy stats into ReplaySimulatedGame**

In `ReplaySimulatedGame`, the Scelight `PlayerStats` tracker event already carries
the full stats. Extract the additional 9 economy fields. For upgrades, process
`UpgradeCompleteEvent` from Scelight tracker events.

- [ ] **Step 7: Run all tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — all existing tests pass with the new fields defaulting to EMPTY/empty

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): extend GameState with economy stats + upgrade tracking for ONNX features"
```

---

### Task 2: Feature infrastructure — WindowSnapshot, TemporalWindowAccumulator, MapCharacteristics

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/WindowSnapshot.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/MapCharacteristics.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java`
- Create: `quarkmind-sc2/src/main/resources/classifier/norm_stats.json`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulatorTest.java`

**Interfaces:**
- Produces: `WindowSnapshot(float[] playerFeatures, float[] opponentFeatures, float scoutingVisibility)` — 134+134+1 per tick
- Produces: `TemporalWindowAccumulator.addSnapshot(WindowSnapshot)`, `getWindowedFeatures()` → 10 × 269
- Produces: `MapCharacteristics(float rushDistance, float expansions, float size, float choke)` with `toArray(hasPlayer, hasOpponent)` → float[6]
- Produces: `FeatureIndexMaps` — static maps from UnitType/BuildingType enum → feature index

- [ ] **Step 1: Write failing test for TemporalWindowAccumulator**

```java
// TemporalWindowAccumulatorTest.java
package io.quarkmind.plugin.scouting;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class TemporalWindowAccumulatorTest {

    @Test
    void emptyAccumulator_returnsZeroPaddedWindows() {
        var acc = new TemporalWindowAccumulator();
        var windows = acc.getWindowedFeatures();
        assertThat(windows).hasSize(10);
        for (float[] w : windows) {
            assertThat(w).hasSize(269);
            assertThat(w).containsOnly(0.0f);
        }
    }

    @Test
    void twoMinutesOfTicks_populatesFourWindows() {
        var acc = new TemporalWindowAccumulator();
        // 2 minutes = 120 seconds ≈ 120 ticks at ~1s per tick
        for (int i = 0; i < 120; i++) {
            var player = new float[134];
            player[0] = 1.0f; // one building
            var opponent = new float[134];
            acc.addSnapshot(new WindowSnapshot(player, opponent, 0.5f));
        }
        var windows = acc.getWindowedFeatures();
        assertThat(windows).hasSize(10);
        // First 4 windows should be non-zero
        assertThat(windows.get(0)[0]).isGreaterThan(0);
        assertThat(windows.get(3)[0]).isGreaterThan(0);
        // Windows 4-9 should be zero-padded
        assertThat(windows.get(4)).containsOnly(0.0f);
        assertThat(windows.get(9)).containsOnly(0.0f);
    }

    @Test
    void scoutingMask_appliedToOpponentFeatures() {
        var acc = new TemporalWindowAccumulator();
        for (int i = 0; i < 30; i++) { // one window
            var player = new float[134];
            var opponent = new float[134];
            opponent[53] = 10.0f; // 10 marines
            acc.addSnapshot(new WindowSnapshot(player, opponent, 0.3f));
        }
        var windows = acc.getWindowedFeatures();
        // Opponent marines at index 134+53 = 187, masked by 0.3
        assertThat(windows.get(0)[187]).isCloseTo(3.0f, within(0.1f));
        // has_vision flag at index 268
        assertThat(windows.get(0)[268]).isEqualTo(1.0f);
    }

    @Test
    void reset_clearsAllState() {
        var acc = new TemporalWindowAccumulator();
        acc.addSnapshot(new WindowSnapshot(new float[134], new float[134], 0.5f));
        acc.reset();
        var windows = acc.getWindowedFeatures();
        assertThat(windows.get(0)).containsOnly(0.0f);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalWindowAccumulatorTest -q`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement WindowSnapshot record**

```java
package io.quarkmind.plugin.scouting;

public record WindowSnapshot(
    float[] playerFeatures,   // 134 elements
    float[] opponentFeatures, // 134 elements
    float scoutingVisibility  // [0, 1]
) {}
```

- [ ] **Step 4: Implement TemporalWindowAccumulator**

```java
package io.quarkmind.plugin.scouting;

import java.util.ArrayList;
import java.util.List;

public class TemporalWindowAccumulator {
    static final int MAX_WINDOWS = 10;
    static final int WINDOW_SECONDS = 30;
    static final int TICKS_PER_WINDOW = 30; // 30s / ~1s per tick (SC2Data.LOOPS_PER_TICK=22, ~22.4 loops/s)
    static final int FEATURES_PER_PLAYER = 134;
    static final int FEATURES_PER_WINDOW = 2 * FEATURES_PER_PLAYER + 1; // 269

    private final List<WindowSnapshot> tickSnapshots = new ArrayList<>();

    public void addSnapshot(WindowSnapshot snapshot) {
        tickSnapshots.add(snapshot);
    }

    public List<float[]> getWindowedFeatures() {
        List<float[]> result = new ArrayList<>(MAX_WINDOWS);
        for (int w = 0; w < MAX_WINDOWS; w++) {
            int startTick = w * TICKS_PER_WINDOW;
            int endTick = Math.min(startTick + TICKS_PER_WINDOW, tickSnapshots.size());
            if (startTick >= tickSnapshots.size()) {
                result.add(new float[FEATURES_PER_WINDOW]);
                continue;
            }
            float[] window = new float[FEATURES_PER_WINDOW];
            int count = endTick - startTick;
            boolean anyVision = false;
            for (int t = startTick; t < endTick; t++) {
                var snap = tickSnapshots.get(t);
                for (int f = 0; f < FEATURES_PER_PLAYER; f++) {
                    window[f] += snap.playerFeatures()[f];
                    window[FEATURES_PER_PLAYER + f] +=
                        snap.opponentFeatures()[f] * snap.scoutingVisibility();
                }
                if (snap.scoutingVisibility() > 0) anyVision = true;
            }
            for (int f = 0; f < 2 * FEATURES_PER_PLAYER; f++) {
                window[f] /= count;
            }
            window[FEATURES_PER_WINDOW - 1] = anyVision ? 1.0f : 0.0f;
            result.add(window);
        }
        return result;
    }

    public void reset() {
        tickSnapshots.clear();
    }
}
```

- [ ] **Step 5: Implement MapCharacteristics**

```java
package io.quarkmind.plugin.scouting;

import java.util.Map;

public record MapCharacteristics(
    float rushDistance, float expansions, float size, float choke
) {
    public static final MapCharacteristics DEFAULT =
        new MapCharacteristics(0.5f, 0.5f, 0.5f, 0.5f);

    public float[] toArray(boolean hasPlayer, boolean hasOpponent) {
        return new float[] {
            rushDistance, expansions, size, choke,
            hasPlayer ? 1.0f : 0.0f,
            hasOpponent ? 1.0f : 0.0f
        };
    }

    // 15 known maps from neocortex MAP_CATALOG
    private static final Map<String, MapCharacteristics> CATALOG = Map.ofEntries(
        Map.entry("Abyssal Reef", new MapCharacteristics(0.5f, 0.8f, 1.0f, 1.0f)),
        Map.entry("Acolyte", new MapCharacteristics(0.0f, 0.6f, 0.0f, 1.0f)),
        Map.entry("Ascension to Aiur", new MapCharacteristics(0.5f, 0.8f, 1.0f, 1.0f)),
        Map.entry("Catalyst", new MapCharacteristics(0.0f, 0.6f, 0.5f, 0.0f)),
        Map.entry("Dusk Towers", new MapCharacteristics(0.5f, 0.6f, 0.5f, 1.0f)),
        Map.entry("Frost", new MapCharacteristics(0.5f, 0.8f, 1.0f, 1.0f)),
        Map.entry("Habitation Station", new MapCharacteristics(0.0f, 0.4f, 0.0f, 1.0f)),
        Map.entry("King Sejong Station", new MapCharacteristics(0.5f, 0.6f, 0.5f, 1.0f)),
        Map.entry("Newkirk Precinct", new MapCharacteristics(0.5f, 0.6f, 0.5f, 1.0f)),
        Map.entry("Odyssey", new MapCharacteristics(1.0f, 0.8f, 1.0f, 0.0f)),
        Map.entry("Paladino Terminal", new MapCharacteristics(0.0f, 0.4f, 0.0f, 1.0f)),
        Map.entry("Proxima Station", new MapCharacteristics(0.5f, 0.6f, 0.5f, 0.0f)),
        Map.entry("Sequencer", new MapCharacteristics(0.0f, 0.6f, 0.5f, 1.0f)),
        Map.entry("Whirlwind", new MapCharacteristics(1.0f, 0.8f, 1.0f, 0.0f)),
        Map.entry("Worldship", new MapCharacteristics(0.5f, 0.8f, 1.0f, 1.0f))
    );

    public static MapCharacteristics forMap(String mapName) {
        return CATALOG.getOrDefault(mapName, DEFAULT);
    }

    // For replay validation: map name is not available in MapInfo.
    // Use DEFAULT for replays. For live SC2, derive from SC2 API map name.
    // Future: compute from MapInfo geometry (expansion count, start-to-start distance).
}
```

- [ ] **Step 6: Implement FeatureIndexMaps**

Static lookup maps from `UnitType`/`BuildingType` → feature index, matching the
exact ordering from `sc2egset_extractor.py` (53 buildings, 53 units). Include
the upgrade name list (15 entries). Units/buildings not in the map are ignored.

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.UnitType;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

public final class FeatureIndexMaps {
    static final int N_BUILDINGS = 53;
    static final int N_UNITS = 53;
    static final int N_STATS = 13;
    static final int N_UPGRADES = 15;
    static final int N_FEATURES_PER_PLAYER = N_BUILDINGS + N_UNITS + N_STATS + N_UPGRADES;

    // Building enum → feature index [0..52]
    static final Map<BuildingType, Integer> BUILDING_INDEX = buildBuildingIndex();
    // Unit enum → feature index [53..105]
    static final Map<UnitType, Integer> UNIT_INDEX = buildUnitIndex();
    // Upgrade name → feature index [119..133]
    static final List<String> UPGRADE_NAMES = List.of(
        "Stimpack", "ShieldWall", "PunisherGrenades", "BansheeCloak",
        "TerranVehicleWeaponsLevel1", "PersonalCloaking", "DrillClaws",
        "zerglingmovementspeed", "GlialReconstitution", "CentrificalHooks",
        "Burrow", "WarpGateResearch", "BlinkTech", "Charge", "AdeptPiercingAttack"
    );

    private static Map<BuildingType, Integer> buildBuildingIndex() {
        // Map each BuildingType to its index in the 53-building feature list
        // Ordering matches sc2egset_extractor.py BUILDINGS list exactly
        var map = new EnumMap<BuildingType, Integer>(BuildingType.class);
        // ... map entries for all 53 buildings matching the Python order
        return Map.copyOf(map);
    }

    private static Map<UnitType, Integer> buildUnitIndex() {
        // Map each UnitType to its index in the 53-unit feature list
        // Ordering matches sc2egset_extractor.py UNITS list exactly
        var map = new EnumMap<UnitType, Integer>(UnitType.class);
        // ... map entries for all 53 units matching the Python order
        return Map.copyOf(map);
    }

    private FeatureIndexMaps() {}
}
```

The implementer must populate the maps with every entry from the building list
(53) and unit list (53) in the spec, matching the exact Python ordering. Use
`ide_find_class` to find the Java `BuildingType` and `UnitType` enums and map
each Python name to its Java equivalent.

- [ ] **Step 7: Export normalization stats**

Convert `norm_stats.npz` from neocortex to JSON format. Write a Python script
to `/tmp/export_norm.py`:

```python
import numpy as np, json
data = np.load('path/to/norm_stats.npz')
result = {"mean": data['mean'].tolist(), "std": data['std'].tolist()}
with open('norm_stats.json', 'w') as f:
    json.dump(result, f)
```

Place output at `quarkmind-sc2/src/main/resources/classifier/norm_stats.json`.

- [ ] **Step 8: Run tests and commit**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalWindowAccumulatorTest -q`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): feature infrastructure — WindowSnapshot, accumulator, MapCharacteristics, norm stats"
```

---

### Task 3: Rewrite StrategyFeatureExtractor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractor.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `TemporalWindowAccumulator.getWindowedFeatures()` → `List<float[]>` (10 × 269)
- Consumes: `MapCharacteristics.toArray(boolean, boolean)` → `float[6]`
- Consumes: `norm_stats.json` resource for z-score normalization
- Produces: `StrategyFeatures(Map<String, float[][]> tensors)` — top-level type in `io.quarkmind.plugin.scouting`, `"temporal"` → `[1][2690]`, `"map"` → `[1][6]`

- [ ] **Step 1: Write failing tests**

```java
// StrategyFeatureExtractorTest.java
package io.quarkmind.plugin.scouting;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class StrategyFeatureExtractorTest {

    @Test
    void extract_producesCorrectTensorDimensions() {
        var extractor = new StrategyFeatureExtractor();
        var accumulator = new TemporalWindowAccumulator();
        // Populate 6 windows (3 minutes ≈ 180 ticks at ~1s/tick)
        for (int i = 0; i < 180; i++) {
            accumulator.addSnapshot(new WindowSnapshot(
                new float[134], new float[134], 0.5f));
        }
        var result = extractor.extract(
            accumulator.getWindowedFeatures(),
            MapCharacteristics.DEFAULT);
        assertThat(result.tensors()).containsKeys("temporal", "map");
        assertThat(result.tensors().get("temporal")).hasNumberOfRows(1);
        assertThat(result.tensors().get("temporal")[0]).hasSize(2690);
        assertThat(result.tensors().get("map")).hasNumberOfRows(1);
        assertThat(result.tensors().get("map")[0]).hasSize(6);
    }

    @Test
    void extract_normalizesTemporalFeatures() {
        var extractor = new StrategyFeatureExtractor();
        var accumulator = new TemporalWindowAccumulator();
        for (int i = 0; i < 30; i++) {
            var player = new float[134];
            player[0] = 1.0f; // one building
            accumulator.addSnapshot(new WindowSnapshot(
                player, new float[134], 0.0f));
        }
        var result = extractor.extract(
            accumulator.getWindowedFeatures(),
            MapCharacteristics.DEFAULT);
        // Feature at index 0 should be normalized (not raw 1.0)
        float raw = result.tensors().get("temporal")[0][0];
        assertThat(raw).isNotEqualTo(1.0f);
    }

    @Test
    void extract_hasVisionFlagNotNormalized() {
        var extractor = new StrategyFeatureExtractor();
        var accumulator = new TemporalWindowAccumulator();
        for (int i = 0; i < 30; i++) {
            accumulator.addSnapshot(new WindowSnapshot(
                new float[134], new float[134], 1.0f));
        }
        var result = extractor.extract(
            accumulator.getWindowedFeatures(),
            MapCharacteristics.DEFAULT);
        // has_vision at index 268 within first window = index 268
        assertThat(result.tensors().get("temporal")[0][268]).isEqualTo(1.0f);
    }

    @Test
    void extract_mapFeaturesIncludeAvailabilityFlags() {
        var extractor = new StrategyFeatureExtractor();
        var accumulator = new TemporalWindowAccumulator();
        for (int i = 0; i < 30; i++) {
            var player = new float[134];
            player[0] = 1.0f;
            accumulator.addSnapshot(new WindowSnapshot(
                player, new float[134], 0.0f));
        }
        var result = extractor.extract(
            accumulator.getWindowedFeatures(),
            MapCharacteristics.DEFAULT);
        float[] map = result.tensors().get("map")[0];
        assertThat(map[4]).isEqualTo(1.0f);  // has_player
        assertThat(map[5]).isEqualTo(0.0f);  // has_opponent (all zeros)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyFeatureExtractorTest -q`
Expected: FAIL — method signatures don't match

- [ ] **Step 3: Rewrite StrategyFeatureExtractor**

Move `StrategyFeatures` out of `StrategyFeatureExtractor` into its own top-level
file at `io.quarkmind.plugin.scouting.StrategyFeatures`. It crosses component
boundaries (used by both extractor and classifier).

Replace the entire class body. The new `extract()` takes windowed features
(from `TemporalWindowAccumulator.getWindowedFeatures()`) and map characteristics.

```java
public class StrategyFeatureExtractor {

    public record StrategyFeatures(Map<String, float[][]> tensors) {}

    private final float[] normMean;
    private final float[] normStd;

    public StrategyFeatureExtractor() {
        // Load norm_stats.json from classpath
        // Parse mean[] and std[] arrays
        // has_vision index (268 per window) forced to mean=0, std=1
    }

    public StrategyFeatures extract(List<float[]> windowedFeatures,
                                     MapCharacteristics map) {
        // 1. Flatten 10 × 269 into [1][2690]
        float[] temporal = new float[2690];
        for (int w = 0; w < 10; w++) {
            System.arraycopy(windowedFeatures.get(w), 0,
                temporal, w * 269, 269);
        }
        // 2. Compute availability flags BEFORE normalization (R1-04)
        boolean hasPlayer = hasNonZeroBlock(temporal, 0, 134);
        boolean hasOpponent = hasNonZeroBlock(temporal, 134, 268);
        // 3. Apply z-score normalization — ONLY to populated windows (R1-03)
        //    Zero-padded windows must stay all-zero for the model's padding mask
        for (int w = 0; w < 10; w++) {
            int base = w * 269;
            boolean isPopulated = false;
            for (int f = 0; f < 269; f++) {
                if (temporal[base + f] != 0.0f) { isPopulated = true; break; }
            }
            if (!isPopulated) continue; // skip zero-padded windows
            for (int f = 0; f < 269; f++) {
                if (f == 268) continue; // has_vision — not normalized
                if (normStd[f] > 0) {
                    temporal[base + f] = (temporal[base + f] - normMean[f]) / normStd[f];
                }
            }
        }
        // 4. Build map features
        float[] mapFeatures = map.toArray(hasPlayer, hasOpponent);

        return new StrategyFeatures(Map.of(
            "temporal", new float[][] { temporal },
            "map", new float[][] { mapFeatures }
        ));
    }

    private boolean hasNonZeroBlock(float[] data, int start, int end) {
        // Check across all windows for any nonzero in the range
        for (int w = 0; w < 10; w++) {
            for (int f = start; f < end; f++) {
                if (data[w * 269 + f] != 0.0f) return true;
            }
        }
        return false;
    }
}
```

- [ ] **Step 4: Update callers of the old extract() signature**

Use `ide_find_references` on the old `extract(List<EnemyUnitFirstSeen>, double)`
method to find all callers. Update `DroolsScoutingTask` and any test callers.
The old single-key map return is replaced by `StrategyFeatures`.

- [ ] **Step 5: Run tests and commit**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyFeatureExtractorTest -q`
Expected: PASS

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS (or identify broken callers from step 4)

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): rewrite StrategyFeatureExtractor — temporal windowing + normalization"
```

---

## Batch 2: Cascade Wiring

### Task 4: StrategyArchetype new values + ONNX label mapping + per-race cascade

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyArchetype.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/OnnxLabelMapping.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadingPatternClassifier.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadingPatternClassifierTest.java`

**Interfaces:**
- Consumes: `Race` enum, `TensorClassifier.classify(Map<String, float[][]>)` → `ClassificationResult`
- Produces: `OnnxLabelMapping.resolve(String label, Race race)` → `StrategyArchetype`
- Produces: `CascadingPatternClassifier.classify(evidence, revisions, StrategyFeatures, Race, frame, prevFrame, ctx)` → `CascadeResult`

- [ ] **Step 1: Add new StrategyArchetype enum values**

Only 4 genuinely new values needed (others already exist or map to existing):

```java
// Add after existing EARLY entries:
PROTOSS_DT_RUSH(Race.PROTOSS, GamePhase.EARLY, ArchetypeCategory.RUSH),

// Add in MID section:
TERRAN_AIR_SUPERIORITY(Race.TERRAN, GamePhase.MID, ArchetypeCategory.COMPOSITION),
ZERG_HYDRA_PUSH(Race.ZERG, GamePhase.MID, ArchetypeCategory.TIMING),
PROTOSS_COLOSSUS_PUSH(Race.PROTOSS, GamePhase.MID, ArchetypeCategory.TIMING),
PROTOSS_AIR_SUPERIORITY(Race.PROTOSS, GamePhase.MID, ArchetypeCategory.COMPOSITION),
```

Note: `PROTOSS_DT_HARASS` already exists — `PROTOSS_DT_RUSH` is a distinct
archetype (all-in vs harassment). `ZERG_MUTALISK_HARASS` already exists and maps
from ONNX label "MUTA_HARASS". `TERRAN_BIO_TIMING`, `TERRAN_MECH_PUSH`,
`PROTOSS_BLINK_STALKER`, `PROTOSS_CANNON_RUSH`, `ZERG_LING_BANE` already exist.

- [ ] **Step 2: Write failing test for OnnxLabelMapping**

```java
@Test
void resolve_terranRush_mapsToBroadestArchetype() {
    assertThat(OnnxLabelMapping.resolve("RUSH", Race.TERRAN))
        .isEqualTo(StrategyArchetype.TERRAN_MARINE_RUSH);
}

@Test
void resolve_zergMutaHarass_mapsToExistingMutaliskHarass() {
    assertThat(OnnxLabelMapping.resolve("MUTA_HARASS", Race.ZERG))
        .isEqualTo(StrategyArchetype.ZERG_MUTALISK_HARASS);
}

@Test
void resolve_unknownLabel_returnsNull() {
    assertThat(OnnxLabelMapping.resolve("NONEXISTENT", Race.TERRAN)).isNull();
}
```

- [ ] **Step 3: Implement OnnxLabelMapping**

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.Race;
import io.quarkmind.domain.StrategyArchetype;
import java.util.List;
import java.util.Map;

public final class OnnxLabelMapping {
    // Per-race label lists matching neocortex post-consolidation classes
    public static final List<String> VS_TERRAN_LABELS = List.of(
        "RUSH", "BANSHEE_HARASS", "AIR_SUPERIORITY", "MECH_PUSH", "BIO_TIMING");
    public static final List<String> VS_ZERG_LABELS = List.of(
        "RUSH", "ROACH_RUSH", "LING_BANE", "MUTA_HARASS", "HYDRA_PUSH", "MACRO_ECONOMY");
    public static final List<String> VS_PROTOSS_LABELS = List.of(
        "RUSH", "PROXY", "CANNON_RUSH", "DT_RUSH", "BLINK_STALKER", "COLOSSUS_PUSH",
        "AIR_SUPERIORITY");

    private static final Map<String, StrategyArchetype> TERRAN_MAP = Map.of(
        "RUSH", StrategyArchetype.TERRAN_MARINE_RUSH,
        "BANSHEE_HARASS", StrategyArchetype.TERRAN_BANSHEE_HARASS,
        "AIR_SUPERIORITY", StrategyArchetype.TERRAN_AIR_SUPERIORITY,
        "MECH_PUSH", StrategyArchetype.TERRAN_MECH_PUSH,
        "BIO_TIMING", StrategyArchetype.TERRAN_BIO_TIMING);

    private static final Map<String, StrategyArchetype> ZERG_MAP = Map.of(
        "RUSH", StrategyArchetype.ZERG_ZERGLING_RUSH,
        "ROACH_RUSH", StrategyArchetype.ZERG_ROACH_RUSH,
        "LING_BANE", StrategyArchetype.ZERG_LING_BANE,
        "MUTA_HARASS", StrategyArchetype.ZERG_MUTALISK_HARASS,
        "HYDRA_PUSH", StrategyArchetype.ZERG_HYDRA_PUSH,
        "MACRO_ECONOMY", StrategyArchetype.ZERG_MACRO);

    private static final Map<String, StrategyArchetype> PROTOSS_MAP = Map.ofEntries(
        Map.entry("RUSH", StrategyArchetype.PROTOSS_GATEWAY_RUSH),
        Map.entry("PROXY", StrategyArchetype.PROTOSS_PROXY_GATE),
        Map.entry("CANNON_RUSH", StrategyArchetype.PROTOSS_CANNON_RUSH),
        Map.entry("DT_RUSH", StrategyArchetype.PROTOSS_DT_RUSH),
        Map.entry("BLINK_STALKER", StrategyArchetype.PROTOSS_BLINK_STALKER),
        Map.entry("COLOSSUS_PUSH", StrategyArchetype.PROTOSS_COLOSSUS_PUSH),
        Map.entry("AIR_SUPERIORITY", StrategyArchetype.PROTOSS_AIR_SUPERIORITY));

    public static StrategyArchetype resolve(String label, Race race) {
        return switch (race) {
            case TERRAN  -> TERRAN_MAP.get(label);
            case ZERG    -> ZERG_MAP.get(label);
            case PROTOSS -> PROTOSS_MAP.get(label);
        };
    }

    public static List<String> labelsForRace(Race race) {
        return switch (race) {
            case TERRAN  -> VS_TERRAN_LABELS;
            case ZERG    -> VS_ZERG_LABELS;
            case PROTOSS -> VS_PROTOSS_LABELS;
        };
    }

    private OnnxLabelMapping() {}
}
```

- [ ] **Step 4: Update CascadingPatternClassifier for per-race models**

Replace the single `@Inference("strategy-classifier")` injection with three:

```java
@Inject
public CascadingPatternClassifier(
    @ConfigProperty(...) double droolsThreshold,
    @ConfigProperty(...) double onnxThreshold,
    @Inference("strategy-vs-terran") Instance<InferenceModel> terranModel,
    @Inference("strategy-vs-zerg") Instance<InferenceModel> zergModel,
    @Inference("strategy-vs-protoss") Instance<InferenceModel> protossModel)
```

Store three `TensorClassifier` instances in an `EnumMap<Race, TensorClassifier>`.
Each initialized with `OnnxLabelMapping.labelsForRace(race)` as labels.

Update `classify()` signature to accept `StrategyFeatures onnxFeatures` and
`Race enemyRace`. In the ONNX tier, select classifier by race. Use
`OnnxLabelMapping.resolve()` instead of `StrategyArchetype.valueOf()` for the
ONNX result label.

Update test constructors similarly — add a `Race → TensorClassifier` map
parameter.

- [ ] **Step 5: Update existing CascadingPatternClassifierTest**

Update all test methods for the new `classify()` signature. Each test call now
passes `null` for `onnxFeatures` and `Race.PROTOSS` for race (arbitrary — tests
exercise Drools tier which is race-agnostic). Update the `onnxResolvesWhenDroolsBelowThreshold` test to use per-race labels.

- [ ] **Step 6: Run tests and commit**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CascadingPatternClassifierTest -q`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): per-race ONNX cascade — OnnxLabelMapping + 5 new StrategyArchetype values"
```

---

### Task 5: Wire DroolsScoutingTask + ONNX model config + update tests

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java`
- Modify: `quarkmind-sc2/src/main/resources/application.properties`
- Copy: neocortex ONNX models → `quarkmind-sc2/src/test/resources/models/strategy/`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java`

**Interfaces:**
- Consumes: `CascadingPatternClassifier.classify(evidence, revisions, StrategyFeatures, Race, frame, prevFrame, ctx)`
- Consumes: `TemporalWindowAccumulator`, `StrategyFeatureExtractor`

- [ ] **Step 1: Add TemporalWindowAccumulator to DroolsScoutingTask**

Add `private final TemporalWindowAccumulator windowAccumulator = new TemporalWindowAccumulator();` field. Reset in `resetDispatchState()`. In `execute()`, after processing enemy units through `ScoutingSessionManager`, build a `WindowSnapshot` from the current `GameState` (using `FeatureIndexMaps` to convert units/buildings to feature vectors) and call `windowAccumulator.addSnapshot()`.

- [ ] **Step 2: Update the classify() call in execute()**

Replace the old `featureExtractor.extract(sessionManager.unitBufferSnapshot(), gameTimeMin)` with:

```java
var windowedFeatures = windowAccumulator.getWindowedFeatures();
var strategyFeatures = featureExtractor.extract(windowedFeatures, mapCharacteristics);
Race enemyRace = resolveEnemyRace(ctx);
CascadeResult cascadeResult = cascadingClassifier.classify(
    patternData.getEvidence(), patternData.getRevisions(),
    strategyFeatures, enemyRace, frame, prevFrame, ctx);
```

Add a `resolveEnemyRace(CaseContext)` helper that reads enemy race from
`CaseContext` key `game.enemy.race`. Also add an `ENEMY_RACE` constant to
`QuarkMindCaseFile` and populate it during game initialization:
- In `GameStateTranslator.translate()` — derive from the first scouted enemy
  unit's type via `UnitType.race()`, or from the SC2 lobby data in live games.
- In replay test harnesses — set from the matchup metadata (IEM10 carries
  matchup strings like "PvT"; AI Arena is always PvP).
- For the validation test in Task 6 — pass race directly from the game entry's
  matchup, bypassing CaseContext (test-only path).

- [ ] **Step 3: Copy ONNX model artifacts to test resources**

```bash
cp /Users/mdproctor/claude/casehub/neocortex/inference-runtime/src/test/resources/models/strategy/*.onnx /Users/mdproctor/claude/casehub/quarkmind/quarkmind-sc2/src/test/resources/models/strategy/
```

- [ ] **Step 4: Add model config to application.properties**

```properties
# %test profile — ONNX strategy models
%test.casehub.inference.models.strategy-vs-terran.model-path=models/strategy/strategy_vs_terran.onnx
%test.casehub.inference.models.strategy-vs-zerg.model-path=models/strategy/strategy_vs_zerg.onnx
%test.casehub.inference.models.strategy-vs-protoss.model-path=models/strategy/strategy_vs_protoss.onnx
```

- [ ] **Step 5: Update DroolsScoutingTaskTest**

Update test setup to construct `CascadingPatternClassifier` with the new
constructor signature. Update test calls to `execute()` to ensure `CaseContext`
carries `game.enemy.race`.

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): wire DroolsScoutingTask for temporal features + per-race ONNX models"
```

---

## Batch 3: Validation and Benchmarking

### Task 6: Extract ReplayClassificationTestSupport + CascadeValidationCalibrationTest

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/ReplayClassificationTestSupport.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/PatternClassificationCalibrationTest.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadeValidationCalibrationTest.java`

**Interfaces:**
- Consumes: `CascadingPatternClassifier` (injected via CDI in @QuarkusTest)
- Consumes: `ReplayClassificationTestSupport.deriveGroundTruth()`, `loadIEM10Games()`, `loadAIArenaGames()`
- Produces: stdout accuracy tables, tier hit rate report, comparison baselines

- [ ] **Step 1: Extract shared infrastructure**

Move `deriveGroundTruth()`, `loadIEM10Games()`, `loadAIArenaGames()`, and
`ClassificationResult` from `PatternClassificationCalibrationTest` into
`ReplayClassificationTestSupport`. Make them `static` utility methods.

- [ ] **Step 2: Refactor PatternClassificationCalibrationTest to use support class**

Replace inline implementations with calls to `ReplayClassificationTestSupport`.
Run existing calibration tests to verify no behavioral change:

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=PatternClassificationCalibrationTest -q`
Expected: PASS — same behavior

- [ ] **Step 3: Write CascadeValidationCalibrationTest**

`@QuarkusTest`, `@Tag("benchmark")`. Inject `CascadingPatternClassifier`.

The test has one main method: `validateCascadeAccuracy()`.

Construct a `TemporalWindowAccumulator` and `StrategyFeatureExtractor` locally
in the test (these are plain objects, not CDI beans). For each mode:
  - Configure thresholds on the classifier
  - For each of 59 games:
    - `cascadingClassifier.reset()` AND `accumulator.reset()`
    - Run game tick-by-tick, building `WindowSnapshot` from `GameState` each tick
      and feeding to the accumulator
    - At each target minute (1-5), extract `StrategyFeatures` from the
      accumulator and call `cascadingClassifier.classify()` with the game's
      enemy race (from matchup metadata)
    - Compare top prediction to ground truth
    - Record result per matchup

Print tables to stdout: per-mode accuracy, tier hit rates (cascade mode),
comparison baselines, AI Arena separate section.

**Assertions:**
```java
assertThat(droolsOnlyRushAccuracyMin3).as("Drools-only rush ≥ 70%").isGreaterThanOrEqualTo(0.70);
assertThat(aiArenaCascadeAccuracyMin4).as("AI Arena cascade ≥ 80%").isGreaterThanOrEqualTo(0.80);
```

- [ ] **Step 4: Run cascade validation**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=CascadeValidationCalibrationTest -q`
Expected: PASS — assertions met, tables printed

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/test/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): cascade validation test — Drools/ONNX/cascade accuracy + tier hit rates"
```

---

### Task 7: CascadeLatencyBenchmarkTest + benchmark snapshot

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadeLatencyBenchmarkTest.java`
- Create: `quarkmind-sc2/docs/benchmarks/YYYY-MM-DD-cascade-validation.md`

**Interfaces:**
- Consumes: `CascadingPatternClassifier` (test constructor), `StrategyFeatureExtractor`, `TensorClassifier`

- [ ] **Step 1: Write CascadeLatencyBenchmarkTest**

Plain JUnit, `@Tag("benchmark")`. Construct classifier directly.

```java
@Tag("benchmark")
class CascadeLatencyBenchmarkTest {

    @Test
    void cascadeLatency_p99Under10ms(@TempDir Path tmpDir) throws Exception {
        // Load one ONNX model
        // Construct CascadingPatternClassifier with test constructor
        // Prepare realistic StrategyFeatures from a snapshot
        // 100 warmup, 1000 measured iterations
        // Assert p99 < 10ms
        // Assert feature extraction p99 < 1ms
        // Print breakdown table
    }
}
```

Follow the same pattern as `StrategyClassifierOnnxTest.latencyUnderThreshold()`:
extract model from test resources, warmup, measure nanos, sort, read percentiles.

- [ ] **Step 2: Run latency benchmark**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=CascadeLatencyBenchmarkTest -q`
Expected: PASS

- [ ] **Step 3: Commit benchmark results snapshot**

Copy the stdout output from both Task 6 and Task 7 into a dated file at
`docs/benchmarks/YYYY-MM-DD-cascade-validation.md` (use today's date).

- [ ] **Step 4: Run -Preport divergence check**

Run: `mvn test -pl quarkmind-sc2 -Preport -q`
Expected: PASS — classifier changes don't regress replay validation

- [ ] **Step 5: Final commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/ docs/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#213): cascade latency benchmark + validation baseline snapshot"
```

---

## References

- [2026-08-22-cascade-validation-design.md] — design spec this plan implements
- `CascadingPatternClassifier.java` — three-tier cascade (modified in Task 4)
- `StrategyFeatureExtractor.java` — feature extractor (rewritten in Task 3)
- `PatternClassificationCalibrationTest.java` — existing calibration test (refactored in Task 6)
- `DroolsScoutingTask.java:294-322` — cascade invocation site (modified in Task 5)
- `StrategyArchetype.java` — enum (extended in Task 4)
- `GameState.java` — domain record (extended in Task 1)
- `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py` — feature definitions
- `neocortex/inference-runtime/.../StrategyClassifierOnnxTest.java` — model shapes + latency test pattern
- GE-20260528-6ebb38 — SC2EGSet JSON extra UnitBorn at loop 0 gotcha
- PP-20260522-572156 — SC2Data calibration protocol
- GitHub #213
