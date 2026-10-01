# Auto-Spawned Unit Event Synthesis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #324 — Synthesize auto-spawned unit events (Larva, MULE, Interceptor, Broodling)
**Issue group:** #324

**Goal:** Synthesize UnitBorn events for MULE (command-based), Larva (timer-based), and Interceptor (timer-based) in the StrippedReplayFeatureExtractor, and wire them into the classifier pipeline.

**Architecture:** MULE calldown is a CmdEvent dispatched through `AbilityMapping.dispatchHuman()`. Larva and Interceptor are timer-based post-loop synthesis methods in `StrippedReplayFeatureExtractor.extract()`, following the `emitWarpGateAutoMorph()` precedent. A global `larvaConsumptionLoops` list (not per-base) tracks when Larva are consumed to enforce the 3-per-base auto-spawn cap.

**Tech Stack:** Java 21, Quarkus, Scelight replay parser, Python classifier pipeline

## Global Constraints

- Timer constants (LARVA_SPAWN_INTERVAL, INTERCEPTOR_BUILD_TIME) MUST be calibrated from oracle replay data — never use `seconds × 22.4` formula (protocol PP-20260522-572156)
- Extraction logic belongs in dedicated classes, not SimulatedGame subclasses (protocol PP-20260528-612dee)
- All UnitType enum additions require entries in ALL SC2Data exhaustive maps (static initializer throws ExceptionInInitializerError on missing entries)
- Python UNITS list and Java FeatureIndexMaps.buildUnitIndex() ordering MUST match exactly
- MULE abilLink must be discovered from oracle replays before implementation (not from bot-replay constants)

---

## Batch 1: Diagnostic Discovery — Calibrate Constants

### Task 1: MULE abilLink Discovery Test

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/MuleAbilLinkDiscoveryTest.java`

**Interfaces:**
- Consumes: `RepParserEngine.parseReplay()`, `RepContent.GAME_EVENTS`, `RepContent.TRACKER_EVENTS`, oracle replay files in `replays/oracle/`
- Produces: printed abilLink constant (used manually in Task 4)

- [ ] **Step 1: Write the discovery test**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.trackerevents.UnitBornEvent;
import hu.scelight.sc2.rep.repproc.RepParserEngine;
import hu.scelight.sc2.rep.s2prot.type.Event;
import hu.scelight.util.gui.GuiUtils;
import io.quarkmind.sc2.mock.Sc2ReplayShared;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.assertFalse;

@Tag("diagnostic")
class MuleAbilLinkDiscoveryTest {

    private static final Path ORACLE_DIR = Path.of("replays/oracle");

    @Test
    @EnabledIf("oracleReplaysExist")
    void discoverMuleCalldownAbilLink() throws IOException {
        Map<Integer, Integer> abilLinkCounts = new TreeMap<>();
        int totalMuleBornEvents = 0;
        int replaysWithMule = 0;

        try (Stream<Path> paths = Files.list(ORACLE_DIR)) {
            for (Path replayPath : paths.filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                Replay replay = RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.GAME_EVENTS,
                               hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.TRACKER_EVENTS));
                if (replay == null || replay.trackerEvents == null) continue;

                // Find oracle MULE UnitBorn events
                List<UnitBornEvent> muleBorns = new ArrayList<>();
                for (var te : replay.trackerEvents.getEvents()) {
                    if (te instanceof UnitBornEvent ub
                        && "MULE".equalsIgnoreCase(ub.getUnitTypeName().toString().trim())) {
                        muleBorns.add(ub);
                    }
                }
                if (muleBorns.isEmpty()) continue;
                replaysWithMule++;
                totalMuleBornEvents += muleBorns.size();

                // Scan CmdEvents near each MULE birth for candidate abilLinks
                Event[] gameEvents = replay.gameEvents.getEvents();
                for (UnitBornEvent mule : muleBorns) {
                    long birthLoop = mule.getLoop();
                    int controlPlayer = mule.getControlPlayerId();
                    // Look for CmdEvents within [-50, +10] loops of birth
                    for (Event ge : gameEvents) {
                        if (ge instanceof hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent cmd) {
                            if (cmd.getLoop() >= birthLoop - 50 && cmd.getLoop() <= birthLoop + 10) {
                                Integer abl = cmd.getAbilLink();
                                if (abl != null && abl > 0) {
                                    abilLinkCounts.merge(abl, 1, Integer::sum);
                                }
                            }
                        }
                    }
                }
            }
        }

        System.out.println("=== MULE AbilLink Discovery ===");
        System.out.println("Replays with MULE births: " + replaysWithMule);
        System.out.println("Total MULE UnitBorn events: " + totalMuleBornEvents);
        System.out.println();
        System.out.println("Candidate abilLinks (sorted by frequency):");
        abilLinkCounts.entrySet().stream()
            .sorted(Map.Entry.<Integer, Integer>comparingByValue().reversed())
            .limit(10)
            .forEach(e -> System.out.printf("  abilLink=%d  count=%d%n", e.getKey(), e.getValue()));

        assertFalse(abilLinkCounts.isEmpty(), "Should find at least one candidate abilLink");
    }

    static boolean oracleReplaysExist() {
        return Files.isDirectory(ORACLE_DIR);
    }
}
```

- [ ] **Step 2: Run test to discover MULE abilLink**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=MuleAbilLinkDiscoveryTest -q`
Expected: prints candidate abilLinks with frequency counts. The highest-frequency candidate is the MULE calldown abilLink.

- [ ] **Step 3: Record the discovered abilLink**

Note the discovered value. It will be used as `ABIL_MULE_CALLDOWN` in Task 4.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/MuleAbilLinkDiscoveryTest.java
git commit -m "feat(diagnostic): MULE calldown abilLink discovery test Refs #324"
```

### Task 2: Auto-Spawn Timer Calibration Test

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AutoSpawnCalibrationTest.java`

**Interfaces:**
- Consumes: `RepParserEngine.parseReplay()`, oracle replay files, `UnitBornEvent`, `UnitInitEvent`, `UnitDoneEvent`
- Produces: printed constants for LARVA_SPAWN_INTERVAL, INTERCEPTOR_BUILD_TIME, and Inject Larva abilLink (used manually in Tasks 5-6)

- [ ] **Step 1: Write the calibration test**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.trackerevents.UnitBornEvent;
import hu.scelight.sc2.rep.model.trackerevents.UnitDoneEvent;
import hu.scelight.sc2.rep.model.trackerevents.UnitInitEvent;
import hu.scelight.sc2.rep.repproc.RepParserEngine;
import hu.scelight.sc2.rep.s2prot.type.Event;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.assertFalse;

@Tag("diagnostic")
class AutoSpawnCalibrationTest {

    private static final Path ORACLE_DIR = Path.of("replays/oracle");

    @Test
    @EnabledIf("oracleReplaysExist")
    void discoverLarvaSpawnInterval() throws IOException {
        List<Integer> intervals = new ArrayList<>();

        try (Stream<Path> paths = Files.list(ORACLE_DIR)) {
            for (Path replayPath : paths.filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                Replay replay = RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.TRACKER_EVENTS));
                if (replay == null || replay.trackerEvents == null) continue;

                // Collect Larva UnitBorn events per player
                Map<Integer, List<Long>> larvaBornsByPlayer = new HashMap<>();
                for (var te : replay.trackerEvents.getEvents()) {
                    if (te instanceof UnitBornEvent ub
                        && "Larva".equalsIgnoreCase(ub.getUnitTypeName().toString().trim())) {
                        larvaBornsByPlayer.computeIfAbsent(ub.getControlPlayerId(), k -> new ArrayList<>())
                            .add(ub.getLoop());
                    }
                }

                // Compute inter-birth intervals (excluding clusters of 3 = Inject)
                for (List<Long> loops : larvaBornsByPlayer.values()) {
                    Collections.sort(loops);
                    for (int i = 1; i < loops.size(); i++) {
                        int diff = (int) (loops.get(i) - loops.get(i - 1));
                        // Inject Larva produces 3 at the same loop — skip diff=0
                        // Auto-spawn interval is typically 200-300 loops
                        if (diff > 50 && diff < 500) {
                            intervals.add(diff);
                        }
                    }
                }
            }
        }

        // Modal analysis
        Map<Integer, Integer> histogram = new TreeMap<>();
        for (int interval : intervals) {
            histogram.merge(interval, 1, Integer::sum);
        }

        System.out.println("=== Larva Spawn Interval Calibration ===");
        System.out.println("Total interval samples: " + intervals.size());
        System.out.println();
        System.out.println("Top intervals (loop count → occurrences):");
        histogram.entrySet().stream()
            .sorted(Map.Entry.<Integer, Integer>comparingByValue().reversed())
            .limit(15)
            .forEach(e -> System.out.printf("  %d loops (%.1fs)  count=%d%n",
                e.getKey(), e.getKey() / 22.4, e.getValue()));

        assertFalse(intervals.isEmpty(), "Should find Larva spawn intervals");
    }

    @Test
    @EnabledIf("oracleReplaysExist")
    void discoverInterceptorBuildTime() throws IOException {
        List<Integer> buildTimes = new ArrayList<>();

        try (Stream<Path> paths = Files.list(ORACLE_DIR)) {
            for (Path replayPath : paths.filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                Replay replay = RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.TRACKER_EVENTS));
                if (replay == null || replay.trackerEvents == null) continue;

                // Find Carrier births and Interceptor births per player
                Map<Integer, List<Long>> carrierBirths = new HashMap<>();
                Map<Integer, List<Long>> interceptorBirths = new HashMap<>();
                for (var te : replay.trackerEvents.getEvents()) {
                    if (te instanceof UnitBornEvent ub) {
                        String name = ub.getUnitTypeName().toString().trim();
                        if ("Carrier".equalsIgnoreCase(name)) {
                            carrierBirths.computeIfAbsent(ub.getControlPlayerId(), k -> new ArrayList<>())
                                .add(ub.getLoop());
                        } else if ("Interceptor".equalsIgnoreCase(name)) {
                            interceptorBirths.computeIfAbsent(ub.getControlPlayerId(), k -> new ArrayList<>())
                                .add(ub.getLoop());
                        }
                    }
                }

                // For each Carrier, compute intervals between consecutive Interceptor births
                for (var entry : carrierBirths.entrySet()) {
                    int player = entry.getKey();
                    List<Long> intBirths = interceptorBirths.getOrDefault(player, List.of());
                    if (intBirths.size() < 2) continue;
                    Collections.sort(intBirths);
                    for (int i = 1; i < intBirths.size(); i++) {
                        int diff = (int) (intBirths.get(i) - intBirths.get(i - 1));
                        if (diff > 50 && diff < 500) {
                            buildTimes.add(diff);
                        }
                    }
                }
            }
        }

        Map<Integer, Integer> histogram = new TreeMap<>();
        for (int bt : buildTimes) {
            histogram.merge(bt, 1, Integer::sum);
        }

        System.out.println("=== Interceptor Build Time Calibration ===");
        System.out.println("Total build time samples: " + buildTimes.size());
        System.out.println();
        System.out.println("Top build times (loop count → occurrences):");
        histogram.entrySet().stream()
            .sorted(Map.Entry.<Integer, Integer>comparingByValue().reversed())
            .limit(15)
            .forEach(e -> System.out.printf("  %d loops (%.1fs)  count=%d%n",
                e.getKey(), e.getKey() / 22.4, e.getValue()));

        assertFalse(buildTimes.isEmpty(), "Should find Interceptor build time samples");
    }

    @Test
    @EnabledIf("oracleReplaysExist")
    void discoverInjectLarvaAbilLink() throws IOException {
        Map<Integer, Integer> abilLinkCounts = new TreeMap<>();
        int totalInjectClusters = 0;

        try (Stream<Path> paths = Files.list(ORACLE_DIR)) {
            for (Path replayPath : paths.filter(p -> p.toString().endsWith(".SC2Replay")).toList()) {
                Replay replay = RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.GAME_EVENTS,
                               hu.scelight.sc2.rep.model.initdata.gamedesc.RepContent.TRACKER_EVENTS));
                if (replay == null || replay.trackerEvents == null) continue;

                // Find Larva birth clusters (3+ at same loop = Inject)
                Map<Long, Integer> larvaBornCountByLoop = new TreeMap<>();
                for (var te : replay.trackerEvents.getEvents()) {
                    if (te instanceof UnitBornEvent ub
                        && "Larva".equalsIgnoreCase(ub.getUnitTypeName().toString().trim())) {
                        larvaBornCountByLoop.merge(ub.getLoop(), 1, Integer::sum);
                    }
                }
                List<Long> injectLoops = new ArrayList<>();
                for (var entry : larvaBornCountByLoop.entrySet()) {
                    if (entry.getValue() >= 3) {
                        injectLoops.add(entry.getKey());
                        totalInjectClusters++;
                    }
                }

                // Correlate Inject Larva clusters with CmdEvents
                Event[] gameEvents = replay.gameEvents.getEvents();
                for (long injectLoop : injectLoops) {
                    for (Event ge : gameEvents) {
                        if (ge instanceof hu.scelight.sc2.rep.model.gameevents.cmd.CmdEvent cmd) {
                            if (cmd.getLoop() >= injectLoop - 100 && cmd.getLoop() <= injectLoop + 10) {
                                Integer abl = cmd.getAbilLink();
                                if (abl != null && abl > 0) {
                                    abilLinkCounts.merge(abl, 1, Integer::sum);
                                }
                            }
                        }
                    }
                }
            }
        }

        System.out.println("=== Inject Larva AbilLink Discovery ===");
        System.out.println("Total Inject clusters (3+ Larva at same loop): " + totalInjectClusters);
        System.out.println();
        System.out.println("Candidate abilLinks:");
        abilLinkCounts.entrySet().stream()
            .sorted(Map.Entry.<Integer, Integer>comparingByValue().reversed())
            .limit(10)
            .forEach(e -> System.out.printf("  abilLink=%d  count=%d%n", e.getKey(), e.getValue()));

        assertFalse(abilLinkCounts.isEmpty(), "Should find Inject Larva candidate abilLinks");
    }

    static boolean oracleReplaysExist() {
        return Files.isDirectory(ORACLE_DIR);
    }
}
```

- [ ] **Step 2: Run all three diagnostic methods**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=AutoSpawnCalibrationTest -q`
Expected: prints calibrated LARVA_SPAWN_INTERVAL, INTERCEPTOR_BUILD_TIME, and Inject Larva abilLink.

- [ ] **Step 3: Record discovered constants**

Note all three values. They will be used in Tasks 5-6.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AutoSpawnCalibrationTest.java
git commit -m "feat(diagnostic): auto-spawn timer calibration and Inject Larva abilLink discovery Refs #324"
```

---

## Batch 2: MULE Synthesis — Command-Based

### Task 3: Add LARVA to UnitType Enum and SC2Data

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/UnitType.java:22` (add LARVA after BANELING)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:140-218` (add LARVA entry to UNIT_TRAIN_TIMES)

**Interfaces:**
- Consumes: nothing
- Produces: `UnitType.LARVA` enum value available project-wide; `SC2Data.trainTimeInLoops(UnitType.LARVA)` returns 0

- [ ] **Step 1: Write the failing test**

Add to existing `SC2DataTest` (or create inline verification):

```java
@Test
void larvaTrainTimeIsZero() {
    assertEquals(0, SC2Data.trainTimeInLoops(UnitType.LARVA));
}

@Test
void larvaIsZergRace() {
    assertEquals(Race.ZERG, UnitType.LARVA.race());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataTest#larvaTrainTimeIsZero -q`
Expected: FAIL — `UnitType.LARVA` does not exist.

- [ ] **Step 3: Add LARVA to UnitType enum**

In `quarkmind-sc2/src/main/java/io/quarkmind/domain/UnitType.java`, after line 21 (`BANELING(Race.ZERG),`), add:

```java
    LARVA(Race.ZERG),
```

- [ ] **Step 4: Add LARVA to SC2Data.UNIT_TRAIN_TIMES**

In `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java`, in the `UNIT_TRAIN_TIMES` static block after the BANELING entry (line 184), add:

```java
        map.put(UnitType.LARVA, 0);  // Larva don't train — auto-spawned by timer
```

Also add entries to `UNIT_SPEEDS` map if it has an exhaustive check:

```java
        // In UNIT_SPEEDS — Larva is immobile, but UNIT_SPEEDS uses getOrDefault so no entry needed
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataTest#larvaTrainTimeIsZero -q`
Expected: PASS

- [ ] **Step 6: Verify full build passes**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: compiles without ExceptionInInitializerError (UNIT_TRAIN_TIMES exhaustive check passes for LARVA)

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/UnitType.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/domain/SC2DataTest.java
git commit -m "feat: add LARVA to UnitType enum and SC2Data with trainTime=0 Refs #324"
```

### Task 4: Wire MULE Calldown into AbilityMapping

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java:91` (add ABIL_MULE_CALLDOWN constant)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java:425-473` (add case in dispatchHuman switch)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:40-100` (add MULE to UNIT_PYTHON_NAMES)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:209` (change MULE trainTime from 672 to 0)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java`

**Interfaces:**
- Consumes: discovered MULE abilLink from Task 1
- Produces: `AbilityMapping.dispatchHuman()` returns `IntentCommand(TrainIntent(MULE))` for MULE calldown CmdEvents; `UNIT_PYTHON_NAMES.get(UnitType.MULE)` returns `"MULE"`

- [ ] **Step 1: Write the failing test**

Add to `AbilityMappingTest.java`:

```java
@Test
void muleCalldownProducesTrainIntent() {
    var mapping = new AbilityMapping(1, true, Race.TERRAN);
    // Use the discovered abilLink value (replace DISCOVERED_MULE_ABIL with actual value)
    CmdEvent cmd = createCmdEvent(1, DISCOVERED_MULE_ABIL, 0, 100);
    List<ReplayCommand> commands = mapping.process(cmd);

    assertFalse(commands.isEmpty(), "MULE calldown should produce a command");
    assertInstanceOf(ReplayCommand.IntentCommand.class, commands.get(0));
    var ic = (ReplayCommand.IntentCommand) commands.get(0);
    assertInstanceOf(TrainIntent.class, ic.intent().intent());
    assertEquals(UnitType.MULE, ((TrainIntent) ic.intent().intent()).unitType());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest#muleCalldownProducesTrainIntent -q`
Expected: FAIL — no case for MULE abilLink in dispatchHuman.

- [ ] **Step 3: Add ABIL_MULE_CALLDOWN constant**

In `AbilityMapping.java` after line 91 (`ABIL_ORACLE_STASIS_WARD`), add:

```java
    private static final int ABIL_MULE_CALLDOWN = DISCOVERED_VALUE; // calibrated: MuleAbilLinkDiscoveryTest
```

- [ ] **Step 4: Add case in dispatchHuman switch**

In `AbilityMapping.java` in the `dispatchHuman` method (line 425), before the `default -> null;` case, add:

```java
            case ABIL_MULE_CALLDOWN -> isRace(Race.TERRAN) ? trainIntent(loop, UnitType.MULE) : null;
```

- [ ] **Step 5: Add MULE to UNIT_PYTHON_NAMES**

In `StrippedReplayFeatureExtractor.java` after line 61 (`map.put(UnitType.BATTLECRUISER, "Battlecruiser");`), add:

```java
        map.put(UnitType.MULE, "MULE");
```

- [ ] **Step 6: Fix MULE trainTime to 0**

In `SC2Data.java` line 209, change:

```java
        map.put(UnitType.MULE, 672);  // uncalibrated
```

to:

```java
        map.put(UnitType.MULE, 0);  // instant calldown — calibrated: MuleAbilLinkDiscoveryTest
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=AbilityMappingTest#muleCalldownProducesTrainIntent -q`
Expected: PASS

- [ ] **Step 8: Run full test suite to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all existing tests pass

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/AbilityMappingTest.java
git commit -m "feat: wire MULE calldown abilLink into AbilityMapping.dispatchHuman Refs #324"
```

---

## Batch 3: Larva Synthesis — Timer-Based

### Task 5: Add Larva Consumption Tracking and Starting Hatchery

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:968-989` (add `larvaConsumptionLoops` to PlayerState)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:257-267` (add starting Hatchery to trackedBuildings)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:191-212` (record Larva consumption loop during train handling)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (add LARVA_SPAWN_INTERVAL constant)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: calibrated LARVA_SPAWN_INTERVAL from Task 2, `PlayerState`, `TrackedBuilding`, `SyntheticEvent`
- Produces: `PlayerState.larvaConsumptionLoops` populated during main loop; starting Hatchery in `trackedBuildings` with `doneLoop=0`; `SC2Data.LARVA_SPAWN_INTERVAL` constant

- [ ] **Step 1: Write the failing test for starting Hatchery tracking**

Add to `StrippedReplayFeatureExtractorTest.java`:

```java
@Test
void zergStartingHatcheryInTrackedBuildings() {
    // Use processTrainForTest or a helper that exposes PlayerState
    // Verify that after init, a Zerg player has a TrackedBuilding for Hatchery with doneLoop=0
    var extractor = new StrippedReplayFeatureExtractor();
    // This test requires either a test helper or testing via the extract output
    // Simplest: verify that Larva UnitBorn events appear from loop 0 + interval
    // (This will be tested via the emitAutoSpawnedLarva test in the next step)
}
```

- [ ] **Step 2: Add LARVA_SPAWN_INTERVAL to SC2Data**

In `SC2Data.java`, add after `MULE_LIFETIME_LOOPS` (line 72):

```java
    /**
     * Larva auto-spawn interval in game loops.
     * Each Hatchery/Lair/Hive spawns one Larva every LARVA_SPAWN_INTERVAL loops,
     * up to a cap of 3 auto-spawned Larva per base.
     * Calibrated: AutoSpawnCalibrationTest (oracle replays).
     */
    public static final int LARVA_SPAWN_INTERVAL = DISCOVERED_VALUE; // replace with calibrated value
```

- [ ] **Step 3: Add larvaConsumptionLoops to PlayerState**

In `StrippedReplayFeatureExtractor.java` in the `PlayerState` class (line 968), add:

```java
        final List<Long> larvaConsumptionLoops = new ArrayList<>();
```

- [ ] **Step 4: Add starting Hatchery to trackedBuildings**

In `initStartingBuildings()` (line 257), inside the `race == Race.ZERG` branch, add:

```java
            state.trackedBuildings.add(new TrackedBuilding(0, "Hatchery", 0));
```

Tag 0 is used for the starting Hatchery (before `tagCounter` starts at 1).

- [ ] **Step 5: Record Larva consumption in the main loop**

In `extract()` (line 191-212), inside the `TrainIntent` handling block, after the `for (int r = 0; r < repeatCount; r++)` loop, add consumption tracking:

```java
                                    if (abilLink != null && abilLink == ABIL_LARVA) {
                                        for (int r2 = 0; r2 < repeatCount; r2++) {
                                            state.larvaConsumptionLoops.add(ti.loop());
                                        }
                                    }
```

- [ ] **Step 6: Add UNIT_PYTHON_NAMES entries for Larva and Interceptor**

In `StrippedReplayFeatureExtractor.java` UNIT_PYTHON_NAMES static block, add after the Overseer entry (line 79):

```java
        map.put(UnitType.LARVA, "Larva");
```

And after Observer (line 99):

```java
        map.put(UnitType.INTERCEPTOR, "Interceptor");
```

- [ ] **Step 7: Run build to verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: compiles successfully

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java
git commit -m "feat: add Larva consumption tracking, starting Hatchery, and spawn interval constant Refs #324"
```

### Task 6: Implement emitAutoSpawnedLarva Post-Loop Method

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java:241-243` (call emitAutoSpawnedLarva after WarpGate auto-morph)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java` (add emitAutoSpawnedLarva method)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `PlayerState.trackedBuildings`, `PlayerState.larvaConsumptionLoops`, `SC2Data.LARVA_SPAWN_INTERVAL`, `SyntheticEvent`, `EventOrdinal.UNIT_BORN`
- Produces: `emitAutoSpawnedLarva(int playerId, Race race, PlayerState state, List<SyntheticEvent> events, int tagCounter, long elapsedLoops) → int` (returns updated tagCounter)

- [ ] **Step 1: Write the failing test — basic Larva spawning**

Add to `StrippedReplayFeatureExtractorTest.java`:

```java
@Test
void larvaAutoSpawnFromStartingHatchery() {
    // Create a minimal Zerg replay scenario with no train commands
    // Expect Larva UnitBorn events at LARVA_SPAWN_INTERVAL intervals, capped at 3
    var extractor = new StrippedReplayFeatureExtractor();
    // Use a test helper that calls emitAutoSpawnedLarva directly
    // (add a package-private test method similar to processTrainForTest)
    var state = new Object() {}; // Will use test helper
    // Assert: 3 Larva UnitBorn events appear at intervals, then spawning stops
}
```

Note: the exact test structure will depend on adding a test helper method similar to `processTrainForTest`. The test should verify:
1. Starting Hatchery at doneLoop=0 produces Larva at intervals
2. Cap at 3 per base stops further spawning
3. Consumption decrements count and resumes spawning

- [ ] **Step 2: Implement emitAutoSpawnedLarva**

Add method to `StrippedReplayFeatureExtractor` after `emitWarpGateAutoMorph` (line 484):

```java
    private int emitAutoSpawnedLarva(int playerId, Race race, PlayerState state,
                                     List<SyntheticEvent> events, int tagCounter,
                                     long elapsedLoops) {
        if (race != Race.ZERG) return tagCounter;

        // Collect all hatchery-type buildings (starting + built during game)
        List<TrackedBuilding> hatcheries = state.trackedBuildings.stream()
            .filter(b -> "Hatchery".equals(b.name()) || "Lair".equals(b.name()) || "Hive".equals(b.name()))
            .toList();

        if (hatcheries.isEmpty()) return tagCounter;

        // Build sorted consumption timeline
        List<Long> consumptions = new ArrayList<>(state.larvaConsumptionLoops);
        Collections.sort(consumptions);
        int consumptionIdx = 0;

        // Per-base spawn state
        int[] spawnedPerBase = new int[hatcheries.size()];
        long[] nextSpawnLoop = new long[hatcheries.size()];
        for (int i = 0; i < hatcheries.size(); i++) {
            nextSpawnLoop[i] = hatcheries.get(i).doneLoop() + SC2Data.LARVA_SPAWN_INTERVAL;
        }

        // Global running Larva count (spawned minus consumed)
        int globalLarvaCount = 0;

        // Walk through time, spawning and consuming
        long currentLoop = 0;
        while (currentLoop <= elapsedLoops) {
            // Find the next event (spawn or consumption)
            long nextConsumptionLoop = consumptionIdx < consumptions.size()
                ? consumptions.get(consumptionIdx) : Long.MAX_VALUE;

            long earliestSpawn = Long.MAX_VALUE;
            int earliestBase = -1;
            for (int i = 0; i < hatcheries.size(); i++) {
                if (spawnedPerBase[i] < 3 && nextSpawnLoop[i] < earliestSpawn) {
                    earliestSpawn = nextSpawnLoop[i];
                    earliestBase = i;
                }
            }

            long nextEvent = Math.min(nextConsumptionLoop, earliestSpawn);
            if (nextEvent > elapsedLoops || nextEvent == Long.MAX_VALUE) break;

            currentLoop = nextEvent;

            // Process consumption first (if at same loop)
            while (consumptionIdx < consumptions.size()
                   && consumptions.get(consumptionIdx) <= currentLoop) {
                globalLarvaCount = Math.max(0, globalLarvaCount - 1);
                // Decrement the base with most spawned larva (heuristic)
                int maxBase = 0;
                for (int i = 1; i < spawnedPerBase.length; i++) {
                    if (spawnedPerBase[i] > spawnedPerBase[maxBase]) maxBase = i;
                }
                if (spawnedPerBase[maxBase] > 0) spawnedPerBase[maxBase]--;
                consumptionIdx++;
            }

            // Process spawns at this loop
            for (int i = 0; i < hatcheries.size(); i++) {
                while (nextSpawnLoop[i] <= currentLoop
                       && nextSpawnLoop[i] <= elapsedLoops
                       && spawnedPerBase[i] < 3) {
                    int tag = tagCounter++;
                    events.add(new SyntheticEvent(nextSpawnLoop[i], EventOrdinal.UNIT_BORN, playerId,
                        Map.of("evtTypeName", "UnitBorn",
                            "loop", nextSpawnLoop[i],
                            "controlPlayerId", playerId,
                            "unitTypeName", "Larva",
                            "unitTagIndex", tag,
                            "unitTagRecycle", 0)));
                    spawnedPerBase[i]++;
                    globalLarvaCount++;
                    nextSpawnLoop[i] += SC2Data.LARVA_SPAWN_INTERVAL;
                }
            }
        }

        return tagCounter;
    }
```

- [ ] **Step 3: Wire into extract() post-loop**

In `extract()`, after the WarpGate auto-morph call (line 243), add:

```java
            tagCounter = emitAutoSpawnedLarva(playerId, playerRace, state,
                                              syntheticEvents, tagCounter,
                                              replay.header.getElapsedGameLoops() != null
                                                  ? replay.header.getElapsedGameLoops() : 0);
```

- [ ] **Step 4: Add test helper for direct testing**

Add package-private test method:

```java
    List<Map<String, Object>> emitAutoSpawnedLarvaForTest(int playerId, Race race,
                                                          List<Long> consumptionLoops,
                                                          List<long[]> hatcheryDoneLoops,
                                                          long elapsedLoops, int startTag) {
        var state = new PlayerState();
        for (long[] h : hatcheryDoneLoops) {
            state.trackedBuildings.add(new TrackedBuilding((int) h[0], "Hatchery", h[1]));
        }
        state.larvaConsumptionLoops.addAll(consumptionLoops);
        var events = new ArrayList<SyntheticEvent>();
        emitAutoSpawnedLarva(playerId, race, state, events, startTag, elapsedLoops);
        return events.stream().map(SyntheticEvent::data).toList();
    }
```

- [ ] **Step 5: Write and run unit tests**

```java
@Test
void larvaAutoSpawnCapsAtThreePerBase() {
    var extractor = new StrippedReplayFeatureExtractor();
    // One hatchery at loop 0, no consumption, game lasts 10000 loops
    var events = extractor.emitAutoSpawnedLarvaForTest(
        1, Race.ZERG, List.of(), List.of(new long[]{0, 0}), 10000, 1);
    long larvaCount = events.stream()
        .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
        .filter(e -> "Larva".equals(e.get("unitTypeName")))
        .count();
    assertEquals(3, larvaCount, "Should cap at 3 Larva per base");
}

@Test
void larvaConsumptionResumesSpawning() {
    var extractor = new StrippedReplayFeatureExtractor();
    int interval = SC2Data.LARVA_SPAWN_INTERVAL;
    // Consume 1 Larva after 3 have spawned → should get 4 total
    long consumeLoop = (long) interval * 4; // after 3 have spawned
    var events = extractor.emitAutoSpawnedLarvaForTest(
        1, Race.ZERG, List.of(consumeLoop), List.of(new long[]{0, 0}), 20000, 1);
    long larvaCount = events.stream()
        .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
        .filter(e -> "Larva".equals(e.get("unitTypeName")))
        .count();
    assertTrue(larvaCount > 3, "Consuming Larva should allow more spawning");
}

@Test
void larvaMultipleBasesSpawnIndependently() {
    var extractor = new StrippedReplayFeatureExtractor();
    // Two hatcheries, no consumption
    var events = extractor.emitAutoSpawnedLarvaForTest(
        1, Race.ZERG, List.of(),
        List.of(new long[]{0, 0}, new long[]{1, 2000}), 20000, 1);
    long larvaCount = events.stream()
        .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
        .filter(e -> "Larva".equals(e.get("unitTypeName")))
        .count();
    assertEquals(6, larvaCount, "Two bases should produce 6 Larva (3 each)");
}

@Test
void nonZergProducesNoLarva() {
    var extractor = new StrippedReplayFeatureExtractor();
    var events = extractor.emitAutoSpawnedLarvaForTest(
        1, Race.TERRAN, List.of(), List.of(new long[]{0, 0}), 10000, 1);
    assertTrue(events.isEmpty());
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorTest -q`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java
git commit -m "feat: implement Larva auto-spawn synthesis with per-base cap and consumption tracking Refs #324"
```

---

## Batch 4: Interceptor Synthesis and Classifier Integration

### Task 7: Implement emitAutoSpawnedInterceptor Post-Loop Method

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java` (add emitAutoSpawnedInterceptor method + wire into extract())
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (add INTERCEPTOR_BUILD_TIME constant)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java`

**Interfaces:**
- Consumes: calibrated INTERCEPTOR_BUILD_TIME from Task 2, existing `SyntheticEvent` list (to find Carrier births)
- Produces: `emitAutoSpawnedInterceptor(int playerId, Race race, List<SyntheticEvent> existingEvents, List<SyntheticEvent> newEvents, int tagCounter, long elapsedLoops) → int`

- [ ] **Step 1: Add INTERCEPTOR_BUILD_TIME to SC2Data**

In `SC2Data.java`, after `LARVA_SPAWN_INTERVAL`, add:

```java
    /**
     * Interceptor auto-build time in game loops.
     * Carriers auto-build Interceptors up to a cap of 8.
     * Calibrated: AutoSpawnCalibrationTest (oracle replays).
     */
    public static final int INTERCEPTOR_BUILD_TIME = DISCOVERED_VALUE; // replace with calibrated value
```

- [ ] **Step 2: Write the failing test**

```java
@Test
void interceptorAutoSpawnFromCarrier() {
    var extractor = new StrippedReplayFeatureExtractor();
    var events = extractor.emitAutoSpawnedInterceptorForTest(
        1, Race.PROTOSS, 500, 20000, 1);
    long interceptorCount = events.stream()
        .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
        .filter(e -> "Interceptor".equals(e.get("unitTypeName")))
        .count();
    assertEquals(8, interceptorCount, "Should cap at 8 Interceptors per Carrier");
}

@Test
void interceptorTimingStartsAfterCarrierBirth() {
    var extractor = new StrippedReplayFeatureExtractor();
    int buildTime = SC2Data.INTERCEPTOR_BUILD_TIME;
    var events = extractor.emitAutoSpawnedInterceptorForTest(
        1, Race.PROTOSS, 1000, 1000 + buildTime * 2, 1);
    // With only 2 build times worth of game time, should get exactly 2 interceptors
    long interceptorCount = events.stream()
        .filter(e -> "UnitBorn".equals(e.get("evtTypeName")))
        .filter(e -> "Interceptor".equals(e.get("unitTypeName")))
        .count();
    assertEquals(2, interceptorCount);
}

@Test
void nonProtossProducesNoInterceptors() {
    var extractor = new StrippedReplayFeatureExtractor();
    var events = extractor.emitAutoSpawnedInterceptorForTest(
        1, Race.ZERG, 500, 20000, 1);
    assertTrue(events.isEmpty());
}
```

- [ ] **Step 3: Implement emitAutoSpawnedInterceptor and test helper**

```java
    private int emitAutoSpawnedInterceptor(int playerId, Race race,
                                           List<SyntheticEvent> existingEvents,
                                           List<SyntheticEvent> newEvents,
                                           int tagCounter, long elapsedLoops) {
        if (race != Race.PROTOSS) return tagCounter;

        // Find Carrier UnitBorn events for this player in existing events
        List<Long> carrierBirthLoops = existingEvents.stream()
            .filter(e -> e.playerId() == playerId)
            .filter(e -> "Carrier".equals(e.data().get("unitTypeName")))
            .filter(e -> "UnitBorn".equals(e.data().get("evtTypeName")))
            .map(SyntheticEvent::loop)
            .toList();

        for (long carrierBirth : carrierBirthLoops) {
            int built = 0;
            long nextBuild = carrierBirth + SC2Data.INTERCEPTOR_BUILD_TIME;
            while (built < 8 && nextBuild <= elapsedLoops) {
                int tag = tagCounter++;
                newEvents.add(new SyntheticEvent(nextBuild, EventOrdinal.UNIT_BORN, playerId,
                    Map.of("evtTypeName", "UnitBorn",
                        "loop", nextBuild,
                        "controlPlayerId", playerId,
                        "unitTypeName", "Interceptor",
                        "unitTagIndex", tag,
                        "unitTagRecycle", 0)));
                built++;
                nextBuild += SC2Data.INTERCEPTOR_BUILD_TIME;
            }
        }

        return tagCounter;
    }

    List<Map<String, Object>> emitAutoSpawnedInterceptorForTest(int playerId, Race race,
                                                                 long carrierBirthLoop,
                                                                 long elapsedLoops, int startTag) {
        var existing = new ArrayList<SyntheticEvent>();
        existing.add(new SyntheticEvent(carrierBirthLoop, EventOrdinal.UNIT_BORN, playerId,
            Map.of("evtTypeName", "UnitBorn", "loop", carrierBirthLoop,
                "controlPlayerId", playerId, "unitTypeName", "Carrier",
                "unitTagIndex", 0, "unitTagRecycle", 0)));
        var newEvents = new ArrayList<SyntheticEvent>();
        emitAutoSpawnedInterceptor(playerId, race, existing, newEvents, startTag, elapsedLoops);
        return newEvents.stream().map(SyntheticEvent::data).toList();
    }
```

- [ ] **Step 4: Wire into extract() post-loop**

In `extract()`, after the `emitAutoSpawnedLarva` call, add:

```java
            tagCounter = emitAutoSpawnedInterceptor(playerId, playerRace,
                                                     syntheticEvents, syntheticEvents,
                                                     tagCounter,
                                                     replay.header.getElapsedGameLoops() != null
                                                         ? replay.header.getElapsedGameLoops() : 0);
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorTest -q`
Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorTest.java
git commit -m "feat: implement Interceptor auto-build synthesis with 8-per-Carrier cap Refs #324"
```

### Task 8: Classifier Pipeline Integration

**Files:**
- Modify: `quarkmind-classifier/src/sc2egset_extractor.py:46-59` (append to UNITS list)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java:13,114-171` (N_UNITS + buildUnitIndex)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureAlignmentTest.java`

**Interfaces:**
- Consumes: `UnitType.LARVA`, `UnitType.MULE`, `UnitType.INTERCEPTOR`
- Produces: Python `UNIT_IDX["Larva"] == 53`, `UNIT_IDX["MULE"] == 54`, `UNIT_IDX["Interceptor"] == 55`; Java `UNIT_INDEX.get(UnitType.LARVA) == 53`, etc.; `N_UNITS == 56`

- [ ] **Step 1: Write the failing test**

Add to `FeatureAlignmentTest.java` (or add a new test method):

```java
@Test
void newAutoSpawnUnitsInUnitIndex() {
    assertEquals(56, FeatureIndexMaps.N_UNITS);
    assertEquals(53, (int) FeatureIndexMaps.UNIT_INDEX.get(UnitType.LARVA));
    assertEquals(54, (int) FeatureIndexMaps.UNIT_INDEX.get(UnitType.MULE));
    assertEquals(55, (int) FeatureIndexMaps.UNIT_INDEX.get(UnitType.INTERCEPTOR));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureAlignmentTest#newAutoSpawnUnitsInUnitIndex -q`
Expected: FAIL — N_UNITS is 53, indices don't exist.

- [ ] **Step 3: Update FeatureIndexMaps**

In `FeatureIndexMaps.java`:
- Line 13: change `N_UNITS = 53` to `N_UNITS = 56`
- After line 170 (`map.put(UnitType.OBSERVER, 52);`), add:

```java
        map.put(UnitType.LARVA, 53);
        map.put(UnitType.MULE, 54);
        map.put(UnitType.INTERCEPTOR, 55);
```

- [ ] **Step 4: Update Python UNITS list**

In `quarkmind-classifier/src/sc2egset_extractor.py`, after line 58 (`"Observer",`), add:

```python
    "Larva", "MULE", "Interceptor",
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=FeatureAlignmentTest -q`
Expected: PASS

Also run Python tests:

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/ -v -k "unit" --no-header`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java quarkmind-classifier/src/sc2egset_extractor.py quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/FeatureAlignmentTest.java
git commit -m "feat: add Larva, MULE, Interceptor to classifier pipeline (N_UNITS 53→56) Refs #324"
```

---

## Batch 5: Validation

### Task 9: Extend StrippedReplayValidationTest

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java` (add per-type threshold assertions for auto-spawn types)

**Interfaces:**
- Consumes: oracle replays, `StrippedReplayFeatureExtractor.extract()`
- Produces: per-type divergence report with Larva threshold assertion (within 20% of oracle)

- [ ] **Step 1: Read existing StrippedReplayValidationTest to understand assertion structure**

Read the existing test to understand how oracle vs Java counts are compared. The test currently prints a divergence report but has only `isGreaterThan(0)` assertions.

- [ ] **Step 2: Add per-type threshold assertion for Larva**

After the existing assertions, add:

```java
// Auto-spawn accuracy assertions
if (oracleLarvaCounts > 0) {
    double larvaRatio = (double) javaLarvaCounts / oracleLarvaCounts;
    System.out.printf("Larva accuracy: %.1f%% (java=%d, oracle=%d)%n",
        larvaRatio * 100, javaLarvaCounts, oracleLarvaCounts);
    assertThat(larvaRatio).as("Larva count within 20%% of oracle")
        .isBetween(0.8, 1.2);
}
```

- [ ] **Step 3: Verify Larva, MULE, Interceptor appear in the divergence report**

These types already appear on the oracle side of the report. The Java side should now produce non-zero counts for them. No code change needed — just verify by running.

- [ ] **Step 4: Run validation test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrippedReplayValidationTest -q`
Expected: report prints Larva/MULE/Interceptor divergence. Larva ratio is between 0.8 and 1.2.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayValidationTest.java
git commit -m "feat: add auto-spawn type accuracy assertions to validation test Refs #324"
```

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests pass

---

## References

- [2026-10-01-auto-spawned-unit-events-design.md] — design spec this plan implements
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java` — main extraction pipeline, SyntheticEvent:963, PlayerState:968, emitWarpGateAutoMorph:452, initStartingBuildings:257, extract:161
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` — dispatchHuman:425, abilLink constants:44-91
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/UnitType.java` — enum definition, MULE:33, INTERCEPTOR:13
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` — UNIT_TRAIN_TIMES:140-218, trainTimeInLoops:231
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java` — N_UNITS:13, buildUnitIndex:114-172
- `quarkmind-classifier/src/sc2egset_extractor.py` — UNITS:46-59, UNIT_IDX:81
- PP-20260522-572156 — train-times-require-calibration protocol
- PP-20260528-612dee — extractor-separate-from-simulated-game protocol
- GitHub #324 — focal issue
- GitHub #318 — parent epic
