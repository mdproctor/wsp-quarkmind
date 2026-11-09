# Worker Income Direction Analysis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #591 — Worker income direction analysis
**Issue group:** #591 (within Epic #534)

**Goal:** Fix the 19-34% income deficit by extending the efficiency factor sweep above 1.0, finding the rate-optimal factor per race, and applying it through RaceModel.

**Architecture:** Extend MiningEfficiencyCalibrationTest to sweep 0.85-1.30 and optimize for mineralsCollectionRate MAPE (rate, not balance). Add `miningEfficiencyFactor()` and `gasEfficiencyFactor()` to RaceModel interface with per-race implementations. Apply factors in EmulatedGame.tick() where worker income is consumed. Calibration playbooks against real SC2 deferred to a session with SC2 available.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- RaceModel is a public interface — adding methods with defaults is backward-compatible
- WorkerSimulation is package-private — no changes needed (factor applied in EmulatedGame)
- Efficiency factors default to 1.0 — no behaviour change until calibrated values are set
- The sweep uses oracle replays (118, @Tag("diagnostic")) — no SC2 required

---

## Batch 1: Extended factor sweep

### Task 1: Extend MiningEfficiencyCalibrationTest above 1.0 with rate optimization

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/MiningEfficiencyCalibrationTest.java`

**Interfaces:**
- Consumes: `StandaloneEmulatedGameBaselineTest.runStandalone()`, `DivergenceReport.TickSnapshot`, `PlayerEconomyStats`
- Produces: Console output — per-race optimal factor for both balance and rate MAPE

- [ ] **Step 1: Extend CANDIDATE_FACTORS array to include values above 1.0**

In `MiningEfficiencyCalibrationTest.java`, replace the `CANDIDATE_FACTORS` array:

```java
private static final double[] CANDIDATE_FACTORS = {
    0.85, 0.86, 0.87, 0.88, 0.89,
    0.90, 0.91, 0.92, 0.93, 0.94,
    0.95, 0.96, 0.97, 0.98, 0.99,
    1.00, 1.02, 1.04, 1.06, 1.08,
    1.10, 1.12, 1.14, 1.16, 1.18,
    1.20, 1.22, 1.24, 1.26, 1.28, 1.30
};
```

- [ ] **Step 2: Add rate MAPE sweep alongside balance MAPE**

Add a `CheckpointData` field for GT and EM mineral collection rate:

```java
record CheckpointData(Race race, int checkpoint,
                      double gtMinerals, double emMinerals,
                      double cumulativeMineralIncome,
                      double gtMineralRate, double emMineralRate) {}
```

Update the data collection in the loop to capture rate fields:

```java
double gtRate = snap.groundTruthEconomy().mineralsCollectionRate();
double emRate = snap.emulatedEconomy().mineralsCollectionRate();

allData.add(new CheckpointData(race, cp, gtMinerals, emMinerals,
    cumulativeIncome, gtRate, emRate));
```

- [ ] **Step 3: Add rate MAPE computation to the sweep output**

After the existing balance MAPE sweep, add a second sweep block that optimizes rate MAPE. For rate, the linear approximation is: `adjustedRate = emRate * factor`.

```java
System.out.printf("%n--- %s RATE MAPE (%d samples) ---%n", race, raceData.size());
System.out.printf("%-8s", "Factor");
for (int cp : CHECKPOINTS_MIN) {
    System.out.printf("  %d-min MAPE", cp);
}
System.out.printf("  Combined%n");

double bestRateMape = Double.MAX_VALUE;
double bestRateFactor = 1.0;

for (double factor : CANDIDATE_FACTORS) {
    System.out.printf("%-8.2f", factor);
    double combinedAbsErr = 0;
    double combinedGt = 0;

    for (int cp : CHECKPOINTS_MIN) {
        List<CheckpointData> cpData = raceData.stream()
            .filter(d -> d.checkpoint() == cp).toList();
        double absErrSum = 0;
        double gtSum = 0;
        for (CheckpointData d : cpData) {
            double adjustedRate = d.emMineralRate() * factor;
            absErrSum += Math.abs(d.gtMineralRate() - adjustedRate);
            gtSum += Math.abs(d.gtMineralRate());
        }
        double mape = gtSum > 0 ? absErrSum / gtSum * 100.0 : 0;
        System.out.printf("  %8.1f%%", mape);
        combinedAbsErr += absErrSum;
        combinedGt += gtSum;
    }

    double combinedMape = combinedGt > 0 ? combinedAbsErr / combinedGt * 100.0 : 0;
    System.out.printf("  %8.1f%%%n", combinedMape);

    if (combinedMape < bestRateMape) {
        bestRateMape = combinedMape;
        bestRateFactor = factor;
    }
}
System.out.printf(">>> RATE OPTIMAL: %.2f (combined rate MAPE: %.1f%%)%n%n",
    bestRateFactor, bestRateMape);
```

- [ ] **Step 4: Run the extended sweep**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=MiningEfficiencyCalibrationTest`
Expected: PASS — prints balance and rate MAPE for factors 0.85-1.30 per race. The rate-optimal factor should be > 1.0.

Record the optimal per-race factor from the output — these will be used in Task 2.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/MiningEfficiencyCalibrationTest.java
git commit -m "feat(#591): extend efficiency factor sweep to 1.30 with rate MAPE optimization

Refs #591

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Apply per-race efficiency factors

### Task 2: Add efficiency factor methods to RaceModel and apply in EmulatedGame

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:203-209`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java` (or new test)

**Interfaces:**
- Consumes: Optimal factors from Task 1 sweep output
- Produces: `RaceModel.miningEfficiencyFactor()` (default 1.0), `RaceModel.gasEfficiencyFactor()` (default 1.0), per-race overrides

- [ ] **Step 1: Write failing test — efficiency factor multiplies mineral income**

Add to `EconomyTrackerTest.java` (or a new test file — EconomyTrackerTest tests the tracker, but this tests EmulatedGame integration):

Create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` — check if it exists and add to it, or create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/MiningEfficiencyFactorTest.java`:

```java
@Test
void miningEfficiencyFactorScalesIncome() {
    EmulatedGame game = new EmulatedGame();
    // Use default Protoss race model
    game.reset();
    
    // Run for 50 ticks to accumulate some income
    for (int i = 0; i < 50; i++) {
        game.tick();
    }
    GameState baseline = game.snapshot();
    int baselineRate = baseline.playerEconomy().mineralsCollectionRate();
    
    // Factor > 1.0 should produce higher rate
    assertThat(baselineRate).as("baseline rate with factor=1.0").isGreaterThan(0);
}
```

Note: The exact test structure depends on how the factor is wired. The key assertion: with a higher factor, the mineral rate increases proportionally.

- [ ] **Step 2: Add default methods to RaceModel interface**

Add to `RaceModel.java`:

```java
default double miningEfficiencyFactor() { return 1.0; }
default double gasEfficiencyFactor() { return 1.0; }
```

- [ ] **Step 3: Apply factor in EmulatedGame.tick()**

In `EmulatedGame.java`, modify the income computation (lines ~203-207). Replace:

```java
double tickMineralIncome = workerIncome.mineralIncome();
double tickGasIncome = workerIncome.vespeneIncome();
```

with:

```java
double tickMineralIncome = workerIncome.mineralIncome() * playerRaceModel.miningEfficiencyFactor();
double tickGasIncome = workerIncome.vespeneIncome() * playerRaceModel.gasEfficiencyFactor();
```

- [ ] **Step 4: Override factors in per-race models with calibrated values**

Using the optimal factors from Task 1's rate MAPE sweep (exact values TBD from sweep output), override in each RaceModel implementation. Example (values are placeholders — replace with actual sweep results):

In `ProtossRaceModel.java`:
```java
@Override
public double miningEfficiencyFactor() { return 1.08; }  // from sweep
```

In `ZergRaceModel.java`:
```java
@Override
public double miningEfficiencyFactor() { return 1.12; }  // from sweep
```

In `TerranRaceModel.java`:
```java
@Override
public double miningEfficiencyFactor() { return 1.05; }  // from sweep
```

Gas factors: leave at 1.0 initially. A separate gas factor sweep can be added once mineral factors are validated.

- [ ] **Step 5: Run tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest,EmulatedGameTest,WorkerSimulationTest -q`
Expected: PASS — default factor is 1.0, so existing tests are unaffected

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/RaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat(#591): per-race mining efficiency factors via RaceModel

Refs #591

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Validation and docs

### Task 3: Run baselines and update docs

**Files:**
- Modify: `docs/sc2-physics-constants.md`

**Interfaces:**
- Consumes: StandaloneEmulatedGameBaselineTest output, DivergenceRegressionTest output
- Produces: Updated physics constants doc

- [ ] **Step 1: Run standalone baseline with new factors**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StandaloneEmulatedGameBaselineTest`
Expected: PASS — rate MAPE should improve significantly

- [ ] **Step 2: Run regression test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest`
Expected: PASS — all thresholds met

- [ ] **Step 3: Update docs/sc2-physics-constants.md**

Update section 6 (Mining Efficiency Factors) with the new calibrated values:

```markdown
## 6. Mining Efficiency Factors

Per-race mineral mining efficiency (rate-MAPE-minimising factor, calibrated post sub-tick):

| Race | Factor | Source | Confidence |
|------|--------|--------|------------|
| Protoss | X.XX | MiningEfficiencyCalibrationTest rate sweep | **Calibrated** — #591 |
| Zerg | X.XX | MiningEfficiencyCalibrationTest rate sweep | **Calibrated** — #591 |
| Terran | X.XX | MiningEfficiencyCalibrationTest rate sweep | **Calibrated** — #591 |
```

Update section 16 (Economy Baseline) with new MAPE numbers.

Update section 14 — note that factors are now applied in EmulatedGame.tick() via `playerRaceModel.miningEfficiencyFactor()`, not as a post-hoc adjustment.

- [ ] **Step 4: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#591): update physics constants for recalibrated mining efficiency factors

Refs #591

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-09-worker-income-direction-design.md] — design spec
- [MiningEfficiencyCalibrationTest.java] — existing sweep, extended in Task 1
- [RaceModel.java:22] — interface for per-race mechanics
- [ProtossRaceModel.java:17] — Protoss implementation
- [TerranRaceModel.java:18] — Terran implementation
- [ZergRaceModel.java:19] — Zerg implementation
- [EmulatedGame.java:203-209] — income consumption in tick()
- [WorkerSimulation.java:304] — cargo deposit point
- [docs/sc2-physics-constants.md §6] — current efficiency factors (reverted, not applied)
- [GitHub #591] — focal issue
- [GitHub #566] — original efficiency factor calibration (pre-sub-tick, reverted)
- [GitHub #534] — parent epic
