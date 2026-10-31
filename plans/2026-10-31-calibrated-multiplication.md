# Calibrated Multiplication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #405 — P2.5-3e: Calibrated multiplication — reduce SCV/Overlord over-count without CmdEvent gap regression
**Issue group:** #405

**Goal:** Replace uniform building-count multiplication in `StrippedReplayFeatureExtractor` with per-type calibration factors, then validate whether this improves classifier accuracy for the vs_zerg matchup.

**Architecture:** The extractor gains an optional `Map<UnitType, Double>` of calibration factors injected via a setter. When factors are present, they replace building-count multiplication. A new `@Tag("report")` test derives factors from oracle data and writes a JSON resource file. Validation runs the full reconstitution → training → cross-version pipeline.

**Tech Stack:** Java 21, JUnit 5, Jackson (JSON), Python (classifier pipeline)

## Global Constraints

- All changes in `StrippedReplayFeatureExtractor` — no changes to `TrackerIntentExtractor` or `EmulatedGame`
- Morph multiplication (`morphMultiplier()`) is not changed — it uses selection size, not building count
- TECHLAB_UNITS bypass remains unchanged (those types always get repeatCount=1)
- Existing tests must continue passing without calibration factors loaded (fallback to building-count logic)
- Factor file is optional — extractor works identically to current behaviour when no factors are set

---

## Batch 1: Factor injection into extractor

### Task 1: Add calibration factor support to StrippedReplayFeatureExtractor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorCalibrationTest.java` (new — plain JUnit, not `@QuarkusTest`)

**Interfaces:**
- Produces: `setCalibrationFactors(Map<UnitType, Double>)` — setter on `StrippedReplayFeatureExtractor`
- Produces: `getCalibrationFactors()` — getter returning current factors (nullable)

- [ ] **Step 1: Write failing test — factor-based multiplication replaces building count**

Create `StrippedReplayFeatureExtractorCalibrationTest.java`:

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class StrippedReplayFeatureExtractorCalibrationTest {

    @Test
    void calibrationFactorOverridesMultiplication() {
        var extractor = new StrippedReplayFeatureExtractor();
        extractor.setCalibrationFactors(Map.of(UnitType.MARINE, 2.0));

        // processTrainForTest uses repeatCount=1 (no building context)
        // With calibration factor 2.0, should produce 2 units
        var events = extractor.processTrainForTest(
            new io.quarkmind.sc2.TrainIntent(UnitType.MARINE),
            1000L, 1, 100, 0L);

        long unitBornCount = events.stream()
            .filter(e -> "UnitBorn".equals(e.get("_event")))
            .count();
        assertEquals(2, unitBornCount, "factor 2.0 should produce 2 UnitBorn events");
    }

    @Test
    void noFactorFallsBackToBuildingCount() {
        var extractor = new StrippedReplayFeatureExtractor();
        // No calibration factors set — should use existing building-count logic
        assertNull(extractor.getCalibrationFactors());
    }

    @Test
    void factorRoundsCorrectly() {
        var extractor = new StrippedReplayFeatureExtractor();

        // 1.44 rounds to 1
        extractor.setCalibrationFactors(Map.of(UnitType.SCV, 1.44));
        var events1 = extractor.processTrainForTest(
            new io.quarkmind.sc2.TrainIntent(UnitType.SCV),
            1000L, 1, 100, 0L);
        long count1 = events1.stream()
            .filter(e -> "UnitBorn".equals(e.get("_event")))
            .count();
        assertEquals(1, count1, "1.44 rounds to 1");

        // 1.6 rounds to 2
        extractor.setCalibrationFactors(Map.of(UnitType.SCV, 1.6));
        var events2 = extractor.processTrainForTest(
            new io.quarkmind.sc2.TrainIntent(UnitType.SCV),
            1000L, 1, 200, 0L);
        long count2 = events2.stream()
            .filter(e -> "UnitBorn".equals(e.get("_event")))
            .count();
        assertEquals(2, count2, "1.6 rounds to 2");
    }

    @Test
    void techlabUnitsIgnoreCalibrationFactors() {
        var extractor = new StrippedReplayFeatureExtractor();
        extractor.setCalibrationFactors(Map.of(UnitType.SIEGE_TANK, 3.0));

        var events = extractor.processTrainForTest(
            new io.quarkmind.sc2.TrainIntent(UnitType.SIEGE_TANK),
            1000L, 1, 100, 0L);
        long count = events.stream()
            .filter(e -> "UnitBorn".equals(e.get("_event")))
            .count();
        assertEquals(1, count, "TECHLAB_UNITS should always produce 1 regardless of factor");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorCalibrationTest -q`
Expected: Compilation error — `setCalibrationFactors` does not exist

- [ ] **Step 3: Add calibration factor field and setter to StrippedReplayFeatureExtractor**

Add to `StrippedReplayFeatureExtractor.java` after the `MAX_MULTIPLICATION` field (line 143):

```java
private Map<UnitType, Double> calibrationFactors;

public void setCalibrationFactors(Map<UnitType, Double> factors) {
    this.calibrationFactors = factors;
}

public Map<UnitType, Double> getCalibrationFactors() {
    return calibrationFactors;
}
```

- [ ] **Step 4: Modify the multiplication logic in `extract()` to use calibration factors**

In `extract()` method, replace lines 210-216 with:

```java
int repeatCount = 1;
if (!TECHLAB_UNITS.contains(train.unitType())) {
    if (calibrationFactors != null
            && calibrationFactors.containsKey(train.unitType())) {
        repeatCount = Math.max(1,
            (int) Math.round(calibrationFactors.get(train.unitType())));
    } else if (abilLink != null) {
        int capKey = (abilLink == ABIL_HATCHERY_QUEEN
                || abilLink == ABIL_LAIR_QUEEN) ? ABIL_LARVA : abilLink;
        int buildingCount = state.productionBuildingCounts
                .getOrDefault(capKey, 0);
        if (buildingCount > 0) {
            repeatCount = Math.min(buildingCount, MAX_MULTIPLICATION);
        }
    }
}
```

Note: the TECHLAB_UNITS check moves to wrap both branches, ensuring techlab units always get repeatCount=1 regardless of calibration factors.

- [ ] **Step 5: Also apply factor in processTrainForTest path**

The `processTrainForTest` method (line 722) calls `handleTrain` directly without going through the multiplication logic. For the test to work, we need to apply the factor there too. Modify `processTrainForTest`:

```java
List<Map<String, Object>> processTrainForTest(TrainIntent train, long commandLoop,
                                              int playerId, int startTag,
                                              long warpGateCompletionLoop) {
    List<SyntheticEvent> events = new ArrayList<>();
    var state = new PlayerState();
    state.warpGateCompletionLoop = warpGateCompletionLoop;
    int repeatCount = 1;
    if (!TECHLAB_UNITS.contains(train.unitType())
            && calibrationFactors != null
            && calibrationFactors.containsKey(train.unitType())) {
        repeatCount = Math.max(1,
            (int) Math.round(calibrationFactors.get(train.unitType())));
    }
    int tagCounter = startTag;
    for (int r = 0; r < repeatCount; r++) {
        tagCounter = handleTrain(train, commandLoop, playerId, state, events, tagCounter);
    }
    return events.stream().map(SyntheticEvent::data).toList();
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayFeatureExtractorCalibrationTest -q`
Expected: All 4 tests PASS

- [ ] **Step 7: Run existing extractor tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrippedReplayValidationTest,ReplayCommandExtractorTest,TerranReplayCommandExtractorTest -q`
Expected: All PASS (no calibration factors loaded = fallback to building-count logic)

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractor.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrippedReplayFeatureExtractorCalibrationTest.java
git commit -m "feat(#405): add calibration factor support to StrippedReplayFeatureExtractor

Per-type calibration factors replace uniform building-count multiplication
when set. TECHLAB_UNITS bypass preserved. Fallback to existing logic when
no factors loaded.

Refs #405"
```

### Task 2: Add calibration factor pass-through in ReplayFeatureExtractor

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayFeatureExtractor.java`

**Interfaces:**
- Consumes: `StrippedReplayFeatureExtractor.setCalibrationFactors(Map<UnitType, Double>)`
- Produces: `ReplayFeatureExtractor.setCalibrationFactors(Map<UnitType, Double>)` — pass-through to stripped extractor

- [ ] **Step 1: Add pass-through setter to ReplayFeatureExtractor**

```java
public void setCalibrationFactors(Map<UnitType, Double> factors) {
    strippedExtractor.setCalibrationFactors(factors);
}
```

Add the `UnitType` import: `import io.quarkmind.domain.UnitType;`

- [ ] **Step 2: Run existing tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReconstitutionExportTest -q`
Expected: Compile check passes (no functional change — no factors set)

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayFeatureExtractor.java
git commit -m "feat(#405): pass-through calibration factors in ReplayFeatureExtractor

Refs #405"
```

## Batch 2: Factor derivation and JSON resource

### Task 3: Create CalibrationFactorDerivationTest to derive and write factors

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CalibrationFactorDerivationTest.java`
- Create: `quarkmind-sc2/src/main/resources/calibration/multiplication-factors.json` (output of test run)

**Interfaces:**
- Consumes: `StrippedReplayFeatureExtractor.extract(Path)` (with no factors = uncalibrated)
- Consumes: Oracle replays at `../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored`
- Consumes: Stripped replays at `../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3/replays`
- Produces: `multiplication-factors.json` written to `src/main/resources/calibration/`

- [ ] **Step 1: Write the CalibrationFactorDerivationTest**

```java
package io.quarkmind.sc2.replay;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelightapi.sc2.rep.model.trackerevents.IBaseUnitEvent;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Collectors;

@Tag("report")
class CalibrationFactorDerivationTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");
    private static final Path STRIPPED_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3/replays");
    private static final Path OUTPUT_PATH = Path.of(
        "src/main/resources/calibration/multiplication-factors.json");

    private static final int LOOPS_PER_MINUTE = 1344;
    private static final int CHECKPOINT_LOOP = 5 * LOOPS_PER_MINUTE;

    private static final Set<String> COMBAT_SPAWN_UNITS = Set.of(
        "Broodling", "BroodlingEscort", "Locust", "InfestorTerran",
        "Changeling", "ChangelingMarine", "ChangelingMarineShield",
        "ChangelingZealot", "ChangelingZergling", "ChangelingZerglingWings",
        "ParasiticBombDummy", "KD8Charge", "AutoTurret", "PointDefenseDrone");

    private static final Set<String> AUTO_SPAWN_UNITS = Set.of("Larva", "Interceptor");

    static boolean datasetsExist() {
        return Files.isDirectory(ORACLE_DIR) && Files.isDirectory(STRIPPED_DIR);
    }

    @Test
    @EnabledIf("datasetsExist")
    void deriveCalibrationFactors() throws Exception {
        Map<String, Long> oracleTotals = new TreeMap<>();
        Map<String, Long> uncalibratedTotals = new TreeMap<>();
        int replayCount = 0;

        var extractor = new StrippedReplayFeatureExtractor();
        // No calibration factors — this gives uncalibrated counts

        List<Path> oracleReplays;
        try (var stream = Files.list(ORACLE_DIR)) {
            oracleReplays = stream
                .filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        for (Path oracleReplay : oracleReplays) {
            Path stripped = STRIPPED_DIR.resolve(oracleReplay.getFileName());
            if (!Files.exists(stripped)) continue;

            // Oracle counts from tracker events
            Replay oracleRep = RepParserEngine.parseReplay(oracleReplay,
                EnumSet.of(RepContent.TRACKER_EVENTS));
            if (oracleRep == null || oracleRep.trackerEvents == null) continue;

            Map<String, Long> oracleCounts = new TreeMap<>();
            for (var evt : oracleRep.trackerEvents.getEvents()) {
                if (evt.loop > CHECKPOINT_LOOP) break;
                if (evt instanceof IBaseUnitEvent ue
                        && "UnitBornEvent".equals(evt.getClass().getSimpleName())) {
                    String name = ue.getUnit().unitTypeName.toString();
                    if (COMBAT_SPAWN_UNITS.contains(name)) continue;
                    if (AUTO_SPAWN_UNITS.contains(name)) continue;
                    oracleCounts.merge(name, 1L, Long::sum);
                }
            }

            // Uncalibrated counts from stripped extractor
            Map<String, Object> javaJson;
            try {
                javaJson = extractor.extract(stripped);
            } catch (Exception e) {
                continue;
            }

            Map<String, Long> javaCounts = new TreeMap<>();
            @SuppressWarnings("unchecked")
            var events = (List<Map<String, Object>>) javaJson.get("events");
            if (events != null) {
                for (var event : events) {
                    if (!"UnitBorn".equals(event.get("_event"))) continue;
                    Object loopObj = event.get("_gameloop");
                    if (loopObj instanceof Number n && n.longValue() > CHECKPOINT_LOOP) continue;
                    String name = (String) event.get("m_unitTypeName");
                    if (name == null) continue;
                    if (COMBAT_SPAWN_UNITS.contains(name)) continue;
                    if (AUTO_SPAWN_UNITS.contains(name)) continue;
                    javaCounts.merge(name, 1L, Long::sum);
                }
            }

            oracleCounts.forEach((k, v) -> oracleTotals.merge(k, v, Long::sum));
            javaCounts.forEach((k, v) -> uncalibratedTotals.merge(k, v, Long::sum));
            replayCount++;
        }

        // Derive factors
        Map<String, Double> factors = new TreeMap<>();
        for (var entry : oracleTotals.entrySet()) {
            String type = entry.getKey();
            long oracle = entry.getValue();
            long uncalibrated = uncalibratedTotals.getOrDefault(type, 0L);
            if (uncalibrated == 0 || oracle == 0) continue;
            double factor = (double) oracle / uncalibrated;
            factors.put(type, Math.round(factor * 100.0) / 100.0);
        }

        // Print report
        System.out.printf("%nCalibration factors derived from %d replays at 5-min:%n%n", replayCount);
        System.out.printf("%-25s %8s %12s %8s%n", "Type", "Oracle", "Uncalibrated", "Factor");
        System.out.println("-".repeat(60));
        for (var entry : factors.entrySet()) {
            System.out.printf("%-25s %8d %12d %8.2f%n",
                entry.getKey(),
                oracleTotals.getOrDefault(entry.getKey(), 0L),
                uncalibratedTotals.getOrDefault(entry.getKey(), 0L),
                entry.getValue());
        }

        // Write JSON
        Files.createDirectories(OUTPUT_PATH.getParent());
        var output = new LinkedHashMap<String, Object>();
        output.put("version", "oracle-v4.9.3-" + replayCount);
        output.put("dataset", "oracle");
        output.put("replay_count", replayCount);
        output.put("checkpoint_minutes", 5);
        output.put("factors", factors);

        var mapper = new ObjectMapper().enable(SerializationFeature.INDENT_OUTPUT);
        mapper.writeValue(OUTPUT_PATH.toFile(), output);
        System.out.printf("%nWrote factors to %s%n", OUTPUT_PATH);
    }
}
```

- [ ] **Step 2: Run the derivation test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=CalibrationFactorDerivationTest -q`
Expected: Factors printed and `multiplication-factors.json` written

- [ ] **Step 3: Review the generated factors**

Read `quarkmind-sc2/src/main/resources/calibration/multiplication-factors.json` and verify:
- SCV factor should be < current building-count multiplication (reducing over-count)
- Overlord factor should be < current (reducing over-count)
- No factors should be negative or extremely large (>10)

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/CalibrationFactorDerivationTest.java
git add quarkmind-sc2/src/main/resources/calibration/multiplication-factors.json
git commit -m "feat(#405): derive per-type calibration factors from oracle dataset

CalibrationFactorDerivationTest (@Tag report) runs StrippedReplayFeatureExtractor
without multiplication against 118 oracle replays, compares against tracker
ground truth at 5-min, and writes per-type factors to multiplication-factors.json.

Refs #405"
```

## Batch 3: Validation pipeline

### Task 4: Run oracle accuracy baseline with calibrated factors

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/OracleAccuracyBaselineTest.java` — add a calibrated variant

**Interfaces:**
- Consumes: `StrippedReplayFeatureExtractor.setCalibrationFactors(Map<UnitType, Double>)`
- Consumes: `multiplication-factors.json`

- [ ] **Step 1: Add a calibrated test method to OracleAccuracyBaselineTest**

Add a JSON loader method and a second `@Test` method that runs the baseline with calibrated factors:

```java
private static Map<UnitType, Double> loadCalibrationFactors() throws Exception {
    var mapper = new com.fasterxml.jackson.databind.ObjectMapper();
    Path factorPath = Path.of("src/main/resources/calibration/multiplication-factors.json");
    if (!Files.exists(factorPath)) return null;

    @SuppressWarnings("unchecked")
    var json = mapper.readValue(factorPath.toFile(), Map.class);
    @SuppressWarnings("unchecked")
    var rawFactors = (Map<String, Number>) json.get("factors");
    if (rawFactors == null) return null;

    Map<UnitType, Double> factors = new java.util.EnumMap<>(UnitType.class);
    for (var entry : rawFactors.entrySet()) {
        try {
            UnitType ut = UnitType.valueOf(entry.getKey().toUpperCase());
            factors.put(ut, entry.getValue().doubleValue());
        } catch (IllegalArgumentException ignored) {
            // Unit name in JSON doesn't match UnitType enum — skip
        }
    }
    return factors;
}

@Test
@EnabledIf("oracleAndOriginalsExist")
void oracleAccuracyCalibratedBaseline() throws Exception {
    Map<UnitType, Double> factors = loadCalibrationFactors();
    org.junit.jupiter.api.Assumptions.assumeTrue(factors != null,
        "No calibration factors file found");

    var extractor = new StrippedReplayFeatureExtractor();
    extractor.setCalibrationFactors(factors);
    var stats = new BaselineStats();

    List<Path> oracleReplays;
    try (var stream = Files.list(ORACLE_DIR)) {
        oracleReplays = stream
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().toList();
    }

    for (Path oracleReplay : oracleReplays) {
        Path stripped = STRIPPED_DIR.resolve(oracleReplay.getFileName());
        if (!Files.exists(stripped)) continue;
        processReplay(extractor, oracleReplay, stripped, stats);
    }

    System.out.println("\n=== CALIBRATED BASELINE ===");
    printReport(stats);
}
```

- [ ] **Step 2: Run the calibrated baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=OracleAccuracyBaselineTest#oracleAccuracyCalibratedBaseline -q`
Expected: Report printed showing per-type divergences with calibrated factors. Compare SCV/Overlord over-counts against the uncalibrated baseline.

- [ ] **Step 3: Record the comparison in the issue**

Capture the calibrated vs uncalibrated T1 accuracy, SCV diff, and Overlord diff. Post as a comment on #405.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/OracleAccuracyBaselineTest.java
git commit -m "feat(#405): add calibrated oracle accuracy baseline variant

Loads multiplication-factors.json and runs the oracle accuracy baseline
with calibrated factors for comparison.

Refs #405"
```

### Task 5: Re-extract reconstituted features and retrain classifier

**Files:**
- No code changes — this is a pipeline execution task
- Output: Updated reconstituted JSON in `quarkmind-classifier/data/reconstituted/`
- Output: Retrained ONNX models in `quarkmind-classifier/models/`

**Interfaces:**
- Consumes: `ReplayFeatureExtractor.setCalibrationFactors(Map<UnitType, Double>)`
- Consumes: `multiplication-factors.json`

- [ ] **Step 1: Wire calibration factors into ReconstitutionExportTest**

Modify `ReconstitutionExportTest.java` to load and apply calibration factors:

After `var extractor = new ReplayFeatureExtractor();` (line 53), add:

```java
Path factorPath = Path.of("src/main/resources/calibration/multiplication-factors.json");
if (Files.exists(factorPath)) {
    var mapper = new ObjectMapper();
    @SuppressWarnings("unchecked")
    var json = mapper.readValue(factorPath.toFile(), Map.class);
    @SuppressWarnings("unchecked")
    var rawFactors = (Map<String, Number>) json.get("factors");
    if (rawFactors != null) {
        Map<io.quarkmind.domain.UnitType, Double> factors =
            new java.util.EnumMap<>(io.quarkmind.domain.UnitType.class);
        for (var entry : rawFactors.entrySet()) {
            try {
                var ut = io.quarkmind.domain.UnitType.valueOf(
                    entry.getKey().toUpperCase());
                factors.put(ut, entry.getValue().doubleValue());
            } catch (IllegalArgumentException ignored) {}
        }
        extractor.setCalibrationFactors(factors);
        System.out.println("Loaded " + factors.size() + " calibration factors");
    }
}
```

- [ ] **Step 2: Re-extract reconstituted data**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=ReconstitutionExportTest -q`
Expected: All datasets re-extracted with calibrated factors. Console prints "Loaded N calibration factors".

- [ ] **Step 3: Retrain classifier**

```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.run_pipeline --data combined
```
Expected: Training completes, models written to `models/`

- [ ] **Step 4: Run cross-version validation**

```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.cross_version_validation --write-report
```
Expected: `docs/benchmarks/cross-version-generalisation.md` updated with new accuracy numbers.

- [ ] **Step 5: Compare results against decision gate targets**

Read the updated `cross-version-generalisation.md` and compare:

| Metric | Baseline | Target | Actual |
|--------|----------|--------|--------|
| vs_zerg fine 3min | 47% | >55% | ? |
| vs_zerg fine 5min | 61% | >70% | ? |
| vs_terran fine 5min | 91% | ≥90% | ? |
| vs_protoss fine 5min | 87% | ≥85% | ? |
| Coarse accuracy 5min | 92% | ≥90% | ? |

- [ ] **Step 6: Decision gate**

If targets met: proceed to Task 6 (finalise).
If not met: revert calibration factor loading in `ReconstitutionExportTest`, re-extract without factors, retrain, and close #405 as won't-fix with assessment data.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReconstitutionExportTest.java
git commit -m "feat(#405): wire calibration factors into reconstitution export

ReconstitutionExportTest loads multiplication-factors.json when present
and applies calibrated multiplication to stripped replay extraction.

Refs #405"
```

### Task 6: Finalise — update baselines and run regression suite

**Files:**
- Modify: `docs/benchmarks/oracle-accuracy-baseline.md` (updated by test output)
- Modify: `docs/benchmarks/cross-version-generalisation.md` (updated by validation)

**Interfaces:**
- Consumes: Updated classifier models and reconstituted data from Task 5

- [ ] **Step 1: Run DivergenceRegressionTest to confirm no threshold violations**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest -q`
Expected: All assertions pass

- [ ] **Step 2: Update oracle accuracy baseline**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=OracleAccuracyBaselineTest#oracleAccuracyBaseline -q`
This regenerates `docs/benchmarks/oracle-accuracy-baseline.md` with the calibrated extractor as the new default.

- [ ] **Step 3: Commit all baseline updates**

```bash
git add docs/benchmarks/
git add quarkmind-classifier/docs/benchmarks/ 2>/dev/null || true
git commit -m "docs(#405): update accuracy baselines after calibrated multiplication

Oracle accuracy and cross-version generalisation baselines updated to
reflect per-type calibrated multiplication factors.

Refs #405"
```

- [ ] **Step 4: Run full test suite to confirm no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass

- [ ] **Step 5: Final commit if any remaining changes**

```bash
git status
# If any uncommitted changes, commit them
```

## References

- [2026-10-31-calibrated-multiplication-design.md] — design spec this plan implements
- [StrippedReplayFeatureExtractor.java:210-226] — current multiplication logic
- [StrippedReplayFeatureExtractor.java:143] — MAX_MULTIPLICATION constant
- [ReplayFeatureExtractor.java:13] — stripped extractor instantiation
- [OracleAccuracyBaselineTest.java:100-121] — oracle accuracy test structure
- [ReconstitutionExportTest.java:53] — extractor instantiation in reconstitution
- [docs/benchmarks/oracle-accuracy-baseline.md] — current per-type divergences
- [docs/benchmarks/cross-version-generalisation.md] — current classifier accuracy
- [docs/protocols/calibration-milestones-are-waypoints.md] — waypoint assessment protocol
- [GitHub #405] — focal issue
- [GitHub #404] — pool-based approach failure (context)
- [GitHub #481] — parent epic
