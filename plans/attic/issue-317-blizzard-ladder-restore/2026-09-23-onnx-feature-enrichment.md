# ONNX Feature Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #306 — Retrain ONNX strategy classifier against corrected spatial features
**Issue group:** #306

**Goal:** Enrich the ONNX strategy classifier's feature vector with spatial, temporal delta, and structural ratio features (269 → 298 per window), then retrain all three per-matchup models.

**Architecture:** Extend the existing two-phase feature pipeline (Java inference + Python training) with 29 new features per window across three dimensions: spatial intent (WHERE — 15 features), temporal deltas (HOW FAST — 8), and structural ratios (HOW MUCH — 6). Single ONNX model per matchup with modality dropout for graceful degradation.

**Tech Stack:** Java 21 (Quarkus), Python 3 (PyTorch, ONNX Runtime), SC2EGSet replay data

## Global Constraints

- Java and Python feature extractors must produce identical feature vectors for the same game state (alignment test is the regression gate)
- `norm_stats.json` must be regenerated — all 298 entries recomputed
- `SC2Data` is the authoritative source for supply costs, worker/base classification, tech tiers — no duplicate lookup tables
- Economy stats are pre-scaled by `/1000.0f` in both Java and Python
- Spatial features use binary fog-of-war visibility, not continuous scouting mask
- `has_vision` at index 297 is the only non-normalizable feature (mean=0, std=1)

---

## Batch 1: Java Foundation — SC2Data and Constants

### Task 1: SC2Data prerequisite fix and classification helpers

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:970` (techTier)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/SC2DataTest.java`

**Interfaces:**
- Produces: `SC2Data.techTier(CYBERNETICS_CORE)` → `OptionalInt.of(1)`, `SC2Data.isProductionBuilding(BuildingType)` → `boolean`, `SC2Data.isTechBuilding(BuildingType)` → `boolean`

- [ ] **Step 1: Write failing test for CYBERNETICS_CORE tech tier**

```java
@Test
void techTier_cyberneticsCore_returnsTier1() {
    assertThat(SC2Data.techTier(BuildingType.CYBERNETICS_CORE))
        .isPresent()
        .hasValue(1);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataTest#techTier_cyberneticsCore_returnsTier1 -q`
Expected: FAIL — currently returns `OptionalInt.empty()`

- [ ] **Step 3: Fix techTier() — add CYBERNETICS_CORE to Protoss T1 tier**

In `SC2Data.techTier()` (line ~970), add `case CYBERNETICS_CORE -> OptionalInt.of(1)` to the Protoss tier 1 block (alongside Gateway, Forge).

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataTest#techTier_cyberneticsCore_returnsTier1 -q`
Expected: PASS

- [ ] **Step 5: Write tests for isProductionBuilding and isTechBuilding**

```java
@Test
void isProductionBuilding_barracks_true() {
    assertThat(SC2Data.isProductionBuilding(BuildingType.BARRACKS)).isTrue();
}

@Test
void isProductionBuilding_engineeringBay_false() {
    assertThat(SC2Data.isProductionBuilding(BuildingType.ENGINEERING_BAY)).isFalse();
}

@Test
void isTechBuilding_spawningPool_true() {
    assertThat(SC2Data.isTechBuilding(BuildingType.SPAWNING_POOL)).isTrue();
}

@Test
void isTechBuilding_hatchery_false_isBase() {
    assertThat(SC2Data.isTechBuilding(BuildingType.HATCHERY)).isFalse();
}

@Test
void isTechBuilding_barracks_false_isProduction() {
    assertThat(SC2Data.isTechBuilding(BuildingType.BARRACKS)).isFalse();
}
```

- [ ] **Step 6: Implement isProductionBuilding and isTechBuilding**

```java
public static boolean isProductionBuilding(BuildingType type) {
    return switch (type) {
        case BARRACKS, FACTORY, STARPORT,
             HATCHERY, LAIR, HIVE,
             GATEWAY, ROBOTICS_FACILITY, STARGATE -> true;
        default -> false;
    };
}

public static boolean isTechBuilding(BuildingType type) {
    return techTier(type).isPresent()
        && !isProductionBuilding(type)
        && !isBase(type);
}
```

- [ ] **Step 7: Run all SC2Data tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataTest -q`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/domain/SC2DataTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: SC2Data.techTier fix for CYBERNETICS_CORE + isProductionBuilding/isTechBuilding Refs #306"
```

### Task 2: Update FeatureIndexMaps constants and generate supply_costs.json

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java`
- Create: `quarkmind-sc2/src/main/resources/classifier/supply_costs.json`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureIndexMapsTest.java` (new or extend existing)

**Interfaces:**
- Consumes: `SC2Data.supplyCost(UnitType)` from Task 1
- Produces: `FeatureIndexMaps.N_TICK_FEATURES_PER_PLAYER` = 144, `FeatureIndexMaps.N_FEATURES_PER_PLAYER` = 148, `FeatureIndexMaps.FEATURES_PER_WINDOW` = 298, `FeatureIndexMaps.SPATIAL_OFFSET` = 134, `FeatureIndexMaps.RATIO_OFFSET` = 141, `FeatureIndexMaps.DELTA_OFFSET` = 144

- [ ] **Step 1: Write tests for new constants**

```java
@Test
void featureLayout_constants() {
    assertThat(FeatureIndexMaps.N_BUILDINGS).isEqualTo(53);
    assertThat(FeatureIndexMaps.N_UNITS).isEqualTo(53);
    assertThat(FeatureIndexMaps.N_STATS).isEqualTo(13);
    assertThat(FeatureIndexMaps.N_UPGRADES).isEqualTo(15);
    assertThat(FeatureIndexMaps.N_SPATIAL).isEqualTo(7);
    assertThat(FeatureIndexMaps.N_RATIOS).isEqualTo(3);
    assertThat(FeatureIndexMaps.N_DELTAS).isEqualTo(4);
    assertThat(FeatureIndexMaps.N_TICK_FEATURES_PER_PLAYER).isEqualTo(144);
    assertThat(FeatureIndexMaps.N_FEATURES_PER_PLAYER).isEqualTo(148);
    assertThat(FeatureIndexMaps.FEATURES_PER_WINDOW).isEqualTo(298);
    assertThat(FeatureIndexMaps.SPATIAL_OFFSET)
        .isEqualTo(53 + 53 + 13 + 15); // 134
    assertThat(FeatureIndexMaps.RATIO_OFFSET)
        .isEqualTo(134 + 7); // 141
    assertThat(FeatureIndexMaps.DELTA_OFFSET)
        .isEqualTo(141 + 3); // 144
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureIndexMapsTest#featureLayout_constants -q`
Expected: FAIL — constants don't exist yet

- [ ] **Step 3: Add constants to FeatureIndexMaps**

```java
static final int N_SPATIAL = 7;
static final int N_RATIOS = 3;
static final int N_DELTAS = 4;
static final int N_TICK_FEATURES_PER_PLAYER =
    N_BUILDINGS + N_UNITS + N_STATS + N_UPGRADES + N_SPATIAL + N_RATIOS;
static final int N_FEATURES_PER_PLAYER = N_TICK_FEATURES_PER_PLAYER + N_DELTAS;
static final int FEATURES_PER_WINDOW = 2 * N_FEATURES_PER_PLAYER + 2; // +army_gap +has_vision
static final int SPATIAL_OFFSET = N_BUILDINGS + N_UNITS + N_STATS + N_UPGRADES;
static final int RATIO_OFFSET = SPATIAL_OFFSET + N_SPATIAL;
static final int DELTA_OFFSET = RATIO_OFFSET + N_RATIOS;
static final int ARMY_GAP_INDEX = 2 * N_FEATURES_PER_PLAYER;
static final int HAS_VISION_INDEX = ARMY_GAP_INDEX + 1;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureIndexMapsTest -q`
Expected: PASS

- [ ] **Step 5: Generate supply_costs.json**

Write a script to `/tmp/gen_supply_costs.py` that reads the Java `SC2Data.supplyCost()` output and generates the JSON. Alternatively, write a quick JUnit test that generates it:

```java
@Test
void generateSupplyCosts() throws Exception {
    var costs = new java.util.LinkedHashMap<String, Integer>();
    for (UnitType type : UnitType.values()) {
        costs.put(type.name(), SC2Data.supplyCost(type));
    }
    var json = new com.fasterxml.jackson.databind.ObjectMapper()
        .writerWithDefaultPrettyPrinter()
        .writeValueAsString(costs);
    java.nio.file.Files.writeString(
        java.nio.file.Path.of("src/main/resources/classifier/supply_costs.json"), json);
}
```

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureIndexMapsTest#generateSupplyCosts -q`

Verify the file was created at `quarkmind-sc2/src/main/resources/classifier/supply_costs.json`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureIndexMapsTest.java quarkmind-sc2/src/main/resources/classifier/supply_costs.json
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: FeatureIndexMaps enriched constants (298 per window) + supply_costs.json Refs #306"
```

---

## Batch 2: Java Per-Tick Feature Extraction

### Task 3: Extend WindowSnapshot and buildSnapshot() with spatial + ratio features

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/WindowSnapshot.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:483` (buildSnapshot)
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java` (or similar static test)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `FeatureIndexMaps.N_TICK_FEATURES_PER_PLAYER` = 144, `FeatureIndexMaps.SPATIAL_OFFSET` = 134, `FeatureIndexMaps.RATIO_OFFSET` = 141, `SC2Data.supplyCost()`, `SC2Data.isWorker()`, `SC2Data.isBase()`, `Point2d.centroidOf()`, `GameState.mapInfo()` → `MapInfo.playerStart()`, `MapInfo.enemyStart()`, `MapInfo.mapWidth()`, `MapInfo.mapHeight()`
- Produces: `WindowSnapshot` with `playerFeatures[0:144]` and `opponentFeatures[0:144]` where indices 134-140 are spatial and 141-143 are ratios

- [ ] **Step 1: Write failing test for spatial feature computation**

```java
@Test
void buildSnapshot_spatialFeatures_armyCentroidAndDistances() {
    Point2d playerStart = new Point2d(30, 30);
    Point2d enemyStart = new Point2d(120, 120);
    MapInfo mapInfo = new MapInfo(playerStart, enemyStart, 150, 150);
    float mapDiag = (float) Math.sqrt(150 * 150 + 150 * 150);

    List<Unit> myUnits = List.of(
        new Unit("u1", UnitType.ZEALOT, new Point2d(50, 50), 100, 100, 50, 50, 0, 0),
        new Unit("u2", UnitType.STALKER, new Point2d(70, 70), 80, 80, 80, 80, 0, 0)
    );
    List<Unit> enemyUnits = List.of(
        new Unit("e1", UnitType.ZERGLING, new Point2d(100, 100), 35, 35, 0, 0, 0, 0)
    );
    GameState gs = GameState.builder()
        .myUnits(myUnits).enemyUnits(enemyUnits)
        .myBuildings(List.of()).enemyBuildings(List.of())
        .playerEconomy(PlayerEconomyStats.EMPTY)
        .enemyEconomy(PlayerEconomyStats.EMPTY)
        .mapInfo(mapInfo)
        .build();

    WindowSnapshot snap = DroolsScoutingTask.buildSnapshot(gs);

    // Player centroid = (60, 60), normalized by map dims
    assertThat(snap.playerFeatures()[134]).isCloseTo(60f / 150f, within(1e-5f));
    assertThat(snap.playerFeatures()[135]).isCloseTo(60f / 150f, within(1e-5f));
    // Player dist to own base
    float playerDistOwn = (float) new Point2d(60, 60).distanceTo(playerStart) / mapDiag;
    assertThat(snap.playerFeatures()[136]).isCloseTo(playerDistOwn, within(1e-5f));
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `playerFeatures` array is still 134 long

- [ ] **Step 3: Update WindowSnapshot record to accept 144-element arrays**

No change to the record itself — it already takes `float[] playerFeatures` and `float[] opponentFeatures`. The arrays just need to be larger (144 instead of 134).

- [ ] **Step 4: Extend buildSnapshot() with spatial feature computation**

In `DroolsScoutingTask.buildSnapshot(GameState gs)`, change the array size from `N_FEATURES_PER_PLAYER` (134) to `N_TICK_FEATURES_PER_PLAYER` (144), then after the existing composition/economy/upgrade extraction, add:

```java
// --- Spatial features (indices 134-140) ---
MapInfo map = gs.mapInfo();
float mapDiag = (float) Math.sqrt(
    (double) map.mapWidth() * map.mapWidth() + (double) map.mapHeight() * map.mapHeight());

// Player spatial
List<Unit> playerArmyUnits = gs.myUnits().stream()
    .filter(u -> !SC2Data.isWorker(u.type())).toList();
computeSpatialFeatures(player, FeatureIndexMaps.SPATIAL_OFFSET,
    playerArmyUnits, gs.myBuildings(), map.playerStart(), map.enemyStart(),
    map.mapWidth(), map.mapHeight(), mapDiag);

// Opponent spatial
List<Unit> enemyArmyUnits = gs.enemyUnits().stream()
    .filter(u -> !SC2Data.isWorker(u.type())).toList();
computeSpatialFeatures(opponent, FeatureIndexMaps.SPATIAL_OFFSET,
    enemyArmyUnits, gs.enemyBuildings(), map.enemyStart(), map.playerStart(),
    map.mapWidth(), map.mapHeight(), mapDiag);

// --- Ratio features (indices 141-143) ---
computeRatioFeatures(player, FeatureIndexMaps.RATIO_OFFSET,
    gs.myUnits(), gs.myBuildings(), gs.playerEconomy());
computeRatioFeatures(opponent, FeatureIndexMaps.RATIO_OFFSET,
    gs.enemyUnits(), gs.enemyBuildings(), gs.enemyEconomy());
```

Add the helper methods:

```java
private static void computeSpatialFeatures(
        float[] features, int offset,
        List<Unit> armyUnits, List<Building> buildings,
        Point2d ownBase, Point2d enemyBase,
        int mapWidth, int mapHeight, float mapDiag) {
    if (armyUnits.isEmpty()) {
        // All spatial features = 0 (spec edge case)
        return;
    }
    Point2d centroid = Point2d.centroidOf(armyUnits);
    if (centroid == null) return;
    features[offset]     = centroid.x() / mapWidth;      // centroid_x
    features[offset + 1] = centroid.y() / mapHeight;     // centroid_y
    features[offset + 2] = (float) centroid.distanceTo(ownBase) / mapDiag;   // dist_own
    features[offset + 3] = (float) centroid.distanceTo(enemyBase) / mapDiag; // dist_enemy
    // spread = sqrt(var_x + var_y)
    double varX = 0, varY = 0;
    for (Unit u : armyUnits) {
        varX += (u.position().x() - centroid.x()) * (u.position().x() - centroid.x());
        varY += (u.position().y() - centroid.y()) * (u.position().y() - centroid.y());
    }
    varX /= armyUnits.size();
    varY /= armyUnits.size();
    features[offset + 4] = (float) Math.sqrt(varX + varY) / mapDiag; // spread
    // max forward distance toward enemy
    double maxForward = 0;
    for (Unit u : armyUnits) {
        double dist = u.position().distanceTo(enemyBase);
        double forward = ownBase.distanceTo(enemyBase) - dist;
        if (forward > maxForward) maxForward = forward;
    }
    features[offset + 5] = (float) maxForward / mapDiag; // max_forward_dist
    // proxy building score
    if (!buildings.isEmpty()) {
        int proxied = 0;
        for (Building b : buildings) {
            if (b.position().distanceTo(enemyBase) < b.position().distanceTo(ownBase)) {
                proxied++;
            }
        }
        features[offset + 6] = (float) proxied / buildings.size();
    }
}

private static void computeRatioFeatures(
        float[] features, int offset,
        List<Unit> allUnits, List<Building> buildings,
        PlayerEconomyStats eco) {
    // army_supply_ratio
    float armySupply = 0;
    for (Unit u : allUnits) {
        if (!SC2Data.isWorker(u.type())) {
            armySupply += SC2Data.supplyCost(u.type());
        }
    }
    float foodUsed = Math.max(eco.foodUsed() / 1000.0f, 1e-6f);
    features[offset] = armySupply / foodUsed; // NOTE: foodUsed is pre-scaled
    // worker_saturation
    int baseCount = 0;
    for (Building b : buildings) {
        if (SC2Data.isBase(b.type())) baseCount++;
    }
    float satDenom = Math.max(baseCount * 16.0f, 1.0f);
    features[offset + 1] = (eco.workersActiveCount() / 1000.0f) / satDenom;
    // gas_mineral_ratio
    float gasSpent = (eco.vespeneUsedCurrentArmy() + eco.vespeneUsedCurrentEconomy()
        + eco.vespeneUsedCurrentTechnology()) / 1000.0f;
    float minSpent = (eco.mineralsUsedCurrentArmy() + eco.mineralsUsedCurrentEconomy()
        + eco.mineralsUsedCurrentTechnology()) / 1000.0f;
    float totalSpent = Math.max(gasSpent + minSpent, 1e-6f);
    features[offset + 2] = gasSpent / totalSpent;
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DroolsScoutingTaskTest -q`
Expected: PASS (new test + existing tests)

- [ ] **Step 6: Write additional edge case tests**

```java
@Test
void buildSnapshot_noArmyUnits_spatialAllZeros() {
    // Only workers, no army — spatial features should all be 0
    // ...
}

@Test
void buildSnapshot_proxyBuilding_score() {
    // Buildings near enemy base — proxy_building_score > 0
    // ...
}

@Test
void buildSnapshot_ratioFeatures_rushProfile() {
    // High army supply, low workers, no gas — rush profile
    // ...
}
```

- [ ] **Step 7: Run full test suite for scouting package**

Run: `mvn test -pl quarkmind-sc2 -Dtest="DroolsScoutingTask*" -q`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: buildSnapshot() spatial + ratio features (134→144 per player) Refs #306"
```

---

## Batch 3: Java Windowed Assembly

### Task 4: Update TemporalWindowAccumulator with two-phase assembly

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulatorTest.java`

**Interfaces:**
- Consumes: `WindowSnapshot` with 144-element playerFeatures/opponentFeatures from Task 3, `FeatureIndexMaps.N_TICK_FEATURES_PER_PLAYER` = 144, `FeatureIndexMaps.N_FEATURES_PER_PLAYER` = 148, `FeatureIndexMaps.FEATURES_PER_WINDOW` = 298
- Produces: `List<float[]>` where each `float[298]` contains per-player features (148 each) + army_gap + has_vision

- [ ] **Step 1: Write failing test for two-phase assembly**

```java
@Test
void getWindowedFeatures_deltasComputedBetweenWindows() {
    TemporalWindowAccumulator acc = new TemporalWindowAccumulator();
    // Add 120 snapshots (2 windows of 60 ticks)
    // Window 0: army supply = 4 (2 zealots)
    // Window 1: army supply = 8 (4 zealots)
    // delta_army_supply for window 1 should be 4
    // ...
    List<float[]> windows = acc.getWindowedFeatures();
    assertThat(windows.get(0).length).isEqualTo(298);
    // Window 0 delta = 0 (no previous window)
    assertThat(windows.get(0)[FeatureIndexMaps.DELTA_OFFSET]).isEqualTo(0f);
    // Window 1 delta_army_supply > 0
    assertThat(windows.get(1)[FeatureIndexMaps.DELTA_OFFSET]).isGreaterThan(0f);
}

@Test
void getWindowedFeatures_armyGapComputed() {
    // Player army at (50, 50), opponent at (100, 100)
    // army_gap should be distance between centroids / map diagonal
    // ...
    List<float[]> windows = acc.getWindowedFeatures();
    int armyGapIdx = FeatureIndexMaps.ARMY_GAP_INDEX;
    assertThat(windows.get(0)[armyGapIdx]).isGreaterThan(0f);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — current `getWindowedFeatures()` produces 269-element arrays

- [ ] **Step 3: Implement two-phase window assembly**

Rewrite `getWindowedFeatures()`:

Phase 1 — tick averaging: Average `playerFeatures[0:144]` normally. For opponent features, average indices 0-133 with visibility scaling (existing behavior) and indices 134-143 (spatial/ratios) WITHOUT visibility scaling (binary visibility — non-zero values are already from visible units only).

Phase 2 — window assembly: After tick-averaging each window, compute:
- Player deltas (indices 144-147): compare current window's averaged army supply, worker count, production building count, tech building count against previous window's values
- Opponent deltas (same 4 features, mirrored at opponent offset)
- `army_gap` (index 296): distance between averaged player centroid (from indices 134-135) and opponent centroid
- `has_vision` (index 297): whether any tick in the window had visibility > 0

Key change: update `FEATURES_PER_WINDOW` from `2 * FEATURES_PER_PLAYER + 1` to `2 * N_FEATURES_PER_PLAYER + 2` = 298.

For delta computation, the accumulator needs to derive aggregate counts from the averaged per-tick features. Army supply requires summing `unit_count * supply_cost` across unit type indices — implement a helper `computeArmySupply(float[] features)` that iterates UNIT_INDEX entries and multiplies by `SC2Data.supplyCost()`. Similarly for worker count, production building count, and tech building count.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalWindowAccumulatorTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulatorTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: TemporalWindowAccumulator two-phase assembly (deltas, army_gap, 298-wide) Refs #306"
```

### Task 5: Update StrategyFeatureExtractor for 298-feature windows

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractor.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractorTest.java`

**Interfaces:**
- Consumes: 298-element window arrays from `TemporalWindowAccumulator`, `norm_stats.json` (298 entries — not yet regenerated, so test with a placeholder)
- Produces: `StrategyFeatures` with 298-wide temporal tensor + map features

- [ ] **Step 1: Write test for 298-feature normalization**

```java
@Test
void extract_298features_normalizesAllExceptHasVision() {
    // Create windowed features with 298 elements per window
    // Verify normalization skips index 297 (has_vision)
    // Verify normalization applies to index 296 (army_gap)
    // ...
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — current extractor uses `FEATURES_PER_WINDOW = 269`

- [ ] **Step 3: Update StrategyFeatureExtractor**

Change `FEATURES_PER_WINDOW` and `FLATTENED_SIZE` to use the new constants from `TemporalWindowAccumulator`. Update `HAS_VISION_OFFSET` to point to index 297 (was last element, still last element). The normalization loop already skips `HAS_VISION_OFFSET` — no logic change needed, just the constant value.

Update `norm_stats.json` loading to expect 298-element mean/std arrays. For now, create a test-only `norm_stats.json` with 298 entries (pad the existing 269 with zeros for mean and ones for std) so tests can run before the Python pipeline regenerates the real stats.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyFeatureExtractorTest -q`
Expected: PASS

- [ ] **Step 5: Run all scouting package tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest="io.quarkmind.plugin.scouting.*" -q`
Expected: All PASS (some existing tests may need array-size adjustments)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractorTest.java quarkmind-sc2/src/main/resources/classifier/norm_stats.json
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: StrategyFeatureExtractor updated for 298-feature windows Refs #306"
```

---

## Batch 4: Python Feature Extraction (neocortex repo)

### Task 6: Extend sc2egset_extractor.py with position tracking + spatial + ratio features

**Files:**
- Modify: `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py`
- Create: `neocortex/evaluation/strategy_classifier/tests/test_spatial_features.py`

**Interfaces:**
- Consumes: SC2EGSet JSON replay format (`UnitBornEvent`, `UnitPositionsEvent`), `supply_costs.json` from quarkmind-sc2
- Produces: `N_FEATURES_PER_PLAYER = 148`, `N_TICK_FEATURES_PER_PLAYER = 144`, per-second feature arrays with spatial (indices 134-140) and ratio (indices 141-143) features

- [ ] **Step 1: Write failing test for position extraction**

```python
def test_extract_replay_includes_spatial_features():
    """Verify spatial features appear at indices 134-140 in per-second arrays."""
    # Load a known IEM10 replay JSON
    replay = load_test_replay("test_replay.json")
    data = extract_replay(replay)
    assert data is not None
    # Feature vector should be 144 wide (was 134)
    assert data.player1_features.shape[1] == 144
    # At some second with units, spatial features should be non-zero
    nonzero_sec = find_first_nonzero(data.player1_features[:, 134])
    assert nonzero_sec is not None, "No spatial features found"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest neocortex/evaluation/strategy_classifier/tests/test_spatial_features.py -v`
Expected: FAIL — feature arrays are 134 wide

- [ ] **Step 3: Implement position tracking in extract_replay()**

In `extract_replay()`:

1. Add `unit_positions: dict[int, tuple[float, float]] = {}` alongside existing `unit_counts`.
2. On `UnitBorn` events (already handled for counts), also store position: `unit_positions[tag] = (event["x"], event["y"])`.
3. Add handling for `SUnitPositionsEvent` (tracker event type 8): parse the packed position array and update `unit_positions` for each unit tag.
4. On `UnitDied`: remove from `unit_positions`.
5. Expand per-second feature arrays from 134 to 144.
6. After setting counts/stats/upgrades (existing code), compute spatial aggregates:
   - Centroid from army unit positions (filter out workers using a `WORKER_UNITS` set)
   - Spread, distances, proxy building score
7. Compute ratio features from counts + stats.

Update constants:
```python
N_TICK_FEATURES_PER_PLAYER = N_BUILDINGS + N_UNITS + N_STATS + N_UPGRADES + N_SPATIAL + N_RATIOS
N_FEATURES_PER_PLAYER = N_TICK_FEATURES_PER_PLAYER + N_DELTAS  # 148
```

Where `N_SPATIAL = 7`, `N_RATIOS = 3`, `N_DELTAS = 4`.

Load supply costs: read `supply_costs.json` at module level (path from `Paths.supply_costs` or default relative path).

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest neocortex/evaluation/strategy_classifier/tests/test_spatial_features.py -v`
Expected: PASS

- [ ] **Step 5: Write ratio feature tests**

```python
def test_army_supply_ratio_rush_profile():
    """High army, low workers = ratio close to 1.0."""
    # ...

def test_gas_mineral_ratio():
    """Gas-heavy build = ratio > 0.5."""
    # ...
```

- [ ] **Step 6: Run and verify**

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex add evaluation/strategy_classifier/sc2egset_extractor.py evaluation/strategy_classifier/tests/
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat: sc2egset_extractor position tracking + spatial + ratio features (134→144) Refs casehubio/quarkmind#306"
```

### Task 7: Update feature_engineering.py with two-phase window assembly

**Files:**
- Modify: `neocortex/evaluation/strategy_classifier/feature_engineering.py`
- Create or modify: `neocortex/evaluation/strategy_classifier/tests/test_feature_engineering.py`

**Interfaces:**
- Consumes: Per-second feature arrays (144 per player) from Task 6, `N_FEATURES_PER_PLAYER` = 148, `N_TICK_FEATURES_PER_PLAYER` = 144
- Produces: Per-window feature arrays (298 per window) with deltas, army_gap, has_vision

- [ ] **Step 1: Write failing test for two-phase assembly**

```python
def test_build_temporal_features_298_wide():
    n_tick = 144
    player = np.random.randn(300, n_tick).astype(np.float32)
    opponent = np.random.randn(300, n_tick).astype(np.float32)
    mask = np.ones(300, dtype=np.float32)
    result = build_temporal_features(player, opponent, mask, 5, HyperParams())
    assert result.shape == (10, 298)

def test_deltas_window0_zeros():
    # Window 0 deltas should all be 0
    # ...

def test_binary_mask_applied_to_opponent_spatial():
    # Opponent spatial features (indices 134-143) should use binary mask
    # ...
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — current output is 269 wide

- [ ] **Step 3: Implement two-phase window assembly**

Rewrite `build_temporal_features()`:

```python
def build_temporal_features(
    player_features: np.ndarray,    # (seconds, 144)
    opponent_features: np.ndarray,  # (seconds, 144)
    scouting_mask: np.ndarray,      # (seconds,)
    minute: int,
    hp: HyperParams,
) -> np.ndarray:
    N_TICK = N_TICK_FEATURES_PER_PLAYER  # 144
    N_PLAYER = N_FEATURES_PER_PLAYER     # 148
    COMP_END = 134  # count/stat/upgrade features
    f_window = 2 * N_PLAYER + 2  # 298

    result = np.zeros((hp.max_windows, f_window), dtype=np.float32)
    prev_player_agg = None
    prev_opp_agg = None

    for w in range(min(n_windows, hp.max_windows)):
        start = w * seconds_per_window
        end = min(start + seconds_per_window, len(player_features))
        if start >= len(player_features):
            break

        # Phase 1: tick averaging
        player_window = player_features[start:end].mean(axis=0)  # (144,)
        mask_window = scouting_mask[start:end]
        has_vision = float(mask_window.any())
        binary_mask = (mask_window > 0).astype(np.float32)

        # Opponent: continuous mask for comp features, binary for spatial/ratio
        opp_slice = opponent_features[start:end]
        opp_comp = (opp_slice[:, :COMP_END] * mask_window[:, np.newaxis]).mean(axis=0)
        opp_spatial_ratio = (opp_slice[:, COMP_END:] * binary_mask[:, np.newaxis]).mean(axis=0)
        opponent_window = np.concatenate([opp_comp, opp_spatial_ratio])  # (144,)

        # Phase 2: deltas
        player_deltas = compute_deltas(player_window, prev_player_agg)
        opp_deltas = compute_deltas(opponent_window, prev_opp_agg)
        prev_player_agg = player_window.copy()
        prev_opp_agg = opponent_window.copy()

        # Army gap from averaged centroids
        p_cx, p_cy = player_window[134], player_window[135]
        o_cx, o_cy = opponent_window[134 - COMP_END + COMP_END], opponent_window[135 - COMP_END + COMP_END]
        # Note: opponent spatial starts at index 134 in the 144-wide array
        o_cx_abs, o_cy_abs = opponent_window[134 - COMP_END + COMP_END], ...
        # Simplified: opponent centroid indices within opponent_window are same as player
        army_gap = compute_army_gap(player_window, opponent_window)

        # Assemble window vector
        result[w, :N_TICK] = player_window           # player tick features (0:144)
        result[w, N_TICK:N_PLAYER] = player_deltas   # player deltas (144:148)
        result[w, N_PLAYER:N_PLAYER+N_TICK] = opponent_window  # opponent tick (148:292)
        result[w, N_PLAYER+N_TICK:2*N_PLAYER] = opp_deltas     # opponent deltas (292:296)
        result[w, 2*N_PLAYER] = army_gap             # army_gap (296)
        result[w, 2*N_PLAYER+1] = has_vision         # has_vision (297)

    return result
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex add evaluation/strategy_classifier/feature_engineering.py evaluation/strategy_classifier/tests/
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat: feature_engineering two-phase window assembly (269→298) Refs casehubio/quarkmind#306"
```

### Task 8: Update normalize.py, dataset.py, export_onnx.py

**Files:**
- Modify: `neocortex/evaluation/strategy_classifier/normalize.py:86` (vis_idx fix)
- Modify: `neocortex/evaluation/strategy_classifier/dataset.py:73` (modality dropout update)
- Modify: `neocortex/evaluation/strategy_classifier/export_onnx.py` (manifest f_temporal)

**Interfaces:**
- Consumes: `N_FEATURES_PER_PLAYER` = 148, `N_TICK_FEATURES_PER_PLAYER` = 144
- Produces: Correct normalization stats (298 entries), correct modality dropout, correct ONNX manifest

- [ ] **Step 1: Fix vis_idx in normalize.py**

Change line 86 from:
```python
vis_idx = 2 * N_PLAYER_FEATURES
```
to:
```python
vis_idx = 2 * N_PLAYER_FEATURES + 1  # has_vision is after army_gap
```

- [ ] **Step 2: Update ModalityDropoutDataset in dataset.py**

After zeroing opponent block `temporal[:, n:2*n]`, also zero:
- `temporal[:, 2*n]` (army_gap — depends on opponent centroid)
- `temporal[:, 2*n+1]` (has_vision — meaningless when opponent dropped)

After zeroing player block `temporal[:, :n]`, also zero:
- `temporal[:, 2*n]` (army_gap — depends on player centroid)

- [ ] **Step 3: Write tests for both fixes**

```python
def test_normalize_has_vision_index_correct():
    """has_vision at index 297 should have mean=0, std=1."""
    # ...

def test_modality_dropout_zeros_army_gap():
    """When opponent is dropped, army_gap should also be zeroed."""
    # ...
```

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex add evaluation/strategy_classifier/normalize.py evaluation/strategy_classifier/dataset.py evaluation/strategy_classifier/export_onnx.py
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat: normalize vis_idx fix, modality dropout for army_gap, manifest update Refs casehubio/quarkmind#306"
```

---

## Batch 5: Alignment, Training, and Calibration

### Task 9: Write FeatureAlignmentTest

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureAlignmentTest.java`
- Create: `quarkmind-sc2/src/test/resources/classifier/alignment_reference.npz` (generated by Python)

**Interfaces:**
- Consumes: Java feature pipeline (Tasks 3-5), Python feature pipeline (Tasks 6-8)
- Produces: Regression gate test that asserts feature-level equality

- [ ] **Step 1: Generate Python reference fixture**

Run the Python pipeline on a known IEM10 replay at a fixed game time (e.g., minute 3), save the 298-element feature window to `alignment_reference.npz`:

```python
# Script: generate_alignment_fixture.py
# Extracts features from a specific IEM10 replay at 3-min mark
# Saves to alignment_reference.npz with keys: temporal (10, 298), map (6,)
```

- [ ] **Step 2: Write FeatureAlignmentTest**

```java
@Test
void javaFeatures_matchPythonReference() throws Exception {
    // Load the same replay used by the Python fixture
    var game = new IEM10JsonSimulatedGame(IEM10_ZIP, "specific_game_name");
    // Run to 3-min mark using the Java pipeline
    // Extract features
    // Load alignment_reference.npz
    // Assert per-feature equality within 1e-5
}
```

- [ ] **Step 3: Run test**

Expected: PASS if both pipelines produce identical features.
If FAIL: debug which feature index diverges and fix the alignment.

- [ ] **Step 4: Write supply_costs alignment check**

```java
@Test
void supplyCosts_matchJsonArtifact() throws Exception {
    var json = new ObjectMapper().readValue(
        getClass().getResourceAsStream("/classifier/supply_costs.json"),
        new TypeReference<Map<String, Integer>>() {});
    for (UnitType type : UnitType.values()) {
        assertThat(json.get(type.name()))
            .as("Supply cost for " + type.name())
            .isEqualTo(SC2Data.supplyCost(type));
    }
}
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureAlignmentTest.java quarkmind-sc2/src/test/resources/classifier/
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: FeatureAlignmentTest — Java/Python feature parity regression gate Refs #306"
```

### Task 10: Train models, deploy, and write OnnxClassificationCalibrationTest

**Files:**
- Run: `neocortex/evaluation/strategy_classifier/run_pipeline.py` (training)
- Update: `quarkmind-sc2/src/test/resources/models/strategy/strategy_vs_*.onnx` (3 models)
- Update: `quarkmind-sc2/src/main/resources/classifier/norm_stats.json`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/OnnxClassificationCalibrationTest.java`

**Interfaces:**
- Consumes: Trained ONNX models, enriched feature pipeline, replay datasets
- Produces: Calibration evidence (accuracy >= 70% for rush/air-threat at 3-min mark)

- [ ] **Step 1: Regenerate training data**

Run the Python extractor on all SC2EGSet data with the enriched features:
```bash
python3 -m evaluation.strategy_classifier.sc2egset_extractor --zip <path_to_sc2egset.zip>
```

- [ ] **Step 2: Normalize and merge datasets**

```bash
python3 -m evaluation.strategy_classifier.normalize --sources sc2egset spawningtool msc
```

This generates the new `norm_stats.npz` with 298-entry mean/std arrays.

- [ ] **Step 3: Convert norm_stats.npz to norm_stats.json for Java**

```python
import numpy as np, json
d = np.load("evaluation/strategy_classifier/data/combined/norm_stats.npz")
json.dump({"mean": d["mean"].tolist(), "std": d["std"].tolist()},
          open("norm_stats.json", "w"))
```

Copy to `quarkmind-sc2/src/main/resources/classifier/norm_stats.json`.

- [ ] **Step 4: Run training pipeline**

```bash
python3 -m evaluation.strategy_classifier.run_pipeline --data combined
```

This trains all 3 matchup models and exports ONNX files.

- [ ] **Step 5: Copy trained models to quarkmind**

Copy `evaluation/strategy_classifier/output/strategy_vs_terran.onnx`, `strategy_vs_zerg.onnx`, `strategy_vs_protoss.onnx` to `quarkmind-sc2/src/test/resources/models/strategy/`.

- [ ] **Step 6: Write OnnxClassificationCalibrationTest**

A plain JUnit test (not `@QuarkusTest`) that:

```java
@Tag("benchmark")
class OnnxClassificationCalibrationTest {

    @Test
    void onnxAccuracy_rushAndAirThreat_atLeast70Percent() throws Exception {
        // For each matchup model:
        // 1. Load ONNX model via TensorClassifier
        // 2. Run replays through the Java feature pipeline
        // 3. Classify with ONNX
        // 4. Compare against ground truth (deriveGroundTruth)
        // 5. Assert >= 70% accuracy for rush + air-threat
    }
}
```

- [ ] **Step 7: Run calibration test**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=OnnxClassificationCalibrationTest -q`
Expected: PASS with >= 70% accuracy

- [ ] **Step 8: Run FeatureAlignmentTest to confirm alignment**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureAlignmentTest -q`
Expected: PASS

- [ ] **Step 9: Commit all artifacts**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/test/resources/models/strategy/ quarkmind-sc2/src/main/resources/classifier/norm_stats.json quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/OnnxClassificationCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: retrained ONNX models with enriched features + OnnxClassificationCalibrationTest Refs #306"
```

---

## References

- `specs/issue-306-retrain-onnx-spatial/2026-09-23-onnx-feature-enrichment-design.md` — design spec this plan implements
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:970` — techTier(), supplyCost(), isWorker(), isBase()
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/Point2d.java:11` — centroidOf(List<Positionable>)
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/GameState.java` — mapInfo(), myUnits(), enemyUnits()
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:483` — buildSnapshot()
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java` — feature layout constants
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java` — windowed assembly
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractor.java` — normalization
- `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py:120` — extract_replay()
- `neocortex/evaluation/strategy_classifier/feature_engineering.py:8` — build_temporal_features()
- `neocortex/evaluation/strategy_classifier/normalize.py:86` — vis_idx computation
- `neocortex/evaluation/strategy_classifier/dataset.py:73` — ModalityDropoutDataset
- `neocortex/evaluation/strategy_classifier/model.py:28` — StrategyClassifier split at N_PLAYER_FEATURES
- `docs/protocols/sc2data-spatial-constants-require-calibration.md` — spatial calibration protocol
- GitHub #306, #298, #300, #208, #212
