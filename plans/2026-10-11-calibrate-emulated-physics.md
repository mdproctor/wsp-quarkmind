# Calibrate EmulatedGame Physics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #386 — Calibrate EmulatedGame physics from tournament ground truth
**Issue group:** #386

**Goal:** Raise EmulatedGame accuracy to ≥80% units, ≤50% economy MAPE, and significant upgrade improvement at the 5-minute checkpoint against 118 oracle replays.

**Architecture:** Two-phase staged approach. Phase 1 fixes EmulatedGame internals (economy measurement bug, EGG phantom filtering, upgrade calibration, MULE verification). Phase 2 fixes extraction pipeline if unit accuracy still <80% after Phase 1.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Quarkus (test profile), SC2 replay parsing via Scelight fork

## Global Constraints

- All timing constants must be calibrated from replay ground truth, not derived from `seconds × 22.4` (protocol PP-20260522-572156)
- No `@QuarkusTest` for tests that can be plain JUnit
- Domain model (`domain/`) must remain plain Java — no CDI imports
- Each fix must have a calibration test documenting its source data
- Run `DivergenceRegressionTest` after each batch to verify no regression

---

## Batch 1: Economy Fix + EGG Phantom Filtering

### Task 1: Fix EconomyTracker to track gross mineral/gas income

The economy MAPE is ~200% because `EconomyTracker.tickUpdate()` (line 49) computes `mineralRate` as `minerals_now - minerals_7_ticks_ago` — the net balance delta. SC2's `mineralsCollectionRate` PlayerStats field reports gross collection rate (minerals mined regardless of spending). When the player spends minerals, the net delta goes to zero while gross collection continues.

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java:8-100`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:150-163` (income calculation), `187-188` (tickUpdate call)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java:8-75`

**Interfaces:**
- Consumes: `SC2Data.mineralIncomePerTick(int workers)`, `SC2Data.gasIncomePerTick(int workers)` — existing
- Produces: `EconomyTracker.recordIncome(double mineralIncome, int vespeneIncome)` — new method called by EmulatedGame after each income tick. `PlayerEconomyStats.mineralsCollectionRate()` will now return gross-income-based rate instead of net-delta rate.

- [ ] **Step 1: Write failing test — gross income tracking**

Add a test to `EconomyTrackerTest` that verifies `mineralsCollectionRate` reflects gross income even when spending reduces the balance.

```java
@Test
void collectionRateReflectsGrossIncomeNotNetDelta() {
    var tracker = new EconomyTracker();
    // Simulate 8 ticks of income: 60 minerals/tick gross
    for (int i = 0; i < 8; i++) {
        tracker.recordIncome(60.0, 10);
        // Balance drops due to spending (simulated by passing decreasing balance)
        tracker.tickUpdate(50.0 - i * 5, 10, 23, 14, 12);
    }
    var stats = tracker.currentStats(10, 80, 23, 14, 12);
    // Rate should reflect gross income (~60/tick), not net delta (negative)
    assertThat(stats.mineralsCollectionRate()).isGreaterThanOrEqualTo(50);
    assertThat(stats.vespeneCollectionRate()).isGreaterThanOrEqualTo(5);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest#collectionRateReflectsGrossIncomeNotNetDelta -q`
Expected: FAIL — `recordIncome` method does not exist yet.

- [ ] **Step 3: Add gross income accumulators to EconomyTracker**

Add fields and `recordIncome()` method to `EconomyTracker`:

```java
private double grossMinerals;
private int grossVespene;
private int incomeTicks;

void recordIncome(double mineralIncome, int vespeneIncome) {
    grossMinerals += mineralIncome;
    grossVespene += vespeneIncome;
    incomeTicks++;
}
```

Update `tickUpdate()` to compute `mineralRate` and `vespeneRate` from gross accumulators over the rate interval instead of from balance delta:

```java
void tickUpdate(double minerals, int vespene,
                int supply, int supplyUsed, int workerCount) {
    ticksSinceRateUpdate++;
    if (ticksSinceRateUpdate >= RATE_INTERVAL_TICKS) {
        mineralRate = (int) (grossMinerals - prevGrossMinerals);
        vespeneRate = Math.max(0, grossVespene - prevGrossVespene);
        prevGrossMinerals = grossMinerals;
        prevGrossVespene = grossVespene;
        ticksSinceRateUpdate = 0;
    }
}
```

Replace `prevMinerals`/`prevVespene` (balance-based) with `prevGrossMinerals`/`prevGrossVespene` (gross-based).

Update `reset()` to clear the new fields.

- [ ] **Step 4: Wire recordIncome in EmulatedGame**

In `EmulatedGame.java`, after the per-base mineral income loop (line ~152) and per-geyser gas income loop (line ~161), call `economyTracker.recordIncome()` with the same income values:

```java
// After mineral income loop — accumulate gross income
double mineralIncome = SC2Data.mineralIncomePerTick(workersAtBase);
friendly.addMinerals(mineralIncome);
totalMineralIncome += mineralIncome;

// After gas income loop
int gasIncome = SC2Data.gasIncomePerTick(workersOnThisGeyser);
friendly.addVespene(gasIncome);
totalGasIncome += gasIncome;

// After all income computed for this tick:
economyTracker.recordIncome(totalMineralIncome, totalGasIncome);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest -q`
Expected: ALL PASS including the new test.

- [ ] **Step 6: Update existing EconomyTrackerTest assertions**

The `computesCollectionRateFromDelta` test (line 44) may need updating since it tests the old net-delta behavior. Update it to also call `recordIncome()` alongside `tickUpdate()` so the rate is gross-based.

- [ ] **Step 7: Run full test suite to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS. Watch for failures in tests that depend on `mineralsCollectionRate` or `vespeneCollectionRate` values.

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java
git commit -m "feat: track gross mineral/gas income in EconomyTracker instead of net balance delta

EconomyTracker.tickUpdate() computed mineralRate as balance delta
(income minus spending), but SC2's mineralsCollectionRate PlayerStats
field reports gross collection. Heavy spending drove the rate to zero
while actual mining continued, causing 24x economy MAPE gap.

Refs #386"
```

### Task 2: Filter EGG phantoms from accuracy baseline

Zerg EGG units appear as 73 phantom units in the emulated count (0 in ground truth) because EmulatedGame tracks eggs during morphing. The baseline comparison counts them as unit divergence.

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java:96-97` (accumulateTypes call)
- Test: Self-verifying — the baseline test itself measures accuracy; filtering EGGs will increase the unit accuracy number.

**Interfaces:**
- Consumes: `DivergenceReport.TickSnapshot.emulatedUnitsByType()` — `Map<UnitType, Integer>`
- Produces: No new interfaces — filter is internal to the test's accumulation logic.

- [ ] **Step 1: Identify transient unit types to filter**

Check `UnitType` enum for transient types: `EGG`, `LAIR_COCOON` (Lair morphing), `HIVE_COCOON` (Hive morphing), `GREATER_SPIRE_COCOON`, `BANELING_COCOON`, `RAVAGER_COCOON`, `BROODLORD_COCOON`, `LURKER_COCOON`, `OVERSEER_COCOON`. Use `ide_find_class` to check the exact values in `UnitType`.

- [ ] **Step 2: Add transient type filter to accumulateTypes**

In `EmulatedGameAccuracyBaselineTest.accumulateTypes()` (line 144), filter out transient unit types from both maps before accumulating:

```java
private static final Set<UnitType> TRANSIENT_TYPES = Set.of(
    UnitType.EGG, UnitType.LAIR_COCOON, UnitType.HIVE_COCOON,
    UnitType.GREATER_SPIRE_COCOON, UnitType.BANELING_COCOON,
    UnitType.RAVAGER_COCOON, UnitType.BROODLORD_COCOON,
    UnitType.LURKER_COCOON, UnitType.OVERSEER_COCOON
);

private static void accumulateTypes(Map<UnitType, Integer> gt,
        Map<UnitType, Integer> em, Map<UnitType, long[]> accum) {
    var allTypes = new HashSet<>(gt.keySet());
    allTypes.addAll(em.keySet());
    allTypes.removeAll(TRANSIENT_TYPES);
    // ... rest unchanged
}
```

Verify the exact enum constant names by checking `UnitType.java` — they may differ from the names listed above.

- [ ] **Step 3: Run the accuracy baseline to verify improvement**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: Unit accuracy should improve (was 78.7%). The EGG filtering removes false-positive divergence.

- [ ] **Step 4: Run DivergenceRegressionTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceRegressionTest -q`
Expected: PASS — unit accuracy should still meet ≥70% floor.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java
git commit -m "fix: filter transient Zerg morphing types from accuracy baseline unit counts

EGG, cocoon, and other transient morphing states were counted as real
units in the emulated snapshot, adding 73+ phantom units to the
divergence. Ground truth correctly excludes these.

Refs #386"
```

---

## Batch 2: Upgrade Calibration

### Task 3: Calibrate upgrade timing constants from oracle replays

All 73 upgrade timing constants in `SC2Data.UPGRADE_TIMES` (lines 367-458) use `(int)(N * GAME_LOOPS_PER_SECOND)` — formula-derived estimates that violate protocol PP-20260522-572156. Calibrate from UpgradeComplete tracker events in the oracle replay dataset.

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/mock/SC2UpgradeTimeCalibrationTest.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java:367-458` (UPGRADE_TIMES map)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/mock/SC2UpgradeTimeCalibrationTest.java`

**Interfaces:**
- Consumes: Oracle replays at `quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored/` — `TrackerEventFeatureExtractor` to extract UpgradeComplete events
- Produces: Calibrated values in `SC2Data.UPGRADE_TIMES` — consumed by `EmulatedGame.handleResearch()` at line 792

- [ ] **Step 1: Write SC2UpgradeTimeCalibrationTest**

Follow the pattern of `SC2TrainTimeCalibrationTest`. The test should:
1. Walk all oracle replay files
2. Parse each replay for tracker events
3. For each `UpgradeCompleteEvent`, record `(upgradeType, duration)` where `duration = completeLoop - startLoop`
4. Compute the modal duration per upgrade type
5. Print a table: `UpgradeType | Modal Duration (loops) | Sample Count | Current SC2Data Value`
6. Tag as `@Tag("report")` so it runs with `-Preport`, not in the default suite

```java
@Tag("report")
class SC2UpgradeTimeCalibrationTest {
    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");

    @Test
    void calibrateUpgradeTimesFromOracleReplays() {
        // Walk oracle replays
        // Extract UpgradeComplete tracker events
        // Group by UpgradeType, compute modal duration
        // Print calibration table
        // Assert: at least 10 upgrade types have samples
    }
}
```

Look at `SC2TrainTimeCalibrationTest` for the exact replay parsing and modal computation pattern to replicate.

- [ ] **Step 2: Run the calibration test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2UpgradeTimeCalibrationTest -q`
Expected: Prints calibration table showing modal durations. Some upgrade types may have few samples — note which ones.

- [ ] **Step 3: Update SC2Data.UPGRADE_TIMES with calibrated values**

Replace each `(int)(N * GAME_LOOPS_PER_SECOND)` entry in `UPGRADE_TIMES` (lines 367-458) with the calibrated modal value from the test output. For upgrade types with fewer than 3 samples in the oracle data, keep the existing estimate but add a comment marking it as uncalibrated.

- [ ] **Step 4: Write assertion test for calibrated values**

Add assertions to `SC2UpgradeTimeCalibrationTest` that verify each calibrated SC2Data value matches the oracle modal within 5%:

```java
@Test
void calibratedValuesMatchOracleModal() {
    // For each upgrade type with ≥3 oracle samples:
    // assertThat(SC2Data.upgradeTimeInLoops(type))
    //     .isCloseTo(oracleModal, withinPercentage(5));
}
```

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS. Updated timing constants may change EmulatedGame behavior — watch for new test failures.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/mock/SC2UpgradeTimeCalibrationTest.java
git commit -m "feat: calibrate upgrade timing constants from oracle replay ground truth

All 73 UPGRADE_TIMES entries used formula-derived estimates
(seconds × GAME_LOOPS_PER_SECOND). Replaced with modal durations
from UpgradeComplete tracker events across 118 oracle replays.

Refs #386"
```

### Task 4: Diagnose and fix upgrade intent generation

Upgrade accuracy is 3.4% — the completion logic works (`handleResearch()` at line 757) but upgrade intents may not be reaching EmulatedGame. Diagnose whether this is an EmulatedGame issue or an extraction pipeline gap.

**Files:**
- Investigate: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:757-796` (handleResearch)
- Investigate: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java` — how upgrades reach the emulator
- Potentially modify: extraction pipeline files if intent generation is the bottleneck
- Test: verify via baseline test

**Interfaces:**
- Consumes: `ResearchIntent` — the intent type that triggers `handleResearch()`
- Produces: Upgrade completions visible in `EmulatedGame.snapshot()`

- [ ] **Step 1: Diagnose the intent flow**

Trace how the baseline test feeds upgrades to EmulatedGame. Check:
1. Does `EmulatedGameAccuracyBaselineTest` use replay commands to drive the emulator?
2. Are `ResearchIntent` objects being generated from replay data?
3. Add temporary logging in `handleResearch()` to count how many upgrade intents arrive during a baseline run.

- [ ] **Step 2: Based on diagnosis, implement the fix**

If upgrade intents aren't generated: this is an extraction pipeline issue. Check how `ReplayFeatureExtractor` or the replay command pipeline handles upgrade CmdEvents. Look for missing abilLink mappings for research commands.

If upgrade intents arrive but `handleResearch()` rejects them: check the guard conditions (`hasUpgrade()`, `buildingResearching` flag) for false rejections.

- [ ] **Step 3: Write a test for the fix**

Write a test that feeds a known upgrade sequence to EmulatedGame and verifies the upgrade completes:

```java
@Test
void researchCompletesWithCalibratedTiming() {
    var game = new EmulatedGame(Race.PROTOSS);
    // Set up: give the player a Forge (required for ground weapons)
    // Issue: ResearchIntent for PROTOSS_GROUND_WEAPONS_1
    // Tick forward by SC2Data.upgradeTimeInLoops(PROTOSS_GROUND_WEAPONS_1) / LOOPS_PER_TICK
    // Assert: snapshot shows the upgrade completed
}
```

- [ ] **Step 4: Run accuracy baseline to measure improvement**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: Upgrade accuracy significantly improved from 3.4%.

- [ ] **Step 5: Commit**

```bash
git add <modified files>
git commit -m "fix: upgrade intent generation and completion in EmulatedGame

<describe the specific fix based on diagnosis>

Refs #386"
```

---

## Batch 3: MULE Verification + Baseline Measurement

### Task 5: Verify MULE calldown end-to-end

Recent commits added MULE support, but the baseline shows 0% MULE accuracy (69 GT vs 0 emulated). Verify MULE calldown → spawn → counting pipeline works.

**Files:**
- Investigate: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — search for MULE handling
- Investigate: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` — MULE_LIFETIME constant (1434 loops)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` or new test

**Interfaces:**
- Consumes: `CalldownIntent` or `AbilityIntent` for MULE — check the exact intent type
- Produces: MULE units in `EmulatedGame.snapshot().friendlyUnits()`

- [ ] **Step 1: Trace MULE lifecycle in EmulatedGame**

Search for `MULE` in EmulatedGame.java. Check:
1. Is there a handler for MULE calldown intent?
2. Does it spawn a MULE unit?
3. Is the MULE included in `snapshot()` unit counts?
4. Is the MULE removed after MULE_LIFETIME (1434 loops)?

- [ ] **Step 2: Write test for MULE calldown**

```java
@Test
void muleCalldownProducesCountedUnit() {
    var game = new EmulatedGame(Race.TERRAN);
    // Set up: OrbitalCommand built, 50 energy
    // Issue: MULE calldown intent
    // Assert: snapshot contains 1 MULE
    // Tick past MULE_LIFETIME
    // Assert: snapshot contains 0 MULEs
}
```

- [ ] **Step 3: Fix any issues found**

If MULE isn't counted: add it to the unit snapshot. If MULE calldown intent isn't handled: add the handler. If MULE is spawned but baseline doesn't see it: check the timing.

- [ ] **Step 4: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest -q`
Expected: PASS including MULE test.

- [ ] **Step 5: Commit**

```bash
git add <modified files>
git commit -m "fix: MULE calldown end-to-end verification

<describe what was found and fixed>

Refs #386"
```

### Task 6: Re-run baselines and update regression thresholds

After all Phase 1 fixes, re-run accuracy baselines to measure combined improvement. Update regression thresholds to the new baselines.

**Files:**
- Run: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java:43-46` (thresholds)
- Modify: `docs/benchmarks/emulated-game-accuracy-baseline.md` (auto-generated by baseline test)

**Interfaces:**
- Consumes: All Phase 1 changes
- Produces: Updated baseline numbers and tightened regression thresholds

- [ ] **Step 1: Run EmulatedGameAccuracyBaselineTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameAccuracyBaselineTest -q`
Record the new accuracy numbers for all 4 categories at all checkpoints.

- [ ] **Step 2: Evaluate Phase 2 gate**

Check the unit accuracy at the 5-min checkpoint:
- If ≥80%: Phase 2 is skipped. Proceed to step 3.
- If <80%: Phase 2 is needed. Note which unit types are the remaining top divergence sources. Proceed to Batch 4.

- [ ] **Step 3: Run DivergenceRegressionTest**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceRegressionTest -q`
Expected: PASS — all categories should meet existing floors.

- [ ] **Step 4: Tighten regression thresholds**

Update the threshold constants in `DivergenceRegressionTest` (lines 43-46) to reflect the new baselines minus 10% margin:

```java
// Before: permissive floors
private static final double UNIT_ACCURACY_FLOOR = 0.70;
private static final double ECONOMY_MAPE_CEILING = 420.0;

// After: tightened to new baselines (adjust based on actual results)
private static final double UNIT_ACCURACY_FLOOR = <new_accuracy - 0.05>;
private static final double ECONOMY_MAPE_CEILING = <new_mape + 20>;
```

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS with tightened thresholds.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java docs/benchmarks/emulated-game-accuracy-baseline.md
git commit -m "feat: tighten regression thresholds after Phase 1 calibration

Unit accuracy: <old> → <new>
Economy MAPE: <old> → <new>
Upgrade accuracy: <old> → <new>

Refs #386"
```

---

## Batch 4: Phase 2 — Extraction Pipeline (Conditional)

**Skip this batch entirely if Task 6 Step 2 shows unit accuracy ≥80%.**

### Task 7: Fix Queen intent generation from replay CmdEvents

145 ground truth Queens vs 26 emulated across 118 replays. Queens train from Hatcheries — the CmdEvent extraction under-counts Queen production commands.

**Files:**
- Investigate: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/` — replay command extraction classes
- Investigate: ability mappings for Queen production abilLink
- Modify: extraction pipeline files (determined by diagnosis)
- Test: run `EmulatedGameAccuracyBaselineTest` to measure improvement

**Interfaces:**
- Consumes: Replay CmdEvents with Queen production abilLinks
- Produces: `TrainIntent(UnitType.QUEEN)` reaching EmulatedGame

- [ ] **Step 1: Diagnose Queen under-counting**

Run `AbilityMappingCoverageDiagnosticTest` (or similar diagnostic) to check Queen abilLink coverage. Look for:
1. Is the Queen abilLink mapped in `AbilityMapping`?
2. Are Queen CmdEvents present in replays but filtered out?
3. Is selection-based multiplication under-counting Hatchery selections?

- [ ] **Step 2: Fix the extraction gap**

Based on diagnosis: add missing abilLink mapping, fix selection multiplication, or fix Hatchery-specific extraction logic.

- [ ] **Step 3: Write regression test**

Write a test that processes a known replay with Queens and verifies the correct count of Queen TrainIntents is generated.

- [ ] **Step 4: Run baseline to verify improvement**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: Queen accuracy significantly improved from 17.9%.

- [ ] **Step 5: Commit**

```bash
git add <modified files>
git commit -m "fix: Queen intent generation from replay CmdEvents

<describe specific abilLink/extraction fix>

Refs #386"
```

### Task 8: Fix morph unit intent generation

Baneling at 0% and Roach at 35.4% — morph and train intents under-counted from replay CmdEvents.

**Files:**
- Investigate: Same extraction pipeline as Task 7
- Modify: extraction pipeline files (determined by diagnosis)
- Test: run `EmulatedGameAccuracyBaselineTest` to measure improvement

**Interfaces:**
- Consumes: Replay CmdEvents for morph/train commands
- Produces: Correct `TrainIntent`/`MorphIntent` counts for Baneling, Roach

- [ ] **Step 1: Diagnose morph under-counting**

Same diagnostic approach as Task 7. Check abilLink mappings for Baneling morph (abilLink 73 — known from `AbilityDiscoveryCalibrationTest`) and Roach train commands.

- [ ] **Step 2: Fix extraction gaps**

Add missing abilLink mappings or fix morph-specific extraction logic.

- [ ] **Step 3: Run baseline to verify improvement**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: Unit accuracy ≥80% after all Phase 2 fixes.

- [ ] **Step 4: Final regression check and threshold update**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceRegressionTest -q`
Update thresholds if needed (same pattern as Task 6 Step 4).

- [ ] **Step 5: Commit**

```bash
git add <modified files>
git commit -m "fix: morph/train intent generation for Baneling and Roach

<describe specific fixes>

Refs #386"
```

---

## References

- [2026-10-11-calibrate-emulated-physics-design.md] — design spec this plan implements
- [EconomyTracker.java:49-61] — net balance delta bug (tickUpdate)
- [EmulatedGame.java:150-163] — income calculation loop
- [EmulatedGame.java:757-796] — handleResearch upgrade processing
- [SC2Data.java:367-458] — UPGRADE_TIMES map (73 uncalibrated entries)
- [EmulatedGameAccuracyBaselineTest.java:96-97] — unit type accumulation (no EGG filter)
- [DivergenceRegressionTest.java:43-46] — regression thresholds
- [SC2TrainTimeCalibrationTest.java] — existing calibration pattern to replicate
- [PP-20260522-572156] — protocol: timing constants require replay calibration
- [GitHub #386] — focal issue
- [GitHub #384] — parent issue (classifier pipeline epic)
