# Physics-First Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #763 — High-saturation contention model gap
**Issue group:** #763

**Goal:** Build a reusable per-loop SC2 worker physics profiler and use it to measure foundation mining constants directly from SC2, replacing aggregate-fit values.

**Architecture:** A `WorkerPhysicsProfilerTest` drives SC2 at per-loop resolution via `SC2FrameCallback`, recording per-worker position and cargo. A `WorkerPhysicsAnalyser` infers mining phases from state transitions and computes physical constants. Phase 1 measures zero-contention constants (8w). Phase 2 measures contention behavior (16w/24w/36w).

**Tech Stack:** ocraft-s2client (SC2 bot API), PlaybookSC2Adapter (playbook execution), JUnit 5

## Global Constraints

- SC2 client must be running for `@Tag("sc2")` tests
- All measurements on TorchesAIE map (apples-to-apples with existing baselines)
- Per-loop resolution via `RequestStep` with `step_size=1` (existing `gameLoop()` method)
- Tag-based worker tracking, not index-based (SC2 observation array reorders per tick, §1b)
- Minimum N ≥ 30 cycles per constant for statistical significance
- Multi-run: minimum 3 runs per scenario

---

## Batch 1: Profiler Infrastructure + 8w Recording

### Task 1: WorkerPhysicsProfilerTest — per-loop mineral worker recording at 8w

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsProfilerTest.java`
- Reference: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/GasCycleProfilerTest.java` (pattern)
- Reference: `quarkmind-sc2/src/test/resources/playbooks/cal-mineral-baseline-8w.yaml`

**Interfaces:**
- Consumes: `QuarkusSC2Transport` (SC2 connection), `PlaybookSC2Adapter` (playbook execution), `SC2FrameCallback` (per-loop callback)
- Produces: `List<LoopMineralObs>` per-loop per-worker observations; JSON file at `src/test/resources/profiler/mineral-8w-run1.json`

- [ ] **Step 1: Create the per-loop observation record**

```java
record LoopMineralObs(int gameLoop, long tag, float x, float y, int mineralCargo, String orderAbility) {}
```

Follows `GasCycleProfilerTest.LoopWorkerObs` pattern. `mineralCargo` from unit's cargo count. `orderAbility` from first order's ability ID (or "NONE").

- [ ] **Step 2: Write the profiler test**

```java
@Tag("sc2")
class WorkerPhysicsProfilerTest {

    @Test
    void profileMineralWorkers8w() throws Exception {
        assumeTrue(QuarkusSC2Transport.isSC2Available(),
                "StarCraft II not installed — skipping live SC2 test");

        QuarkusSC2Transport transport = new QuarkusSC2Transport();
        transport.sc2Port = 8168;
        transport.mapName = "TorchesAIE_v4";
        transport.difficultyStr = "VERY_EASY";
        transport.aiRaceStr = "RANDOM";
        transport.botRaceStr = "PROTOSS";
        transport.connectRetryCount = 60;
        transport.connectRetryIntervalMs = 5000;
        transport.connect();
        transport.createGame();
        transport.joinGame();

        List<LoopMineralObs> loopData = new ArrayList<>();
        Map<Long, float[]> patchPositions = new LinkedHashMap<>();
        Map<Long, float[]> thPositions = new LinkedHashMap<>();
        int[] loopRef = {0};
        int targetWorkers = 8;
        int measurementStartLoop = -1;
        int measurementTicks = 50;

        PlaybookSC2Adapter playbook = new PlaybookSC2Adapter(
                transport, "playbooks/cal-mineral-baseline-8w.yaml");

        SC2FrameCallback profiler = new SC2FrameCallback() {
            @Override
            public void onGameStart(ResponseGameInfo gameInfo) {
                playbook.onGameStart(gameInfo);
            }

            @Override
            public void onStep(Observation obs) throws InterruptedException {
                playbook.onStep(obs);
                ObservationRaw raw = obs.getRaw().orElseThrow();
                var allUnits = raw.getUnits();

                // Count mineral workers
                int workerCount = 0;
                for (var u : allUnits) {
                    if (u.getAlliance() != Alliance.SELF) continue;
                    Units type = (Units) u.getType();
                    if (type == Units.PROTOSS_PROBE) workerCount++;
                    if (type == Units.PROTOSS_NEXUS)
                        thPositions.putIfAbsent(u.getTag().getValue(),
                            new float[]{u.getPosition().getX(), u.getPosition().getY()});
                    if (type == Units.NEUTRAL_MINERAL_FIELD || type == Units.NEUTRAL_MINERAL_FIELD750
                            || type == Units.NEUTRAL_RICH_MINERAL_FIELD
                            || type == Units.NEUTRAL_RICH_MINERAL_FIELD750)
                        patchPositions.putIfAbsent(u.getTag().getValue(),
                            new float[]{u.getPosition().getX(), u.getPosition().getY()});
                }

                // Start measurement window once target count reached and stabilised
                if (workerCount == targetWorkers && measurementStartLoop < 0
                        && loopRef[0] > 100 * SC2Data.LOOPS_PER_TICK) {
                    measurementStartLoop = loopRef[0];
                }

                // Record per-loop per-worker data during measurement window
                if (measurementStartLoop >= 0
                        && loopRef[0] < measurementStartLoop + measurementTicks * SC2Data.LOOPS_PER_TICK) {
                    for (var u : allUnits) {
                        if (u.getAlliance() != Alliance.SELF) continue;
                        if ((Units) u.getType() != Units.PROTOSS_PROBE) continue;
                        int cargo = u.getMineralContents().orElse(0);
                        String order = u.getOrders().stream().findFirst()
                                .map(o -> o.getAbility().toString()).orElse("NONE");
                        loopData.add(new LoopMineralObs(loopRef[0], u.getTag().getValue(),
                                u.getPosition().getX(), u.getPosition().getY(), cargo, order));
                    }
                }

                loopRef[0]++;
            }

            @Override
            public void onGameEnd(GameResult result) { playbook.onGameEnd(result); }
        };

        transport.runGameLoop(profiler);
        while (!transport.isRunning()) { Thread.sleep(50); }
        while (transport.isRunning()) { Thread.sleep(100); }

        assertFalse(loopData.isEmpty(), "Should have recorded per-loop data");
        assertTrue(measurementStartLoop >= 0, "Should have reached target worker count");

        // Write JSON
        var mapper = new com.fasterxml.jackson.databind.ObjectMapper()
                .enable(com.fasterxml.jackson.databind.SerializationFeature.INDENT_OUTPUT);
        var output = Map.of(
                "scenario", "mineral-8w",
                "map", "TorchesAIE_v4",
                "targetWorkers", targetWorkers,
                "measurementStartLoop", measurementStartLoop,
                "measurementTicks", measurementTicks,
                "observations", loopData,
                "patchPositions", patchPositions,
                "thPositions", thPositions);
        java.nio.file.Path outPath = java.nio.file.Path.of(
                "src/test/resources/profiler/mineral-8w-run1.json");
        java.nio.file.Files.createDirectories(outPath.getParent());
        mapper.writeValue(outPath.toFile(), output);
        System.out.printf("Wrote %d observations to %s%n", loopData.size(), outPath);
    }
}
```

- [ ] **Step 3: Run the profiler against real SC2**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsProfilerTest#profileMineralWorkers8w -Dquarkus.profile=sc2`
Expected: JSON file written with per-loop worker data. Verify file exists and contains observations.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsProfilerTest.java
git add quarkmind-sc2/src/test/resources/profiler/
git commit -m "feat(#763): add WorkerPhysicsProfilerTest — per-loop mineral worker recording Refs #763"
```

### Task 2: WorkerPhysicsAnalyser — phase decomposition from per-loop data

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyser.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyserTest.java`
- Reference: `quarkmind-sc2/src/test/resources/profiler/mineral-8w-run1.json` (from Task 1)

**Interfaces:**
- Consumes: Profiler JSON (List of LoopMineralObs, patch/TH positions)
- Produces: `CycleDecomposition` record per worker per cycle; `ConstantReport` with measured values, mean, σ, N

- [ ] **Step 1: Write failing test for cargo-based cycle detection**

```java
class WorkerPhysicsAnalyserTest {

    @Test
    void detectsCyclesFromCargoFlips() {
        // Synthetic data: worker at patch (cargo 0), then cargo flips to 5, then walks to TH, cargo flips to 0
        var obs = List.of(
            new LoopMineralObs(100, 1L, 10.0f, 10.0f, 0, "HARVEST_GATHER"),  // at patch, mining
            new LoopMineralObs(122, 1L, 10.0f, 10.0f, 5, "HARVEST_GATHER"),  // cargo flip 0→5
            new LoopMineralObs(123, 1L, 10.5f, 10.0f, 5, "HARVEST_RETURN"),  // walking to TH
            new LoopMineralObs(155, 1L, 8.0f, 8.0f, 5, "HARVEST_RETURN"),    // at TH
            new LoopMineralObs(167, 1L, 8.0f, 8.0f, 0, "HARVEST_GATHER"),    // cargo flip 5→0 (deposit)
            new LoopMineralObs(168, 1L, 8.5f, 8.5f, 0, "HARVEST_GATHER")     // walking to patch
        );
        var patches = Map.of(100L, new float[]{10.0f, 10.0f});
        var ths = Map.of(200L, new float[]{8.0f, 8.0f});

        var analyser = new WorkerPhysicsAnalyser(patches, ths, 0.75f);
        var cycles = analyser.detectCycles(obs, 1L);

        assertEquals(1, cycles.size());
        // Deposit at loop 167, next deposit would be the next cycle boundary
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsAnalyserTest#detectsCyclesFromCargoFlips`
Expected: FAIL — class not found

- [ ] **Step 3: Implement WorkerPhysicsAnalyser**

```java
class WorkerPhysicsAnalyser {

    record CycleDecomposition(long workerTag, int cycleIndex,
            int depositLoop, int departThLoop, int arrivePatchLoop,
            int cargoFlipLoop, int departPatchLoop, int arriveThLoop, int nextDepositLoop,
            int walkToPatch, int approachAndMine, int walkToBase, int deposit, int totalCycle) {}

    record ConstantReport(String name, double mean, double sigma, int sampleSize, double min, double max) {}

    private final Map<Long, float[]> patchPositions;
    private final Map<Long, float[]> thPositions;
    private final float positionTolerance;

    WorkerPhysicsAnalyser(Map<Long, float[]> patches, Map<Long, float[]> ths, float tolerance) {
        this.patchPositions = patches;
        this.thPositions = ths;
        this.positionTolerance = tolerance;
    }

    List<CycleDecomposition> detectCycles(List<LoopMineralObs> obs, long workerTag) {
        // Filter to this worker, sorted by gameLoop
        var workerObs = obs.stream()
                .filter(o -> o.tag() == workerTag)
                .sorted(java.util.Comparator.comparingInt(LoopMineralObs::gameLoop))
                .toList();

        // Find cargo flip loops (0→5 and 5→0)
        List<int[]> cargoFlips = new ArrayList<>(); // [loop, fromCargo, toCargo]
        for (int i = 1; i < workerObs.size(); i++) {
            var prev = workerObs.get(i - 1);
            var curr = workerObs.get(i);
            if (prev.mineralCargo() != curr.mineralCargo()) {
                cargoFlips.add(new int[]{curr.gameLoop(), prev.mineralCargo(), curr.mineralCargo()});
            }
        }

        // Build cycles between consecutive deposit flips (5→0)
        List<CycleDecomposition> cycles = new ArrayList<>();
        List<Integer> depositLoops = cargoFlips.stream()
                .filter(f -> f[1] > 0 && f[2] == 0)
                .map(f -> f[0]).toList();
        List<Integer> pickupLoops = cargoFlips.stream()
                .filter(f -> f[1] == 0 && f[2] > 0)
                .map(f -> f[0]).toList();

        for (int d = 0; d < depositLoops.size() - 1; d++) {
            int deposit = depositLoops.get(d);
            int nextDeposit = depositLoops.get(d + 1);
            // Find the pickup (0→5) between these two deposits
            int pickup = pickupLoops.stream()
                    .filter(p -> p > deposit && p < nextDeposit)
                    .findFirst().orElse(-1);
            if (pickup < 0) continue;

            // Infer walk phases from position transitions
            int departTh = findDepartureLoop(workerObs, deposit, true);
            int arrivePatch = findArrivalLoop(workerObs, departTh, pickup, false);
            int departPatch = findDepartureLoop(workerObs, pickup, false);
            int arriveTh = findArrivalLoop(workerObs, departPatch, nextDeposit, true);

            int walkToPatch = arrivePatch - departTh;
            int approachAndMine = pickup - arrivePatch;
            int walkToBase = arriveTh - departPatch;
            int depositDuration = nextDeposit - arriveTh;
            int total = nextDeposit - deposit;

            cycles.add(new CycleDecomposition(workerTag, d, deposit, departTh, arrivePatch,
                    pickup, departPatch, arriveTh, nextDeposit,
                    walkToPatch, approachAndMine, walkToBase, depositDuration, total));
        }
        return cycles;
    }

    private int findDepartureLoop(List<LoopMineralObs> obs, int fromLoop, boolean fromTh) {
        // Find first loop after fromLoop where position starts moving
        for (int i = 0; i < obs.size() - 1; i++) {
            if (obs.get(i).gameLoop() < fromLoop) continue;
            float dx = obs.get(i + 1).x() - obs.get(i).x();
            float dy = obs.get(i + 1).y() - obs.get(i).y();
            if (Math.sqrt(dx * dx + dy * dy) > 0.1f) return obs.get(i).gameLoop();
        }
        return fromLoop;
    }

    private int findArrivalLoop(List<LoopMineralObs> obs, int fromLoop, int beforeLoop, boolean atTh) {
        Map<Long, float[]> targets = atTh ? thPositions : patchPositions;
        for (int i = 0; i < obs.size(); i++) {
            var o = obs.get(i);
            if (o.gameLoop() < fromLoop || o.gameLoop() > beforeLoop) continue;
            for (float[] pos : targets.values()) {
                float dx = o.x() - pos[0];
                float dy = o.y() - pos[1];
                if (Math.sqrt(dx * dx + dy * dy) <= positionTolerance) return o.gameLoop();
            }
        }
        return beforeLoop;
    }

    List<ConstantReport> computeConstants(List<CycleDecomposition> allCycles) {
        return List.of(
            report("walkToPatch", allCycles.stream().mapToDouble(CycleDecomposition::walkToPatch)),
            report("approachAndMine", allCycles.stream().mapToDouble(CycleDecomposition::approachAndMine)),
            report("walkToBase", allCycles.stream().mapToDouble(CycleDecomposition::walkToBase)),
            report("deposit", allCycles.stream().mapToDouble(CycleDecomposition::deposit)),
            report("totalCycle", allCycles.stream().mapToDouble(CycleDecomposition::totalCycle))
        );
    }

    private ConstantReport report(String name, java.util.stream.DoubleStream values) {
        double[] arr = values.toArray();
        double mean = java.util.Arrays.stream(arr).average().orElse(0);
        double variance = java.util.Arrays.stream(arr).map(v -> (v - mean) * (v - mean)).average().orElse(0);
        double sigma = Math.sqrt(variance);
        double min = java.util.Arrays.stream(arr).min().orElse(0);
        double max = java.util.Arrays.stream(arr).max().orElse(0);
        return new ConstantReport(name, mean, sigma, arr.length, min, max);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsAnalyserTest#detectsCyclesFromCargoFlips`
Expected: PASS

- [ ] **Step 5: Add test for constant computation from real profiler data**

```java
@Test
void computeConstantsFromProfilerData() throws Exception {
    var mapper = new com.fasterxml.jackson.databind.ObjectMapper();
    try (var is = getClass().getClassLoader().getResourceAsStream("profiler/mineral-8w-run1.json")) {
        assumeTrue(is != null, "Profiler data not recorded — run WorkerPhysicsProfilerTest first");
        var data = mapper.readTree(is);
        // Parse observations, patches, TH positions from JSON
        // Run analyser, print constant report
        // Assert N ≥ 30 cycles total
    }
}
```

- [ ] **Step 6: Run against real profiler data and print report**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsAnalyserTest#computeConstantsFromProfilerData`
Expected: Report with measured approach+mine, walk, deposit durations with mean ± σ.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyser.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyserTest.java
git commit -m "feat(#763): add WorkerPhysicsAnalyser — cargo-primary phase decomposition Refs #763"
```

## Batch 2: Phase 1 Constants — Measure and Replace

### Task 3: Validate foundation constants and update physics doc

**Files:**
- Modify: `docs/sc2-physics-constants.md` — update §3 constants with measured values, add §20 Measurement Backlog
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` — update defaults if measured values differ
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java` — update compile-time constants if measured values differ

**Interfaces:**
- Consumes: Constant report from Task 2 analyser
- Produces: Updated physics constants with **Measured** derivation method

- [ ] **Step 1: Compare measured values to current constants**

From the Task 2 report, extract:
- `miningDuration` = `approachAndMine` - `approach` (but we can't separate them yet — record `approachAndMine` as the combined value)
- `deposit` directly measured
- `walkToPatch` and `walkToBase` for cycle validation

Print comparison table:
```
| Constant | Current | Measured (mean) | σ | N | Delta |
```

- [ ] **Step 2: If measured deposit differs from 12 by >1 loop, update the constant**

Update `WorkerPhysicsConfig.DEFAULT_DEPOSIT_DURATION_LOOPS` to the measured value.
Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest,SC2PlaybookCalibrationTest,WorkerSimulationTest,EmulatedGameTest`
Expected: All tests pass. If any calibration checkpoint worsens by >1%, apply escape clause (investigate, don't revert).

- [ ] **Step 3: Update physics doc with measured values and derivation method**

Add `Derivation method` column to §3 table. Mark measured constants as **Measured**. Add §20 Measurement Backlog with remaining constants.

- [ ] **Step 4: Commit**

```bash
git add docs/sc2-physics-constants.md quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/
git commit -m "feat(#763): Phase 1 — replace aggregate-fit constants with SC2-measured values Refs #763"
```

## Batch 3: Phase 2 — Contention Measurement

### Task 4: Record 16w and 36w profiler data

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsProfilerTest.java` — add 16w and 36w scenarios

**Interfaces:**
- Consumes: Existing playbooks `cal-mineral-saturated-16w.yaml`, `cal-single-base-probes.yaml` (for 36w)
- Produces: JSON files `profiler/mineral-16w-run1.json`, `profiler/mineral-36w-run1.json`

- [ ] **Step 1: Add profileMineralWorkers16w test method**

Same pattern as 8w but with `targetWorkers = 16` and playbook `cal-mineral-saturated-16w.yaml`. Measurement window starts after probe count stabilises at 16.

- [ ] **Step 2: Add profileMineralWorkers36w test method**

Uses `cal-single-base-probes.yaml` (trains to 36). Measurement window starts after probe count reaches 36 (near end of playbook). Shorter window (20 ticks) since 36w is reached late.

- [ ] **Step 3: Run both profilers**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsProfilerTest -Dquarkus.profile=sc2`

- [ ] **Step 4: Commit profiler data**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsProfilerTest.java
git add quarkmind-sc2/src/test/resources/profiler/
git commit -m "feat(#763): record 16w and 36w per-loop profiler data Refs #763"
```

### Task 5: Measure approach absorption from contention data

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyserTest.java` — add contention analysis
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — update `MIN_APPROACH_AFTER_WAIT` with measured value
- Modify: `docs/sc2-physics-constants.md` — update §3 with measured approach absorption

**Interfaces:**
- Consumes: 8w profiler data (zero-contention baseline), 16w profiler data (contention tier)
- Produces: Measured `MIN_APPROACH_AFTER_WAIT` value; updated `approachAndMine` for waiting vs non-waiting workers

- [ ] **Step 1: Write test comparing waiting vs non-waiting approach at 16w**

```java
@Test
void measureApproachAbsorptionAt16w() throws Exception {
    var baseline8w = loadProfilerData("profiler/mineral-8w-run1.json");
    var contention16w = loadProfilerData("profiler/mineral-16w-run1.json");

    var analyser8w = createAnalyser(baseline8w);
    var analyser16w = createAnalyser(contention16w);

    // Get baseline approachAndMine at 8w (no waiting)
    var cycles8w = getAllCycles(analyser8w, baseline8w);
    double baselineApproachAndMine = cycles8w.stream()
            .mapToDouble(CycleDecomposition::approachAndMine).average().orElse(0);

    // Get 16w cycles, classify by wait time
    var cycles16w = getAllCycles(analyser16w, contention16w);
    // A cycle "waited" if arrivePatchLoop is significantly before cargoFlipLoop - mining_duration
    double miningDuration = 22; // validated in Phase 1
    var waited = cycles16w.stream()
            .filter(c -> c.approachAndMine() > baselineApproachAndMine + 5)
            .toList();
    var notWaited = cycles16w.stream()
            .filter(c -> c.approachAndMine() <= baselineApproachAndMine + 5)
            .toList();

    System.out.printf("8w baseline approachAndMine: %.1f loops (N=%d)%n",
            baselineApproachAndMine, cycles8w.size());
    System.out.printf("16w non-waited approachAndMine: %.1f loops (N=%d)%n",
            notWaited.stream().mapToDouble(CycleDecomposition::approachAndMine).average().orElse(0),
            notWaited.size());
    System.out.printf("16w waited approachAndMine: %.1f loops (N=%d)%n",
            waited.stream().mapToDouble(CycleDecomposition::approachAndMine).average().orElse(0),
            waited.size());

    // The irreducible minimum = waited approachAndMine - mining_duration
    // This is what MIN_APPROACH_AFTER_WAIT should be
    double measuredMinApproach = waited.stream()
            .mapToDouble(c -> c.approachAndMine() - miningDuration)
            .average().orElse(0);
    System.out.printf("Measured MIN_APPROACH_AFTER_WAIT: %.1f loops%n", measuredMinApproach);
}
```

- [ ] **Step 2: Run analysis**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerPhysicsAnalyserTest#measureApproachAbsorptionAt16w`
Expected: Report with measured approach for waiting vs non-waiting workers.

- [ ] **Step 3: Update MIN_APPROACH_AFTER_WAIT with measured value**

Replace the 0 (or swept 12) with the measured value from Step 2.

- [ ] **Step 4: Run full calibration suite to validate**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest,SC2PlaybookCalibrationTest,WorkerSimulationTest,EmulatedGameTest`
Expected: All pass. Report pinned single-base and two-base deltas.

- [ ] **Step 5: Update physics doc and commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/WorkerPhysicsAnalyserTest.java
git add docs/sc2-physics-constants.md
git commit -m "feat(#763): Phase 2 — measured MIN_APPROACH_AFTER_WAIT from SC2 per-loop data Refs #763"
```

## References

- [2026-11-25-physics-first-calibration-design.md] — design spec this plan implements
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/GasCycleProfilerTest.java` — existing per-loop gas profiler (pattern)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/SC2FrameCallback.java` — per-loop callback interface
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java` — SC2 connection
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/PlaybookSC2Adapter.java` — playbook-driven SC2 control
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` — constants to validate/replace
- `docs/sc2-physics-constants.md` — constant documentation
- GitHub #763 — focal issue
