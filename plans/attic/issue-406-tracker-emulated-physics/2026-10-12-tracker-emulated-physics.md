# Tracker-Driven EmulatedGame Physics Validation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #406 — Milestone 2: Tracker-event-driven EmulatedGame physics validation
**Issue group:** #406

**Goal:** Build a tracker-event-driven validation path that feeds EmulatedGame complete inputs from oracle replay ground truth, isolating physics accuracy from input completeness.

**Architecture:** Create a `TrackerIntentExtractor` that converts oracle tracker events (UnitBorn, UnitInit+UnitDone, Upgrade) into `TimedIntent`s by back-calculating command loops from completion events. Feed these intents through the existing `ReplayValidationHarness.run(groundTruth, intents, tickLimit, race)` API — the harness already accepts `List<TimedIntent>`. Create a new baseline test that reports per-type accuracy at checkpoints, using the same accumulation pattern as `EmulatedGameAccuracyBaselineTest`.

**Tech Stack:** Java 21, Scelight replay parser (ocraft), existing SC2Data train/build/upgrade time constants, existing ReplayValidationHarness

## Global Constraints

- Intent types are sealed (`Intent permits BuildIntent, TrainIntent, ...`) — any new permits clause entry must update every `switch` over `Intent` in the same commit
- Train/build/upgrade times must come from `SC2Data` calibrated constants, not from `seconds × 22.4` formulae (protocol: `sc2data-train-times-require-calibration`)
- Tracker event name → domain type mapping must use `Sc2ReplayShared.toUnitType()` and `Sc2ReplayShared.toBuildingType()` — no duplicate mapping tables
- Calibration targets are waypoints per protocol `calibration-milestones-are-waypoints` — reaching them triggers assessment, not closure
- `TrackerIntentExtractor` must be a separate class from `SimulatedGame` (protocol: `extractor-separate-from-simulated-game`)

---

## Batch 1: TrackerIntentExtractor — unit and building intents

### Task 1: TrackerIntentExtractor with UnitBorn → TrainIntent conversion

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java`
- Read: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/mock/Sc2ReplayShared.java` (name mapping)
- Read: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (train times)

**Interfaces:**
- Consumes: `Sc2ReplayShared.toUnitType(String)`, `Sc2ReplayShared.isBuilding(String)`, `SC2Data.trainTimeInLoops(UnitType)`, `SC2Data.buildTimeInLoops(BuildingType)`, `Sc2ReplayShared.toBuildingType(String)`
- Produces: `TrackerIntentExtractor.extract(Path replayPath, int playerId) → List<TimedIntent>` — static method matching `ReplayCommandExtractor` pattern

- [ ] **Step 1: Write failing test — Protoss unit extraction from a known oracle replay**

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.sc2.intent.*;
import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;
import java.nio.file.*;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class TrackerIntentExtractorIntentTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");

    static boolean oracleExists() {
        if (!Files.isDirectory(ORACLE_DIR)) return false;
        try (var stream = Files.list(ORACLE_DIR)) {
            return stream.anyMatch(p -> p.toString().endsWith(".SC2Replay"));
        } catch (Exception e) { return false; }
    }

    @Test
    @EnabledIf("oracleExists")
    void extractsTrainIntentsFromUnitBorn() throws Exception {
        Path replay = Files.list(ORACLE_DIR)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().findFirst().orElseThrow();

        List<TimedIntent> intents = TrackerIntentExtractor.extract(replay, 1);

        assertFalse(intents.isEmpty(), "Should produce intents from tracker events");

        long trainCount = intents.stream()
            .filter(ti -> ti.intent() instanceof TrainIntent).count();
        assertTrue(trainCount > 0, "Should have TrainIntents from UnitBorn events");

        // Intents must be sorted by loop
        for (int i = 1; i < intents.size(); i++) {
            assertTrue(intents.get(i).loop() >= intents.get(i - 1).loop(),
                "Intents must be sorted by loop");
        }

        // All TrainIntents should have non-negative loops (back-calculated from completion)
        intents.stream()
            .filter(ti -> ti.intent() instanceof TrainIntent)
            .forEach(ti -> assertTrue(ti.loop() >= 0,
                "Back-calculated loop must be non-negative: " + ti));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest -q`
Expected: FAIL — `TrackerIntentExtractor` class does not exist

- [ ] **Step 3: Implement TrackerIntentExtractor with UnitBorn → TrainIntent**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.repproc.RepParserEngine;
import hu.scelight.sc2.rep.s2prot.Event;
import hu.scelight.sc2.rep.s2prot.type.IBaseUnitEvent;
import hu.scelight.sc2.rep.s2prot.type.ITrackerEvents;
import hu.scelight.sc2.rep.s2prot.type.IUpgradeEvent;
import hu.sllauncher.util.repconst.RepContent;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.*;
import io.quarkmind.sc2.mock.Sc2ReplayShared;

import java.nio.file.Path;
import java.util.*;

public final class TrackerIntentExtractor {

    private TrackerIntentExtractor() {}

    public static List<TimedIntent> extract(Path replayPath, int playerId) {
        Replay replay;
        try {
            replay = RepParserEngine.parseReplay(replayPath,
                EnumSet.of(RepContent.TRACKER_EVENTS, RepContent.DETAILS));
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot parse replay: " + replayPath, e);
        }
        if (replay == null || replay.trackerEvents == null) {
            throw new IllegalArgumentException("No tracker events in replay: " + replayPath);
        }

        Race race = resolvePlayerRace(replay, playerId);
        List<TimedIntent> intents = new ArrayList<>();
        record PendingBuilding(String typeName, int playerId, long initLoop,
                               int tagIndex, int tagRecycle) {}
        Map<Long, PendingBuilding> pendingBuildings = new HashMap<>();

        for (Event raw : replay.trackerEvents.getEvents()) {
            int id = raw.getId();
            switch (id) {
                case ITrackerEvents.ID_UNIT_BORN -> {
                    if (!(raw instanceof IBaseUnitEvent ue)) continue;
                    Integer pid = ue.getControlPlayerId();
                    if (pid == null || pid != playerId) continue;
                    String typeName = String.valueOf(ue.getUnitTypeName()).trim();
                    if (Sc2ReplayShared.isBuilding(typeName)) continue;

                    UnitType unitType = Sc2ReplayShared.toUnitType(typeName);
                    if (unitType == UnitType.UNKNOWN) continue;

                    int completionLoop = ue.getLoop();
                    int trainTime = SC2Data.trainTimeInLoops(unitType);
                    int commandLoop = Math.max(0, completionLoop - trainTime);

                    String tag = ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle();
                    intents.add(new TimedIntent(commandLoop,
                        new TrainIntent(tag, unitType)));
                }
                case ITrackerEvents.ID_UNIT_INIT -> {
                    if (!(raw instanceof IBaseUnitEvent ue)) continue;
                    Integer pid = ue.getControlPlayerId();
                    if (pid == null || pid != playerId) continue;
                    String typeName = String.valueOf(ue.getUnitTypeName()).trim();
                    if (!Sc2ReplayShared.isBuilding(typeName)) continue;

                    int tagIdx = ue.getUnitTagIndex() != null ? ue.getUnitTagIndex() : 0;
                    int tagRec = ue.getUnitTagRecycle() != null ? ue.getUnitTagRecycle() : 0;
                    long compositeTag = ((long) tagIdx << 32) | (tagRec & 0xFFFFFFFFL);
                    pendingBuildings.put(compositeTag,
                        new PendingBuilding(typeName, pid, ue.getLoop(), tagIdx, tagRec));
                }
                case ITrackerEvents.ID_UNIT_DONE -> {
                    Integer tagIdx = raw.get("unitTagIndex");
                    Integer tagRec = raw.get("unitTagRecycle");
                    if (tagIdx == null || tagRec == null) continue;
                    long compositeTag = ((long) tagIdx << 32) | (tagRec & 0xFFFFFFFFL);
                    PendingBuilding pb = pendingBuildings.remove(compositeTag);
                    if (pb == null) continue;

                    BuildingType bt = Sc2ReplayShared.toBuildingType(pb.typeName());
                    if (bt == null) continue;

                    String tag = pb.tagIndex() + "-" + pb.tagRecycle();
                    intents.add(new TimedIntent(pb.initLoop(),
                        new BuildIntent(tag, bt, null)));
                }
                // Upgrades handled in Task 2 after UpgradeType.fromTrackerName is added
                default -> {}
            }
        }

        intents.sort(Comparator.comparingLong(TimedIntent::loop));
        return Collections.unmodifiableList(intents);
    }

    private static Race resolvePlayerRace(Replay replay, int playerId) {
        if (replay.details != null) {
            var players = replay.details.getPlayerList();
            if (players.length >= playerId) {
                var scRace = players[playerId - 1].getRace();
                if (scRace == hu.scelight.sc2.rep.model.details.Race.TERRAN) return Race.TERRAN;
                if (scRace == hu.scelight.sc2.rep.model.details.Race.ZERG) return Race.ZERG;
            }
        }
        return Race.PROTOSS;
    }
}
```

Note: This implementation requires `UpgradeType.fromTrackerName(String)` — see Task 2. For this task, the upgrade conversion will produce no results until that method exists. The test in Step 1 only asserts on TrainIntents, which don't need upgrade mapping.

Note: `Sc2ReplayShared.isBuilding(String)` and `Sc2ReplayShared.toBuildingType(String)` already exist. Verify via `ide_find_symbol` before implementing — do NOT create duplicate mappings.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest -q`
Expected: PASS

- [ ] **Step 5: Write test — BuildIntents from UnitInit+UnitDone**

Add to `TrackerIntentExtractorIntentTest`:

```java
@Test
@EnabledIf("oracleExists")
void extractsBuildIntentsFromUnitInitDone() throws Exception {
    Path replay = Files.list(ORACLE_DIR)
        .filter(p -> p.toString().endsWith(".SC2Replay"))
        .sorted().findFirst().orElseThrow();

    List<TimedIntent> intents = TrackerIntentExtractor.extract(replay, 1);

    long buildCount = intents.stream()
        .filter(ti -> ti.intent() instanceof BuildIntent).count();
    assertTrue(buildCount > 0, "Should have BuildIntents from UnitInit/UnitDone events");

    // BuildIntent loops should be the UnitInit loop (start of construction)
    intents.stream()
        .filter(ti -> ti.intent() instanceof BuildIntent)
        .forEach(ti -> assertTrue(ti.loop() >= 0,
            "BuildIntent loop (UnitInit) must be non-negative: " + ti));
}
```

- [ ] **Step 6: Run test — should pass (already implemented in Step 3)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java
git commit -m "feat: add TrackerIntentExtractor — UnitBorn→TrainIntent, UnitInit/Done→BuildIntent

Converts oracle tracker events into TimedIntents with back-calculated
command loops. 100% input coverage vs 43% from CmdEvent path.

Refs #406"
```

### Task 2: UpgradeType.fromTrackerName and ResearchIntent conversion

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/UpgradeType.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java` (add test)

**Interfaces:**
- Produces: `UpgradeType.fromTrackerName(String) → UpgradeType` (nullable) — maps tracker event upgrade names to enum values

- [ ] **Step 1: Write failing test — ResearchIntents from Upgrade events**

Add to `TrackerIntentExtractorIntentTest`:

```java
@Test
@EnabledIf("oracleExists")
void extractsResearchIntentsFromUpgradeEvents() throws Exception {
    // Use all oracle replays to ensure we encounter at least one upgrade
    List<Path> replays;
    try (var stream = Files.list(ORACLE_DIR)) {
        replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().limit(10).toList();
    }

    long totalResearch = 0;
    for (Path replay : replays) {
        for (int pid = 1; pid <= 2; pid++) {
            try {
                List<TimedIntent> intents = TrackerIntentExtractor.extract(replay, pid);
                totalResearch += intents.stream()
                    .filter(ti -> ti.intent() instanceof ResearchIntent).count();
            } catch (Exception e) { /* skip unparseable */ }
        }
    }

    assertTrue(totalResearch > 0,
        "Should have ResearchIntents from Upgrade tracker events across 10 replays");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#extractsResearchIntentsFromUpgradeEvents -q`
Expected: FAIL — `UpgradeType.fromTrackerName` does not exist

- [ ] **Step 3: Add `fromTrackerName` to UpgradeType and upgrade handling to TrackerIntentExtractor**

The tracker event field `upgradeTypeName` uses SC2 internal names. Build the mapping from the existing `UPGRADE_NAME_MAP` in `ReplayCommandExtractor` plus the Python `upgrade_mapping.py` generated by `UpgradeMappingGeneratorTest`. The tracker names match the Python keys (e.g., `"ProtossGroundWeaponsLevel1"`, `"Stimpack"`, `"TerranInfantryWeaponsLevel1"`).

```java
public static UpgradeType fromTrackerName(String trackerName) {
    return TRACKER_NAME_MAP.get(trackerName);
}

private static final Map<String, UpgradeType> TRACKER_NAME_MAP = Map.ofEntries(
    // Build this from the full UpgradeType enum — each enum constant's name
    // maps to one or more tracker event strings.
    // Use UpgradeMappingGeneratorTest output as reference.
    Map.entry("Stimpack", UpgradeType.STIMPACK),
    Map.entry("ShieldWall", UpgradeType.COMBAT_SHIELD),
    // ... all 82 entries from UpgradeType enum
);
```

Populate the map by reading the existing `quarkmind-classifier/src/upgrade_mapping.py` for the canonical tracker name → enum name mapping (generated by `UpgradeMappingGeneratorTest`).

Also add the Upgrade event handling to `TrackerIntentExtractor.extract()` — replace the placeholder comment added in Task 1:

```java
case ITrackerEvents.ID_UPGRADE -> {
    if (!(raw instanceof IUpgradeEvent up)) continue;
    if (up.getPlayerId() == null || up.getPlayerId() != playerId) continue;
    String upgradeName = String.valueOf(up.getUpgradeTypeName()).trim();
    UpgradeType ut = UpgradeType.fromTrackerName(upgradeName);
    if (ut == null) continue;

    int completionLoop = up.getLoop();
    int upgradeTime = SC2Data.upgradeTimeInLoops(ut);
    int commandLoop = Math.max(0, completionLoop - upgradeTime);

    intents.add(new TimedIntent(commandLoop,
        new ResearchIntent("tracker", ut)));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/UpgradeType.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java
git commit -m "feat: add UpgradeType.fromTrackerName for tracker→ResearchIntent conversion

Maps SC2 tracker event upgradeTypeName strings to UpgradeType enum.
Enables TrackerIntentExtractor to produce ResearchIntents.

Refs #406"
```

### Task 3: Coverage comparison test — tracker vs CmdEvent intent counts

**Files:**
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java` (add test)

**Interfaces:**
- Consumes: `TrackerIntentExtractor.extract(Path, int)`, `ReplayCommandExtractor.extract(Path, int)`

- [ ] **Step 1: Write test comparing tracker vs CmdEvent intent counts**

```java
@Test
@EnabledIf("oracleExists")
void trackerIntentsCoverMoreThanCmdEvents() throws Exception {
    Path replay = Files.list(ORACLE_DIR)
        .filter(p -> p.toString().endsWith(".SC2Replay"))
        .sorted().findFirst().orElseThrow();

    List<TimedIntent> trackerIntents = TrackerIntentExtractor.extract(replay, 1);
    List<TimedIntent> cmdIntents = ReplayCommandExtractor.extract(replay, 1).intents();

    long trackerTrainCount = trackerIntents.stream()
        .filter(ti -> ti.intent() instanceof TrainIntent).count();
    long cmdTrainCount = cmdIntents.stream()
        .filter(ti -> ti.intent() instanceof TrainIntent).count();

    System.out.printf("Tracker TrainIntents: %d, CmdEvent TrainIntents: %d (%.1f%% more)%n",
        trackerTrainCount, cmdTrainCount,
        cmdTrainCount > 0 ? (trackerTrainCount - cmdTrainCount) * 100.0 / cmdTrainCount : 100.0);

    assertTrue(trackerTrainCount >= cmdTrainCount,
        "Tracker path should produce at least as many TrainIntents as CmdEvent path");
}
```

- [ ] **Step 2: Run test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TrackerIntentExtractorIntentTest#trackerIntentsCoverMoreThanCmdEvents -q`
Expected: PASS — tracker path produces ≥ CmdEvent intents

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerIntentExtractorIntentTest.java
git commit -m "test: coverage comparison — tracker vs CmdEvent intent counts

Validates tracker path produces at least as many TrainIntents as
the CmdEvent path (expected: ~2.3x more at 43% CmdEvent capture).

Refs #406"
```

---

## Batch 2: Tracker-driven validation harness and baseline test

### Task 4: TrackerDrivenAccuracyBaselineTest

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerDrivenAccuracyBaselineTest.java`

**Interfaces:**
- Consumes: `TrackerIntentExtractor.extract(Path, int)`, `ReplayValidationHarness.run(SimulatedGame, List<TimedIntent>, int, Race)`, `ReplaySimulatedGame(Path, int)`
- Produces: Accuracy baseline report written to `docs/benchmarks/tracker-driven-accuracy-baseline.md`

This test follows the exact pattern of `EmulatedGameAccuracyBaselineTest` but substitutes `TrackerIntentExtractor.extract()` for `ReplayCommandExtractor.extract().intents()`.

- [ ] **Step 1: Write the test**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.details.Player;
import hu.scelight.sc2.rep.repproc.RepParserEngine;
import hu.sllauncher.util.repconst.RepContent;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

@Tag("report")
class TrackerDrivenAccuracyBaselineTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");
    private static final int TICKS_PER_MINUTE =
        (int) (60 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
    private static final int[] CHECKPOINTS_MIN = {1, 3, 5, 7};
    private static final int TICK_LIMIT = TICKS_PER_MINUTE * 8;
    private static final Set<UnitType> TRANSIENT_TYPES = EnumSet.of(UnitType.EGG);

    static boolean oracleExists() {
        if (!Files.isDirectory(ORACLE_DIR)) return false;
        try (var stream = Files.list(ORACLE_DIR)) {
            return stream.anyMatch(p -> p.toString().endsWith(".SC2Replay"));
        } catch (Exception e) { return false; }
    }

    @Test
    @EnabledIf("oracleExists")
    void trackerDrivenAccuracyBaseline() throws IOException {
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        // Same accumulator pattern as EmulatedGameAccuracyBaselineTest
        Map<Integer, Map<UnitType, long[]>> unitAccum = new TreeMap<>();
        Map<Integer, Map<BuildingType, long[]>> bldgAccum = new TreeMap<>();
        Map<Integer, int[]> upgradeAccum = new TreeMap<>();
        Map<Integer, double[][]> economyAccum = new TreeMap<>();
        int totalPlayerRuns = 0;

        for (int cp : CHECKPOINTS_MIN) {
            unitAccum.put(cp, new EnumMap<>(UnitType.class));
            bldgAccum.put(cp, new EnumMap<>(BuildingType.class));
            upgradeAccum.put(cp, new int[]{0, 0});
            economyAccum.put(cp, new double[13][2]);
        }

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;
            if (replay.details.getPlayerList().length < 2) continue;

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    ReplaySimulatedGame groundTruth = new ReplaySimulatedGame(replayPath, playerId);
                    List<TimedIntent> intents = TrackerIntentExtractor.extract(replayPath, playerId);
                    Race race = resolveRace(replay, playerId);

                    DivergenceReport report = ReplayValidationHarness.run(
                        groundTruth, intents, TICK_LIMIT, race);

                    // Accumulate at each checkpoint — same logic as EmulatedGameAccuracyBaselineTest
                    for (int cp : CHECKPOINTS_MIN) {
                        int tickIndex = cp * TICKS_PER_MINUTE - 1;
                        if (tickIndex >= report.ticks().size()) continue;
                        var snap = report.ticks().get(tickIndex);

                        // Units
                        for (var entry : snap.groundTruthUnitsByType().entrySet()) {
                            if (TRANSIENT_TYPES.contains(entry.getKey())) continue;
                            long[] acc = unitAccum.get(cp).computeIfAbsent(
                                entry.getKey(), k -> new long[]{0, 0});
                            acc[0] += entry.getValue();
                            int emVal = snap.emulatedUnitsByType()
                                .getOrDefault(entry.getKey(), 0);
                            acc[1] += Math.min(emVal, entry.getValue());
                        }
                        // Buildings
                        for (var entry : snap.groundTruthBuildingsByType().entrySet()) {
                            long[] acc = bldgAccum.get(cp).computeIfAbsent(
                                entry.getKey(), k -> new long[]{0, 0});
                            acc[0] += entry.getValue();
                            int emVal = snap.emulatedBuildingsByType()
                                .getOrDefault(entry.getKey(), 0);
                            acc[1] += Math.min(emVal, entry.getValue());
                        }
                        // Upgrades
                        int gtSize = snap.groundTruthUpgrades().size();
                        int matchCount = (int) snap.groundTruthUpgrades().stream()
                            .filter(snap.emulatedUpgrades()::contains).count();
                        upgradeAccum.get(cp)[0] += gtSize;
                        upgradeAccum.get(cp)[1] += matchCount;
                    }
                    totalPlayerRuns++;
                } catch (Exception e) { /* skip failed replays */ }
            }
        }

        // Render report — write to docs/benchmarks/
        StringBuilder sb = new StringBuilder();
        sb.append("# Tracker-Driven EmulatedGame Accuracy Baseline\n\n");
        sb.append(String.format("**Dataset:** Oracle (118 replays, v4.9.3)%n"));
        sb.append(String.format("**Processed:** %d player runs%n", totalPlayerRuns));
        sb.append(String.format("**Method:** Tracker-event-driven (100%% input coverage)%n%n"));

        for (int cp : CHECKPOINTS_MIN) {
            sb.append(String.format("## %d-minute checkpoint%n%n", cp));
            // Unit accuracy table
            sb.append("### Units\n\n| Type | GT | Emulated | Accuracy |\n");
            sb.append("|---|---|---|---|\n");
            long totalGt = 0, totalEm = 0;
            for (var entry : new TreeMap<>(unitAccum.get(cp)).entrySet()) {
                long gt = entry.getValue()[0];
                long em = entry.getValue()[1];
                totalGt += gt;
                totalEm += em;
                double acc = gt > 0 ? em * 100.0 / gt : 100.0;
                sb.append(String.format("| %s | %d | %d | %.1f%% |%n",
                    entry.getKey(), gt, em, acc));
            }
            double totalAcc = totalGt > 0 ? totalEm * 100.0 / totalGt : 100.0;
            sb.append(String.format("| **TOTAL** | %d | %d | **%.1f%%** |%n%n",
                totalGt, totalEm, totalAcc));

            // Upgrade accuracy
            int[] upAcc = upgradeAccum.get(cp);
            double upPct = upAcc[0] > 0 ? upAcc[1] * 100.0 / upAcc[0] : 100.0;
            sb.append(String.format("### Upgrades%nAccuracy: %.1f%% (%d/%d)%n%n",
                upPct, upAcc[1], upAcc[0]));
        }

        String report = sb.toString();
        System.out.println(report);

        Path outputPath = Path.of("docs/benchmarks/tracker-driven-accuracy-baseline.md");
        Files.createDirectories(outputPath.getParent());
        Files.writeString(outputPath, report);

        // Sanity assertion: at 5-min with 100% inputs, unit accuracy should beat CmdEvent baseline
        long gt5 = unitAccum.get(5).values().stream().mapToLong(a -> a[0]).sum();
        long em5 = unitAccum.get(5).values().stream().mapToLong(a -> a[1]).sum();
        double unitAcc5 = gt5 > 0 ? em5 * 100.0 / gt5 : 0;
        assertTrue(unitAcc5 > 50.0,
            "Tracker-driven unit accuracy at 5min should be > 50%: " + unitAcc5);
    }

    private static Race resolveRace(Replay replay, int playerId) {
        var players = replay.details.getPlayerList();
        if (players.length >= playerId) {
            var scRace = players[playerId - 1].getRace();
            if (scRace == hu.scelight.sc2.rep.model.details.Race.TERRAN) return Race.TERRAN;
            if (scRace == hu.scelight.sc2.rep.model.details.Race.ZERG) return Race.ZERG;
        }
        return Race.PROTOSS;
    }
}
```

- [ ] **Step 2: Run test to verify it compiles and passes**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
Expected: PASS — writes baseline report to `docs/benchmarks/tracker-driven-accuracy-baseline.md`

- [ ] **Step 3: Review the baseline report**

Read `docs/benchmarks/tracker-driven-accuracy-baseline.md` and compare against the CmdEvent baseline (`docs/benchmarks/emulated-game-accuracy-baseline.md`). The tracker-driven numbers should be higher since inputs are 100% complete. Document the comparison.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerDrivenAccuracyBaselineTest.java
git add docs/benchmarks/tracker-driven-accuracy-baseline.md
git commit -m "feat: add TrackerDrivenAccuracyBaselineTest — 100%% input coverage baseline

Runs the validation harness with tracker-derived intents against 118
oracle replays. Reports per-type accuracy at 1/3/5/7 min checkpoints.
Isolates EmulatedGame physics accuracy from input completeness.

Refs #406"
```

---

## Batch 3: Physics calibration (iterative)

### Task 5: Analyse tracker-driven baseline and identify physics gaps

**Files:**
- Read: `docs/benchmarks/tracker-driven-accuracy-baseline.md` (from Task 4)
- Read: `docs/benchmarks/emulated-game-accuracy-baseline.md` (existing CmdEvent baseline)

**Interfaces:**
- Consumes: Both baseline reports
- Produces: Prioritised list of physics gaps to fix (documented in issue #406 comments)

This is a judgment task, not a code task. After running the tracker-driven baseline (Task 4), compare per-type accuracy against the CmdEvent baseline. The delta reveals which inaccuracies were hiding behind missing inputs vs genuine physics bugs.

- [ ] **Step 1: Compare baselines**

Read both baseline reports. For each unit type, building type, and upgrade:
- CmdEvent accuracy (existing)
- Tracker accuracy (new)
- Delta (improvement from complete inputs)
- Remaining gap (tracker accuracy < 100% = physics error)

- [ ] **Step 2: Identify top physics gaps**

Types where tracker accuracy is < 95% represent physics bugs, not input problems. Rank by:
1. Impact (GT count × accuracy gap)
2. Tractability (is the fix in train timing, build timing, or economy model?)

- [ ] **Step 3: Post gap analysis to issue #406**

```bash
gh issue comment 406 --repo casehubio/quarkmind --body "## Tracker-Driven Baseline Analysis
[paste the comparison table and prioritised gap list]"
```

- [ ] **Step 4: Create child issues for top gaps**

Each physics fix gets its own issue (XS or S scale) — e.g., "Fix Stalker train time: tracker baseline shows 87% accuracy at 5min" or "Fix economy MAPE: mineral collection rate diverges at 3min".

### Task 6: Fix physics gaps (per-gap, iterative)

This task repeats for each physics gap identified in Task 5. Each iteration follows the same TDD cycle.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` (timing constants)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (physics logic)
- Test: Unit-specific calibration test

**For each physics gap:**

- [ ] **Step 1: Write a failing test that isolates the gap**

Example for a train time mismatch:

```java
@Test
void stalkerTrainTimeShouldMatchTracker() {
    // From tracker baseline: Stalkers appear N loops late
    int expectedTrainTime = /* value from tracker data */;
    assertEquals(expectedTrainTime, SC2Data.trainTimeInLoops(UnitType.STALKER),
        "Train time should match empirical tracker observation");
}
```

- [ ] **Step 2: Run test — verify it fails**
- [ ] **Step 3: Fix the constant or physics logic**
- [ ] **Step 4: Run test — verify it passes**
- [ ] **Step 5: Re-run TrackerDrivenAccuracyBaselineTest to confirm improvement**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
Verify: The affected type's accuracy improves.

- [ ] **Step 6: Commit with calibration source**

```bash
git commit -m "fix: calibrate [UnitType] train time from tracker ground truth

Empirical value [N] loops from [M] observations across 118 oracle replays.
Previous value [old] was [source].

Refs #406, Closes #[child-issue]"
```

---

## References

- [#406](https://github.com/casehubio/quarkmind/issues/406) — Milestone 2 issue with full spec
- [#386](https://github.com/casehubio/quarkmind/issues/386) — Milestone 1 (predecessor, closed)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerEventFeatureExtractor.java` — existing tracker event extraction (JSON output)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:55-138` — existing harness (accepts List<TimedIntent>)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayCommandExtractor.java:29-86` — existing CmdEvent intent extraction
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java` — existing CmdEvent baseline test pattern
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:280-286` — train time constants
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/mock/Sc2ReplayShared.java:55` — tracker name → UnitType mapping
- Protocol: `calibration-milestones-are-waypoints.md`
- Protocol: `sc2data-train-times-require-calibration.md`
- Protocol: `extractor-separate-from-simulated-game.md`
- Protocol: `unit-morph-emits-unitborn.md`
