# CreepTumor and Special Building UnitInit Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #325 — Add CreepTumor and special building types to UnitInit coverage
**Issue group:** #318, #325

**Goal:** Increase StrippedReplayFeatureExtractor building coverage by adding
ability-placed structures (CreepTumor, NydusCanal, OracleStasisTrap, AssimilatorRich),
auto-spread tumor synthesis, and fixing existing building under-counting.

**Architecture:** Ability-placed structures route through existing
`BuildCommand → handleBuild()` via new AbilityMapping dispatch cases. Auto-spread
tumors use a single-spread chain synthesis step (each tumor spreads exactly once after
maturation). Building under-counting diagnosed via coverage diagnostic, then fixed.

**Tech Stack:** Java 21, Quarkus, s2prot replay parsing, plain JUnit tests

## Global Constraints

- Protocol `enum-switch-exhaustive-required` — no `default ->` on BuildingType switches
- Protocol `extractor-separate-from-simulated-game` — extraction logic stays in extractor classes
- Oracle Python names must exactly match oracle replay data format
- All tests are plain JUnit (no `@QuarkusTest`) unless CDI is required
- Build times marked "estimate" are calibrated from oracle data when diagnostic runs

---

## Batch 1: Enum Foundation + Protocol Compliance

### Task 1: Add BuildingType enum values and update all BuildingType switches

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/BuildingType.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:254-309,500-507,700-755,759-814,1021-1026,1048-1064,1069-1099,1112-1118,1120-1127`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:630,790-801,893-943`
- Test: existing `SC2DataTest`, `StrippedReplayFeatureExtractorTest`

**Interfaces:**
- Produces: `BuildingType.CREEP_TUMOR`, `BuildingType.CREEP_TUMOR_QUEEN`, `BuildingType.ORACLE_STASIS_TRAP`, `BuildingType.ASSIMILATOR_RICH` — used by Tasks 2-5
- Produces: `buildingTypeToPythonName(CREEP_TUMOR) → "CreepTumor"` etc — used by Task 4

- [ ] **Step 1: Write failing test — new BuildingType values have correct Python names**

```java
// In StrippedReplayFeatureExtractorTest or a new section of it
@Test
void newBuildingTypesHavePythonNames() {
    assertNotNull(StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.CREEP_TUMOR));
    assertEquals("CreepTumor",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.CREEP_TUMOR));
    assertEquals("CreepTumorQueen",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.CREEP_TUMOR_QUEEN));
    assertEquals("NydusCanal",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.NYDUS_CANAL));
    assertEquals("OracleStasisTrap",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.ORACLE_STASIS_TRAP));
    assertEquals("AssimilatorRich",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.ASSIMILATOR_RICH));
    assertEquals("LurkerDenMP",
        StrippedReplayFeatureExtractor.buildingTypeToPythonName(BuildingType.LURKER_DEN));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorTest#newBuildingTypesHavePythonNames -q`
Expected: FAIL — `CREEP_TUMOR` does not exist in BuildingType

- [ ] **Step 3: Add enum values to BuildingType**

In `BuildingType.java`, add after `EXTRACTOR`:

```java
// Ability-placed structures
CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP, ASSIMILATOR_RICH,
UNKNOWN
```

Remove the existing `UNKNOWN` at the end (it stays, just moves after the new values).

- [ ] **Step 4: Attempt compilation — expect failures in default-armed switches**

Run: `mvn compile -pl quarkmind-sc2 -q`

If compilation succeeds (because defaults swallow new values), proceed to Step 5.
The defaults are functionally wrong but syntactically valid — we remove them now.

- [ ] **Step 5: Update buildingTypeToPythonName — add cases, remove default**

Replace the `default -> null` at the end of `buildingTypeToPythonName()` with explicit cases:

```java
case NYDUS_CANAL -> "NydusCanal";
case LURKER_DEN -> "LurkerDenMP";
case CREEP_TUMOR -> "CreepTumor";
case CREEP_TUMOR_QUEEN -> "CreepTumorQueen";
case ORACLE_STASIS_TRAP -> "OracleStasisTrap";
case ASSIMILATOR_RICH -> "AssimilatorRich";
case UNKNOWN -> null;
```

- [ ] **Step 6: Update SC2Data.buildTimeInLoops — add cases, remove default**

Replace `default -> 880` with:

```java
case CREEP_TUMOR, CREEP_TUMOR_QUEEN -> 224;  // estimate: ~10s
case ORACLE_STASIS_TRAP -> 67;               // estimate: ~3s
case ASSIMILATOR_RICH -> 480;                // same as ASSIMILATOR
case UNKNOWN -> 880;
```

- [ ] **Step 7: Update SC2Data.mineralCost(BuildingType) — add cases, remove default**

Replace `default -> 100` with:

```java
case CREEP_TUMOR, CREEP_TUMOR_QUEEN -> 0;   // energy-only (25 Queen energy)
case ORACLE_STASIS_TRAP -> 0;                // energy-only (50 Oracle energy)
case ASSIMILATOR_RICH -> 75;                 // same as ASSIMILATOR
case UNKNOWN -> 100;
```

- [ ] **Step 8: Update SC2Data.maxBuildingHealth — add cases, remove default**

Replace `default -> 500` with:

```java
case CREEP_TUMOR, CREEP_TUMOR_QUEEN -> 50;
case ORACLE_STASIS_TRAP -> 30;
case ASSIMILATOR_RICH -> 450;                // same as ASSIMILATOR
case UNKNOWN -> 500;
```

- [ ] **Step 9: Update SC2Data.supplyBonus — add cases, remove default**

Replace `default -> 0` with:

```java
// List ALL remaining BuildingType values not in the cases above.
// Let the compiler guide you — any missing value triggers an error.
case NEXUS, COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS,
     GATEWAY, CYBERNETICS_CORE, ASSIMILATOR, ASSIMILATOR_RICH,
     ROBOTICS_FACILITY, STARGATE, FORGE, TWILIGHT_COUNCIL,
     PHOTON_CANNON, SHIELD_BATTERY, DARK_SHRINE, TEMPLAR_ARCHIVES,
     FLEET_BEACON, ROBOTICS_BAY,
     BARRACKS, ENGINEERING_BAY, ARMORY, MISSILE_TURRET, BUNKER,
     SENSOR_TOWER, GHOST_ACADEMY, FACTORY, STARPORT, FUSION_CORE, REFINERY,
     SPAWNING_POOL, EVOLUTION_CHAMBER, ROACH_WARREN, BANELING_NEST,
     SPINE_CRAWLER, SPORE_CRAWLER, HYDRALISK_DEN, LURKER_DEN,
     INFESTATION_PIT, SPIRE, GREATER_SPIRE, NYDUS_NETWORK, NYDUS_CANAL,
     ULTRALISK_CAVERN, EXTRACTOR,
     CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP, UNKNOWN -> 0;
```

NOTE: Keep the existing `PYLON -> 8`, `SUPPLY_DEPOT -> 8`, `HATCHERY, LAIR, HIVE -> 6`
cases. The new explicit case lists ALL remaining values that return 0. Remove any
duplicates that already appear in the named cases above (ORBITAL_COMMAND,
PLANETARY_FORTRESS appear in both — the compiler will flag this).

- [ ] **Step 10: Update SC2Data.sightRange(BuildingType) — add cases, remove default**

Replace `default -> 9` with explicit cases:

```java
case ASSIMILATOR, ASSIMILATOR_RICH, REFINERY, EXTRACTOR -> 6;
case CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP -> 0;
case UNKNOWN -> 9;
// All remaining building types explicitly → 9
case NEXUS, PYLON, GATEWAY, CYBERNETICS_CORE, ROBOTICS_FACILITY,
     STARGATE, FORGE, TWILIGHT_COUNCIL, PHOTON_CANNON, SHIELD_BATTERY,
     DARK_SHRINE, TEMPLAR_ARCHIVES, FLEET_BEACON, ROBOTICS_BAY,
     COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS,
     SUPPLY_DEPOT, BARRACKS, ENGINEERING_BAY, ARMORY, MISSILE_TURRET,
     BUNKER, SENSOR_TOWER, GHOST_ACADEMY, FACTORY, STARPORT, FUSION_CORE,
     HATCHERY, LAIR, HIVE, SPAWNING_POOL, EVOLUTION_CHAMBER,
     ROACH_WARREN, BANELING_NEST, SPINE_CRAWLER, SPORE_CRAWLER,
     HYDRALISK_DEN, LURKER_DEN, INFESTATION_PIT, SPIRE, GREATER_SPIRE,
     NYDUS_NETWORK, NYDUS_CANAL, ULTRALISK_CAVERN -> 9;
```

- [ ] **Step 11: Update SC2Data.buildingRadius — add cases, remove default**

Replace `default -> 1.5f` with:

```java
case CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP -> 0.5f;
case ASSIMILATOR_RICH -> 1.0f;                // same as ASSIMILATOR
case UNKNOWN -> 1.5f;
// Enumerate all remaining medium buildings that were in the default
case BARRACKS, FACTORY, STARPORT, GATEWAY, CYBERNETICS_CORE,
     FORGE, TWILIGHT_COUNCIL, DARK_SHRINE, TEMPLAR_ARCHIVES,
     FLEET_BEACON, ROBOTICS_BAY, ROBOTICS_FACILITY, STARGATE,
     ENGINEERING_BAY, ARMORY, GHOST_ACADEMY, FUSION_CORE, BUNKER,
     SPAWNING_POOL, ROACH_WARREN, BANELING_NEST,
     HYDRALISK_DEN, LURKER_DEN, INFESTATION_PIT, SPIRE,
     GREATER_SPIRE, NYDUS_NETWORK, NEXUS -> 1.5f;
```

Wait — NEXUS is 2.5f (large). Only list buildings that were genuinely caught by default.
Check existing explicit cases and list only the remaining ones. The compiler will catch
any missing values.

- [ ] **Step 12: Update SC2Data.techTier — add cases, remove default**

Replace `default -> OptionalInt.empty()` with:

```java
case NEXUS, PYLON, ASSIMILATOR, ASSIMILATOR_RICH,
     COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS,
     SUPPLY_DEPOT, MISSILE_TURRET, BUNKER, SENSOR_TOWER, REFINERY,
     HATCHERY, LAIR, HIVE, SPINE_CRAWLER, SPORE_CRAWLER,
     NYDUS_CANAL, EXTRACTOR,
     PHOTON_CANNON, SHIELD_BATTERY,
     CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP,
     UNKNOWN -> OptionalInt.empty();
```

- [ ] **Step 13: Update SC2Data.isBase — add cases, remove default**

Replace `default -> false` with:

```java
case PYLON, GATEWAY, CYBERNETICS_CORE, ASSIMILATOR, ASSIMILATOR_RICH,
     ROBOTICS_FACILITY, STARGATE, FORGE, TWILIGHT_COUNCIL,
     PHOTON_CANNON, SHIELD_BATTERY, DARK_SHRINE, TEMPLAR_ARCHIVES,
     FLEET_BEACON, ROBOTICS_BAY,
     SUPPLY_DEPOT, BARRACKS, ENGINEERING_BAY, ARMORY, MISSILE_TURRET,
     BUNKER, SENSOR_TOWER, GHOST_ACADEMY, FACTORY, STARPORT,
     FUSION_CORE, REFINERY,
     SPAWNING_POOL, EVOLUTION_CHAMBER, ROACH_WARREN, BANELING_NEST,
     SPINE_CRAWLER, SPORE_CRAWLER, HYDRALISK_DEN, LURKER_DEN,
     INFESTATION_PIT, SPIRE, GREATER_SPIRE, NYDUS_NETWORK, NYDUS_CANAL,
     ULTRALISK_CAVERN, EXTRACTOR,
     CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP, UNKNOWN -> false;
```

- [ ] **Step 14: Update SC2Data.isProductionBuilding — add cases, remove default**

Replace `default -> false` with explicit `-> false` for all remaining values.

- [ ] **Step 15: Update gasCostForBuilding in extractor — add cases, remove default**

Replace `default -> 0` with:

```java
case NEXUS, PYLON, GATEWAY, ASSIMILATOR, ASSIMILATOR_RICH,
     ROBOTICS_FACILITY, FORGE, PHOTON_CANNON, SHIELD_BATTERY,
     ROBOTICS_BAY, FLEET_BEACON,
     COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS,
     SUPPLY_DEPOT, BARRACKS, ENGINEERING_BAY, MISSILE_TURRET,
     BUNKER, SENSOR_TOWER, FACTORY, REFINERY, ARMORY,
     HATCHERY, SPAWNING_POOL, EVOLUTION_CHAMBER, ROACH_WARREN,
     BANELING_NEST, SPINE_CRAWLER, SPORE_CRAWLER, HYDRALISK_DEN,
     NYDUS_NETWORK, NYDUS_CANAL, EXTRACTOR,
     CREEP_TUMOR, CREEP_TUMOR_QUEEN, ORACLE_STASIS_TRAP, UNKNOWN -> 0;
```

- [ ] **Step 16: Update GAS_BUILDINGS — add AssimilatorRich**

Change line 630:

```java
private static final Set<String> GAS_BUILDINGS = Set.of(
    "Assimilator", "AssimilatorRich", "Refinery", "Extractor");
```

- [ ] **Step 17: Run test to verify Python name test passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorTest#newBuildingTypesHavePythonNames -q`
Expected: PASS

- [ ] **Step 18: Verify full compilation and existing tests pass**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS (all switches exhaustive)

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All existing tests pass

- [ ] **Step 19: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/BuildingType.java \
       quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java
git commit -m "feat: add CreepTumor, OracleStasisTrap, AssimilatorRich BuildingType values

Adds 4 new BuildingType enum values. Updates all 11 BuildingType switch
expressions to explicit exhaustive form per protocol PP-20260913-4d73d1.
Adds buildingTypeToPythonName cases for NYDUS_CANAL, LURKER_DEN, and the
4 new types. AssimilatorRich added to GAS_BUILDINGS.

Refs #325"
```

---

## Batch 2: Diagnostic Discovery + AbilityMapping Dispatch

### Task 2: Write abilLink discovery diagnostic for ability-placed structures

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CreepTumorAbilityDiscoveryTest.java`

**Interfaces:**
- Consumes: oracle replays in `quarkmind-sc2/src/test/resources/4.9.3_oracle/restored/`
- Produces: discovered abilLink values (console output) — used by Task 3

- [ ] **Step 1: Write diagnostic test**

```java
package io.quarkmind.sc2.replay;

import mpq.RepContent;
import mpq.Replay;
import mpq.event.Event;
import mpq.event.gameevent.CmdEvent;
import mpq.event.tracker.UnitInitEvent;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.*;
import java.util.*;

@Tag("diagnostic")
class CreepTumorAbilityDiscoveryTest {

    private static final Set<String> TARGET_TYPES = Set.of(
        "CreepTumor", "CreepTumorQueen", "CreepTumorBurrowed",
        "NydusCanal", "NydusWorm",
        "OracleStasisTrap", "StasisTrap"
    );

    @Test
    void discoverAbilLinksForAbilityPlacedStructures() throws Exception {
        Path oracleDir = Path.of("src/test/resources/4.9.3_oracle/restored");
        if (!Files.isDirectory(oracleDir)) {
            System.out.println("Oracle directory not found — skipping");
            return;
        }

        Map<String, Map<Integer, Integer>> typeToAbilLinkCounts = new TreeMap<>();

        try (var replays = Files.list(oracleDir)) {
            for (Path replayPath : replays.filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                Replay replay = mpq.RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(RepContent.GAME_EVENTS, RepContent.TRACKER_EVENTS));
                if (replay == null) continue;

                // Collect oracle UnitInit events for target types
                List<UnitInitEvent> targetInits = new ArrayList<>();
                for (var evt : replay.trackerEvents.getEvents()) {
                    if (evt instanceof UnitInitEvent ui) {
                        String typeName = ui.getUnitTypeName();
                        if (typeName != null && TARGET_TYPES.contains(typeName)) {
                            targetInits.add(ui);
                        }
                    }
                }

                if (targetInits.isEmpty()) continue;

                // Collect all CmdEvents
                List<CmdEvent> cmds = new ArrayList<>();
                for (var evt : replay.gameEvents.getEvents()) {
                    if (evt instanceof CmdEvent cmd && cmd.getAbilLink() != null) {
                        cmds.add(cmd);
                    }
                }

                // Correlate: for each target UnitInit, find CmdEvents within ±50 loops
                for (UnitInitEvent ui : targetInits) {
                    long initLoop = ui.getLoop();
                    int playerId = ui.getControlPlayerId();
                    String typeName = ui.getUnitTypeName();

                    for (CmdEvent cmd : cmds) {
                        if (cmd.getUserId() == playerId - 1
                            && Math.abs(cmd.getLoop() - initLoop) <= 50) {
                            typeToAbilLinkCounts
                                .computeIfAbsent(typeName, k -> new TreeMap<>())
                                .merge(cmd.getAbilLink(), 1, Integer::sum);
                        }
                    }
                }
            }
        }

        System.out.println("\n=== Ability-Placed Structure abilLink Discovery ===\n");
        for (var entry : typeToAbilLinkCounts.entrySet()) {
            System.out.printf("%s:%n", entry.getKey());
            entry.getValue().entrySet().stream()
                .sorted(Map.Entry.<Integer, Integer>comparingByValue().reversed())
                .forEach(e -> System.out.printf("  abilLink=%d  count=%d%n",
                    e.getKey(), e.getValue()));
        }
    }
}
```

- [ ] **Step 2: Run diagnostic**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=CreepTumorAbilityDiscoveryTest -q`
Expected: Console output showing abilLink correlations per building type.

Record the top-correlated abilLink for each type — these are the values for Task 3.

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CreepTumorAbilityDiscoveryTest.java
git commit -m "feat: add CreepTumor abilLink discovery diagnostic

Correlates CmdEvent abilLinks with oracle UnitInit events for CreepTumor,
NydusCanal, and OracleStasisTrap across oracle replays.

Refs #325"
```

### Task 3: Wire AbilityMapping dispatch for discovered abilLinks

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java:81-87,420-460`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java`

**Interfaces:**
- Consumes: abilLink values discovered in Task 2
- Consumes: `buildCommand()` helper method (AbilityMapping.java:462)
- Produces: `BuildCommand` for CreepTumorQueen, NydusCanal, OracleStasisTrap — consumed by handleBuild in extract()

- [ ] **Step 1: Write failing test — AbilityMapping dispatches CreepTumor abilLink**

```java
// In AbilityMappingTest
@Test
void queenCreepTumorEmitsBuildCommand() {
    // Use the abilLink discovered in Task 2
    // Example assumes abilLink = 216 (replace with actual discovered value)
    int ABIL_QUEEN_CREEP_TUMOR = ???; // from diagnostic output
    var mapping = new AbilityMapping(1, true, Race.ZERG);
    CmdEvent cmd = createCmdEvent(0, ABIL_QUEEN_CREEP_TUMOR, 0,
        createTargetPoint(30.0f, 40.0f));
    List<ReplayCommand> result = mapping.process(cmd);
    assertEquals(1, result.size());
    assertInstanceOf(ReplayCommand.BuildCommand.class, result.get(0));
    var bc = (ReplayCommand.BuildCommand) result.get(0);
    assertEquals("CreepTumorQueen", bc.buildingName());
}
```

Write similar tests for NydusCanal and OracleStasisTrap (if abilLinks were discovered).

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest#queenCreepTumorEmitsBuildCommand -q`
Expected: FAIL — abilLink falls through to `default -> null`

- [ ] **Step 3: Add abilLink constants and dispatch cases in AbilityMapping**

After the existing morph constants (line 87), add:

```java
private static final int ABIL_QUEEN_CREEP_TUMOR = ???; // from diagnostic
private static final int ABIL_NYDUS_SPAWN = ???;       // from diagnostic
private static final int ABIL_ORACLE_STASIS_WARD = ???; // from diagnostic
```

In `dispatchHuman()`, before `default -> null` (line 458), add:

```java
case ABIL_QUEEN_CREEP_TUMOR -> isRace(Race.ZERG)
    ? buildCommand(loop, "CreepTumorQueen", event) : null;
case ABIL_NYDUS_SPAWN -> isRace(Race.ZERG)
    ? buildCommand(loop, "NydusCanal", event) : null;
case ABIL_ORACLE_STASIS_WARD -> isRace(Race.PROTOSS)
    ? buildCommand(loop, "OracleStasisTrap", event) : null;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest -q`
Expected: All tests pass including new dispatch tests

- [ ] **Step 5: Verify no regression**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java
git commit -m "feat: dispatch CreepTumor, NydusCanal, OracleStasisTrap via AbilityMapping

Adds abilLink constants discovered from oracle replay diagnostic.
Ability-placed structures route through buildCommand() helper to emit
BuildCommand, flowing through existing handleBuild() pipeline.

Refs #325"
```

---

## Batch 3: Auto-Spread Synthesis + Building Diagnostics

### Task 4: Implement CreepTumor single-spread chain synthesis

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:240-249`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CreepTumorSynthesisTest.java`

**Interfaces:**
- Consumes: `SyntheticEvent` record, `EventOrdinal` enum (StrippedReplayFeatureExtractor inner types)
- Produces: `synthesizeCreepTumorSpread(int, List<SyntheticEvent>, int, int) → int` — called from extract()

- [ ] **Step 1: Write failing test — single Queen tumor produces one auto-spread child**

```java
package io.quarkmind.sc2.replay;

import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class CreepTumorSynthesisTest {

    @Test
    void singleQueenTumorProducesOneAutoSpreadChild() {
        var extractor = new StrippedReplayFeatureExtractor();
        // Simulate: Queen places tumor at loop 1000, build time 224
        // UnitDone at 1224 (maturation)
        // Auto-spread at 1224 + SPREAD_DELAY
        // Child matures and spreads once more
        List<Map<String, Object>> events = extractor.synthesizeCreepTumorSpreadForTest(
            1,   // playerId
            List.of(seedTumorDone(1, "CreepTumorQueen", 1224)),
            100, // startTag
            5000 // gameLength
        );

        // Should produce at least one auto-spread CreepTumor UnitInit + UnitDone
        long creepTumorInits = events.stream()
            .filter(e -> "UnitInit".equals(e.get("evtTypeName"))
                      && "CreepTumor".equals(e.get("unitTypeName")))
            .count();
        assertTrue(creepTumorInits >= 1,
            "Expected at least 1 auto-spread CreepTumor, got " + creepTumorInits);

        // Each init should have a matching done
        long creepTumorDones = events.stream()
            .filter(e -> "UnitDone".equals(e.get("evtTypeName"))
                      && "CreepTumor".equals(e.get("unitTypeName")))
            .count();
        assertEquals(creepTumorInits, creepTumorDones);
    }

    @Test
    void eachTumorSpreadsExactlyOnce() {
        var extractor = new StrippedReplayFeatureExtractor();
        // Long game with one seed — chain should be linear, not exponential
        List<Map<String, Object>> events = extractor.synthesizeCreepTumorSpreadForTest(
            1,
            List.of(seedTumorDone(1, "CreepTumorQueen", 500)),
            100,
            20000 // long game
        );

        long initCount = events.stream()
            .filter(e -> "UnitInit".equals(e.get("evtTypeName"))
                      && "CreepTumor".equals(e.get("unitTypeName")))
            .count();

        // Linear chain from one seed: each tumor spreads once.
        // With ~336 loop spread delay + 224 build time = ~560 loops per generation.
        // 20000 game length / 560 ≈ 35 generations max, but capped.
        // Key assertion: count should be reasonable (not exponential thousands)
        assertTrue(initCount > 0, "Should produce auto-spread tumors");
        assertTrue(initCount <= 50,
            "Linear spread should not produce exponential growth, got " + initCount);
    }

    @Test
    void spreadStopsAtGameLength() {
        var extractor = new StrippedReplayFeatureExtractor();
        int gameLength = 2000;
        List<Map<String, Object>> events = extractor.synthesizeCreepTumorSpreadForTest(
            1,
            List.of(seedTumorDone(1, "CreepTumorQueen", 500)),
            100,
            gameLength
        );

        boolean allWithinBounds = events.stream()
            .allMatch(e -> ((Number) e.get("loop")).longValue() <= gameLength);
        assertTrue(allWithinBounds, "No events should exceed game length");
    }

    private Map<String, Object> seedTumorDone(int playerId, String typeName, long loop) {
        return Map.of(
            "evtTypeName", "UnitDone",
            "loop", loop,
            "controlPlayerId", playerId,
            "unitTypeName", typeName,
            "unitTagIndex", 99,
            "unitTagRecycle", 0
        );
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CreepTumorSynthesisTest -q`
Expected: FAIL — `synthesizeCreepTumorSpreadForTest` does not exist

- [ ] **Step 3: Implement synthesis method and test bridge**

Add to `StrippedReplayFeatureExtractor`:

```java
private static final int CREEP_TUMOR_BUILD_TIME = 224;
private static final int CREEP_SPREAD_DELAY = 336;  // ~15s at Faster
private static final int MAX_ACTIVE_TUMORS = 20;

private int synthesizeCreepTumorSpread(int playerId, List<SyntheticEvent> events,
                                       int tagCounter, int gameLength) {
    // Collect seed tumors (UnitDone for CreepTumor/CreepTumorQueen)
    List<Long> seedMaturationLoops = events.stream()
        .filter(e -> e.playerId() == playerId
                  && e.ordinal() == EventOrdinal.UNIT_DONE
                  && isCreepTumor((String) e.data().get("unitTypeName")))
        .map(SyntheticEvent::loop)
        .sorted()
        .toList();

    if (seedMaturationLoops.isEmpty()) return tagCounter;

    // Priority queue: (spreadLoop, hasSpread=false)
    PriorityQueue<Long> pendingSpreads = new PriorityQueue<>();
    int activeTumors = seedMaturationLoops.size();

    for (long matLoop : seedMaturationLoops) {
        long spreadLoop = matLoop + CREEP_SPREAD_DELAY;
        if (spreadLoop <= gameLength) {
            pendingSpreads.add(spreadLoop);
        }
    }

    while (!pendingSpreads.isEmpty()) {
        long spreadLoop = pendingSpreads.poll();
        if (spreadLoop > gameLength) break;
        if (activeTumors >= MAX_ACTIVE_TUMORS) continue;

        int tag = tagCounter++;
        events.add(new SyntheticEvent(spreadLoop, EventOrdinal.UNIT_INIT, playerId,
            Map.of("evtTypeName", "UnitInit",
                   "loop", spreadLoop,
                   "controlPlayerId", playerId,
                   "unitTypeName", "CreepTumor",
                   "unitTagIndex", tag,
                   "unitTagRecycle", 0)));

        long doneLoop = spreadLoop + CREEP_TUMOR_BUILD_TIME;
        if (doneLoop <= gameLength) {
            events.add(new SyntheticEvent(doneLoop, EventOrdinal.UNIT_DONE, playerId,
                Map.of("evtTypeName", "UnitDone",
                       "loop", doneLoop,
                       "controlPlayerId", playerId,
                       "unitTypeName", "CreepTumor",
                       "unitTagIndex", tag,
                       "unitTagRecycle", 0)));

            activeTumors++;
            // Child schedules its own single spread
            long childSpreadLoop = doneLoop + CREEP_SPREAD_DELAY;
            if (childSpreadLoop <= gameLength) {
                pendingSpreads.add(childSpreadLoop);
            }
        }
    }

    return tagCounter;
}

private static boolean isCreepTumor(String typeName) {
    return "CreepTumor".equals(typeName) || "CreepTumorQueen".equals(typeName);
}
```

Add test bridge method:

```java
List<Map<String, Object>> synthesizeCreepTumorSpreadForTest(
        int playerId, List<Map<String, Object>> seedDoneEvents,
        int startTag, int gameLength) {
    List<SyntheticEvent> events = new ArrayList<>();
    for (Map<String, Object> seed : seedDoneEvents) {
        events.add(new SyntheticEvent(
            ((Number) seed.get("loop")).longValue(),
            EventOrdinal.UNIT_DONE,
            (int) seed.get("controlPlayerId"),
            seed));
    }
    synthesizeCreepTumorSpread(playerId, events, startTag, gameLength);
    return events.stream()
        .filter(e -> e.ordinal() != EventOrdinal.UNIT_DONE
                  || !seedDoneEvents.contains(e.data()))
        .map(SyntheticEvent::data)
        .toList();
}
```

- [ ] **Step 4: Wire synthesis into extract() loop**

Inside the per-player loop, modify the `elapsedLoops` guard block:

```java
if (elapsedLoops != null && elapsedLoops > 0) {
    generatePlayerStats(playerId, state, syntheticEvents, elapsedLoops);
    tagCounter = synthesizeCreepTumorSpread(playerId, syntheticEvents,
                                             tagCounter, elapsedLoops);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CreepTumorSynthesisTest -q`
Expected: All 3 tests PASS

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass (no regressions)

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CreepTumorSynthesisTest.java
git commit -m "feat: implement single-spread CreepTumor chain synthesis

Each tumor spreads exactly once after maturation (matching SC2 mechanics),
producing linear chain growth. Per-player cap of 20 active tumors.
Synthesis runs inside elapsedLoops guard after generatePlayerStats.

Refs #325"
```

### Task 5: Write building coverage diagnostic and fix under-counting

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/BuildingCoverageDiagnosticTest.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` (if gaps found)

**Interfaces:**
- Consumes: oracle replays, StrippedReplayFeatureExtractor
- Produces: diagnostic output + targeted fixes for under-counting

- [ ] **Step 1: Write diagnostic test**

```java
package io.quarkmind.sc2.replay;

import mpq.RepContent;
import mpq.Replay;
import mpq.event.tracker.UnitInitEvent;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.*;
import java.util.*;

@Tag("diagnostic")
class BuildingCoverageDiagnosticTest {

    private static final Set<String> BUILDING_TYPES = Set.of(
        "Nexus", "Pylon", "Gateway", "CyberneticsCore", "Assimilator",
        "RoboticsFacility", "Stargate", "Forge", "TwilightCouncil",
        "PhotonCannon", "ShieldBattery", "DarkShrine", "TemplarArchive",
        "FleetBeacon", "RoboticsBay",
        "CommandCenter", "OrbitalCommand", "PlanetaryFortress",
        "SupplyDepot", "Barracks", "EngineeringBay", "Armory",
        "MissileTurret", "Bunker", "SensorTower", "GhostAcademy",
        "Factory", "Starport", "FusionCore", "Refinery",
        "Hatchery", "SpawningPool", "EvolutionChamber", "RoachWarren",
        "BanelingNest", "SpineCrawler", "SporeCrawler", "HydraliskDen",
        "LurkerDenMP", "InfestationPit", "Spire", "NydusNetwork",
        "UltraliskCavern", "Extractor",
        "CreepTumor", "CreepTumorQueen", "NydusCanal",
        "OracleStasisTrap", "AssimilatorRich"
    );

    @Test
    void compareBuildingCoverageAgainstOracle() throws Exception {
        Path oracleDir = Path.of("src/test/resources/4.9.3_oracle/restored");
        Path strippedDir = Path.of("src/test/resources/4.9.3_oracle/stripped");
        if (!Files.isDirectory(oracleDir) || !Files.isDirectory(strippedDir)) {
            System.out.println("Oracle/stripped directories not found — skipping");
            return;
        }

        Map<String, Integer> oracleCounts = new TreeMap<>();
        Map<String, Integer> javaCounts = new TreeMap<>();
        var extractor = new StrippedReplayFeatureExtractor();

        try (var replays = Files.list(oracleDir)) {
            for (Path oraclePath : replays
                    .filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                // Oracle counts
                Replay oracle = mpq.RepParserEngine.parseReplay(oraclePath,
                    EnumSet.of(RepContent.TRACKER_EVENTS));
                if (oracle == null) continue;
                for (var evt : oracle.trackerEvents.getEvents()) {
                    if (evt instanceof UnitInitEvent ui) {
                        String name = ui.getUnitTypeName();
                        if (name != null && BUILDING_TYPES.contains(name)) {
                            oracleCounts.merge(name, 1, Integer::sum);
                        }
                    }
                }

                // Java counts
                Path strippedPath = strippedDir.resolve(oraclePath.getFileName());
                if (!Files.exists(strippedPath)) continue;
                try {
                    Map<String, Object> result = extractor.extract(strippedPath);
                    @SuppressWarnings("unchecked")
                    List<Map<String, Object>> events =
                        (List<Map<String, Object>>) result.get("events");
                    if (events == null) continue;
                    for (Map<String, Object> event : events) {
                        if ("UnitInit".equals(event.get("evtTypeName"))) {
                            String name = (String) event.get("unitTypeName");
                            if (name != null && BUILDING_TYPES.contains(name)) {
                                javaCounts.merge(name, 1, Integer::sum);
                            }
                        }
                    }
                } catch (Exception e) {
                    System.err.println("Failed: " + strippedPath + " — " + e.getMessage());
                }
            }
        }

        System.out.println("\n=== Building UnitInit Coverage: Oracle vs Java ===\n");
        System.out.printf("%-25s %8s %8s %8s%n", "BuildingType", "Oracle", "Java", "Coverage");
        System.out.println("-".repeat(55));

        Set<String> allTypes = new TreeSet<>(oracleCounts.keySet());
        allTypes.addAll(javaCounts.keySet());
        for (String type : allTypes) {
            int oracle = oracleCounts.getOrDefault(type, 0);
            int java = javaCounts.getOrDefault(type, 0);
            String coverage = oracle > 0
                ? String.format("%.0f%%", 100.0 * java / oracle)
                : "N/A";
            System.out.printf("%-25s %8d %8d %8s%n", type, oracle, java, coverage);
        }
    }
}
```

- [ ] **Step 2: Run diagnostic**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=BuildingCoverageDiagnosticTest -q`
Expected: Console output showing per-type building coverage.

Analyze output: identify which building types are still below 90% and what the
gap root cause is. Common fixes:
- Missing abilCmdIndex values in SCV_BUILD_BUILDINGS, PROBE_BUILD_BUILDINGS, or
  DRONE_BUILD_BUILDINGS maps
- CmdUpdateTargetPointEvent not handled for building repositioning

- [ ] **Step 3: Apply targeted fixes based on diagnostic findings**

This step depends on diagnostic output. Likely changes:
- Add missing abilCmdIndex entries to build maps in AbilityMapping
- No speculative fixes — only what the diagnostic identifies

- [ ] **Step 4: Re-run diagnostic to verify improvement**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=BuildingCoverageDiagnosticTest -q`
Expected: All building types at or near acceptance criteria (within 10% of oracle)

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/BuildingCoverageDiagnosticTest.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java
git commit -m "feat: add building coverage diagnostic + fix under-counting

Diagnostic compares oracle vs Java UnitInit counts per building type.
[Describe specific fixes applied based on diagnostic findings.]

Refs #325"
```

---

## Batch 4: Validation

### Task 6: Run full validation report and verify acceptance criteria

**Files:**
- No new files — runs existing StrippedReplayValidationTest

**Interfaces:**
- Consumes: all changes from Tasks 1-5

- [ ] **Step 1: Run StrippedReplayValidationTest report**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`
Expected: Divergence report showing improved UnitInit coverage

- [ ] **Step 2: Verify acceptance criteria**

Check report output against acceptance criteria:
- [ ] CreepTumor and CreepTumorQueen appear in UnitInit output with non-zero counts
- [ ] Each existing building type within 10% of oracle count
- [ ] NydusCanal, OracleStasisTrap, AssimilatorRich present in output

- [ ] **Step 3: Run full test suite (final verification)**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass

- [ ] **Step 4: Final commit if any adjustments were needed**

Only commit if validation revealed calibration adjustments needed.

---

## References

- [2026-10-01-creeptumor-special-buildings-design.md] — design spec this plan implements
- StrippedReplayFeatureExtractor.java:160 (extract loop), :326 (handleBuild),
  :630 (GAS_BUILDINGS), :790 (gasCostForBuilding), :893 (buildingTypeToPythonName)
- AbilityMapping.java:420 (dispatchHuman), :462 (buildCommand helper)
- BuildingType.java — enum
- SC2Data.java:254 (buildTimeInLoops), :500 (supplyBonus), :700 (maxBuildingHealth),
  :759 (mineralCost), :1021 (sightRange), :1048 (buildingRadius), :1069 (techTier),
  :1112 (isBase), :1120 (isProductionBuilding)
- AbilityDiscoveryCalibrationTest — pattern for abilLink discovery
- Protocol: enum-switch-exhaustive-required (PP-20260913-4d73d1)
- Protocol: extractor-separate-from-simulated-game (PP-20260528-612dee)
- GitHub #325, #318
