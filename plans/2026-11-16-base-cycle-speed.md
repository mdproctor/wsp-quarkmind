# Base Cycle Speed — Measure-Then-Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #661 — EM base cycle speed: edge-to-edge walks 25% faster than SC2 paths
**Issue group:** #661

**Goal:** Record an 8w SC2 baseline, measure per-worker walk distances from cadence data to identify root causes of the 3-loop/leg gap, then apply targeted fixes.

**Architecture:** Two-phase. Phase 1 adds "kill" action to the playbook system, records an 8w SC2 baseline with worker cadence data, and writes a diagnostic test that decomposes each worker's cycle into mining position, deposit position, walk distance, and path deviation. Phase 2 applies fixes based on Phase 1 measurements — the specific fix depends on what the data shows.

**Tech Stack:** Java 21, JUnit 5, SC2 debug API (DebugKillUnit), existing playbook/calibration infrastructure

## Global Constraints

- No compensation constants — every fix must correspond to a measured physical value
- CDI defaults must match constants (PP-20261116-5f16ed)
- Simulate per-loop, never use formulas
- Gas rebalance after any mineral worker redistribution (PP-20261116-8ee9df)

---

## Batch 1: Playbook "kill" action + 8w baseline infrastructure

### Task 1: Add "kill" action to playbook system

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java:52-64` (add `case "kill"`)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (add `debugKillUnit`)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/PlaybookSC2Adapter.java:194-206` (add `case "kill"`)
- Create: `quarkmind-sc2/src/test/resources/playbooks/cal-mineral-baseline-8w.yaml`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookCalibrationTest.java` (add test)

**Interfaces:**
- Consumes: `EmulatedGame.applyIntent(UnitDiedIntent)`, `WorkerSimulation.onWorkerDied(String tag)`, `QuarkusSC2Transport.sendDebug(RequestDebug)`
- Produces: `EmulatedGame.debugKillUnit(String tag)` — kills a friendly unit by tag, removing from PlayerState and WorkerSimulation. `SC2DeliveryHandler.doKill(stepName, data)` — kills N workers via `debugKillUnit`. PlaybookSC2Adapter handles `"kill"` action via DebugKillUnit debug API.

- [ ] **Step 1: Create the 8w playbook YAML**

```yaml
playbook: cal-mineral-baseline-8w
schema: sc2
meta:
  race: PROTOSS
  duration: 305 ticks
  purpose: Uncontested per-worker mineral rate — 8 workers (1 per patch), no gas, no training

steps:
  - kill: 4
    at: 0 ticks

  - assert: tick-61
    at: 61 ticks
    expect:
      units:
        PROBE: { min: 8, max: 8 }

  - assert: tick-305
    at: 305 ticks
    expect:
      units:
        PROBE: { min: 8, max: 8 }
```

Write to `quarkmind-sc2/src/test/resources/playbooks/cal-mineral-baseline-8w.yaml`.

- [ ] **Step 2: Add `debugKillUnit` to EmulatedGame**

Use `ide_insert_member` to add after `debugCreateBuilding`:

```java
public void debugKillUnit(String tag) {
    Unit unit = friendly.findUnit(tag);
    if (unit == null) return;
    handleUnitDied(new UnitDiedIntent(tag, unit.type()), friendly, friendlyPhysics);
}
```

Verify `PlayerState.findUnit(String tag)` exists. If not, add it:

```java
public Unit findUnit(String tag) {
    return units().stream().filter(u -> u.tag().equals(tag)).findFirst().orElse(null);
}
```

- [ ] **Step 3: Add `doKill` to SC2DeliveryHandler**

Use `ide_insert_member` to add after `doCalldown`:

```java
private StepOutcome doKill(String stepName, Map<String, Object> data) {
    int count = Integer.parseInt((String) data.get("kill"));
    List<io.quarkmind.domain.Unit> workers = game.snapshot().units().stream()
        .filter(u -> io.quarkmind.domain.SC2Data.isWorker(u.type()))
        .limit(count)
        .toList();
    for (io.quarkmind.domain.Unit w : workers) {
        game.debugKillUnit(w.tag());
    }
    return StepOutcome.ok(stepName, Map.of("killed", workers.size()));
}
```

Use `ide_edit_member` on `execute` to add `case "kill" -> doKill(stepName, data);` to the switch.

- [ ] **Step 4: Add `"kill"` case to PlaybookSC2Adapter.onStep**

In the step processing loop (line ~128), add a special case for "kill" before `translateStep`, similar to "morph":

```java
if ("kill".equals(step.action())) {
    int count = Integer.parseInt((String) step.params().get("kill"));
    UnitType workerType = workerForRace(race);
    List<io.quarkmind.domain.Unit> workers = state.myUnits().stream()
        .filter(u -> u.type() == workerType)
        .limit(count)
        .toList();
    if (!workers.isEmpty()) {
        var debugReq = SC2APIProtocol.Debug.DebugKillUnit.newBuilder();
        workers.forEach(w -> debugReq.addTag(Long.parseLong(w.tag())));
        transport.sendDebug(SC2APIProtocol.Sc2Api.RequestDebug.newBuilder()
            .addDebug(SC2APIProtocol.Debug.DebugCommand.newBuilder()
                .setKillUnit(debugReq.build()).build())
            .build());
        fired[i] = true;
        log.infof("[Recorder] tick %d: killed %d workers via debug API", tick, workers.size());
    }
    continue;
}
```

- [ ] **Step 5: Add self-baseline test for the 8w playbook**

Use `ide_insert_member` in `DualRunCalibrationTest`:

```java
@Test
void selfBaselineCalMineralBaseline8w() {
    assertSelfBaseline("playbooks/cal-mineral-baseline-8w.yaml", Race.PROTOSS);
}
```

- [ ] **Step 6: Add calibration test for 8w playbook**

Use `ide_insert_member` in `SC2PlaybookCalibrationTest`:

```java
@Test
void mineralBaseline8w() {
    runner.execute("playbooks/cal-mineral-baseline-8w.yaml");
}
```

- [ ] **Step 7: Run tests and verify**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest#mineralBaseline8w -q`
Expected: PASS — 8w playbook runs, kills 4 workers, asserts 8 workers at checkpoints

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest#selfBaselineCalMineralBaseline8w -q`
Expected: PASS — self-baseline consistency

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: PASS — all existing dual-run tests still pass (no regression)

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/PlaybookSC2Adapter.java \
       quarkmind-sc2/src/test/resources/playbooks/cal-mineral-baseline-8w.yaml \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/DualRunCalibrationTest.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2PlaybookCalibrationTest.java
git commit -m "feat(#661): add 'kill' playbook action + 8w mineral baseline playbook

Refs #661

- Add debugKillUnit(tag) to EmulatedGame
- Add doKill to SC2DeliveryHandler for EM-side kill
- Add kill case to PlaybookSC2Adapter for SC2-side DebugKillUnit
- Create cal-mineral-baseline-8w.yaml playbook (kills 4 of 12, mines at 8w)
- Add self-baseline and calibration tests"
```

### Task 2: Record 8w SC2 baseline

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/PlaybookSC2RecorderTest.java` (add recorder)
- Create: `quarkmind-sc2/src/test/resources/calibration/cal-mineral-baseline-8w.json` (output)
- Create: `quarkmind-sc2/src/test/resources/calibration/cal-worker-cadence-8w.json` (output)

**Interfaces:**
- Consumes: `PlaybookSC2Adapter` with kill support from Task 1, `WorkerCadenceBaseline.toJson()`
- Produces: JSON calibration files used by Task 3 (dual-run test) and Task 4 (cycle decomposition)

- [ ] **Step 1: Add recorder test method**

Use `ide_insert_member` in `PlaybookSC2RecorderTest` after `recordCalWorkerCadence12w`:

```java
@Test
void recordCalMineralBaseline8w() throws Exception {
    assumeTrue(QuarkusSC2Transport.isSC2Available(),
               "StarCraft II not installed — skipping live SC2 test");
    QuarkusSC2Transport transport = new QuarkusSC2Transport();
    transport.sc2Port                = 8168;
    transport.mapName                = "TorchesAIE_v4";
    transport.difficultyStr          = "VERY_EASY";
    transport.aiRaceStr              = "RANDOM";
    transport.botRaceStr             = "PROTOSS";
    transport.connectRetryCount      = 60;
    transport.connectRetryIntervalMs = 5000;
    transport.connect();
    transport.createGame();
    transport.joinGame();

    PlaybookSC2Adapter adapter = new PlaybookSC2Adapter(
            transport, "playbooks/cal-mineral-baseline-8w.yaml");

    transport.runGameLoop(adapter);

    while (!transport.isRunning()) {Thread.sleep(50);}
    while (transport.isRunning()) {Thread.sleep(100);}

    assertFalse(adapter.workerCadenceSnapshots().isEmpty(),
                "Should have recorded worker cadence snapshots");
    assertFalse(adapter.snapshots().isEmpty(),
                "Should have recorded calibration snapshots");

    WorkerCadenceBaseline cadenceBaseline = adapter.buildWorkerCadenceBaseline("unknown");
    String cadenceJson   = cadenceBaseline.toJson();
    Path   cadenceOutput = Path.of("src/test/resources/calibration/cal-worker-cadence-8w.json");
    Files.createDirectories(cadenceOutput.getParent());
    Files.writeString(cadenceOutput, cadenceJson);
    System.out.printf("[Recorder] Worker cadence baseline written to %s (%d bytes)%n",
                      cadenceOutput, cadenceJson.length());

    CalibrationBaseline calBaseline = adapter.buildBaseline("unknown");
    String              calJson     = calBaseline.toJson();
    Path                calOutput   = Path.of("src/test/resources/calibration/cal-mineral-baseline-8w.json");
    Files.writeString(calOutput, calJson);
    System.out.printf("[Recorder] Calibration baseline written to %s (%d bytes)%n",
                      calOutput, calJson.length());

    System.out.printf("[Recorder] Both baselines from same game run — %d ticks%n",
                      adapter.snapshots().size());
}
```

- [ ] **Step 2: Run the recorder against SC2**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PlaybookSC2RecorderTest#recordCalMineralBaseline8w -Dquarkus.profile=sc2 -q`
Expected: SC2 launches, plays 305 ticks with 8 workers, writes both JSON files

Verify outputs exist:
```bash
ls -la quarkmind-sc2/src/test/resources/calibration/cal-mineral-baseline-8w.json
ls -la quarkmind-sc2/src/test/resources/calibration/cal-worker-cadence-8w.json
```

- [ ] **Step 3: Add dual-run test for 8w**

Use `ide_insert_member` in `DualRunCalibrationTest` after `dualRunCalMineralBaseline12wAgainstSC2`:

```java
@Test
void dualRunCalMineralBaseline8wAgainstSC2() {
    CalibrationBaseline sc2Baseline = loadBaseline("calibration/cal-mineral-baseline-8w.json");
    CalibrationResult   result      = calibrate("playbooks/cal-mineral-baseline-8w.yaml", sc2Baseline);
    System.out.println(result.report());
    if (!result.passed()) {
        System.out.printf("[DualRun] Diverged at tick %d: %s expected=%s actual=%s%n",
                          result.divergenceTick(), result.divergenceField(),
                          result.expected(), result.actual());
    }
}
```

- [ ] **Step 4: Run 8w dual-run to measure current gap**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest#dualRunCalMineralBaseline8wAgainstSC2 -q`
Expected: PASS (no structural divergence), econ log shows mineral gap. Record the final tick mineral values — this is the quantified 8w gap.

- [ ] **Step 5: Commit baseline files and tests**

```bash
git add quarkmind-sc2/src/test/resources/calibration/cal-mineral-baseline-8w.json \
       quarkmind-sc2/src/test/resources/calibration/cal-worker-cadence-8w.json \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/PlaybookSC2RecorderTest.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/DualRunCalibrationTest.java
git commit -m "feat(#661): record 8w SC2 baseline with worker cadence data

Refs #661

- Record 8w CalibrationBaseline and WorkerCadenceBaseline from SC2
- Add dual-run test for 8w EM vs SC2 comparison
- 8w mineral gap: EM <X> vs SC2 <Y> (+<Z>%)"
```

## Batch 2: Walk decomposition diagnostic

### Task 3: CycleDecompositionTest — measure root causes from cadence data

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/CycleDecompositionTest.java`

**Interfaces:**
- Consumes: `WorkerCadenceBaseline.fromJson()`, `CalibrationBaseline` (for patch positions), `WorkerSnapshot` (tag, position, action, carrying)
- Produces: Diagnostic report — per-patch mining radius, deposit distance, walk distance, path deviation factor

- [ ] **Step 1: Write the test class shell with baseline loading**

Create `CycleDecompositionTest.java` with `@Tag("diagnostic")`:

```java
package io.quarkmind.sc2.playbook;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.WorkerAction;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.assertFalse;

@Tag("diagnostic")
class CycleDecompositionTest {

    @Test
    void decompose8wCycles() throws IOException {
        WorkerCadenceBaseline cadence = loadCadence("calibration/cal-worker-cadence-8w.json");
        CalibrationBaseline   cal     = loadCalibration("calibration/cal-mineral-baseline-8w.json");

        assertFalse(cadence.cadence().isEmpty(), "Cadence baseline should not be empty");

        float[] basePos     = cal.basePosition();
        Point2d baseCenter  = new Point2d(basePos[0], basePos[1]);

        Map<String, List<CycleRecord>> cyclesPerWorker = extractCycles(cadence);

        System.out.println("=== 8w Cycle Decomposition ===");
        System.out.printf("%-8s | %6s | %8s | %8s | %8s | %8s | %8s | %8s%n",
            "worker", "cycles", "mineRad", "depDist", "walkTo", "walkBack", "straightE2E", "deviation");
        System.out.println("-".repeat(90));

        double totalMineRadius = 0, totalDepositDist = 0, totalDeviation = 0;
        int    totalCycles = 0;

        for (var entry : cyclesPerWorker.entrySet()) {
            String tag = entry.getKey();
            List<CycleRecord> cycles = entry.getValue();
            if (cycles.isEmpty()) continue;

            double avgMineRadius = cycles.stream().mapToDouble(c -> c.miningRadius).average().orElse(0);
            double avgDepDist    = cycles.stream().mapToDouble(c -> c.depositDistance).average().orElse(0);
            double avgWalkTo     = cycles.stream().mapToDouble(c -> c.walkToDistance).average().orElse(0);
            double avgWalkBack   = cycles.stream().mapToDouble(c -> c.walkBackDistance).average().orElse(0);
            double avgStraight   = cycles.stream().mapToDouble(c -> c.straightLineE2E).average().orElse(0);
            double avgDeviation  = avgStraight > 0 ? (avgWalkTo + avgWalkBack) / 2.0 / avgStraight : 1.0;

            System.out.printf("%-8s | %6d | %8.3f | %8.3f | %8.3f | %8.3f | %8.3f | %8.3f%n",
                tag, cycles.size(), avgMineRadius, avgDepDist, avgWalkTo, avgWalkBack, avgStraight, avgDeviation);

            totalMineRadius += avgMineRadius * cycles.size();
            totalDepositDist += avgDepDist * cycles.size();
            totalDeviation += avgDeviation * cycles.size();
            totalCycles += cycles.size();
        }

        if (totalCycles > 0) {
            System.out.println();
            System.out.printf("AGGREGATE (n=%d cycles):%n", totalCycles);
            System.out.printf("  Effective mining radius:    %.3f  (EM uses %.3f)%n",
                totalMineRadius / totalCycles, 1.0);
            System.out.printf("  Effective deposit distance: %.3f  (EM uses %.3f)%n",
                totalDepositDist / totalCycles, 2.75);
            System.out.printf("  Path deviation factor:      %.3f  (1.0 = straight line)%n",
                totalDeviation / totalCycles);
        }
    }

    record CycleRecord(
        double miningRadius,
        double depositDistance,
        double walkToDistance,
        double walkBackDistance,
        double straightLineE2E
    ) {}

    private Map<String, List<CycleRecord>> extractCycles(WorkerCadenceBaseline cadence) {
        Map<String, List<Point2d>> positions = new LinkedHashMap<>();
        Map<String, List<WorkerAction>> actions = new LinkedHashMap<>();
        Map<String, List<Boolean>> carrying = new LinkedHashMap<>();

        for (WorkerCadenceSnapshot snap : cadence.cadence()) {
            for (WorkerSnapshot ws : snap.workers()) {
                positions.computeIfAbsent(ws.tag(), k -> new ArrayList<>()).add(ws.position());
                actions.computeIfAbsent(ws.tag(), k -> new ArrayList<>()).add(ws.action());
                carrying.computeIfAbsent(ws.tag(), k -> new ArrayList<>()).add(ws.carrying());
            }
        }

        Map<String, List<CycleRecord>> result = new LinkedHashMap<>();
        for (String tag : positions.keySet()) {
            result.put(tag, decomposeSingleWorker(
                positions.get(tag), actions.get(tag), carrying.get(tag)));
        }
        return result;
    }

    private List<CycleRecord> decomposeSingleWorker(
            List<Point2d> positions, List<WorkerAction> actions, List<Boolean> carrying) {
        List<CycleRecord> cycles = new ArrayList<>();
        int n = positions.size();

        int i = 10;
        while (i < n) {
            int depositTick = findNextDeposit(i, carrying);
            if (depositTick < 0) break;

            int miningStart = findPreviousMiningStart(depositTick, actions, positions);
            if (miningStart < 0) { i = depositTick + 1; continue; }

            int walkToStart = findWalkToStart(miningStart, actions);
            if (walkToStart < 0) { i = depositTick + 1; continue; }

            int nextWalkToStart = findNextDeposit(depositTick + 1, carrying);

            Point2d minePos    = positions.get(miningStart);
            Point2d depositPos = positions.get(depositTick);

            double walkTo = sumDistances(positions, walkToStart, miningStart);
            double walkBack = sumDistances(positions, miningStart + miningDuration(miningStart, actions, positions), depositTick);

            double mineRadius = Double.NaN;
            double depositDist = Double.NaN;

            double straightE2E = minePos.distanceTo(depositPos);

            cycles.add(new CycleRecord(mineRadius, depositDist, walkTo, walkBack, straightE2E));

            i = depositTick + 1;
        }
        return cycles;
    }

    private int findNextDeposit(int from, List<Boolean> carrying) {
        for (int i = from + 1; i < carrying.size(); i++) {
            if (carrying.get(i - 1) && !carrying.get(i)) return i;
        }
        return -1;
    }

    private int findPreviousMiningStart(int depositTick, List<WorkerAction> actions, List<Point2d> positions) {
        for (int i = depositTick - 1; i >= 0; i--) {
            if (actions.get(i) == WorkerAction.GATHERING_MINERALS && !isMoving(positions, i)) {
                int start = i;
                while (start > 0 && actions.get(start - 1) == WorkerAction.GATHERING_MINERALS
                       && !isMoving(positions, start - 1)) {
                    start--;
                }
                return start;
            }
        }
        return -1;
    }

    private int findWalkToStart(int miningStart, List<WorkerAction> actions) {
        for (int i = miningStart - 1; i >= 0; i--) {
            if (actions.get(i) != WorkerAction.GATHERING_MINERALS || actions.get(i) == WorkerAction.RETURNING_MINERALS) {
                return i + 1;
            }
        }
        return -1;
    }

    private int miningDuration(int miningStart, List<WorkerAction> actions, List<Point2d> positions) {
        int end = miningStart;
        while (end < actions.size() - 1
               && actions.get(end + 1) == WorkerAction.GATHERING_MINERALS
               && !isMoving(positions, end + 1)) {
            end++;
        }
        return end - miningStart;
    }

    private boolean isMoving(List<Point2d> positions, int index) {
        if (index <= 0 || index >= positions.size()) return false;
        return positions.get(index).distanceTo(positions.get(index - 1)) > 0.1;
    }

    private double sumDistances(List<Point2d> positions, int from, int to) {
        double total = 0;
        for (int i = from + 1; i <= to && i < positions.size(); i++) {
            total += positions.get(i).distanceTo(positions.get(i - 1));
        }
        return total;
    }

    private WorkerCadenceBaseline loadCadence(String resource) throws IOException {
        try (InputStream is = getClass().getClassLoader().getResourceAsStream(resource)) {
            if (is == null) throw new IOException("Resource not found: " + resource);
            return WorkerCadenceBaseline.fromJson(new String(is.readAllBytes(), StandardCharsets.UTF_8));
        }
    }

    private CalibrationBaseline loadCalibration(String resource) throws IOException {
        try (InputStream is = getClass().getClassLoader().getResourceAsStream(resource)) {
            if (is == null) throw new IOException("Resource not found: " + resource);
            return CalibrationBaseline.fromJson(new String(is.readAllBytes(), StandardCharsets.UTF_8));
        }
    }
}
```

Note: The cycle decomposition logic above is a starting framework. The exact phase detection depends on what SC2's cadence data shows — the `WorkerAction` enum and `carrying` flag should give clear phase boundaries, but the mining position measurement needs refinement after seeing actual data (the `mineRadius` and `depositDist` calculations are left as `NaN` — populate them once patch positions from the calibration baseline are available to compute distances).

- [ ] **Step 2: Run the diagnostic (after 8w baseline is recorded)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CycleDecompositionTest -Pdiagnostic -q`
Expected: Per-worker per-patch cycle decomposition showing mining radius, deposit distance, walk distances, and path deviation factor.

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/CycleDecompositionTest.java
git commit -m "feat(#661): add CycleDecompositionTest for 8w walk analysis

Refs #661

- Decomposes per-worker mining cycles from cadence data
- Measures effective mining radius, deposit distance, walk distance
- Computes path deviation factor (actual/straight-line)"
```

## Batch 3: Apply fixes based on measurement (deferred — depends on Phase 2 data)

### Task 4: Apply root cause fixes

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` (radii or walk model)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` (if constants change)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java` (if approachDocking or distance changes)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java` (CDI defaults)
- Modify: `docs/sc2-physics-constants.md` (update §3, §14, §15 with measurements)

**Interfaces:**
- Consumes: CycleDecompositionTest output (Phase 2 measurements)
- Produces: Corrected physics constants / walk model

**This task's steps depend on Phase 2 measurement results.** Three scenarios:

**Scenario A — MINERAL_PATCH_RADIUS is the primary cause:**
- [ ] Change `MINERAL_PATCH_RADIUS` from 1.0 to measured value
- [ ] Run `DualRunCalibrationTest` (all baselines) — verify 8w gap closes, 12w/16w stay within tolerance
- [ ] Update CDI default in `WorkerPhysicsConfigProducer` if configurable

**Scenario B — Path deviation is the primary cause:**
- [ ] Add `effectiveWalkFactor` to `MineralPatch`, computed in `computeApproachDocking()` from mineral line geometry
- [ ] Modify `miningTarget()` or walk distance to use effective factor
- [ ] Run all dual-run tests

**Scenario C — Combination:**
- [ ] Fix MINERAL_PATCH_RADIUS partially
- [ ] Add modest per-patch walk factor for the residual
- [ ] Run all dual-run tests

**Common steps (all scenarios):**
- [ ] Run full existing test suite: `mvn test -pl quarkmind-sc2 -q`
- [ ] Run report suite: `mvn test -pl quarkmind-sc2 -Preport -q` (regression check)
- [ ] Update `docs/sc2-physics-constants.md`:
  - §3: Update MINERAL_PATCH_RADIUS or add new constant with measurement source
  - §14: Update mining cycle breakdown with corrected walk times
  - §15: Revise "Worker pathing around mineral line" impact from "small" to measured value
- [ ] Update CLAUDE.md §SC2 dual-run calibration results table with new 8w row
- [ ] Commit:

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ \
       docs/sc2-physics-constants.md \
       CLAUDE.md
git commit -m "fix(#661): correct base cycle speed — <specific fix description>

Closes #661

- <what was measured>
- <what was changed>
- 8w gap: <before>% → <after>%
- 12w: <value>%, 16w: <value>% (within tolerance)"
```

## References

- [2026-11-16-base-cycle-speed-design.md] — design spec this plan implements
- [WorkerSimulation.java:16-55] — edge-to-edge model, radii
- [MineralPatch.java:59-64] — computeApproachDocking
- [SimulatedWorker.java:44-51] — MAX_SPEED, ARRIVAL_THRESHOLD, physics constants
- [SC2DeliveryHandler.java:52-64] — action dispatch switch
- [PlaybookSC2Adapter.java:119-158] — SC2 step processing, morph/kill special cases
- [PlaybookSC2RecorderTest.java:104-150] — recorder pattern (cadence + calibration)
- [sc2-physics-constants.md §3, §14, §15] — mining constants, cycle breakdown, gaps
- [PP-20261116-5f16ed] — CDI defaults must match constants
- [GitHub #661] — EM base cycle speed: edge-to-edge walks 25% faster than SC2 paths
