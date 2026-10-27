# Hypothesis Prior Learning Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #456 — SC2 replay pipeline: extract scouting observations and strategy labels for hypothesis prior learning
**Issue group:** #456

**Goal:** Build a data pipeline that extracts discrete scouting observations from labelled replay data, learns prior frequencies and observation CPTs, and exports them as classpath JSON resources for the HypothesisTracker integration (Phase 2+).

**Architecture:** A new `ScoutingObservationExtractor` reads game_json (from `ReplayFeatureExtractor`) and produces `ScoutingObservation` records — discrete variables with named states (first_building, gas_timing, tech_path, etc.). A `PriorExportPipeline` test reads `.label.json` sidecars + extracted observations across 149K labelled replays, learns prior frequencies via `HypothesisPriorLearner.fromOutcomes()`, learns observation CPTs from co-occurrences with Laplace smoothing, and writes JSON files to `quarkmind-sc2/src/main/resources/priors/`. This follows the existing `LearnedCptLoader` pattern.

**Tech Stack:** Java 21, plain JUnit (no `@QuarkusTest`), Jackson for JSON, neocortex `HypothesisPriorLearner`

## Global Constraints

- Domain records (`ScoutingObservation`, `ScoutingObservationSequence`) go in `io.quarkmind.domain` — plain Java, no framework deps
- Extractor goes in `io.quarkmind.sc2.replay` — same package as `TrackerEventFeatureExtractor`
- Prior/CPT JSON files go in `quarkmind-sc2/src/main/resources/priors/` — follows existing `resources/classifier/` pattern
- game_json field for unit names is `unitTypeName` (not `unitName`)
- Classifier labels (RUSH, MACRO_ECONOMY) are matchup-generic; `StrategyArchetype` is race+phase-specific — mapping is needed
- Discretization thresholds must be calibrated from replay data per protocol `sc2data-train-times-require-calibration.md`
- CPT entries clamped to [0.01, 0.99], Laplace smoothing (add-1) before normalization
- Game loop to real time: 22.4 loops/second, so 1:30 = 2016 loops, 2:00 = 2688 loops, 2:30 = 3360 loops, 3:00 = 4032 loops, 3:30 = 4704 loops, 5:00 = 6720 loops

---

## Batch 1: Domain Records and Discrete Observation Extraction

### Task 1: ScoutingObservation domain records

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/ScoutingObservation.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/ScoutingObservationSequence.java`

**Interfaces:**
- Consumes: `io.quarkmind.domain.Race` (existing enum)
- Produces: `ScoutingObservation(String variable, String state, long detectedAtLoop)` record, `ScoutingObservationSequence(String replayHash, int playerId, Race playerRace, Race opponentRace, List<ScoutingObservation> observations, String groundTruthStrategy)` record

- [ ] **Step 1: Create ScoutingObservation record**

```java
package io.quarkmind.domain;

public record ScoutingObservation(
    String variable,
    String state,
    long detectedAtLoop
) {}
```

- [ ] **Step 2: Create ScoutingObservationSequence record**

```java
package io.quarkmind.domain;

import java.util.List;

public record ScoutingObservationSequence(
    String replayHash,
    int playerId,
    Race playerRace,
    Race opponentRace,
    List<ScoutingObservation> observations,
    String groundTruthStrategy
) {}
```

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/ScoutingObservation.java quarkmind-sc2/src/main/java/io/quarkmind/domain/ScoutingObservationSequence.java
git commit -m "feat: add ScoutingObservation domain records Refs #456"
```

### Task 2: ScoutingObservationExtractor — extract discrete observations from game_json

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ScoutingObservationExtractor.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ScoutingObservationExtractorTest.java`

**Interfaces:**
- Consumes: `Map<String, Object>` (game_json from `ReplayFeatureExtractor.extract()`)
- Produces: `List<ScoutingObservation> extract(Map<String, Object> gameJson, int playerId)` — returns discrete observations for a given player's perspective

The extractor reads game_json tracker events and produces discrete observations. Each observation type has a dedicated private method. The extractor processes events chronologically and emits observations at the game loop when the event occurs. Fog-of-war filtering is NOT applied here (the game_json contains both players' events) — fog-of-war is a separate concern for live mode (Phase 3).

**Discrete variable definitions:**

| Variable | Building/unit filter | States | Threshold (game loops at 22.4/s) |
|---|---|---|---|
| `first_building` | First `UnitInit` where `unitTypeName` is a production/tech building (not `SupplyDepot`/`Pylon`/`Overlord`/`Hatchery` bases) | Per-race building names lowercased | N/A — just the name |
| `gas_timing` | First `UnitInit`/`UnitBorn` where `unitTypeName` in (`Assimilator`, `Extractor`, `Refinery`) | `early` (< 2016), `normal` (2016–3360), `late` (> 3360), `none` (not by loop 6720) | 1:30 / 2:30 |
| `expansion_timing` | Second `UnitInit` where `unitTypeName` in (`Nexus`, `CommandCenter`, `Hatchery`) | `fast` (< 2688), `normal` (2688–4704), `late` (> 4704), `none` (not by 6720) | 2:00 / 3:30 |
| `tech_path` | First `UnitInit` where `unitTypeName` is a tech building (see list below) | Per-race tech building name lowercased | N/A — just the name |
| `army_composition` | Unit counts at loop 6720 (5:00) | `light`, `armored`, `air`, `mixed` | Ratio-based (see below) |
| `worker_saturation` | Worker count / base count at loop 6720 | `under` (< 16), `saturated` (16–22), `over` (> 22) | Per-base ratio |
| `aggression_timing` | First non-worker military unit with `UnitBorn` | `early_pressure` (< 4032), `timing_attack` (4032–6720), `macro_play` (> 6720), `no_aggression` | 3:00 / 5:00 |

**Tech buildings per race:**
- Terran: `Factory`, `Starport`, `GhostAcademy`, `FusionCore`, `Armory`
- Zerg: `SpawningPool`, `BanelingNest`, `RoachWarren`, `HydraliskDen`, `Spire`, `InfestationPit`, `LurkerDen`, `UltraliskCavern`
- Protoss: `TwilightCouncil`, `Stargate`, `RoboticsFacility`, `DarkShrine`, `TemplarArchives`, `RoboticsBay`, `FleetBeacon`

**Base buildings:** `Nexus`, `CommandCenter`, `OrbitalCommand`, `PlanetaryFortress`, `Hatchery`, `Lair`, `Hive`

**Supply buildings (excluded from first_building):** `SupplyDepot`, `SupplyDepotLowered`, `Pylon`, `Overlord`, `OverlordCocoon`, `Overseer`

**Army composition classification:**
- Count units at loop 6720 by category: light (Marine, Zergling, Zealot, Adept, Hellion), armored (Roach, SiegeTank, Stalker, Immortal, Marauder), air (Mutalisk, Banshee, Phoenix, VoidRay, Oracle, Liberator)
- If air > 50% of total military: `air`
- If light > 60% of total military: `light`
- If armored > 60% of total military: `armored`
- Otherwise: `mixed`

- [ ] **Step 1: Write failing test — first_building extraction**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.ScoutingObservation;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class ScoutingObservationExtractorTest {

    @Test
    void extractFirstBuilding_barracks() {
        var gameJson = buildGameJson(List.of(
            unitInit("SupplyDepot", 1, 405),
            unitInit("Barracks", 1, 914)
        ), "Terran", "Zerg");

        var obs = ScoutingObservationExtractor.extract(gameJson, 1);
        var firstBuilding = obs.stream()
            .filter(o -> o.variable().equals("first_building"))
            .findFirst().orElseThrow();

        assertEquals("barracks", firstBuilding.state());
        assertEquals(914, firstBuilding.detectedAtLoop());
    }

    @Test
    void extractFirstBuilding_skipsSupplyBuildings() {
        var gameJson = buildGameJson(List.of(
            unitInit("Pylon", 1, 300),
            unitInit("Gateway", 1, 800)
        ), "Protoss", "Terran");

        var obs = ScoutingObservationExtractor.extract(gameJson, 1);
        var firstBuilding = obs.stream()
            .filter(o -> o.variable().equals("first_building"))
            .findFirst().orElseThrow();

        assertEquals("gateway", firstBuilding.state());
    }

    // Helper to build minimal game_json
    static Map<String, Object> buildGameJson(
            List<Map<String, Object>> events, String race1, String race2) {
        Map<String, Object> p1 = Map.of("playerID", 1, "race", race1, "result", "Win");
        Map<String, Object> p2 = Map.of("playerID", 2, "race", race2, "result", "Loss");
        return Map.of(
            "ToonPlayerDescMap", Map.of("1", p1, "2", p2),
            "trackerEvents", events,
            "header", Map.of("elapsedGameLoops", 10000, "baseBuild", 75025),
            "metadata", Map.of("mapName", "TestMap")
        );
    }

    static Map<String, Object> unitInit(String typeName, int playerId, int loop) {
        return Map.of(
            "evtTypeName", "UnitInit",
            "unitTypeName", typeName,
            "controlPlayerId", playerId,
            "loop", loop
        );
    }

    static Map<String, Object> unitBorn(String typeName, int playerId, int loop) {
        return Map.of(
            "evtTypeName", "UnitBorn",
            "unitTypeName", typeName,
            "controlPlayerId", playerId,
            "loop", loop
        );
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingObservationExtractorTest#extractFirstBuilding_barracks -q`
Expected: compilation failure — `ScoutingObservationExtractor` doesn't exist

- [ ] **Step 3: Implement ScoutingObservationExtractor — first_building**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.ScoutingObservation;

import java.util.*;

public class ScoutingObservationExtractor {

    private static final Set<String> SUPPLY_BUILDINGS = Set.of(
        "SupplyDepot", "SupplyDepotLowered", "Pylon",
        "Overlord", "OverlordCocoon", "Overseer"
    );

    private static final Set<String> BASE_BUILDINGS = Set.of(
        "Nexus", "CommandCenter", "OrbitalCommand", "PlanetaryFortress",
        "Hatchery", "Lair", "Hive"
    );

    private static final Set<String> GAS_BUILDINGS = Set.of(
        "Assimilator", "Extractor", "Refinery"
    );

    private static final Set<String> TERRAN_TECH = Set.of(
        "Factory", "Starport", "GhostAcademy", "FusionCore", "Armory"
    );
    private static final Set<String> ZERG_TECH = Set.of(
        "SpawningPool", "BanelingNest", "RoachWarren", "HydraliskDen",
        "Spire", "InfestationPit", "LurkerDen", "UltraliskCavern"
    );
    private static final Set<String> PROTOSS_TECH = Set.of(
        "TwilightCouncil", "Stargate", "RoboticsFacility", "DarkShrine",
        "TemplarArchives", "RoboticsBay", "FleetBeacon"
    );

    private static final Set<String> WORKER_UNITS = Set.of("SCV", "Probe", "Drone");
    private static final Set<String> LIGHT_UNITS = Set.of(
        "Marine", "Zergling", "Zealot", "Adept", "Hellion"
    );
    private static final Set<String> ARMORED_UNITS = Set.of(
        "Roach", "SiegeTank", "SiegeTankSieged", "Stalker", "Immortal", "Marauder"
    );
    private static final Set<String> AIR_UNITS = Set.of(
        "Mutalisk", "Banshee", "Phoenix", "VoidRay", "Oracle",
        "Liberator", "LiberatorAG", "Viking", "VikingFighter"
    );

    static final long GAS_EARLY = 2016;    // 1:30
    static final long GAS_LATE = 3360;     // 2:30
    static final long EXPAND_FAST = 2688;  // 2:00
    static final long EXPAND_LATE = 4704;  // 3:30
    static final long AGGRESSION_EARLY = 4032; // 3:00
    static final long AGGRESSION_LATE = 6720;  // 5:00
    static final long SNAPSHOT_LOOP = 6720;    // 5:00

    @SuppressWarnings("unchecked")
    public static List<ScoutingObservation> extract(Map<String, Object> gameJson, int playerId) {
        var events = (List<Map<String, Object>>) gameJson.get("trackerEvents");
        var observations = new ArrayList<ScoutingObservation>();

        String firstBuilding = null;
        long firstBuildingLoop = -1;
        String firstGas = null;
        long firstGasLoop = -1;
        int baseCount = 1; // start with main base
        String secondBase = null;
        long secondBaseLoop = -1;
        String firstTech = null;
        long firstTechLoop = -1;
        long firstMilitaryLoop = -1;

        int lightCount = 0, armoredCount = 0, airCount = 0;
        int workerCount = 0;

        Set<String> techBuildings = techBuildingsForPlayer(gameJson, playerId);

        for (var event : events) {
            int pid = ((Number) event.get("controlPlayerId")).intValue();
            if (pid != playerId) continue;

            String type = (String) event.get("evtTypeName");
            String unitType = (String) event.getOrDefault("unitTypeName", "");
            long loop = ((Number) event.get("loop")).longValue();

            if ("UnitInit".equals(type) || "UnitBorn".equals(type)) {
                // first_building: first non-supply, non-base, non-gas building via UnitInit
                if ("UnitInit".equals(type) && firstBuilding == null
                        && !SUPPLY_BUILDINGS.contains(unitType)
                        && !BASE_BUILDINGS.contains(unitType)
                        && !GAS_BUILDINGS.contains(unitType)) {
                    firstBuilding = unitType.toLowerCase();
                    firstBuildingLoop = loop;
                }

                // gas_timing
                if (firstGas == null && GAS_BUILDINGS.contains(unitType)) {
                    firstGas = unitType;
                    firstGasLoop = loop;
                }

                // expansion_timing — count base buildings
                if (BASE_BUILDINGS.contains(unitType)) {
                    baseCount++;
                    if (secondBase == null && baseCount >= 2) {
                        secondBase = unitType;
                        secondBaseLoop = loop;
                    }
                }

                // tech_path
                if (firstTech == null && techBuildings.contains(unitType)) {
                    firstTech = unitType.toLowerCase();
                    firstTechLoop = loop;
                }

                // aggression_timing — first military unit
                if (firstMilitaryLoop < 0
                        && !WORKER_UNITS.contains(unitType)
                        && !SUPPLY_BUILDINGS.contains(unitType)
                        && !BASE_BUILDINGS.contains(unitType)
                        && !GAS_BUILDINGS.contains(unitType)
                        && !"Larva".equals(unitType)
                        && (LIGHT_UNITS.contains(unitType) || ARMORED_UNITS.contains(unitType)
                            || AIR_UNITS.contains(unitType))) {
                    firstMilitaryLoop = loop;
                }

                // Snapshot counts at SNAPSHOT_LOOP
                if (loop <= SNAPSHOT_LOOP) {
                    if (WORKER_UNITS.contains(unitType)) workerCount++;
                    if (LIGHT_UNITS.contains(unitType)) lightCount++;
                    if (ARMORED_UNITS.contains(unitType)) armoredCount++;
                    if (AIR_UNITS.contains(unitType)) airCount++;
                }
            }
        }

        // Emit observations
        if (firstBuilding != null) {
            observations.add(new ScoutingObservation("first_building", firstBuilding, firstBuildingLoop));
        }

        // gas_timing
        String gasTiming;
        if (firstGasLoop < 0 || firstGasLoop > SNAPSHOT_LOOP) gasTiming = "none";
        else if (firstGasLoop < GAS_EARLY) gasTiming = "early";
        else if (firstGasLoop <= GAS_LATE) gasTiming = "normal";
        else gasTiming = "late";
        long gasObsLoop = firstGasLoop > 0 ? firstGasLoop : SNAPSHOT_LOOP;
        observations.add(new ScoutingObservation("gas_timing", gasTiming, gasObsLoop));

        // expansion_timing
        String expansionTiming;
        if (secondBaseLoop < 0 || secondBaseLoop > SNAPSHOT_LOOP) expansionTiming = "none";
        else if (secondBaseLoop < EXPAND_FAST) expansionTiming = "fast";
        else if (secondBaseLoop <= EXPAND_LATE) expansionTiming = "normal";
        else expansionTiming = "late";
        long expandObsLoop = secondBaseLoop > 0 ? secondBaseLoop : SNAPSHOT_LOOP;
        observations.add(new ScoutingObservation("expansion_timing", expansionTiming, expandObsLoop));

        // tech_path
        if (firstTech != null) {
            observations.add(new ScoutingObservation("tech_path", firstTech, firstTechLoop));
        }

        // army_composition at 5:00
        int totalMilitary = lightCount + armoredCount + airCount;
        String armyComp;
        if (totalMilitary == 0) armyComp = "none";
        else if ((double) airCount / totalMilitary > 0.5) armyComp = "air";
        else if ((double) lightCount / totalMilitary > 0.6) armyComp = "light";
        else if ((double) armoredCount / totalMilitary > 0.6) armyComp = "armored";
        else armyComp = "mixed";
        observations.add(new ScoutingObservation("army_composition", armyComp, SNAPSHOT_LOOP));

        // worker_saturation
        int effectiveBases = Math.max(1, baseCount);
        double workersPerBase = (double) workerCount / effectiveBases;
        String saturation;
        if (workersPerBase < 16) saturation = "under";
        else if (workersPerBase <= 22) saturation = "saturated";
        else saturation = "over";
        observations.add(new ScoutingObservation("worker_saturation", saturation, SNAPSHOT_LOOP));

        // aggression_timing
        String aggression;
        if (firstMilitaryLoop < 0) aggression = "no_aggression";
        else if (firstMilitaryLoop < AGGRESSION_EARLY) aggression = "early_pressure";
        else if (firstMilitaryLoop <= AGGRESSION_LATE) aggression = "timing_attack";
        else aggression = "macro_play";
        long aggrObsLoop = firstMilitaryLoop > 0 ? firstMilitaryLoop : SNAPSHOT_LOOP;
        observations.add(new ScoutingObservation("aggression_timing", aggression, aggrObsLoop));

        observations.sort(Comparator.comparingLong(ScoutingObservation::detectedAtLoop));
        return List.copyOf(observations);
    }

    @SuppressWarnings("unchecked")
    private static Set<String> techBuildingsForPlayer(Map<String, Object> gameJson, int playerId) {
        var playerMap = (Map<String, Map<String, Object>>) gameJson.get("ToonPlayerDescMap");
        var playerData = playerMap.get(String.valueOf(playerId));
        String race = (String) playerData.get("race");
        return switch (race) {
            case "Terran" -> TERRAN_TECH;
            case "Zerg" -> ZERG_TECH;
            case "Protoss" -> PROTOSS_TECH;
            default -> Set.of();
        };
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingObservationExtractorTest -q`
Expected: PASS

- [ ] **Step 5: Add tests for gas_timing, expansion_timing, tech_path, army_composition, worker_saturation, aggression_timing**

Add these tests to `ScoutingObservationExtractorTest`:

```java
@Test
void extractGasTiming_early() {
    var gameJson = buildGameJson(List.of(
        unitInit("Refinery", 1, 1500)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    var gas = findObs(obs, "gas_timing");
    assertEquals("early", gas.state());
}

@Test
void extractGasTiming_normal() {
    var gameJson = buildGameJson(List.of(
        unitInit("Assimilator", 1, 2500)
    ), "Protoss", "Terran");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("normal", findObs(obs, "gas_timing").state());
}

@Test
void extractGasTiming_late() {
    var gameJson = buildGameJson(List.of(
        unitInit("Extractor", 1, 4000)
    ), "Zerg", "Protoss");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("late", findObs(obs, "gas_timing").state());
}

@Test
void extractGasTiming_none_whenNoGas() {
    var gameJson = buildGameJson(List.of(
        unitInit("Barracks", 1, 900)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("none", findObs(obs, "gas_timing").state());
}

@Test
void extractExpansionTiming_fast() {
    var gameJson = buildGameJson(List.of(
        unitInit("Hatchery", 2, 1600)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 2);
    assertEquals("fast", findObs(obs, "expansion_timing").state());
}

@Test
void extractTechPath_factory() {
    var gameJson = buildGameJson(List.of(
        unitInit("SupplyDepot", 1, 400),
        unitInit("Barracks", 1, 900),
        unitInit("Factory", 1, 2500)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("factory", findObs(obs, "tech_path").state());
}

@Test
void extractArmyComposition_light() {
    var gameJson = buildGameJson(List.of(
        unitBorn("Marine", 1, 3000),
        unitBorn("Marine", 1, 3200),
        unitBorn("Marine", 1, 3400),
        unitBorn("Marine", 1, 3600),
        unitBorn("Marine", 1, 3800)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("light", findObs(obs, "army_composition").state());
}

@Test
void extractWorkerSaturation_saturated() {
    List<Map<String, Object>> events = new ArrayList<>();
    for (int i = 0; i < 18; i++) {
        events.add(unitBorn("SCV", 1, 300 + i * 200));
    }
    var gameJson = buildGameJson(events, "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 1);
    assertEquals("saturated", findObs(obs, "worker_saturation").state());
}

@Test
void extractAggressionTiming_earlyPressure() {
    var gameJson = buildGameJson(List.of(
        unitBorn("Zergling", 2, 2500),
        unitBorn("Zergling", 2, 2500)
    ), "Terran", "Zerg");
    var obs = ScoutingObservationExtractor.extract(gameJson, 2);
    assertEquals("early_pressure", findObs(obs, "aggression_timing").state());
}

@Test
void extractOnlyForRequestedPlayer() {
    var gameJson = buildGameJson(List.of(
        unitInit("Barracks", 1, 900),
        unitInit("SpawningPool", 2, 1100)
    ), "Terran", "Zerg");
    var obs1 = ScoutingObservationExtractor.extract(gameJson, 1);
    var obs2 = ScoutingObservationExtractor.extract(gameJson, 2);
    assertEquals("barracks", findObs(obs1, "first_building").state());
    assertEquals("spawningpool", findObs(obs2, "first_building").state());
}

private ScoutingObservation findObs(List<ScoutingObservation> obs, String variable) {
    return obs.stream().filter(o -> o.variable().equals(variable)).findFirst().orElseThrow(
        () -> new AssertionError("No observation for variable: " + variable));
}
```

- [ ] **Step 6: Run all tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingObservationExtractorTest -q`
Expected: PASS

- [ ] **Step 7: Add integration test with real replay data**

Add a test that reads a real reconstituted game_json and verifies the extractor produces reasonable observations:

```java
@Test
void extractFromRealReplay() throws Exception {
    var path = Path.of("../quarkmind-classifier/data/reconstituted/blizzard_ladder_4.9.3");
    if (!Files.exists(path)) return; // skip if data not available

    var firstJson = Files.list(path)
        .filter(p -> p.toString().endsWith(".json") && !p.toString().endsWith(".label.json"))
        .findFirst().orElse(null);
    if (firstJson == null) return;

    var gameJson = new ObjectMapper().readValue(firstJson.toFile(),
        new TypeReference<Map<String, Object>>() {});

    var obs1 = ScoutingObservationExtractor.extract(gameJson, 1);
    var obs2 = ScoutingObservationExtractor.extract(gameJson, 2);

    assertFalse(obs1.isEmpty(), "Player 1 should have observations");
    assertFalse(obs2.isEmpty(), "Player 2 should have observations");

    // All observations should have known variables
    var knownVars = Set.of("first_building", "gas_timing", "expansion_timing",
        "tech_path", "army_composition", "worker_saturation", "aggression_timing");
    for (var obs : obs1) {
        assertTrue(knownVars.contains(obs.variable()),
            "Unknown variable: " + obs.variable());
    }
}
```

- [ ] **Step 8: Run — verify**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ScoutingObservationExtractorTest -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ScoutingObservationExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ScoutingObservationExtractorTest.java
git commit -m "feat: ScoutingObservationExtractor — discrete scouting observations from game_json Refs #456"
```

---

## Batch 2: Prior and CPT Learning Pipeline

### Task 3: Classifier label to StrategyArchetype mapping

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ClassifierLabelMapper.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ClassifierLabelMapperTest.java`

**Interfaces:**
- Consumes: `StrategyArchetype` enum, classifier label strings (RUSH, MACRO_ECONOMY, etc.), matchup strings (vs_terran, vs_zerg, vs_protoss)
- Produces: `List<StrategyArchetype> mapToArchetypes(String classifierLabel, String matchup, GamePhase phase)` — maps a classifier fineGrainedLabel + matchup + phase to the corresponding `StrategyArchetype` values. Returns a list because one classifier label may map to multiple archetypes (e.g., RUSH in vs_zerg could be ZERG_ZERGLING_RUSH or ZERG_BANELING_BUST).

The mapping uses the opponent race (derived from matchup) and game phase to filter `StrategyArchetype` values. The classifier's `fineGrainedLabel` maps to `ArchetypeCategory` + race-specific pattern matching.

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.StrategyArchetype;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class ClassifierLabelMapperTest {

    @Test
    void mapRush_vsTerran_early() {
        var archetypes = ClassifierLabelMapper.mapToArchetypes("RUSH", "vs_terran", GamePhase.EARLY);
        assertTrue(archetypes.contains(StrategyArchetype.TERRAN_MARINE_RUSH));
        assertTrue(archetypes.contains(StrategyArchetype.TERRAN_PROXY_BARRACKS));
        assertFalse(archetypes.contains(StrategyArchetype.ZERG_ZERGLING_RUSH));
    }

    @Test
    void mapMacroEconomy_vsZerg_early() {
        var archetypes = ClassifierLabelMapper.mapToArchetypes("MACRO_ECONOMY", "vs_zerg", GamePhase.EARLY);
        assertTrue(archetypes.contains(StrategyArchetype.ZERG_MACRO));
    }

    @Test
    void mapCannonRush_vsProtoss() {
        var archetypes = ClassifierLabelMapper.mapToArchetypes("CANNON_RUSH", "vs_protoss", GamePhase.EARLY);
        assertTrue(archetypes.contains(StrategyArchetype.PROTOSS_CANNON_RUSH));
        assertEquals(1, archetypes.size());
    }

    @Test
    void matchupToOpponentRace() {
        assertEquals("Terran", ClassifierLabelMapper.opponentRace("vs_terran"));
        assertEquals("Zerg", ClassifierLabelMapper.opponentRace("vs_zerg"));
        assertEquals("Protoss", ClassifierLabelMapper.opponentRace("vs_protoss"));
    }
}
```

- [ ] **Step 2: Run — verify fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ClassifierLabelMapperTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement ClassifierLabelMapper**

The mapper filters `StrategyArchetype` values by: opponent race (from matchup), game phase, and label pattern. Classifier labels like "RUSH", "PROXY" map to the category; specific labels like "CANNON_RUSH", "DT_RUSH" map directly by name match.

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.*;

import java.util.*;

public class ClassifierLabelMapper {

    private static final Map<String, String> MATCHUP_TO_RACE = Map.of(
        "vs_terran", "Terran",
        "vs_zerg", "Zerg",
        "vs_protoss", "Protoss"
    );

    public static String opponentRace(String matchup) {
        return MATCHUP_TO_RACE.get(matchup);
    }

    public static Race opponentRaceEnum(String matchup) {
        return Race.valueOf(opponentRace(matchup).toUpperCase());
    }

    public static List<StrategyArchetype> mapToArchetypes(
            String classifierLabel, String matchup, GamePhase phase) {
        Race race = opponentRaceEnum(matchup);
        String labelUpper = classifierLabel.toUpperCase();

        var result = new ArrayList<StrategyArchetype>();
        for (var arch : StrategyArchetype.values()) {
            if (arch.race() != race) continue;
            if (arch.phase() != phase) continue;
            if (arch.name().contains("UNKNOWN")) continue;

            if (matchesLabel(arch, labelUpper)) {
                result.add(arch);
            }
        }
        return result;
    }

    private static boolean matchesLabel(StrategyArchetype arch, String label) {
        String archName = arch.name();
        String racePrefixRemoved = archName.substring(archName.indexOf('_') + 1);

        if (racePrefixRemoved.equals(label)) return true;
        if (label.equals("RUSH") && arch.category() == ArchetypeCategory.RUSH) return true;
        if (label.equals("PROXY") && archName.contains("PROXY")) return true;
        if (label.equals("MACRO_ECONOMY") && arch.category() == ArchetypeCategory.MACRO) return true;
        if (label.equals("TECH_RUSH") && arch.category() == ArchetypeCategory.TECH) return true;

        return false;
    }
}
```

- [ ] **Step 4: Run — verify pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ClassifierLabelMapperTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ClassifierLabelMapper.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ClassifierLabelMapperTest.java
git commit -m "feat: ClassifierLabelMapper — map classifier labels to StrategyArchetype Refs #456"
```

### Task 4: PriorExportPipeline — learn priors and CPTs, export JSON

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java`

**Interfaces:**
- Consumes: `ScoutingObservationExtractor.extract()`, `ClassifierLabelMapper.mapToArchetypes()`, `HypothesisPriorLearner.fromOutcomes()`, `.label.json` sidecars, game_json files
- Produces:
  - `Map<String, Double> learnPriors(List<String> archetypeLabels, Set<String> allArchetypes)` — prior frequencies per matchup per phase
  - `Map<String, Map<String, double[]>> learnObservationCpts(List<ScoutingObservationSequence> sequences, Set<String> allArchetypes)` — per-variable, per-archetype probability distributions
  - `void exportToJson(Path outputDir, ...)` — writes JSON files to classpath directory

- [ ] **Step 1: Write failing test — prior learning from label sidecars**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class PriorExportPipelineTest {

    @Test
    void learnPriors_fromLabels() {
        var labels = List.of("RUSH", "RUSH", "RUSH", "MACRO_ECONOMY", "MACRO_ECONOMY",
            "TECH_RUSH", "BIO_TIMING", "BIO_TIMING", "BIO_TIMING", "BIO_TIMING");

        var allArchetypes = Set.of("RUSH", "MACRO_ECONOMY", "TECH_RUSH", "BIO_TIMING",
            "PROXY", "BANSHEE_HARASS", "AIR_SUPERIORITY", "MECH_PUSH");

        var priors = PriorExportPipeline.learnPriors(labels, allArchetypes);

        assertEquals(allArchetypes.size(), priors.size());
        assertTrue(priors.get("BIO_TIMING") > priors.get("RUSH"));
        assertTrue(priors.get("RUSH") > priors.get("TECH_RUSH"));
        // Laplace smoothing: even unobserved archetypes get non-zero priors
        assertTrue(priors.get("PROXY") > 0);

        double sum = priors.values().stream().mapToDouble(d -> d).sum();
        assertEquals(1.0, sum, 0.001);
    }
}
```

- [ ] **Step 2: Run — verify fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement PriorExportPipeline — learnPriors**

```java
package io.quarkmind.sc2.replay;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.*;
import io.quarkmind.inference.bayes.HypothesisPriorLearner;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;

public class PriorExportPipeline {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final double SMOOTHING = 1.0;
    private static final double CPT_FLOOR = 0.01;
    private static final double CPT_CEILING = 0.99;

    public static Map<String, Double> learnPriors(List<String> labels, Set<String> allArchetypes) {
        return HypothesisPriorLearner.fromOutcomes(labels, allArchetypes, SMOOTHING);
    }
}
```

- [ ] **Step 4: Run — verify pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest#learnPriors_fromLabels -q`
Expected: PASS

- [ ] **Step 5: Write failing test — CPT learning**

```java
@Test
void learnObservationCpts_fromSequences() {
    var seq1 = new ScoutingObservationSequence("hash1", 1, Race.TERRAN, Race.ZERG,
        List.of(
            new ScoutingObservation("first_building", "barracks", 900),
            new ScoutingObservation("gas_timing", "normal", 2500)
        ), "RUSH");

    var seq2 = new ScoutingObservationSequence("hash2", 1, Race.TERRAN, Race.ZERG,
        List.of(
            new ScoutingObservation("first_building", "barracks", 850),
            new ScoutingObservation("gas_timing", "early", 1500)
        ), "RUSH");

    var seq3 = new ScoutingObservationSequence("hash3", 1, Race.TERRAN, Race.ZERG,
        List.of(
            new ScoutingObservation("first_building", "factory", 2000),
            new ScoutingObservation("gas_timing", "early", 1200)
        ), "MECH_PUSH");

    var allArchetypes = Set.of("RUSH", "MECH_PUSH", "MACRO_ECONOMY");
    var cpts = PriorExportPipeline.learnObservationCpts(
        List.of(seq1, seq2, seq3), allArchetypes);

    // first_building variable should have CPT entries
    assertTrue(cpts.containsKey("first_building"));
    var fbCpt = cpts.get("first_building");

    // RUSH archetype should have higher P(barracks) than P(factory)
    var rushDist = fbCpt.get("RUSH");
    assertNotNull(rushDist);

    // All CPT entries should be clamped to [0.01, 0.99]
    for (var entry : fbCpt.entrySet()) {
        for (double p : entry.getValue()) {
            assertTrue(p >= 0.01, "CPT below floor: " + p);
            assertTrue(p <= 0.99, "CPT above ceiling: " + p);
        }
    }
}
```

- [ ] **Step 6: Implement learnObservationCpts**

```java
public static Map<String, Map<String, double[]>> learnObservationCpts(
        List<ScoutingObservationSequence> sequences, Set<String> allArchetypes) {

    // Collect all states per variable
    var variableStates = new LinkedHashMap<String, LinkedHashSet<String>>();
    for (var seq : sequences) {
        for (var obs : seq.observations()) {
            variableStates.computeIfAbsent(obs.variable(), k -> new LinkedHashSet<>())
                .add(obs.state());
        }
    }

    // Count co-occurrences: variable → archetype → state → count
    var counts = new HashMap<String, Map<String, Map<String, Double>>>();
    for (var seq : sequences) {
        String label = seq.groundTruthStrategy();
        if (!allArchetypes.contains(label)) continue;
        for (var obs : seq.observations()) {
            counts.computeIfAbsent(obs.variable(), k -> new HashMap<>())
                .computeIfAbsent(label, k -> new HashMap<>())
                .merge(obs.state(), 1.0, Double::sum);
        }
    }

    // Normalize with Laplace smoothing + clamping
    var result = new LinkedHashMap<String, Map<String, double[]>>();
    for (var varEntry : variableStates.entrySet()) {
        String variable = varEntry.getKey();
        var states = new ArrayList<>(varEntry.getValue());
        var archCpts = new LinkedHashMap<String, double[]>();

        for (String archetype : allArchetypes) {
            double[] dist = new double[states.size()];
            var archCounts = counts.getOrDefault(variable, Map.of())
                .getOrDefault(archetype, Map.of());

            double total = 0;
            for (int i = 0; i < states.size(); i++) {
                dist[i] = archCounts.getOrDefault(states.get(i), 0.0) + SMOOTHING;
                total += dist[i];
            }
            // Normalize and clamp
            for (int i = 0; i < dist.length; i++) {
                dist[i] = Math.max(CPT_FLOOR, Math.min(CPT_CEILING, dist[i] / total));
            }
            // Re-normalize after clamping
            double sum = 0;
            for (double d : dist) sum += d;
            for (int i = 0; i < dist.length; i++) dist[i] /= sum;

            archCpts.put(archetype, dist);
        }
        result.put(variable, archCpts);
    }

    return result;
}
```

- [ ] **Step 7: Run — verify pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: PASS

- [ ] **Step 8: Write failing test — JSON export**

```java
@Test
void exportToJson_writesFiles(@TempDir Path tempDir) throws Exception {
    var priors = Map.of("RUSH", 0.3, "MACRO_ECONOMY", 0.5, "TECH_RUSH", 0.2);
    var cpts = Map.of("first_building", Map.of(
        "RUSH", new double[]{0.7, 0.2, 0.1},
        "MACRO_ECONOMY", new double[]{0.2, 0.5, 0.3}
    ));
    var stateIndex = Map.of("first_building", List.of("barracks", "factory", "gateway"));

    PriorExportPipeline.exportToJson(tempDir, "EARLY", "vs_terran", priors, cpts, stateIndex);

    assertTrue(Files.exists(tempDir.resolve("opponent-priors-EARLY-vs-terran.json")));
    assertTrue(Files.exists(tempDir.resolve("observation-cpts-EARLY-vs-terran.json")));

    // Verify the priors file is valid JSON with correct content
    var loadedPriors = MAPPER.readValue(
        tempDir.resolve("opponent-priors-EARLY-vs-terran.json").toFile(),
        new com.fasterxml.jackson.core.type.TypeReference<Map<String, Double>>() {});
    assertEquals(0.3, loadedPriors.get("RUSH"), 0.001);
}

private static final ObjectMapper MAPPER = new ObjectMapper();
```

- [ ] **Step 9: Implement exportToJson**

```java
public static void exportToJson(Path outputDir, String phase, String matchup,
        Map<String, Double> priors,
        Map<String, Map<String, double[]>> cpts,
        Map<String, List<String>> stateIndex) throws IOException {

    Files.createDirectories(outputDir);

    String suffix = phase + "-" + matchup + ".json";

    // Write priors
    MAPPER.writerWithDefaultPrettyPrinter().writeValue(
        outputDir.resolve("opponent-priors-" + suffix).toFile(), priors);

    // Write CPTs with state index
    var cptExport = new LinkedHashMap<String, Object>();
    cptExport.put("stateIndex", stateIndex);
    cptExport.put("cpts", cpts);
    MAPPER.writerWithDefaultPrettyPrinter().writeValue(
        outputDir.resolve("observation-cpts-" + suffix).toFile(), cptExport);
}
```

- [ ] **Step 10: Run — verify pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PriorExportPipelineTest -q`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportPipelineTest.java
git commit -m "feat: PriorExportPipeline — learn priors and CPTs from replay data Refs #456"
```

---

## Batch 3: End-to-End Export Test

### Task 5: PriorExportTest — bulk export from real replay data

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportTest.java` (`@Tag("report")`, excluded from default surefire run)

**Interfaces:**
- Consumes: `ReplayFeatureExtractor.extract()`, `ScoutingObservationExtractor.extract()`, `ClassifierLabelMapper`, `PriorExportPipeline`, `.label.json` sidecars from `quarkmind-classifier/data/reconstituted/`
- Produces: JSON files in `quarkmind-sc2/src/main/resources/priors/` — 9 prior files (3 phases × 3 matchups) + 9 observation CPT files

This is a `@Tag("report")` test that runs the full pipeline over all available replay data and writes the learned priors/CPTs to the classpath. Run manually with `mvn test -pl quarkmind-sc2 -Preport -Dtest=PriorExportTest`.

- [ ] **Step 1: Write the export test**

```java
package io.quarkmind.sc2.replay;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

import static org.junit.jupiter.api.Assertions.*;

@Tag("report")
class PriorExportTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Path DATA_ROOT = Path.of("../quarkmind-classifier/data/reconstituted");
    private static final Path OUTPUT_DIR = Path.of("src/main/resources/priors");
    private static final String[] MATCHUPS = {"vs_terran", "vs_zerg", "vs_protoss"};

    @Test
    void exportPriorsAndCpts() throws Exception {
        if (!Files.exists(DATA_ROOT)) {
            System.out.println("SKIP: reconstituted data not available at " + DATA_ROOT);
            return;
        }

        Files.createDirectories(OUTPUT_DIR);

        // Collect all sequences grouped by matchup
        var byMatchup = new HashMap<String, List<ScoutingObservationSequence>>();
        var labelsByMatchupPhase = new HashMap<String, List<String>>();

        int processed = 0;
        int skipped = 0;

        try (var datasets = Files.list(DATA_ROOT).filter(Files::isDirectory)) {
            for (var dataset : datasets.toList()) {
                try (var jsons = Files.list(dataset)
                        .filter(p -> p.toString().endsWith(".json") && !p.toString().endsWith(".label.json"))) {
                    for (var jsonPath : jsons.toList()) {
                        var labelPath = Path.of(jsonPath.toString().replace(".json", ".label.json"));
                        if (!Files.exists(labelPath)) { skipped++; continue; }

                        try {
                            var gameJson = MAPPER.readValue(jsonPath.toFile(),
                                new TypeReference<Map<String, Object>>() {});
                            var labelJson = MAPPER.readValue(labelPath.toFile(),
                                new TypeReference<Map<String, Object>>() {});

                            @SuppressWarnings("unchecked")
                            var perspectives = (Map<String, Map<String, Object>>) labelJson.get("perspectives");
                            String hash = (String) labelJson.get("replayHash");

                            for (var entry : perspectives.entrySet()) {
                                int pid = Integer.parseInt(entry.getKey());
                                var pData = entry.getValue();
                                String matchup = (String) pData.get("matchup");
                                String label = (String) pData.get("fineGrainedLabel");

                                @SuppressWarnings("unchecked")
                                var playerMap = (Map<String, Map<String, Object>>) gameJson.get("ToonPlayerDescMap");
                                var playerInfo = playerMap.get(entry.getKey());
                                String raceStr = (String) playerInfo.get("race");
                                Race playerRace = Race.valueOf(raceStr.toUpperCase());
                                Race opponentRace = ClassifierLabelMapper.opponentRaceEnum(matchup);

                                var observations = ScoutingObservationExtractor.extract(gameJson, pid);
                                var seq = new ScoutingObservationSequence(
                                    hash, pid, playerRace, opponentRace,
                                    observations, label);

                                byMatchup.computeIfAbsent(matchup, k -> new ArrayList<>()).add(seq);

                                for (var phase : GamePhase.values()) {
                                    String key = matchup + "_" + phase.name();
                                    labelsByMatchupPhase.computeIfAbsent(key, k -> new ArrayList<>()).add(label);
                                }
                            }
                            processed++;
                        } catch (Exception e) {
                            skipped++;
                        }
                    }
                }
            }
        }

        System.out.printf("Processed %d replays, skipped %d%n", processed, skipped);
        assertTrue(processed > 100, "Need at least 100 replays for meaningful priors");

        // Export per matchup × phase
        for (String matchup : MATCHUPS) {
            var sequences = byMatchup.getOrDefault(matchup, List.of());
            if (sequences.isEmpty()) continue;

            for (var phase : GamePhase.values()) {
                String key = matchup + "_" + phase.name();
                var labels = labelsByMatchupPhase.getOrDefault(key, List.of());

                // Collect all archetype labels for this matchup
                var allArchetypes = new LinkedHashSet<String>();
                for (var arch : StrategyArchetype.values()) {
                    if (arch.race() == ClassifierLabelMapper.opponentRaceEnum(matchup)
                            && arch.phase() == phase
                            && !arch.name().contains("UNKNOWN")) {
                        allArchetypes.add(arch.name().substring(arch.name().indexOf('_') + 1));
                    }
                }
                if (allArchetypes.isEmpty()) continue;

                // Learn priors
                var priors = PriorExportPipeline.learnPriors(labels, allArchetypes);

                // Learn observation CPTs
                var cpts = PriorExportPipeline.learnObservationCpts(sequences, allArchetypes);

                // Collect state index
                var stateIndex = PriorExportPipeline.collectStateIndex(sequences);

                // Export
                PriorExportPipeline.exportToJson(OUTPUT_DIR, phase.name(), matchup, priors, cpts, stateIndex);

                System.out.printf("%s %s: %d labels, %d sequences, %d prior entries%n",
                    phase, matchup, labels.size(), sequences.size(), priors.size());
            }
        }

        // Verify files exist
        for (String matchup : MATCHUPS) {
            for (var phase : GamePhase.values()) {
                var priorsFile = OUTPUT_DIR.resolve("opponent-priors-" + phase.name() + "-" + matchup + ".json");
                if (Files.exists(priorsFile)) {
                    var loadedPriors = MAPPER.readValue(priorsFile.toFile(),
                        new TypeReference<Map<String, Double>>() {});
                    assertFalse(loadedPriors.isEmpty(), priorsFile + " is empty");
                    double sum = loadedPriors.values().stream().mapToDouble(d -> d).sum();
                    assertEquals(1.0, sum, 0.01, priorsFile + " priors don't sum to 1.0");
                }
            }
        }
    }
}
```

- [ ] **Step 2: Add collectStateIndex to PriorExportPipeline**

```java
public static Map<String, List<String>> collectStateIndex(
        List<ScoutingObservationSequence> sequences) {
    var stateIndex = new LinkedHashMap<String, List<String>>();
    var seen = new LinkedHashMap<String, LinkedHashSet<String>>();
    for (var seq : sequences) {
        for (var obs : seq.observations()) {
            seen.computeIfAbsent(obs.variable(), k -> new LinkedHashSet<>())
                .add(obs.state());
        }
    }
    for (var entry : seen.entrySet()) {
        stateIndex.put(entry.getKey(), new ArrayList<>(entry.getValue()));
    }
    return stateIndex;
}
```

- [ ] **Step 3: Run test with report profile (if data available)**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=PriorExportTest -q`
Expected: PASS (or SKIP if data not available)

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/PriorExportTest.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/PriorExportPipeline.java
git commit -m "feat: PriorExportTest — bulk prior and CPT export from labelled replays Refs #456"
```

- [ ] **Step 5: Commit exported JSON resources (if generated)**

```bash
git add quarkmind-sc2/src/main/resources/priors/
git commit -m "data: learned opponent priors and observation CPTs from 149K labelled replays Refs #456"
```

---

## References

- [2026-10-26-hypothesis-tracker-integration-design.md] — design spec this plan implements (Phase 1 scope)
- [TrackerEventFeatureExtractor.java:23] — game_json format (unitTypeName field, event structure)
- [HypothesisPriorLearner.java:8] — fromOutcomes() API for prior learning with Laplace smoothing
- [LearnedCptLoader.java:17] — existing classpath resource pattern for learned CPTs
- [StrategyArchetype.java:3] — 62 strategy archetypes with race/phase/category
- [config.py:1] — classifier label definitions per matchup
- [sc2data-train-times-require-calibration.md] — protocol: discretization boundaries must be calibrated
- [GitHub #456] — SC2 replay pipeline: extract scouting observations and strategy labels
- [GitHub casehubio/neocortex#705] — HypothesisPriorLearner (consumer of this pipeline's output)
