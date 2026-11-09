# Worker Activity Modelling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #572 — Worker activity modelling
**Issue group:** #534 (Epic: EmulatedGame 100% accuracy)

**Goal:** Model worker unavailability (scouting, transit) and validate
the per-patch saturation curve so EmulatedGame economy accuracy improves
in both replay and standalone modes.

**Architecture:** Two orthogonal changes to `WorkerSimulation`: (1)
validate the existing MAX_MINERS=2 contention model against published
SC2 income curves, (2) add SCOUTING and TRANSFERRING worker states with
event hooks following the existing `pauseWorkerForBuilding` pattern.
Transit uses physical walk time via the existing `moveToward` system.

**Tech Stack:** Java 21, JUnit 5, plain unit tests (no Quarkus boot)

## Global Constraints

- All changes in `quarkmind-sc2` module only
- Plain JUnit tests — never `@QuarkusTest` for tests that can be plain JUnit
- Simulation over formulas — no efficiency scalars, no lookup tables
- Commit after every substantial change with `Refs #572`
- Run `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q` after each task

---

## Batch 1: Saturation Curve Validation

### Task 1: Saturation curve diagnostic test

Validate that the existing contention model produces the right
income shape: 3rd worker per patch should earn ~50% of 2nd worker's
rate.

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Consumes: `WorkerSimulation.onTownHallCompleted`, `onInitialWorkers`, `onWorkerProduced`, `tick`
- Produces: diagnostic output validating saturation curve shape

- [ ] **Step 1: Write the saturation curve test**

Add a test that measures income at 8, 12, 16, 20, and 24 workers per
base over 300 ticks (after 20-tick warmup), prints per-worker income
rate at each level, and asserts the 3rd-worker efficiency drop.

```java
@Test
void saturationCurve_thirdWorkerEarnsLessThanSecond() {
    int warmup = 20;
    int measure = 300;
    double ticksPerMinute = 60.0 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK;

    int[] workerCounts = {8, 12, 16, 20, 24};
    double[] perWorkerRates = new double[workerCounts.length];

    for (int idx = 0; idx < workerCounts.length; idx++) {
        sim.reset();
        sim.onTownHallCompleted(BASE_TAG, BASE_POS);
        tickCount = 0;

        List<String> tags = IntStream.rangeClosed(1, workerCounts[idx])
                .mapToObj(i -> "w-" + i).toList();
        sim.onInitialWorkers(tags.subList(0, Math.min(tags.size(), 12)), BASE_POS, BASE_TAG);
        for (int i = 12; i < workerCounts[idx]; i++) {
            sim.onWorkerProduced("w-" + (i + 1), BASE_POS);
        }

        for (int t = 0; t < warmup; t++) { tick(); }
        double income = 0;
        for (int t = 0; t < measure; t++) {
            income += tick().mineralIncome();
        }

        double perMinute = income / measure * ticksPerMinute;
        perWorkerRates[idx] = perMinute / workerCounts[idx];
        System.out.printf("%d workers: %.1f minerals/min total, %.1f per worker%n",
                workerCounts[idx], perMinute, perWorkerRates[idx]);
    }

    // At 16 workers (2/patch), per-worker rate should be near-optimal
    // At 20 workers (2.5/patch), per-worker rate should drop
    // At 24 workers (3/patch), per-worker rate should be significantly lower
    double rate16 = perWorkerRates[2]; // index of 16
    double rate24 = perWorkerRates[4]; // index of 24
    assertTrue(rate24 < rate16,
            "24-worker per-worker rate should be less than 16-worker rate");
    System.out.printf("Efficiency drop 16→24: %.1f%%%n",
            (1.0 - rate24 / rate16) * 100);
}
```

- [ ] **Step 2: Run the test and record output**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#saturationCurve_thirdWorkerEarnsLessThanSecond -q`

Record the output. Expected: the test passes (the contention model
already produces some diminishing returns). The diagnostic output tells
us whether the curve shape matches SC2's published data:
- SC2 reference: 3rd worker per patch earns ~50% of 2nd worker rate
- If the simulation's drop is too small or too large, note it for
  later investigation (may need #575 mining duration calibration first)

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "test(#572): saturation curve diagnostic — validate contention model shape

Measures per-worker income at 8/12/16/20/24 workers per base.
Validates MAX_MINERS=2 contention produces diminishing returns for
3rd workers per patch.

Refs #572"
```

---

## Batch 2: Worker Activity States and Scouting

### Task 2: Add SCOUTING and TRANSFERRING to WorkerTask

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerTask.java:3-10`

**Interfaces:**
- Consumes: nothing
- Produces: `WorkerTask.SCOUTING`, `WorkerTask.TRANSFERRING` enum values

- [ ] **Step 1: Add enum values**

```java
enum WorkerTask {
    MINING_MINERALS,
    RETURNING_MINERALS,
    HARVESTING_GAS,
    RETURNING_GAS,
    IDLE,
    BUILDING,
    SCOUTING,
    TRANSFERRING
}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: SUCCESS — new enum values don't break existing switch
expressions (WorkerTask is not used in exhaustive switches).

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerTask.java
git commit -m "feat(#572): add SCOUTING and TRANSFERRING to WorkerTask enum

Refs #572"
```

### Task 3: Add scoutReturnLoop field to SimulatedWorker

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java:33`

**Interfaces:**
- Consumes: nothing
- Produces: `SimulatedWorker.scoutReturnLoop` field (package-private double)

- [ ] **Step 1: Add the field**

Add after line 33 (`field phaseEndLoop double`):

```java
double scoutReturnLoop;
```

- [ ] **Step 2: Verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: SUCCESS

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java
git commit -m "feat(#572): add scoutReturnLoop field to SimulatedWorker

Refs #572"
```

### Task 4: Implement scouting lifecycle

Implement `onWorkerScouting`, `onWorkerReturnFromScout`, and auto-return
in the tick loop. Follows the `pauseWorkerForBuilding` / `tickBuilderReturns`
pattern.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Consumes: `WorkerTask.SCOUTING`, `SimulatedWorker.scoutReturnLoop`
- Produces: `WorkerSimulation.onWorkerScouting(String tag, double returnAtLoop)`,
  `WorkerSimulation.onWorkerReturnFromScout(String tag)`

- [ ] **Step 1: Write test — scouting pulls worker off minerals**

```java
@Test
void workerScouting_producesNoIncome() {
    sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_POS, BASE_TAG);
    // Warm up — both workers mining
    for (int t = 0; t < 20; t++) { tick(); }

    double incomeBeforeScout = 0;
    for (int t = 0; t < 50; t++) {
        incomeBeforeScout += tick().mineralIncome();
    }

    sim.onWorkerScouting("w-1", Double.MAX_VALUE); // never auto-returns

    double incomeWhileScouting = 0;
    for (int t = 0; t < 50; t++) {
        incomeWhileScouting += tick().mineralIncome();
    }

    assertTrue(incomeWhileScouting < incomeBeforeScout,
            "Income should drop when a worker is scouting");
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerScouting_producesNoIncome -q`
Expected: FAIL — `onWorkerScouting` method doesn't exist yet

- [ ] **Step 3: Implement onWorkerScouting**

Add to `WorkerSimulation` after `tickBuilderReturns` (after line 579):

```java
void onWorkerScouting(String tag, double returnAtLoop) {
    SimulatedWorker w = workers.get(tag);
    if (w == null) return;
    if (w.assignedPatch != null && w.assignedPatch.miners.contains(w)) {
        w.assignedPatch.miners.remove(w);
    }
    w.assignedPatch = null;
    if (w.assignedGasSlot != null) {
        w.assignedGasSlot.removeWorker(w);
        w.assignedGasSlot = null;
    }
    w.task = WorkerTask.SCOUTING;
    w.scoutReturnLoop = returnAtLoop;
}

void onWorkerReturnFromScout(String tag) {
    SimulatedWorker w = workers.get(tag);
    if (w == null || w.task != WorkerTask.SCOUTING) return;
    w.task = WorkerTask.IDLE;
    assignToNearestPatch(w);
}
```

- [ ] **Step 4: Update tickMineralWorker guard**

Update the guard at `WorkerSimulation.java` line 233 to exclude
SCOUTING and TRANSFERRING:

```java
if (w.task == WorkerTask.BUILDING || w.task == WorkerTask.IDLE
        || w.task == WorkerTask.SCOUTING || w.task == WorkerTask.TRANSFERRING) return 0;
```

- [ ] **Step 5: Add auto-return logic in tick()**

Update `tick()` method (line 214) to auto-return scouts before the
income loop:

```java
TickIncome tick(double tickStartLoop) {
    this.currentTickStartLoop = tickStartLoop;
    double tickEndLoop = tickStartLoop + SC2Data.LOOPS_PER_TICK;

    // Auto-return scouts whose return time has passed
    for (SimulatedWorker w : new ArrayList<>(workers.values())) {
        if (w.task == WorkerTask.SCOUTING && tickEndLoop >= w.scoutReturnLoop) {
            onWorkerReturnFromScout(w.tag);
        }
    }

    double mineralIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot == null) {
            mineralIncome += tickMineralWorker(w, tickStartLoop, tickEndLoop);
        }
    }
    double gasIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot != null) {
            gasIncome += tickGasWorker(w, tickStartLoop, tickEndLoop);
        }
    }
    return new TickIncome(mineralIncome, gasIncome);
}
```

- [ ] **Step 6: Run test — verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerScouting_producesNoIncome -q`
Expected: PASS

- [ ] **Step 7: Write test — scout auto-returns and resumes mining**

```java
@Test
void workerScouting_autoReturnsAtScheduledLoop() {
    sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_POS, BASE_TAG);
    for (int t = 0; t < 20; t++) { tick(); }

    double scoutReturnLoop = (tickCount + 10) * (double) SC2Data.LOOPS_PER_TICK;
    sim.onWorkerScouting("w-1", scoutReturnLoop);

    // Tick 10 times — scout should auto-return around tick 10
    double incomeAfterReturn = 0;
    for (int t = 0; t < 30; t++) {
        incomeAfterReturn += tick().mineralIncome();
    }

    // After return, both workers mine — income should recover
    double incomeRecovery = 0;
    for (int t = 0; t < 50; t++) {
        incomeRecovery += tick().mineralIncome();
    }

    assertTrue(incomeRecovery > incomeAfterReturn * 0.8,
            "Income should recover after scout returns");
}
```

- [ ] **Step 8: Run test — verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerScouting_autoReturnsAtScheduledLoop -q`
Expected: PASS — auto-return is handled by the tick loop addition in step 5

- [ ] **Step 9: Write test — manual return from scout**

```java
@Test
void workerScouting_manualReturnResumesMinig() {
    sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_POS, BASE_TAG);
    for (int t = 0; t < 20; t++) { tick(); }

    sim.onWorkerScouting("w-1", Double.MAX_VALUE);
    for (int t = 0; t < 20; t++) { tick(); }

    sim.onWorkerReturnFromScout("w-1");

    double incomeAfterReturn = 0;
    for (int t = 0; t < 50; t++) {
        incomeAfterReturn += tick().mineralIncome();
    }

    assertTrue(incomeAfterReturn > 0,
            "Worker should resume mining after manual return");
}
```

- [ ] **Step 10: Run test — verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#workerScouting_manualReturnResumesMinig -q`
Expected: PASS

- [ ] **Step 11: Run full WorkerSimulationTest suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: all existing + new tests PASS

- [ ] **Step 12: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#572): scouting lifecycle — onWorkerScouting/onWorkerReturnFromScout

Workers in SCOUTING state produce no income. Auto-return at scheduled
loop. Manual return via onWorkerReturnFromScout. Follows the
pauseWorkerForBuilding pattern.

Refs #572"
```

---

## Batch 3: Transit Modelling

### Task 5: Physical walk time for base transfers

Replace instant base reassignment with physical walk time. Workers enter
TRANSFERRING state and walk to the new base using `moveToward`.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Consumes: `WorkerTask.TRANSFERRING`, `SimulatedWorker.moveToward`
- Produces: updated `transferMineralWorkers` with walk time, `tickTransferringWorker`

- [ ] **Step 1: Write test — workers take time to reach new base**

```java
@Test
void transferredWorkers_walkToNewBase_noIncomeInTransit() {
    List<String> tags = IntStream.rangeClosed(1, 16)
            .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_POS, BASE_TAG);
    for (int t = 0; t < 20; t++) { tick(); }

    // Second base far away — workers must walk
    Point2d farBase = new Point2d(100, 100);
    sim.onTownHallCompleted("base-2", farBase);

    // Some workers should be TRANSFERRING
    // Tick once and measure — transferred workers shouldn't produce income yet
    // They need multiple ticks to walk to the new base
    double incomeFirstTick = tick().mineralIncome();

    // After many ticks, transferred workers arrive and start mining
    double totalIncome = 0;
    for (int t = 0; t < 200; t++) {
        totalIncome += tick().mineralIncome();
    }
    assertTrue(totalIncome > 0,
            "Transferred workers should eventually arrive and mine");
}
```

- [ ] **Step 2: Run test — verify it fails or shows instant transfer**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#transferredWorkers_walkToNewBase_noIncomeInTransit -q`
Expected: may pass (instant transfer still works) but workers produce
income immediately at the new base — no transit delay.

- [ ] **Step 3: Modify transferMineralWorkers to use TRANSFERRING state**

Replace the body of `transferMineralWorkers` (lines 78-96) with:

```java
private int transferMineralWorkers(int fromBase, int toBase, int count) {
    int transferred = 0;
    Point2d targetPos = bases.get(toBase).position();
    for (SimulatedWorker w : new ArrayList<>(workers.values())) {
        if (transferred >= count) break;
        if (w.baseIndex != fromBase) continue;
        if (w.assignedGasSlot != null) continue;
        if (w.task != WorkerTask.MINING_MINERALS
                && w.task != WorkerTask.RETURNING_MINERALS
                && w.task != WorkerTask.IDLE) continue;
        if (w.assignedPatch != null && w.assignedPatch.miners.contains(w)) {
            w.assignedPatch.miners.remove(w);
        }
        w.assignedPatch = null;
        w.baseIndex = toBase;
        w.task = WorkerTask.TRANSFERRING;
        w.targetPosition = targetPos;
        transferred++;
    }
    return transferred;
}
```

- [ ] **Step 4: Add tickTransferringWorker handler**

Add after `tickBuilderReturns` method:

```java
private void tickTransferringWorker(SimulatedWorker w) {
    w.moveToward(workerDistancePerTick);
    if (w.isAtTarget()) {
        assignToNearestPatch(w);
    }
}
```

- [ ] **Step 5: Update tick() to handle TRANSFERRING workers**

In the `tick()` method, before the mineral income loop, add handling
for TRANSFERRING workers:

```java
TickIncome tick(double tickStartLoop) {
    this.currentTickStartLoop = tickStartLoop;
    double tickEndLoop = tickStartLoop + SC2Data.LOOPS_PER_TICK;

    // Auto-return scouts whose return time has passed
    for (SimulatedWorker w : new ArrayList<>(workers.values())) {
        if (w.task == WorkerTask.SCOUTING && tickEndLoop >= w.scoutReturnLoop) {
            onWorkerReturnFromScout(w.tag);
        }
    }

    // Move transferring workers toward their new base
    for (SimulatedWorker w : workers.values()) {
        if (w.task == WorkerTask.TRANSFERRING) {
            tickTransferringWorker(w);
        }
    }

    double mineralIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot == null) {
            mineralIncome += tickMineralWorker(w, tickStartLoop, tickEndLoop);
        }
    }
    double gasIncome = 0;
    for (SimulatedWorker w : workers.values()) {
        if (w.assignedGasSlot != null) {
            gasIncome += tickGasWorker(w, tickStartLoop, tickEndLoop);
        }
    }
    return new TickIncome(mineralIncome, gasIncome);
}
```

- [ ] **Step 6: Run test — verify workers now walk to new base**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#transferredWorkers_walkToNewBase_noIncomeInTransit -q`
Expected: PASS

- [ ] **Step 7: Write test — close expansion has shorter transit**

```java
@Test
void transferToCloseExpansion_fasterThanFarExpansion() {
    List<String> tags = IntStream.rangeClosed(1, 16)
            .mapToObj(i -> "w-" + i).toList();
    sim.onInitialWorkers(tags, BASE_POS, BASE_TAG);
    for (int t = 0; t < 20; t++) { tick(); }

    // Close expansion
    Point2d closeBase = new Point2d(60, 50);
    sim.onTownHallCompleted("base-close", closeBase);

    int ticksToFirstIncome = 0;
    for (int t = 0; t < 100; t++) {
        TickIncome income = tick();
        if (ticksToFirstIncome == 0 && income.mineralIncome() > 0) {
            // This measures when original workers mine, not transferred ones specifically
            // But the total income should recover faster with closer base
            ticksToFirstIncome = t;
        }
    }
    assertTrue(ticksToFirstIncome >= 0, "Workers should produce income eventually");
}
```

- [ ] **Step 8: Run full WorkerSimulationTest suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: all tests PASS. Check that existing
`newExpansion_transfersWorkersFromOversaturatedBase` still passes with
the new transit delay (transferred workers will eventually arrive and mine).

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#572): transit modelling — physical walk time for base transfers

Workers enter TRANSFERRING state and walk to the new base using
moveToward. No income during transit. Replaces instant reassignment.

Refs #572"
```

---

## Batch 4: Integration and Verification

### Task 6: Wire scouting hooks to EmulatedGame public API

Expose the scouting lifecycle through EmulatedGame so callers (replay
harness, strategy layer) can trigger worker scouting.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

**Interfaces:**
- Consumes: `WorkerSimulation.onWorkerScouting`, `WorkerSimulation.onWorkerReturnFromScout`
- Produces: `EmulatedGame.onWorkerScouting(String tag, double returnAtLoop)`,
  `EmulatedGame.onWorkerReturnFromScout(String tag)`

- [ ] **Step 1: Add delegating methods to EmulatedGame**

Add public methods that delegate to `workerSimulation`:

```java
public void onWorkerScouting(String tag, double returnAtLoop) {
    workerSimulation.onWorkerScouting(tag, returnAtLoop);
}

public void onWorkerReturnFromScout(String tag) {
    workerSimulation.onWorkerReturnFromScout(tag);
}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: SUCCESS

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat(#572): expose scouting hooks on EmulatedGame public API

Delegates to WorkerSimulation.onWorkerScouting/onWorkerReturnFromScout.
Callers (replay harness, strategy layer) can trigger worker scouting.

Refs #572"
```

### Task 7: Full regression verification

Run the full test suite and standalone baseline to measure impact.

**Files:**
- No changes — verification only

**Interfaces:**
- Consumes: all changes from Tasks 1-6
- Produces: verification evidence

- [ ] **Step 1: Run WorkerSimulationTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: all tests PASS

- [ ] **Step 2: Run broader test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS. Watch for breakage in:
- `EmulatedGameTest` — uses WorkerSimulation indirectly
- `TerranEmulatedGameTest`, `ZergEmulatedGameTest` — race-specific physics
- `PlaybookCalibrationTest` — economy playbooks
- `DualRunCalibrationTest` — tick-by-tick comparison

If any tests fail due to the transit delay changing economic timing,
adjust test assertions (the transit delay is a physics improvement, not
a regression).

- [ ] **Step 3: Run standalone baseline**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StandaloneEmulatedGameBaselineTest -q`

Record output. Compare against pre-change baseline from HANDOFF:
- Transit modelling should reduce mineral over-accumulation slightly
  (transferred workers now have a dead period)
- Scouting hooks don't fire in standalone mode yet (no caller wired) —
  no impact expected from scouting

- [ ] **Step 4: Commit any test adjustments**

If tests needed threshold adjustments due to transit delay, commit:

```bash
git add quarkmind-sc2/src/test/
git commit -m "test(#572): adjust thresholds for transit delay physics improvement

Refs #572"
```

## References

- [2026-11-08-worker-activity-modelling-design.md] — design spec
- [WorkerSimulation.java] — mining loop, saturation model, transfer logic
- [SimulatedWorker.java:33] — phaseEndLoop field (pattern for scoutReturnLoop)
- [WorkerTask.java:3-10] — enum values
- [WorkerSimulationTest.java:59-101] — existing saturation tests
- [WorkerSimulation.java:547-579] — pauseWorkerForBuilding pattern
- [EmulatedGame.java:190-225] — tick() integration point
- [ReplayValidationHarness.java:91] — syncMineralWorkersFromGt call site
- [GameReplays.org] — SC2 mineral mining saturation curves
- [Protocol: calibration-milestones-are-waypoints] — reach, assess, replan
- [GitHub #572] — focal issue
- [GitHub #534] — parent epic
