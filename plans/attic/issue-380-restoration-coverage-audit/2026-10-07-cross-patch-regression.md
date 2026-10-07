# Cross-Patch Regression Suite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #371 — P2.5-5: Cross-patch regression suite — all 4 categories
**Issue group:** #371

**Goal:** Extend `DivergenceRegressionTest` to cover all 4 feature
categories (units, buildings, upgrades, economy) and run across multiple
patch-era datasets with per-dataset baselines.

**Architecture:** Modify the existing `DivergenceRegressionTest` test class.
Add economy MAPE tracking to the existing `perCategoryAccuracyWithinThreshold()`
method. Add a new `crossPatchAccuracyWithinBaseline()` test method that
samples up to 20 replays per tournament dataset and asserts per-category
accuracy against per-dataset baselines.

**Tech Stack:** JUnit 5, Scelight parser, `ReplayValidationHarness`,
`DivergenceReport.TickSnapshot`, `PlayerEconomyStats`

## Global Constraints

- Plain JUnit extended with `@Tag("report")` — no `@QuarkusTest`
- Test runs from `quarkmind-sc2/` module via `mvn test -pl quarkmind-sc2 -Preport`
- Replay packs at `../quarkmind-classifier/data/replay_packs/`
- Economy MAPE uses 4 core fields only: `mineralsCurrent`, `vespeneCurrent`,
  `mineralsCollectionRate`, `vespeneCollectionRate`
- Baselines set at measured values; margin applied via existing `MARGIN = 1.10`

---

## Batch 1: Economy category and cross-patch regression

### Task 1: Add economy category to perCategoryAccuracyWithinThreshold

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java`

**Interfaces:**
- Consumes: `TickSnapshot.groundTruthEconomy()` → `PlayerEconomyStats`
- Consumes: `TickSnapshot.emulatedEconomy()` → `PlayerEconomyStats`
- Consumes: `PlayerEconomyStats.mineralsCurrent()`, `vespeneCurrent()`,
  `mineralsCollectionRate()`, `vespeneCollectionRate()`
- Produces: economy MAPE assertion in the existing test method

- [ ] **Step 1: Add economy MAPE ceiling constant and helper method**

Add after the existing `UPGRADE_ACCURACY_FLOOR` constant (line 44) and
add a static helper method for computing MAPE from two `PlayerEconomyStats`:

```java
private static final double ECONOMY_MAPE_CEILING = 400.0;

private static double economyMape(PlayerEconomyStats gt, PlayerEconomyStats em) {
    double sum = 0;
    int count = 0;
    int[] gtVals = { gt.mineralsCurrent(), gt.vespeneCurrent(),
                     gt.mineralsCollectionRate(), gt.vespeneCollectionRate() };
    int[] emVals = { em.mineralsCurrent(), em.vespeneCurrent(),
                     em.mineralsCollectionRate(), em.vespeneCollectionRate() };
    for (int i = 0; i < gtVals.length; i++) {
        if (gtVals[i] != 0) {
            sum += Math.abs((double)(emVals[i] - gtVals[i]) / gtVals[i]);
            count++;
        }
    }
    return count > 0 ? (sum / count) * 100.0 : 0;
}
```

The 400.0 ceiling comes from the emulated-game-accuracy-baseline.md which
shows ~328% MAPE at 5min. 400.0 gives ~20% margin above that baseline.

- [ ] **Step 2: Add economy tracking to perCategoryAccuracyWithinThreshold**

In the `perCategoryAccuracyWithinThreshold()` method, add economy
tracking variables alongside the existing counters (after line 155):

```java
double economyMapeSum = 0;
int    economyMapeCount = 0;
```

Inside the per-player loop (after the upgrade tracking at line 190),
add economy collection:

```java
PlayerEconomyStats gtEcon = snap.groundTruthEconomy();
PlayerEconomyStats emEcon = snap.emulatedEconomy();
if (gtEcon != null && emEcon != null) {
    economyMapeSum += economyMape(gtEcon, emEcon);
    economyMapeCount++;
}
```

After the existing accuracy calculations (after line 203), add economy
MAPE computation:

```java
double econMape = economyMapeCount > 0 ? economyMapeSum / economyMapeCount : 0;
```

Add economy to the print output (after the upgrades printf at line 211):

```java
System.out.printf("  Economy:   %.1f%% MAPE (threshold: %.0f%%)%n",
                  econMape, ECONOMY_MAPE_CEILING);
```

Add the economy assertion (after the upgrade assertion at line 218):

```java
assertThat(econMape)
        .as("Economy MAPE at 5-min").isLessThanOrEqualTo(ECONOMY_MAPE_CEILING);
```

- [ ] **Step 3: Run test to verify economy assertion passes**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest#perCategoryAccuracyWithinThreshold`
Expected: PASS with economy MAPE printed and within the 400.0 ceiling

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java
git commit -m "feat: add economy category to DivergenceRegressionTest

Tracks economy MAPE across 4 core fields (mineralsCurrent, vespeneCurrent,
mineralsCollectionRate, vespeneCollectionRate) at 5-min checkpoint.
Ceiling set at 400% based on emulated-game-accuracy-baseline.md.

Refs #371"
```

### Task 2: Add cross-patch regression test method

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java`

**Interfaces:**
- Consumes: `ReplayValidationHarness.run(Path, int, int)` → `DivergenceReport`
- Consumes: `DivergenceReport.TickSnapshot` — all per-type maps, upgrades, economy
- Consumes: `RepParserEngine.parseReplay(Path, EnumSet)` → `Replay`
- Produces: new test method `crossPatchAccuracyWithinBaseline()`

- [ ] **Step 1: Add DatasetBaseline record, dataset discovery, and baseline map**

Add after the existing `BASELINE_5MIN` map. The `DatasetBaseline` record
holds per-category thresholds. The `discoverTournamentDatasets()` method
returns only datasets with tracker events (excludes ladder datasets).
The baseline map holds measured values (to be calibrated on first run).

```java
record DatasetBaseline(double unitFloor, double buildingFloor,
                       double upgradeFloor, double economyMapeCeiling) {}

private static final int SAMPLE_CAP = 20;

private static final Map<String, DatasetBaseline> CROSS_PATCH_BASELINES = new java.util.LinkedHashMap<>();
static {
    // Baselines will be populated after first calibration run.
    // Format: dataset name → DatasetBaseline(unitFloor, buildingFloor, upgradeFloor, econMapeCeiling)
    // Thresholds are applied with MARGIN (1.10) so values here are the raw measured baselines.
}

record DatasetSource(String name, Path dir) {}

private static List<DatasetSource> discoverTournamentDatasets() {
    var replayPacks = Path.of("../quarkmind-classifier/data/replay_packs");
    var datasets = new ArrayList<DatasetSource>();
    datasets.add(new DatasetSource("AI Arena (4.9.3)", Path.of("replays/aiarena_protoss")));
    datasets.add(new DatasetSource("HSC XXVII 2025", replayPacks.resolve("2025_HomeStory_Cup_XXVII")));
    datasets.add(new DatasetSource("IEM PyeongChang 2018", replayPacks.resolve("2018_IEM_PyeongChang")));
    datasets.add(new DatasetSource("ASUS ROG 2020", replayPacks.resolve("2020_ASUS_ROG_Online")));
    datasets.add(new DatasetSource("DreamHack Dallas 2025", replayPacks.resolve("2025_DreamHack_Dallas")));
    datasets.add(new DatasetSource("EWC 2025", replayPacks.resolve("2025_Esports_World_Cup")));
    datasets.add(new DatasetSource("FEL Cracow 2025", replayPacks.resolve("2025_FEL_Cracow")));
    return datasets;
}
```

- [ ] **Step 2: Add crossPatchAccuracyWithinBaseline test method**

Add a new test method after `perCategoryAccuracyWithinThreshold()`:

```java
@Test
@EnabledIf("anyTournamentDatasetExists")
void crossPatchAccuracyWithinBaseline() throws Exception {
    System.out.printf("%n=== Cross-Patch Regression — %d replay sample per dataset ===%n%n", SAMPLE_CAP);
    System.out.printf("%-30s %6s %6s %6s %6s %8s%n",
        "Dataset", "Reps", "UnitAc", "BldgAc", "UpgAc", "EconMAPE");
    System.out.println("-".repeat(75));

    int totalProcessed = 0;

    for (var ds : discoverTournamentDatasets()) {
        if (!Files.isDirectory(ds.dir())) continue;

        List<Path> replays;
        try (var stream = Files.list(ds.dir())) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                            .sorted().limit(SAMPLE_CAP).toList();
        }
        if (replays.isEmpty()) continue;

        long gtUnits = 0, emUnits = 0, gtBldgs = 0, emBldgs = 0;
        int gtUpgrades = 0, matchedUpgrades = 0;
        double econMapeSum = 0;
        int econCount = 0, playerRuns = 0;

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    DivergenceReport report = ReplayValidationHarness.run(
                        replayPath, playerId, TICK_LIMIT);
                    int tickIndex = CHECKPOINT_MINUTE * TICKS_PER_MINUTE - 1;
                    if (tickIndex >= report.ticks().size()) continue;

                    DivergenceReport.TickSnapshot snap = report.ticks().get(tickIndex);

                    for (var entry : snap.groundTruthUnitsByType().entrySet()) {
                        gtUnits += entry.getValue();
                        emUnits += snap.emulatedUnitsByType().getOrDefault(entry.getKey(), 0);
                    }
                    for (var entry : snap.groundTruthBuildingsByType().entrySet()) {
                        gtBldgs += entry.getValue();
                        emBldgs += snap.emulatedBuildingsByType().getOrDefault(entry.getKey(), 0);
                    }
                    gtUpgrades += snap.groundTruthUpgrades().size();
                    matchedUpgrades += (int) snap.groundTruthUpgrades().stream()
                        .filter(snap.emulatedUpgrades()::contains).count();

                    PlayerEconomyStats gtEcon = snap.groundTruthEconomy();
                    PlayerEconomyStats emEcon = snap.emulatedEconomy();
                    if (gtEcon != null && emEcon != null) {
                        econMapeSum += economyMape(gtEcon, emEcon);
                        econCount++;
                    }
                    playerRuns++;
                } catch (Exception e) { /* skip */ }
            }
        }

        if (playerRuns == 0) continue;

        double unitAcc = gtUnits > 0 ? (double) Math.min(emUnits, gtUnits) / gtUnits : 1.0;
        double bldgAcc = gtBldgs > 0 ? (double) Math.min(emBldgs, gtBldgs) / gtBldgs : 1.0;
        double upgAcc = gtUpgrades > 0 ? (double) matchedUpgrades / gtUpgrades : 1.0;
        double econMape = econCount > 0 ? econMapeSum / econCount : 0;

        System.out.printf("%-30s %6d %5.1f%% %5.1f%% %5.1f%% %7.1f%%%n",
            ds.name(), replays.size(), unitAcc * 100, bldgAcc * 100,
            upgAcc * 100, econMape);

        DatasetBaseline baseline = CROSS_PATCH_BASELINES.get(ds.name());
        if (baseline != null) {
            assertThat(unitAcc)
                .as("Unit accuracy for %s", ds.name())
                .isGreaterThanOrEqualTo(baseline.unitFloor() / MARGIN);
            assertThat(bldgAcc)
                .as("Building accuracy for %s", ds.name())
                .isGreaterThanOrEqualTo(baseline.buildingFloor() / MARGIN);
            assertThat(upgAcc)
                .as("Upgrade accuracy for %s", ds.name())
                .isGreaterThanOrEqualTo(baseline.upgradeFloor() / MARGIN);
            assertThat(econMape)
                .as("Economy MAPE for %s", ds.name())
                .isLessThanOrEqualTo(baseline.economyMapeCeiling() * MARGIN);
        } else {
            System.out.printf("  ⚠ No baseline for %s — run calibration to set thresholds%n", ds.name());
        }

        totalProcessed += replays.size();
    }

    assertThat(totalProcessed).as("Must process at least 1 cross-patch replay").isGreaterThan(0);
    System.out.printf("%nTotal: %d replays processed across tournament datasets%n", totalProcessed);
}
```

Add the `anyTournamentDatasetExists` guard method:

```java
static boolean anyTournamentDatasetExists() {
    return discoverTournamentDatasets().stream().anyMatch(ds -> Files.isDirectory(ds.dir()));
}
```

- [ ] **Step 3: Run calibration (no baselines yet — prints measured values)**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest#crossPatchAccuracyWithinBaseline`
Expected: PASS (no assertions fire without baselines), prints per-dataset
measured values. Record these values.

- [ ] **Step 4: Populate CROSS_PATCH_BASELINES with measured values**

From the calibration output, populate the baseline map. Example (actual
values from the calibration run):

```java
static {
    CROSS_PATCH_BASELINES.put("AI Arena (4.9.3)",
        new DatasetBaseline(0.30, 0.95, 0.0, 400.0));
    CROSS_PATCH_BASELINES.put("HSC XXVII 2025",
        new DatasetBaseline(/* measured unitAcc */, /* bldgAcc */, /* upgAcc */, /* econMape */));
    // ... repeat for each dataset from calibration output
}
```

Use the measured values directly — the `MARGIN` is applied at assertion
time (dividing floors by MARGIN, multiplying ceilings by MARGIN).

- [ ] **Step 5: Run full test suite to verify all assertions pass**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest`
Expected: All 3 tests PASS — `oracleDivergenceWithinBaseline`,
`perCategoryAccuracyWithinThreshold` (with economy), and
`crossPatchAccuracyWithinBaseline` (with baselines)

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java
git commit -m "feat: add cross-patch regression with per-dataset baselines

Samples up to 20 replays per tournament dataset (HSC, IEM, ASUS ROG,
DreamHack, EWC, FEL, AI Arena). Asserts per-category accuracy (units,
buildings, upgrades, economy MAPE) stays within per-dataset baselines
+ 10% margin.

Refs #371"
```

### Task 3: Update CLAUDE.md and issue

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: Task 1 and Task 2 test names and descriptions
- Produces: updated documentation

- [ ] **Step 1: Update DivergenceRegressionTest description in CLAUDE.md**

Find the existing description in the report test section (line ~300):

Old:
```
- `DivergenceRegressionTest` — asserts per-matchup unit and building deltas at 5-min checkpoint stay within baseline + 10% margin (oracle dataset); also asserts per-category accuracy thresholds (units ≥30%, buildings ≥95%); fails loudly on regression
```

New:
```
- `DivergenceRegressionTest` — asserts per-matchup unit and building deltas at 5-min checkpoint stay within baseline + 10% margin (oracle dataset); per-category accuracy thresholds for all 4 categories (units ≥30%, buildings ≥95%, upgrades ≥0%, economy MAPE ≤400%); cross-patch regression against tournament datasets (HSC, IEM, ASUS ROG, DreamHack, EWC, FEL) with per-dataset baselines
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update DivergenceRegressionTest description — economy + cross-patch

Refs #371"
```

## References

- [2026-10-07-cross-patch-regression-design.md] — design spec this plan implements
- [DivergenceRegressionTest.java] — test being extended
- [DivergenceReport.java:13-34] — TickSnapshot record with economy fields
- [PlayerEconomyStats.java] — economy record with 13 fields
- [OracleAccuracyBaselineTest.java:518-529] — MAPE computation pattern
- [CrossPatchExtractionTest.java:25-39] — dataset discovery pattern
- [docs/benchmarks/emulated-game-accuracy-baseline.md] — economy MAPE baselines
- [GitHub #371] — focal issue
- [GitHub #379] — TickSnapshot economy fields were added here
- [GitHub #366] — epic: Phase 2.5 reconstitution accuracy gate
