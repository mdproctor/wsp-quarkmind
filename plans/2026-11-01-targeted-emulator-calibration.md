# Targeted Emulator Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #484 — Targeted emulator calibration — use SC2 oracle comparison on strategic replay subsets
**Issue group:** #484

**Goal:** Diagnose and fix Terran emulator production divergences (Marine, Hellion, WidowMine, SiegeTank, Medivac, Reaper) to raise tracker-driven unit accuracy from 99.8% to ≥99.9% at 5min, and expand the complete replay dataset via SC2EGSet ingestion.

**Architecture:** Two-phase diagnosis — a ReplayDivergenceScanner identifies which replays have the worst per-unit-type divergences, then a ReplayDivergenceTracer provides per-tick causal timelines via an IntentExecutionListener on EmulatedGame. Physics fixes are validated against TrackerDrivenAccuracyBaselineTest. SC2EGSet ingestion uses the existing Python pipeline.

**Tech Stack:** Java 21, JUnit 5, EmulatedGame, ReplayValidationHarness, TrackerIntentExtractor, Python sc2egset_extractor.py

## Global Constraints

- All new tests are plain JUnit — no `@QuarkusTest` (boot cost is significant)
- Oracle replays are in `../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored`
- Tests guarded by `@EnabledIf("oracleExists")` — CI skips when dataset absent
- Follow `calibration-milestones-are-waypoints` protocol — assess at waypoint, don't declare victory
- SC2Data timing constants calibrated from replays, not formulae

---

## Batch 1: Diagnostic infrastructure

### Task 1: IntentExecutionListener on EmulatedGame

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/IntentExecutionListener.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:362-472` (handleTrain method)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/IntentExecutionListenerTest.java`

**Interfaces:**
- Consumes: `EmulatedGame.handleTrain()`, `TrainIntent`, `Building`
- Produces: `IntentExecutionListener` interface, `EmulatedGame.setIntentExecutionListener(listener)` method. Later tasks register this listener to capture train outcomes.

- [ ] **Step 1: Write the listener interface**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.TrainIntent;

public interface IntentExecutionListener {
    void onTrainOutcome(int tick, TrainIntent intent, String resolvedBuildingTag,
                        boolean success, String rejectionReason);
}
```

- [ ] **Step 2: Write failing test — listener fires on successful train**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.TrainIntent;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class IntentExecutionListenerTest {

    record TrainEvent(int tick, TrainIntent intent, String buildingTag,
                      boolean success, String reason) {}

    @Test
    void listenerFiresOnSuccessfulTrain() {
        EmulatedGame game = new EmulatedGame();
        game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
        game.reset();

        List<TrainEvent> events = new ArrayList<>();
        game.setIntentExecutionListener((tick, intent, bldgTag, success, reason) ->
            events.add(new TrainEvent(tick, intent, bldgTag, success, reason)));

        // Tick once so buildings are initialized
        game.tick();

        // Train a probe from the starting Nexus
        String nexusTag = game.snapshot().myBuildings().stream()
            .filter(b -> b.type() == BuildingType.NEXUS)
            .findFirst().map(Building::tag).orElseThrow();
        game.applyIntent(new TrainIntent(nexusTag, UnitType.PROBE));

        assertFalse(events.isEmpty(), "Listener should have fired");
        assertTrue(events.get(0).success());
        assertEquals(UnitType.PROBE, events.get(0).intent().unitType());
        assertEquals(nexusTag, events.get(0).buildingTag());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=IntentExecutionListenerTest#listenerFiresOnSuccessfulTrain -q`
Expected: FAIL — `setIntentExecutionListener` does not exist

- [ ] **Step 4: Add listener field and setter to EmulatedGame**

Add to `EmulatedGame` (near the `trackerTrainDiagnostics` field, around line 480):

```java
private IntentExecutionListener intentExecutionListener;

public void setIntentExecutionListener(IntentExecutionListener listener) {
    this.intentExecutionListener = listener;
}
```

- [ ] **Step 5: Fire listener in handleTrain — success path**

In `handleTrain(TrainIntent, PlayerState, PhysicsState, long)`, after `recordTrainDiagnostic(t, 1)` (line 456, the "trained" slot), add:

```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, resolvedTag, true, null);
}
```

- [ ] **Step 6: Fire listener in handleTrain — rejection paths**

At each rejection point in `handleTrain`, fire the listener with the reason:

1. After `recordTrainDiagnostic(t, 2)` (building_not_found, line 390):
```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, null, false, "building_not_found");
}
```

2. After the `canTrainFrom` rejection (line 401):
```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, resolvedTag, false, "wrong_building_type");
}
```

3. After `recordTrainDiagnostic(t, 3)` (production_blocked, line 416):
```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, resolvedTag, false, "production_blocked");
}
```

4. After `recordTrainDiagnostic(t, 4)` (insufficient_resources, line 429):
```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, resolvedTag, false, "insufficient_resources");
}
```

5. After `recordTrainDiagnostic(t, 5)` (queue_full, line 447):
```java
if (intentExecutionListener != null) {
    intentExecutionListener.onTrainOutcome(
        (int) gameFrame, t, resolvedTag, false, "queue_full");
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=IntentExecutionListenerTest#listenerFiresOnSuccessfulTrain -q`
Expected: PASS

- [ ] **Step 8: Write failing test — listener fires on rejection**

```java
@Test
void listenerFiresOnBuildingNotFound() {
    EmulatedGame game = new EmulatedGame();
    game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
    game.reset();
    game.tick();

    List<TrainEvent> events = new ArrayList<>();
    game.setIntentExecutionListener((tick, intent, bldgTag, success, reason) ->
        events.add(new TrainEvent(tick, intent, bldgTag, success, reason)));

    // Try to train from a nonexistent building tag
    game.applyIntent(new TrainIntent("r-nonexistent", UnitType.MARINE));

    assertFalse(events.isEmpty(), "Listener should have fired");
    assertFalse(events.get(0).success());
    assertEquals("building_not_found", events.get(0).reason());
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=IntentExecutionListenerTest -q`
Expected: PASS (both tests)

- [ ] **Step 10: Write failing test — no listener means no error**

```java
@Test
void noListenerDoesNotThrow() {
    EmulatedGame game = new EmulatedGame();
    game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
    game.reset();
    game.tick();

    // No listener set — should not throw
    String nexusTag = game.snapshot().myBuildings().stream()
        .filter(b -> b.type() == BuildingType.NEXUS)
        .findFirst().map(Building::tag).orElseThrow();
    assertDoesNotThrow(() -> game.applyIntent(new TrainIntent(nexusTag, UnitType.PROBE)));
}
```

- [ ] **Step 11: Run full test to verify all pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=IntentExecutionListenerTest -q`
Expected: PASS (all 3 tests)

- [ ] **Step 12: Run existing tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest,DualRunCalibrationTest,SC2PlaybookCalibrationTest -q`
Expected: PASS — listener is null-safe, no existing behavior changed

- [ ] **Step 13: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/IntentExecutionListener.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/IntentExecutionListenerTest.java
git commit -m "feat(#484): add IntentExecutionListener to EmulatedGame for diagnostic tracing

Refs #484"
```

### Task 2: ReplayDivergenceScanner

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayDivergenceScannerTest.java`

**Interfaces:**
- Consumes: `ReplayValidationHarness.run(Path, int, int)`, `TrackerIntentExtractor.extract()`, `TrackerIntentExtractor.resolvePlayerRace()`, `DivergenceReport.TickSnapshot` (all existing)
- Produces: stdout report — per-replay per-unit-type delta table for Terran players, sorted by absolute delta descending. Used to identify target replays for Task 3.

- [ ] **Step 1: Write the scanner test**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.s2prot.RepContent;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.emulated.RaceModelFactory;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;

import static org.junit.jupiter.api.Assertions.assertTrue;

@Tag("report")
class ReplayDivergenceScannerTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");
    private static final int TICKS_PER_MINUTE =
        (int) (60 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
    private static final int TICK_LIMIT = TICKS_PER_MINUTE * 8;
    private static final int[] CHECKPOINTS_MIN = {1, 3, 5, 7};
    private static final Set<UnitType> TARGET_TYPES = EnumSet.of(
        UnitType.MARINE, UnitType.HELLION, UnitType.WIDOW_MINE,
        UnitType.SIEGE_TANK, UnitType.MEDIVAC, UnitType.REAPER,
        UnitType.SCV, UnitType.MARAUDER, UnitType.VIKING,
        UnitType.BANSHEE, UnitType.CYCLONE, UnitType.LIBERATOR);

    static boolean oracleExists() {
        if (!Files.isDirectory(ORACLE_DIR)) return false;
        try (var stream = Files.list(ORACLE_DIR)) {
            return stream.anyMatch(p -> p.toString().endsWith(".SC2Replay"));
        } catch (Exception e) { return false; }
    }

    record DivergenceEntry(String replayHash, int playerId, UnitType unitType,
                           int checkpoint, int gt, int em, int delta) {}

    @Test
    @EnabledIf("oracleExists")
    void scanTerranDivergences() throws IOException {
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        List<DivergenceEntry> entries = new ArrayList<>();
        int processed = 0;

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;
            if (replay.details.getPlayerList().length < 2) continue;

            String hash = replayPath.getFileName().toString().replace(".SC2Replay", "");

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    Race race = TrackerIntentExtractor.resolvePlayerRace(replay, playerId);
                    if (race != Race.TERRAN) continue;

                    ReplaySimulatedGame groundTruth = new ReplaySimulatedGame(replayPath, playerId);
                    List<TimedIntent> intents = TrackerIntentExtractor.extract(replayPath, playerId);
                    DivergenceReport report = ReplayValidationHarness.run(
                        groundTruth, intents, TICK_LIMIT, race);

                    for (int cp : CHECKPOINTS_MIN) {
                        int tickIndex = cp * TICKS_PER_MINUTE - 1;
                        if (tickIndex >= report.ticks().size()) continue;
                        var snap = report.ticks().get(tickIndex);

                        for (UnitType ut : TARGET_TYPES) {
                            int gtCount = snap.groundTruthUnitsByType().getOrDefault(ut, 0);
                            int emCount = snap.emulatedUnitsByType().getOrDefault(ut, 0);
                            int delta = emCount - gtCount;
                            if (delta != 0) {
                                entries.add(new DivergenceEntry(
                                    hash, playerId, ut, cp, gtCount, emCount, delta));
                            }
                        }
                    }
                    processed++;
                } catch (Exception e) {
                    System.err.printf("[Scanner] Error on %s player %d: %s%n",
                        replayPath.getFileName(), playerId, e.getMessage());
                }
            }
        }

        // Sort by absolute delta descending, then by checkpoint
        entries.sort(Comparator.comparingInt((DivergenceEntry e) -> Math.abs(e.delta()))
            .reversed().thenComparingInt(DivergenceEntry::checkpoint));

        // Print report
        System.out.println("\n=== Terran Unit Divergence Report ===");
        System.out.printf("Processed: %d Terran player runs across %d replays%n%n",
            processed, replays.size());
        System.out.printf("%-20s | Player | %-12s | CP | %5s | %5s | %5s%n",
            "Replay", "UnitType", "GT", "Em", "Delta");
        System.out.println("-".repeat(78));
        for (var e : entries) {
            System.out.printf("%-20s | %6d | %-12s | %2d | %5d | %5d | %+5d%n",
                e.replayHash().substring(0, Math.min(20, e.replayHash().length())),
                e.playerId(), e.unitType(), e.checkpoint(),
                e.gt(), e.em(), e.delta());
        }

        // Summary by unit type at 5min
        System.out.println("\n=== 5-min Summary by Unit Type ===");
        Map<UnitType, int[]> summary = new EnumMap<>(UnitType.class);
        for (var e : entries) {
            if (e.checkpoint() != 5) continue;
            int[] acc = summary.computeIfAbsent(e.unitType(), k -> new int[]{0, 0, 0});
            acc[0] += e.gt();
            acc[1] += e.em();
            acc[2] += Math.abs(e.delta());
        }
        for (var e : summary.entrySet()) {
            System.out.printf("%-12s: GT=%d Em=%d TotalAbsDelta=%d%n",
                e.getKey(), e.getValue()[0], e.getValue()[1], e.getValue()[2]);
        }

        assertTrue(processed > 0, "Should have processed at least one Terran player run");
    }
}
```

- [ ] **Step 2: Run to verify it compiles and executes**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=ReplayDivergenceScannerTest -q`
Expected: PASS — prints per-replay delta table and 5min summary

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayDivergenceScannerTest.java
git commit -m "feat(#484): add ReplayDivergenceScanner — per-replay Terran unit divergence report

Refs #484"
```

### Task 3: ReplayDivergenceTracer

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayDivergenceTracerTest.java`

**Interfaces:**
- Consumes: `IntentExecutionListener` (from Task 1), `ReplayValidationHarness.run(groundTruth, intents, tickLimit, emulated, race)`, `EmulatedGame.setIntentExecutionListener()`, `DivergenceReport.TickSnapshot`
- Produces: stdout causal timeline — per-tick intent outcomes correlated with GT vs Em count divergence for a specific unit type

- [ ] **Step 1: Write the tracer test**

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.s2prot.RepContent;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.emulated.IntentExecutionListener;
import io.quarkmind.sc2.emulated.RaceModelFactory;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.intent.TrainIntent;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;

import static org.junit.jupiter.api.Assertions.assertNotNull;

@Tag("diagnostic")
class ReplayDivergenceTracerTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");
    private static final int TICKS_PER_MINUTE =
        (int) (60 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
    private static final int TICK_LIMIT = TICKS_PER_MINUTE * 8;

    // Configure these from scanner output before running
    private static final UnitType TRACE_UNIT_TYPE = UnitType.MARINE;

    static boolean oracleExists() {
        if (!Files.isDirectory(ORACLE_DIR)) return false;
        try (var stream = Files.list(ORACLE_DIR)) {
            return stream.anyMatch(p -> p.toString().endsWith(".SC2Replay"));
        } catch (Exception e) { return false; }
    }

    record TrainEvent(int tick, TrainIntent intent, String buildingTag,
                      boolean success, String reason) {}

    @Test
    @EnabledIf("oracleExists")
    void traceWorstDivergence() throws Exception {
        // Find the replay with the worst divergence for TRACE_UNIT_TYPE at 5min
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        Path worstReplay = null;
        int worstPlayerId = -1;
        int worstDelta = 0;

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    Race race = TrackerIntentExtractor.resolvePlayerRace(replay, playerId);
                    if (race != Race.TERRAN) continue;

                    ReplaySimulatedGame gt = new ReplaySimulatedGame(replayPath, playerId);
                    List<TimedIntent> intents = TrackerIntentExtractor.extract(replayPath, playerId);
                    DivergenceReport report = ReplayValidationHarness.run(gt, intents, TICK_LIMIT, race);

                    int tickIndex = 5 * TICKS_PER_MINUTE - 1;
                    if (tickIndex >= report.ticks().size()) continue;
                    var snap = report.ticks().get(tickIndex);

                    int gtCount = snap.groundTruthUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
                    int emCount = snap.emulatedUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
                    int delta = Math.abs(emCount - gtCount);
                    if (delta > worstDelta) {
                        worstDelta = delta;
                        worstReplay = replayPath;
                        worstPlayerId = playerId;
                    }
                } catch (Exception e) { /* skip */ }
            }
        }

        assertNotNull(worstReplay, "Should find at least one Terran replay with divergence");
        System.out.printf("%n=== Tracing %s (player %d) for %s (delta=%d at 5min) ===%n%n",
            worstReplay.getFileName(), worstPlayerId, TRACE_UNIT_TYPE, worstDelta);

        // Re-run with listener
        ReplaySimulatedGame groundTruth = new ReplaySimulatedGame(worstReplay, worstPlayerId);
        List<TimedIntent> intents = TrackerIntentExtractor.extract(worstReplay, worstPlayerId);
        Replay replay = RepParserEngine.parseReplay(worstReplay, EnumSet.of(RepContent.DETAILS));
        Race race = TrackerIntentExtractor.resolvePlayerRace(replay, worstPlayerId);

        EmulatedGame emulated = new EmulatedGame();
        emulated.setPlayerRaceModel(RaceModelFactory.forRace(race));

        List<TrainEvent> trainEvents = new ArrayList<>();
        emulated.setIntentExecutionListener((tick, intent, bldgTag, success, reason) -> {
            if (intent.unitType() == TRACE_UNIT_TYPE) {
                trainEvents.add(new TrainEvent(tick, intent, bldgTag, success, reason));
            }
        });

        DivergenceReport report = ReplayValidationHarness.run(
            groundTruth, intents, TICK_LIMIT, emulated, race);

        // Print causal timeline
        System.out.println("--- Intent Execution Timeline ---");
        System.out.printf("%-6s | %-8s | %-20s | %-7s | %s%n",
            "Tick", "Outcome", "Building", "Success", "Reason");
        System.out.println("-".repeat(70));
        for (var e : trainEvents) {
            System.out.printf("%-6d | %-8s | %-20s | %-7s | %s%n",
                e.tick(), e.intent().unitType(),
                e.buildingTag() != null ? e.buildingTag() : "N/A",
                e.success() ? "OK" : "FAIL",
                e.reason() != null ? e.reason() : "");
        }

        // Print per-minute GT vs Em counts
        System.out.println("\n--- Per-Minute Count Comparison ---");
        for (int cp : new int[]{1, 3, 5, 7}) {
            int tickIndex = cp * TICKS_PER_MINUTE - 1;
            if (tickIndex >= report.ticks().size()) continue;
            var snap = report.ticks().get(tickIndex);
            int gtCount = snap.groundTruthUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
            int emCount = snap.emulatedUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
            System.out.printf("%dmin: GT=%d Em=%d delta=%+d%n", cp, gtCount, emCount, emCount - gtCount);
        }

        // Print production building state at 5min
        System.out.println("\n--- Production Building State at 5min ---");
        int tickIndex5 = 5 * TICKS_PER_MINUTE - 1;
        if (tickIndex5 < report.ticks().size()) {
            var snap5 = report.ticks().get(tickIndex5);
            BuildingType producerType = SC2Data.trainedBy(TRACE_UNIT_TYPE);
            System.out.printf("Producer: %s%n", producerType);
            for (var entry : snap5.emulatedBuildingsByType().entrySet()) {
                if (SC2Data.canTrainFrom(entry.getKey(), producerType) || entry.getKey() == producerType) {
                    int gtBldg = snap5.groundTruthBuildingsByType().getOrDefault(entry.getKey(), 0);
                    System.out.printf("  %s: GT=%d Em=%d%n", entry.getKey(), gtBldg, entry.getValue());
                }
            }
        }

        // Print train diagnostics
        System.out.println("\n--- Train Diagnostics ---");
        var diag = emulated.trackerTrainDiagnostics();
        var unitDiag = diag.get(TRACE_UNIT_TYPE);
        if (unitDiag != null) {
            System.out.printf("received=%d trained=%d bldg404=%d prodBlocked=%d insuffRes=%d queueFull=%d deferOverflow=%d%n",
                unitDiag[0], unitDiag[1], unitDiag[2], unitDiag[3], unitDiag[4], unitDiag[5], unitDiag[6]);
        }

        // Print first divergence tick
        System.out.println("\n--- First Divergence ---");
        for (int tick = 0; tick < report.ticks().size(); tick++) {
            var snap = report.ticks().get(tick);
            int gtCount = snap.groundTruthUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
            int emCount = snap.emulatedUnitsByType().getOrDefault(TRACE_UNIT_TYPE, 0);
            if (gtCount != emCount) {
                System.out.printf("First divergence at tick %d (~%.1f min): GT=%d Em=%d%n",
                    tick, tick / (double) TICKS_PER_MINUTE, gtCount, emCount);
                // Print surrounding intent events
                for (var e : trainEvents) {
                    if (Math.abs(e.tick() - tick) <= 5) {
                        System.out.printf("  tick %d: %s → %s (bldg=%s, reason=%s)%n",
                            e.tick(), e.intent().unitType(),
                            e.success() ? "OK" : "FAIL",
                            e.buildingTag(), e.reason());
                    }
                }
                break;
            }
        }
    }
}
```

- [ ] **Step 2: Run to verify it compiles and executes**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=ReplayDivergenceTracerTest -q`
Expected: PASS — prints causal timeline, per-minute comparison, building state, and first divergence

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayDivergenceTracerTest.java
git commit -m "feat(#484): add ReplayDivergenceTracer — per-tick causal timeline for unit divergences

Refs #484"
```

## Batch 2: Diagnose and fix Terran divergences

### Task 4: Run scanner, diagnose root causes, fix physics

This task is investigative — the exact fixes depend on what the scanner and tracer reveal. The process is documented here; the specific code changes emerge from diagnosis.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (physics fixes — exact locations TBD by tracer output)
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayDivergenceTracerTest.java` (update TRACE_UNIT_TYPE for each unit)

**Interfaces:**
- Consumes: Scanner output (Task 2), Tracer output (Task 3)
- Produces: Fixed EmulatedGame physics, updated accuracy baseline

- [ ] **Step 1: Run the scanner to identify worst replays**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=ReplayDivergenceScannerTest -q`
Read the output. Note the replay hash, player ID, and delta for each unit type.

- [ ] **Step 2: Trace Marine divergence**

Edit `ReplayDivergenceTracerTest`: set `TRACE_UNIT_TYPE = UnitType.MARINE`.
Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=ReplayDivergenceTracerTest -q`
Read the causal timeline. Identify the root cause from the first divergence tick and surrounding intent events.

- [ ] **Step 3: Fix the Marine root cause**

Based on the tracer output, fix `EmulatedGame.handleTrain()` or the relevant physics code. Common patterns:
- Building resolution failure → fix `shortestQueue()` or reactor detection in `handleTrain()`
- Timing issue → fix train time constants via `SC2TrainTimeCalibrationTest`
- Queue management → fix queue limit or deferred intent handling

- [ ] **Step 4: Verify Marine fix**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
Check Marine accuracy at 5min has improved. Check no other unit type regressed.

- [ ] **Step 5: Repeat steps 2-4 for Hellion**

Set `TRACE_UNIT_TYPE = UnitType.HELLION`. Run tracer. Diagnose. Fix. Verify.

- [ ] **Step 6: Repeat steps 2-4 for WidowMine**

Set `TRACE_UNIT_TYPE = UnitType.WIDOW_MINE`. Run tracer. Diagnose. Fix. Verify.

- [ ] **Step 7: Repeat for remaining Terran types (SiegeTank, Medivac, Reaper)**

Each has a delta of 1 — may be a single replay edge case. If the root cause is the same as Marine/Hellion/WidowMine, the earlier fixes may have already resolved them. Re-run scanner to check.

- [ ] **Step 8: Verify ≥99.9% at 5min**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
Check 5min unit accuracy ≥99.9%.

Per `calibration-milestones-are-waypoints`: if we reach 99.9% but the tracer reveals deeper issues, assess and replan.

- [ ] **Step 9: Run regression tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest,SC2PlaybookCalibrationTest,EmulatedGameTest -q`
Expected: PASS — playbook calibration and existing physics unbroken.

- [ ] **Step 10: Update accuracy baseline doc**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=TrackerDrivenAccuracyBaselineTest -q`
The test writes `docs/benchmarks/tracker-driven-accuracy-baseline.md` automatically. Verify the new numbers.

- [ ] **Step 11: Commit all fixes**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add docs/benchmarks/tracker-driven-accuracy-baseline.md
git commit -m "fix(#484): fix Terran emulator production divergences — [describe specific fixes]

[Describe root causes found and fixes applied]

Refs #484"
```

## Batch 3: SC2EGSet dataset expansion

### Task 5: Download and ingest SC2EGSet

**Files:**
- Modify: `quarkmind-classifier/data/sc2egset/` (new tournament ZIPs)
- Modify: `replays/replay-index.md` (document new datasets)

**Interfaces:**
- Consumes: SC2EGSet from Zenodo (`zenodo.org/records/14963484`), existing `sc2egset_extractor.py`
- Produces: Expanded classifier training data in `quarkmind-classifier/data/sc2egset/`

- [ ] **Step 1: Check available disk space**

Run: `df -h /Users/mdproctor/claude/casehub/quarkmind`
Verify ≥10 GB free (SC2EGSet ZIPs ~2-5 GB, extracted features larger).

- [ ] **Step 2: Check what SC2EGSet data already exists**

Run: `ls quarkmind-classifier/data/sc2egset/ | wc -l`
The exploration found 174 entries already present. Determine which tournaments are missing from the full Zenodo archive.

- [ ] **Step 3: Download missing SC2EGSet ZIPs from Zenodo**

Browse `zenodo.org/records/14963484` for the file list. Download any tournament ZIPs not already present.

Note: this is a manual download step — Zenodo may require browser download for large files. Use `curl` or `wget` if direct links are available.

- [ ] **Step 4: Verify extraction works on new data**

```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -c "
from src.sc2egset_extractor import extract_from_zip
import sys
# Test on one new ZIP
data = list(extract_from_zip(sys.argv[1]))
print(f'Extracted {len(data)} replays')
" data/sc2egset/<new-tournament>.zip
```

Expected: prints replay count, no errors.

- [ ] **Step 5: Run full extraction pipeline on new data**

Use `prepare_replay_pack.py` or `sc2egset_extractor.py` to process all new ZIPs. Follow existing conventions in `quarkmind-classifier/` for output format and location.

- [ ] **Step 6: Update replay-index.md**

Add new datasets to `replays/replay-index.md` with source, format, count, and date.

- [ ] **Step 7: Commit**

```bash
git add replays/replay-index.md
git commit -m "feat(#484): expand SC2EGSet dataset — [N] new tournament replays ingested

Refs #484"
```

Note: large data files in `quarkmind-classifier/data/sc2egset/` are not committed to git — only the index doc.

## Batch 4: Documentation and acceptance

### Task 6: Document the divergence → fix → validate cycle

**Files:**
- Create: `docs/guides/emulator-calibration-process.md`

**Interfaces:**
- Consumes: lessons learned from Tasks 1-4
- Produces: repeatable process documentation (acceptance criterion)

- [ ] **Step 1: Write the process doc**

Document:
1. How to run ReplayDivergenceScannerTest to identify divergent unit types and replays
2. How to configure and run ReplayDivergenceTracerTest for per-tick diagnosis
3. How to read the causal timeline output
4. How to fix EmulatedGame physics based on the diagnosis
5. How to validate fixes with TrackerDrivenAccuracyBaselineTest and regression tests
6. When to apply the calibration-milestones-are-waypoints protocol

- [ ] **Step 2: Update CLAUDE.md test listings**

Add `ReplayDivergenceScannerTest` to the report tests list and `ReplayDivergenceTracerTest` to the diagnostic tests list in CLAUDE.md.

- [ ] **Step 3: Commit**

```bash
git add docs/guides/emulator-calibration-process.md CLAUDE.md
git commit -m "docs(#484): document emulator calibration divergence-fix-validate cycle

Refs #484"
```

## References

- [2026-11-01-targeted-emulator-calibration-design.md] — design spec this plan implements
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:362-472] — handleTrain method (listener fire points)
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:76-140] — runInternal tick loop
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerDrivenAccuracyBaselineTest.java] — aggregate accuracy baseline pattern
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/HellionDeficitDiagnosticTest.java] — per-replay diagnostic pattern
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java:13-26] — TickSnapshot record
- [docs/protocols/calibration-milestones-are-waypoints.md] — calibration milestone discipline
- [quarkmind-classifier/src/sc2egset_extractor.py] — SC2EGSet extraction pipeline
- [GitHub #484] — focal issue
- [GitHub #481] — parent epic
- [GitHub #405] — prior work establishing emulator-fix approach
