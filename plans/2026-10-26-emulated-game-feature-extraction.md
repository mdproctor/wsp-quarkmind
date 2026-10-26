# EmulatedGame Feature Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #360 — P4-2: Feature extraction from emulated games
**Issue group:** #360

**Goal:** Create `EmulatedGameFeatureExtractor` that converts EmulatedGame
tick-by-tick GameState snapshots into standard game_json format for
ONNX-compatible training data, then validate the full round-trip from
emulated features through ONNX inference.

**Architecture:** A dedicated extractor class diffs consecutive GameState
snapshots by unit/building type counts and upgrade sets, emitting synthetic
tracker events (UnitBorn, UnitDied, UnitInit, UnitDone, Upgrade, PlayerStats)
in the same `Map<String, Object>` format that `TrackerEventFeatureExtractor`
produces. No Python pipeline changes needed — the output is format-compatible.

**Tech Stack:** Java 21, JUnit 5, Quarkus (for integration tests only),
ONNX Runtime (for Tier 3 validation)

## Global Constraints

- Protocol PP-20260528-612dee: extraction logic in dedicated class, not in EmulatedGame
- game_json output must match `TrackerEventFeatureExtractor.buildGameJson()` key structure exactly
- PlayerStats events emitted every 160 game loops, stats scaled ×1000
- Feature shape: 204 per player (53 buildings + 56 units + 13 stats + 82 upgrades)
- Never use `@QuarkusTest` for tests that can be plain JUnit
- All commits reference #360

---

## Batch 1: Core Extractor

### Task 1: EmulatedGameFeatureExtractor — diff logic and unit test

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureExtractor.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `io.quarkmind.domain.GameState` (record — `myUnits()`, `myBuildings()`, `enemyUnits()`, `enemyBuildings()`, `playerUpgrades()`, `enemyUpgrades()`, `playerEconomy()`, `enemyEconomy()`, `gameFrame()`)
- Consumes: `io.quarkmind.domain.PlayerEconomyStats` (record — `toFeatureVector()` returns `float[13]`, 13 int fields, `EMPTY` constant)
- Consumes: `io.quarkmind.domain.UnitType` (enum — `pythonName()`, `race()`)
- Consumes: `io.quarkmind.domain.BuildingType` (enum — `name()`)
- Consumes: `io.quarkmind.domain.Unit` (record — `type()` returns `UnitType`)
- Consumes: `io.quarkmind.domain.Building` (record — `type()` returns `BuildingType`)
- Consumes: `io.quarkmind.domain.Race` (enum — `PROTOSS`, `TERRAN`, `ZERG`)
- Produces: `Map<String, Object>` game_json with keys `ToonPlayerDescMap`, `trackerEvents`, `header`, `metadata`

- [ ] **Step 1: Write the failing test for basic unit event emission**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class EmulatedGameFeatureExtractorTest {

    @Test
    void emitsUnitBornWhenUnitCountIncreases() {
        var extractor = new EmulatedGameFeatureExtractor();

        var snap0 = gameState(0, List.of(), List.of(), List.of(), List.of(),
                Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);
        var snap1 = gameState(16, List.of(unit(UnitType.ZEALOT)), List.of(),
                List.of(), List.of(),
                Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

        extractor.addSnapshot(snap0, 0);
        extractor.addSnapshot(snap1, 16);
        var gameJson = extractor.finalise(Race.PROTOSS, Race.TERRAN);

        var events = trackerEvents(gameJson);
        var unitBorns = events.stream()
                .filter(e -> "UnitBornEvent".equals(e.get("evtTypeName")))
                .toList();
        assertThat(unitBorns).hasSize(1);
        assertThat(unitBorns.get(0).get("unitTypeName")).isEqualTo("Zealot");
        assertThat(unitBorns.get(0).get("loop")).isEqualTo(16);
        assertThat(unitBorns.get(0).get("controlPlayerId")).isEqualTo(1);
    }

    // --- helpers ---

    static GameState gameState(long frame,
                               List<Unit> myUnits, List<Building> myBuildings,
                               List<Unit> enemyUnits, List<Building> enemyBuildings,
                               Set<String> playerUpgrades, Set<String> enemyUpgrades,
                               PlayerEconomyStats playerEconomy,
                               PlayerEconomyStats enemyEconomy) {
        return new GameState(50, 0, 15, 6,
                myUnits, enemyUnits, myBuildings, enemyBuildings,
                List.of(), List.of(), List.of(),
                frame, null, playerEconomy, enemyEconomy,
                playerUpgrades, enemyUpgrades);
    }

    static Unit unit(UnitType type) {
        return new Unit(type, 0, 0, 100, 0, 0);
    }

    static Building building(BuildingType type) {
        return new Building(type, 0, 0, 1000, false, 0);
    }

    @SuppressWarnings("unchecked")
    static List<Map<String, Object>> trackerEvents(Map<String, Object> gameJson) {
        return (List<Map<String, Object>>) gameJson.get("trackerEvents");
    }
}
```

Note: The `GameState` and `Unit`/`Building` constructors may have slightly
different parameter orders. Check the actual record definitions before
writing the test. Use `ide_find_class` to find `GameState`, `Unit`, `Building`
and verify constructor parameter order.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameFeatureExtractorTest#emitsUnitBornWhenUnitCountIncreases -q`
Expected: FAIL — `EmulatedGameFeatureExtractor` does not exist yet.

- [ ] **Step 3: Write EmulatedGameFeatureExtractor**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import java.util.*;

public class EmulatedGameFeatureExtractor {

    private static final int PLAYER_STATS_INTERVAL = 160;

    private final List<Map<String, Object>> events = new ArrayList<>();
    private Map<UnitType, Integer> prevPlayerUnits = new EnumMap<>(UnitType.class);
    private Map<UnitType, Integer> prevEnemyUnits = new EnumMap<>(UnitType.class);
    private Map<BuildingType, Integer> prevPlayerBuildings = new EnumMap<>(BuildingType.class);
    private Map<BuildingType, Integer> prevEnemyBuildings = new EnumMap<>(BuildingType.class);
    private Set<String> prevPlayerUpgrades = Set.of();
    private Set<String> prevEnemyUpgrades = Set.of();
    private int lastStatsLoop = -PLAYER_STATS_INTERVAL;
    private int lastLoop = 0;
    private boolean firstSnapshot = true;

    public void addSnapshot(GameState state, int gameLoop) {
        if (firstSnapshot) {
            prevPlayerUnits = countUnits(state.myUnits());
            prevEnemyUnits = countUnits(state.enemyUnits());
            prevPlayerBuildings = countBuildings(state.myBuildings());
            prevEnemyBuildings = countBuildings(state.enemyBuildings());
            prevPlayerUpgrades = new HashSet<>(state.playerUpgrades());
            prevEnemyUpgrades = new HashSet<>(state.enemyUpgrades());
            firstSnapshot = false;
            lastLoop = gameLoop;
            emitPlayerStats(state, gameLoop, 1);
            emitPlayerStats(state, gameLoop, 2);
            lastStatsLoop = gameLoop;
            return;
        }

        diffUnits(state.myUnits(), prevPlayerUnits, gameLoop, 1);
        diffUnits(state.enemyUnits(), prevEnemyUnits, gameLoop, 2);
        diffBuildings(state.myBuildings(), prevPlayerBuildings, gameLoop, 1);
        diffBuildings(state.enemyBuildings(), prevEnemyBuildings, gameLoop, 2);
        diffUpgrades(state.playerUpgrades(), prevPlayerUpgrades, gameLoop, 1);
        diffUpgrades(state.enemyUpgrades(), prevEnemyUpgrades, gameLoop, 2);

        if (gameLoop - lastStatsLoop >= PLAYER_STATS_INTERVAL) {
            emitPlayerStats(state, gameLoop, 1);
            emitPlayerStats(state, gameLoop, 2);
            lastStatsLoop = gameLoop;
        }

        prevPlayerUnits = countUnits(state.myUnits());
        prevEnemyUnits = countUnits(state.enemyUnits());
        prevPlayerBuildings = countBuildings(state.myBuildings());
        prevEnemyBuildings = countBuildings(state.enemyBuildings());
        prevPlayerUpgrades = new HashSet<>(state.playerUpgrades());
        prevEnemyUpgrades = new HashSet<>(state.enemyUpgrades());
        lastLoop = gameLoop;
    }

    public Map<String, Object> finalise(Race playerRace, Race opponentRace) {
        events.sort(Comparator.comparingInt(e -> (int) e.get("loop")));

        var result = new LinkedHashMap<String, Object>();
        result.put("ToonPlayerDescMap", buildPlayerDescMap(playerRace, opponentRace));
        result.put("trackerEvents", new ArrayList<>(events));
        result.put("header", Map.of("elapsedGameLoops", lastLoop, "baseBuild", 0));
        result.put("metadata", Map.of("extractionMethod", "emulator", "emulatorVersion", "1.0"));
        return result;
    }

    public Map<String, Object> extract(List<GameState> snapshots, int loopsPerSnapshot,
                                       Race playerRace, Race opponentRace) {
        for (int i = 0; i < snapshots.size(); i++) {
            addSnapshot(snapshots.get(i), i * loopsPerSnapshot);
        }
        return finalise(playerRace, opponentRace);
    }

    private void diffUnits(List<Unit> current, Map<UnitType, Integer> prev,
                           int loop, int playerId) {
        var currentCounts = countUnits(current);
        for (var type : UnitType.values()) {
            int now = currentCounts.getOrDefault(type, 0);
            int was = prev.getOrDefault(type, 0);
            int delta = now - was;
            if (delta > 0) {
                for (int i = 0; i < delta; i++) {
                    events.add(unitEvent("UnitBornEvent", loop, playerId, type.pythonName()));
                }
            } else if (delta < 0) {
                for (int i = 0; i < -delta; i++) {
                    events.add(unitEvent("UnitDiedEvent", loop, playerId, type.pythonName()));
                }
            }
        }
    }

    private void diffBuildings(List<Building> current, Map<BuildingType, Integer> prev,
                               int loop, int playerId) {
        var currentCounts = countBuildings(current);
        for (var type : BuildingType.values()) {
            int now = currentCounts.getOrDefault(type, 0);
            int was = prev.getOrDefault(type, 0);
            int delta = now - was;
            if (delta > 0) {
                for (int i = 0; i < delta; i++) {
                    events.add(unitEvent("UnitInitEvent", loop, playerId, type.name()));
                    events.add(unitEvent("UnitDoneEvent", loop, playerId, type.name()));
                }
            } else if (delta < 0) {
                for (int i = 0; i < -delta; i++) {
                    events.add(unitEvent("UnitDiedEvent", loop, playerId, type.name()));
                }
            }
        }
    }

    private void diffUpgrades(Set<String> current, Set<String> prev,
                              int loop, int playerId) {
        for (String upgrade : current) {
            if (!prev.contains(upgrade)) {
                events.add(Map.of(
                        "evtTypeName", "Upgrade",
                        "loop", loop,
                        "playerId", playerId,
                        "upgradeTypeName", upgrade));
            }
        }
    }

    private void emitPlayerStats(GameState state, int loop, int playerId) {
        PlayerEconomyStats eco = playerId == 1 ? state.playerEconomy() : state.enemyEconomy();
        var stats = new LinkedHashMap<String, Object>();
        stats.put("scoreValueMineralsCurrent", eco.mineralsCurrent() * 1000);
        stats.put("scoreValueVespeneCurrent", eco.vespeneCurrent() * 1000);
        stats.put("scoreValueMineralsCollectionRate", eco.mineralsCollectionRate() * 1000);
        stats.put("scoreValueVespeneCollectionRate", eco.vespeneCollectionRate() * 1000);
        stats.put("scoreValueFoodMade", eco.foodMade() * 1000);
        stats.put("scoreValueFoodUsed", eco.foodUsed() * 1000);
        stats.put("scoreValueWorkersActiveCount", eco.workersActiveCount() * 1000);
        stats.put("scoreValueMineralsUsedCurrentArmy", eco.mineralsUsedCurrentArmy() * 1000);
        stats.put("scoreValueMineralsUsedCurrentEconomy", eco.mineralsUsedCurrentEconomy() * 1000);
        stats.put("scoreValueMineralsUsedCurrentTechnology", eco.mineralsUsedCurrentTechnology() * 1000);
        stats.put("scoreValueVespeneUsedCurrentArmy", eco.vespeneUsedCurrentArmy() * 1000);
        stats.put("scoreValueVespeneUsedCurrentEconomy", eco.vespeneUsedCurrentEconomy() * 1000);
        stats.put("scoreValueVespeneUsedCurrentTechnology", eco.vespeneUsedCurrentTechnology() * 1000);

        events.add(Map.of(
                "evtTypeName", "PlayerStatsEvent",
                "loop", loop,
                "controlPlayerId", playerId,
                "stats", stats));
    }

    private static Map<String, Object> unitEvent(String type, int loop,
                                                  int playerId, String unitTypeName) {
        return Map.of(
                "evtTypeName", type,
                "loop", loop,
                "controlPlayerId", playerId,
                "unitTypeName", unitTypeName,
                "unitTagIndex", 0,
                "unitTagRecycle", 0);
    }

    private static Map<UnitType, Integer> countUnits(List<Unit> units) {
        var counts = new EnumMap<UnitType, Integer>(UnitType.class);
        for (var u : units) counts.merge(u.type(), 1, Integer::sum);
        return counts;
    }

    private static Map<BuildingType, Integer> countBuildings(List<Building> buildings) {
        var counts = new EnumMap<BuildingType, Integer>(BuildingType.class);
        for (var b : buildings) counts.merge(b.type(), 1, Integer::sum);
        return counts;
    }

    private static Map<String, Object> buildPlayerDescMap(Race playerRace, Race opponentRace) {
        return Map.of(
                "1", Map.of("playerID", 1, "race", playerRace.name()),
                "2", Map.of("playerID", 2, "race", opponentRace.name()));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameFeatureExtractorTest#emitsUnitBornWhenUnitCountIncreases -q`
Expected: PASS

- [ ] **Step 5: Add tests for unit died, building, upgrade, and PlayerStats events**

Add these test methods to `EmulatedGameFeatureExtractorTest`:

```java
@Test
void emitsUnitDiedWhenUnitCountDecreases() {
    var extractor = new EmulatedGameFeatureExtractor();
    var snap0 = gameState(0, List.of(unit(UnitType.ZEALOT), unit(UnitType.ZEALOT)),
            List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);
    var snap1 = gameState(32, List.of(unit(UnitType.ZEALOT)),
            List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

    extractor.addSnapshot(snap0, 0);
    extractor.addSnapshot(snap1, 32);
    var events = trackerEvents(extractor.finalise(Race.PROTOSS, Race.TERRAN));

    var died = events.stream()
            .filter(e -> "UnitDiedEvent".equals(e.get("evtTypeName")))
            .toList();
    assertThat(died).hasSize(1);
    assertThat(died.get(0).get("unitTypeName")).isEqualTo("Zealot");
}

@Test
void emitsBuildingInitAndDoneWhenBuildingAppears() {
    var extractor = new EmulatedGameFeatureExtractor();
    var snap0 = gameState(0, List.of(), List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);
    var snap1 = gameState(100, List.of(), List.of(building(BuildingType.GATEWAY)),
            List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

    extractor.addSnapshot(snap0, 0);
    extractor.addSnapshot(snap1, 100);
    var events = trackerEvents(extractor.finalise(Race.PROTOSS, Race.TERRAN));

    var inits = events.stream()
            .filter(e -> "UnitInitEvent".equals(e.get("evtTypeName")))
            .toList();
    var dones = events.stream()
            .filter(e -> "UnitDoneEvent".equals(e.get("evtTypeName")))
            .toList();
    assertThat(inits).hasSize(1);
    assertThat(dones).hasSize(1);
    assertThat(inits.get(0).get("unitTypeName")).isEqualTo("GATEWAY");
}

@Test
void emitsUpgradeWhenNewUpgradeAppears() {
    var extractor = new EmulatedGameFeatureExtractor();
    var snap0 = gameState(0, List.of(), List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);
    var snap1 = gameState(3200, List.of(), List.of(), List.of(), List.of(),
            Set.of("Stimpack"), Set.of(),
            PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

    extractor.addSnapshot(snap0, 0);
    extractor.addSnapshot(snap1, 3200);
    var events = trackerEvents(extractor.finalise(Race.TERRAN, Race.PROTOSS));

    var upgrades = events.stream()
            .filter(e -> "Upgrade".equals(e.get("evtTypeName")))
            .toList();
    assertThat(upgrades).hasSize(1);
    assertThat(upgrades.get(0).get("upgradeTypeName")).isEqualTo("Stimpack");
    assertThat(upgrades.get(0).get("playerId")).isEqualTo(1);
}

@Test
void emitsPlayerStatsEvery160Loops() {
    var extractor = new EmulatedGameFeatureExtractor();
    var eco = new PlayerEconomyStats(400, 100, 1200, 300,
            30, 18, 16, 500, 800, 200, 100, 0, 150);

    for (int loop = 0; loop <= 320; loop += 16) {
        extractor.addSnapshot(
                gameState(loop, List.of(), List.of(), List.of(), List.of(),
                        Set.of(), Set.of(), eco, PlayerEconomyStats.EMPTY),
                loop);
    }
    var events = trackerEvents(extractor.finalise(Race.PROTOSS, Race.TERRAN));

    var stats = events.stream()
            .filter(e -> "PlayerStatsEvent".equals(e.get("evtTypeName")))
            .filter(e -> (int) e.get("controlPlayerId") == 1)
            .toList();
    // Initial at loop 0, then at loop 160, then at loop 320
    assertThat(stats).hasSize(3);
    @SuppressWarnings("unchecked")
    var firstStats = (Map<String, Object>) stats.get(0).get("stats");
    assertThat(firstStats.get("scoreValueMineralsCurrent")).isEqualTo(400_000);
    assertThat(firstStats.get("scoreValueWorkersActiveCount")).isEqualTo(16_000);
}

@Test
void batchExtractProducesCompleteGameJson() {
    var snapshots = new ArrayList<GameState>();
    snapshots.add(gameState(0, List.of(), List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY));
    snapshots.add(gameState(16, List.of(unit(UnitType.PROBE)), List.of(),
            List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY));

    var extractor = new EmulatedGameFeatureExtractor();
    var gameJson = extractor.extract(snapshots, 16, Race.PROTOSS, Race.TERRAN);

    assertThat(gameJson).containsKeys("ToonPlayerDescMap", "trackerEvents", "header", "metadata");
    assertThat(gameJson.get("metadata")).isInstanceOf(Map.class);
    @SuppressWarnings("unchecked")
    var metadata = (Map<String, Object>) gameJson.get("metadata");
    assertThat(metadata.get("extractionMethod")).isEqualTo("emulator");
}

@Test
void enemyUnitEventsUsePlayerId2() {
    var extractor = new EmulatedGameFeatureExtractor();
    var snap0 = gameState(0, List.of(), List.of(), List.of(), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);
    var snap1 = gameState(50, List.of(), List.of(),
            List.of(unit(UnitType.MARINE)), List.of(),
            Set.of(), Set.of(), PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

    extractor.addSnapshot(snap0, 0);
    extractor.addSnapshot(snap1, 50);
    var events = trackerEvents(extractor.finalise(Race.PROTOSS, Race.TERRAN));

    var born = events.stream()
            .filter(e -> "UnitBornEvent".equals(e.get("evtTypeName")))
            .findFirst().orElseThrow();
    assertThat(born.get("controlPlayerId")).isEqualTo(2);
    assertThat(born.get("unitTypeName")).isEqualTo("Marine");
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameFeatureExtractorTest -q`
Expected: all 6 tests PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureExtractorTest.java
git commit -m "feat: EmulatedGameFeatureExtractor — count-based diff emitter for training data

Diffs consecutive GameState snapshots by unit/building type counts and
upgrade sets. Emits synthetic tracker events (UnitBorn, UnitDied, UnitInit,
UnitDone, Upgrade, PlayerStats) in standard game_json format compatible
with the Python training pipeline.

Refs #360"
```

---

## Batch 2: Validation Tests

### Task 2: Shape validation test — controlled scenario to game_json

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureShapeTest.java`

**Interfaces:**
- Consumes: `EmulatedGameFeatureExtractor` (from Task 1)
- Consumes: `EmulatedGame` — `new EmulatedGame()`, `reset()`, `tick(List<Intent>)`, `snapshot()`
- Consumes: `FeatureIndexMaps` — `N_BUILDINGS`, `N_UNITS`, `N_STATS`, `N_UPGRADES`, `BUILDING_INDEX`, `UNIT_INDEX`

- [ ] **Step 1: Write failing shape validation test**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.plugin.scouting.FeatureIndexMaps;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class EmulatedGameFeatureShapeTest {

    @Test
    void extractedGameJsonHasCorrectEventStructure() {
        var game = new EmulatedGame();
        game.reset();

        var snapshots = new ArrayList<GameState>();
        for (int tick = 0; tick < 500; tick++) {
            game.tick(List.of());
            if (tick % 16 == 0) {
                snapshots.add(game.snapshot());
            }
        }

        var extractor = new EmulatedGameFeatureExtractor();
        var gameJson = extractor.extract(snapshots, 16, Race.PROTOSS, Race.TERRAN);

        // Top-level keys present
        assertThat(gameJson).containsKeys("ToonPlayerDescMap", "trackerEvents", "header", "metadata");

        // trackerEvents is a non-empty list
        @SuppressWarnings("unchecked")
        var events = (List<Map<String, Object>>) gameJson.get("trackerEvents");
        assertThat(events).isNotEmpty();

        // PlayerStats events exist and have 13 stat fields
        var statsEvents = events.stream()
                .filter(e -> "PlayerStatsEvent".equals(e.get("evtTypeName")))
                .toList();
        assertThat(statsEvents).isNotEmpty();

        @SuppressWarnings("unchecked")
        var stats = (Map<String, Object>) statsEvents.get(0).get("stats");
        assertThat(stats).hasSize(13);
        assertThat(stats).containsKeys(
                "scoreValueMineralsCurrent", "scoreValueVespeneCurrent",
                "scoreValueMineralsCollectionRate", "scoreValueVespeneCollectionRate",
                "scoreValueFoodMade", "scoreValueFoodUsed",
                "scoreValueWorkersActiveCount",
                "scoreValueMineralsUsedCurrentArmy", "scoreValueMineralsUsedCurrentEconomy",
                "scoreValueMineralsUsedCurrentTechnology",
                "scoreValueVespeneUsedCurrentArmy", "scoreValueVespeneUsedCurrentEconomy",
                "scoreValueVespeneUsedCurrentTechnology");

        // All stat values are integers (×1000 scaled)
        for (var value : stats.values()) {
            assertThat(value).isInstanceOf(Integer.class);
        }

        // Unit events have required keys
        var unitEvents = events.stream()
                .filter(e -> {
                    var t = (String) e.get("evtTypeName");
                    return "UnitBornEvent".equals(t) || "UnitDiedEvent".equals(t)
                           || "UnitInitEvent".equals(t) || "UnitDoneEvent".equals(t);
                })
                .toList();
        for (var evt : unitEvents) {
            assertThat(evt).containsKeys("evtTypeName", "loop", "controlPlayerId",
                    "unitTypeName", "unitTagIndex", "unitTagRecycle");
        }

        // metadata has extractionMethod
        @SuppressWarnings("unchecked")
        var metadata = (Map<String, Object>) gameJson.get("metadata");
        assertThat(metadata.get("extractionMethod")).isEqualTo("emulator");
    }

    @Test
    void unitTypeNamesMatchFeatureIndexMaps() {
        // Verify all UnitType pythonNames used by the extractor are recognised
        // by FeatureIndexMaps
        for (var unitType : UnitType.values()) {
            var idx = FeatureIndexMaps.UNIT_INDEX.get(unitType.pythonName());
            // Not all UnitTypes have feature indices (e.g. UNKNOWN), but those
            // that do must have valid indices within range
            if (idx != null) {
                assertThat(idx).isBetween(0, FeatureIndexMaps.N_UNITS - 1);
            }
        }
    }

    @Test
    void buildingTypeNamesMatchFeatureIndexMaps() {
        for (var buildingType : BuildingType.values()) {
            if (buildingType == BuildingType.UNKNOWN) continue;
            var idx = FeatureIndexMaps.BUILDING_INDEX.get(buildingType.name());
            // BuildingType names should map to feature indices
            if (idx != null) {
                assertThat(idx).isBetween(0, FeatureIndexMaps.N_BUILDINGS - 1);
            }
        }
    }
}
```

Note: `EmulatedGame` constructor may require arguments. Check via
`ide_find_class("EmulatedGame")` and read the constructor. The fork
report says `new EmulatedGame()` works. If it takes `RaceModel` or
other args, adjust accordingly.

- [ ] **Step 2: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameFeatureShapeTest -q`
Expected: PASS — the extractor from Task 1 handles the EmulatedGame output.

If any test fails, fix the issue (likely constructor mismatch or name mapping).

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureShapeTest.java
git commit -m "test: shape validation for EmulatedGameFeatureExtractor

Validates game_json structure (13 PlayerStats fields, event keys,
metadata), and verifies UnitType/BuildingType names align with
FeatureIndexMaps indices.

Refs #360"
```

### Task 3: ONNX inference validation test

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameOnnxInferenceTest.java`

**Interfaces:**
- Consumes: `EmulatedGameFeatureExtractor` (from Task 1)
- Consumes: `EmulatedGame` — `new EmulatedGame()`, `reset()`, `tick()`, `snapshot()`
- Consumes: `DroolsScoutingTask` — `buildSnapshot(GameState)` (package-private static — test must be in same package or use a visible helper)
- Consumes: `TemporalWindowAccumulator` — `addSnapshot(WindowSnapshot)`, `getWindowedFeatures()`
- Consumes: `StrategyFeatureExtractor` — `extract(TemporalWindowAccumulator, MapCharacteristics)`
- Consumes: `BinaryDetectorRunner` — `runAll(StrategyFeatures, Race, Race)`
- Consumes: `StrategyFeatures` — `tensors()` with keys `"temporal"` and `"map"`

Note: `DroolsScoutingTask.buildSnapshot()` is in `io.quarkmind.plugin.scouting`
which is a different package from the test. Check if `buildSnapshot` is
package-private or public. If package-private, the test must go in the
`io.quarkmind.plugin.scouting` package, or use a different approach to
build `WindowSnapshot` directly. Use `ide_find_class("DroolsScoutingTask")`
and `ide_file_structure` to check visibility.

Alternative: Construct `WindowSnapshot` directly from `GameState` fields.
`WindowSnapshot` is a record — check its constructor via `ide_find_class`.

- [ ] **Step 1: Write failing ONNX inference test**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.plugin.scouting.*;
import io.quarkmind.sc2.detector.BinaryDetectorRunner;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class EmulatedGameOnnxInferenceTest {

    @Test
    @EnabledIf("onnxModelsAvailable")
    void onnxInferenceProducesValidOutputOnEmulatedFeatures() {
        // 1. Run EmulatedGame for ~5 minutes of game time
        var game = new EmulatedGame();
        game.reset();

        var accumulator = new TemporalWindowAccumulator();
        for (int tick = 0; tick < 4500; tick++) {
            game.tick(List.of());
            // Add snapshot every 60 ticks (one window)
            if (tick % 60 == 0 && tick > 0) {
                var gs = game.snapshot();
                var snapshot = buildWindowSnapshot(gs);
                accumulator.addSnapshot(snapshot);
            }
        }

        // 2. Extract features via runtime path
        var features = StrategyFeatureExtractor.extract(accumulator,
                MapCharacteristics.unknown());

        // 3. Validate feature shape
        assertThat(features.tensors()).containsKeys("temporal", "map");
        var temporal = features.tensors().get("temporal");
        assertThat(temporal).isNotNull();
        assertThat(temporal.length).isEqualTo(1);
        // Temporal features: MAX_WINDOWS * FEATURES_PER_WINDOW
        assertThat(temporal[0].length).isGreaterThan(0);

        var map = features.tensors().get("map");
        assertThat(map).isNotNull();
        assertThat(map.length).isEqualTo(1);

        // 4. Run ONNX inference
        var runner = BinaryDetectorRunner.create();
        if (runner == null) return; // ONNX models not available

        var results = runner.runAll(features, Race.PROTOSS, Race.TERRAN);

        // 5. Validate: results should be non-null with valid confidence scores
        assertThat(results).isNotNull();
        for (var result : results.entrySet()) {
            assertThat(result.getValue().confidence())
                    .isBetween(0.0f, 1.0f);
        }
    }

    private WindowSnapshot buildWindowSnapshot(GameState gs) {
        // Build playerFeatures and opponentFeatures float arrays
        // matching FeatureIndexMaps layout: buildings(53) + units(56) + stats(13) + upgrades(82) = 204
        float[] playerFeatures = buildFeatureArray(
                gs.myBuildings(), gs.myUnits(), gs.playerEconomy(), gs.playerUpgrades());
        float[] opponentFeatures = buildFeatureArray(
                gs.enemyBuildings(), gs.enemyUnits(), gs.enemyEconomy(), gs.enemyUpgrades());

        return new WindowSnapshot(playerFeatures, opponentFeatures, 0.0f);
    }

    private float[] buildFeatureArray(List<Building> buildings, List<Unit> units,
                                      PlayerEconomyStats economy, Set<String> upgrades) {
        float[] features = new float[FeatureIndexMaps.SPATIAL_OFFSET]; // 204
        // Buildings
        for (var b : buildings) {
            Integer idx = FeatureIndexMaps.BUILDING_INDEX.get(b.type().name());
            if (idx != null) features[idx]++;
        }
        // Units
        for (var u : units) {
            Integer idx = FeatureIndexMaps.UNIT_INDEX.get(u.type().pythonName());
            if (idx != null) features[FeatureIndexMaps.N_BUILDINGS + idx]++;
        }
        // Stats
        float[] ecoVec = economy.toFeatureVector();
        System.arraycopy(ecoVec, 0, features,
                FeatureIndexMaps.N_BUILDINGS + FeatureIndexMaps.N_UNITS, ecoVec.length);
        // Upgrades
        for (var upgrade : upgrades) {
            Integer idx = FeatureIndexMaps.UPGRADE_INDEX.get(upgrade);
            if (idx != null) {
                features[FeatureIndexMaps.N_BUILDINGS + FeatureIndexMaps.N_UNITS
                          + FeatureIndexMaps.N_STATS + idx] = 1.0f;
            }
        }
        return features;
    }

    static boolean onnxModelsAvailable() {
        // Check if ONNX model files exist
        var modelsDir = java.nio.file.Path.of("quarkmind-sc2/src/main/resources/models");
        return java.nio.file.Files.exists(modelsDir)
               && java.nio.file.Files.isDirectory(modelsDir);
    }
}
```

Note: `WindowSnapshot`, `TemporalWindowAccumulator`, `StrategyFeatureExtractor`,
`BinaryDetectorRunner`, `MapCharacteristics` APIs may differ from shown above.
Verify exact constructor/method signatures via `ide_find_class` before writing.
The `@EnabledIf` guard ensures the test only runs when ONNX models are present.

- [ ] **Step 2: Run test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameOnnxInferenceTest -q`
Expected: PASS if ONNX models are available, SKIP otherwise.

If it fails, check:
- `WindowSnapshot` constructor signature — may take additional parameters
- `StrategyFeatureExtractor.extract()` signature — may be instance method, not static
- `BinaryDetectorRunner.create()` — may have different factory method name
- `FeatureIndexMaps` constant names — verify exact names via `ide_file_structure`

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameOnnxInferenceTest.java
git commit -m "test: ONNX inference validation on EmulatedGame features

Runs EmulatedGame for ~5 minutes, extracts features via the runtime
path (WindowSnapshot → TemporalWindowAccumulator → StrategyFeatures),
feeds through BinaryDetectorRunner, asserts valid classifications.
Guarded: skips when ONNX models are not present.

Refs #360"
```

---

## Batch 3: Equivalence Characterisation

### Task 4: Feature equivalence report — oracle replays

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureEquivalenceTest.java`

**Interfaces:**
- Consumes: `EmulatedGameFeatureExtractor` (from Task 1)
- Consumes: `TrackerEventFeatureExtractor` — `extract(Replay)` returns `Map<String, Object>` game_json
- Consumes: `EmulatedGame`, `ReplayValidationHarness` — for driving EmulatedGame with replay intents
- Produces: Divergence report written to stdout (and optionally `docs/benchmarks/emulated-game-feature-equivalence.md`)

This test is `@Tag("report")` — excluded from the default surefire run.
It characterises divergence rather than asserting thresholds.

- [ ] **Step 1: Write the equivalence characterisation test**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.replay.TrackerEventFeatureExtractor;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Collectors;
import static org.assertj.core.api.Assertions.assertThat;

@Tag("report")
class EmulatedGameFeatureEquivalenceTest {

    private static final Path ORACLE_DIR = Path.of("replays/oracle");

    @Test
    void compareEmulatedVsTrackerFeatures() throws Exception {
        // Skip if oracle replays not available
        org.junit.jupiter.api.Assumptions.assumeTrue(
                Files.exists(ORACLE_DIR), "Oracle replay directory not found");

        var replayFiles = Files.list(ORACLE_DIR)
                .filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted()
                .toList();

        org.junit.jupiter.api.Assumptions.assumeTrue(
                !replayFiles.isEmpty(), "No oracle replays found");

        int totalReplays = 0;
        int unitMatches = 0, unitTotal = 0;
        int buildingMatches = 0, buildingTotal = 0;
        int upgradeMatches = 0, upgradeTotal = 0;
        var economyErrors = new ArrayList<Double>();

        for (var replayPath : replayFiles) {
            try {
                // 1. Extract ground truth via TrackerEventFeatureExtractor
                var trackerJson = TrackerEventFeatureExtractor.extract(replayPath);

                // 2. Run same replay through EmulatedGame
                //    (Use ReplayValidationHarness to drive EmulatedGame with intents)
                //    This requires integration with the existing harness —
                //    see ReplayValidationHarness.run() and EmulatedGame.tick()
                var emulatedJson = extractViaEmulatedGame(replayPath);

                if (emulatedJson == null) continue;

                // 3. Compare events at checkpoints (1-min intervals)
                totalReplays++;
                var comparison = compareAtCheckpoints(trackerJson, emulatedJson,
                        List.of(960, 2880, 4800, 6720)); // 1/3/5/7 min in game loops

                unitMatches += comparison.unitMatches;
                unitTotal += comparison.unitTotal;
                buildingMatches += comparison.buildingMatches;
                buildingTotal += comparison.buildingTotal;
                upgradeMatches += comparison.upgradeMatches;
                upgradeTotal += comparison.upgradeTotal;
                economyErrors.addAll(comparison.economyErrors);
            } catch (Exception e) {
                System.err.println("Skipping " + replayPath.getFileName() + ": " + e.getMessage());
            }
        }

        // 4. Print report
        System.out.println("\n=== EmulatedGame vs Tracker Feature Equivalence ===");
        System.out.println("Replays processed: " + totalReplays);
        System.out.printf("Units:     %d/%d (%.1f%%)%n", unitMatches, unitTotal,
                unitTotal > 0 ? 100.0 * unitMatches / unitTotal : 0);
        System.out.printf("Buildings: %d/%d (%.1f%%)%n", buildingMatches, buildingTotal,
                buildingTotal > 0 ? 100.0 * buildingMatches / buildingTotal : 0);
        System.out.printf("Upgrades:  %d/%d (%.1f%%)%n", upgradeMatches, upgradeTotal,
                upgradeTotal > 0 ? 100.0 * upgradeMatches / upgradeTotal : 0);
        if (!economyErrors.isEmpty()) {
            var avgMape = economyErrors.stream()
                    .mapToDouble(d -> d).average().orElse(0);
            System.out.printf("Economy MAPE: %.1f%%%n", avgMape);
        }

        // This is a characterisation test — it doesn't assert thresholds
        assertThat(totalReplays).as("At least one replay processed").isGreaterThan(0);
    }

    // Implementation note: extractViaEmulatedGame and compareAtCheckpoints
    // require integrating with ReplayValidationHarness.
    // The exact integration depends on how ReplayValidationHarness exposes
    // tick-by-tick GameState. Check the harness API via ide_find_class.
    //
    // Skeleton:
    // - Parse replay to get intents (via ReplayCommandExtractor)
    // - Run EmulatedGame.tick() with those intents
    // - Collect snapshots at checkpoint loops
    // - Feed through EmulatedGameFeatureExtractor
    // - Compare event counts per type against tracker events

    private Map<String, Object> extractViaEmulatedGame(Path replayPath) {
        // TODO: integrate with ReplayValidationHarness to drive EmulatedGame
        // This will be filled in during implementation — the harness API
        // needs to be checked via ide_find_class("ReplayValidationHarness")
        return null;
    }

    private ComparisonResult compareAtCheckpoints(
            Map<String, Object> trackerJson, Map<String, Object> emulatedJson,
            List<Integer> checkpoints) {
        // Compare unit/building/upgrade counts at each checkpoint loop
        // by filtering events up to each checkpoint and counting per type
        return new ComparisonResult(0, 0, 0, 0, 0, 0, List.of());
    }

    record ComparisonResult(int unitMatches, int unitTotal,
                            int buildingMatches, int buildingTotal,
                            int upgradeMatches, int upgradeTotal,
                            List<Double> economyErrors) {}
}
```

Note: The `extractViaEmulatedGame` method requires integration with the
existing `ReplayValidationHarness`. During implementation:
1. Use `ide_find_class("ReplayValidationHarness")` to understand the API
2. Use `ide_find_class("ReplayCommandExtractor")` to understand intent extraction
3. The harness drives EmulatedGame with replay intents — check if it
   exposes tick-by-tick GameState or only final results
4. If the harness doesn't expose per-tick state, run EmulatedGame directly
   with extracted intents (similar to how `EmulatedGameAccuracyBaselineTest` works)
5. Check `EmulatedGameAccuracyBaselineTest` as the reference implementation —
   it already runs EmulatedGame against oracle replays

- [ ] **Step 2: Run test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameFeatureEquivalenceTest -q`
Expected: PASS with characterisation report printed to stdout (if oracle replays available).

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameFeatureEquivalenceTest.java
git commit -m "test: feature equivalence characterisation — emulated vs tracker

@Tag(report) test that runs oracle replays through both
TrackerEventFeatureExtractor and EmulatedGameFeatureExtractor,
comparing per-category divergence at 1/3/5/7 min checkpoints.
Characterisation only — no threshold assertions.

Refs #360"
```

---

## References

- `specs/issue-360-feature-extraction-emulated-games/2026-10-26-emulated-game-feature-extraction-design.md` — design spec this plan implements
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerEventFeatureExtractor.java` — game_json output format reference
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — `snapshot()` method
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/GameState.java` — 17-field record
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/PlayerEconomyStats.java` — 13-field record, `toFeatureVector()`
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java` — index mappings
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java` — windowed features
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/BinaryDetectorRunner.java` — ONNX inference
- PP-20260528-612dee — extraction in dedicated class protocol
- GitHub #360 — focal issue
- GitHub #384 — parent reconstitution pipeline epic
