# Rate Cadence Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #590 — Rate measurement cadence alignment
**Issue group:** #590 (within Epic #534)

**Goal:** Eliminate rate MAPE error caused by EconomyTracker sampling at 154 loops (7 ticks) while SC2 PlayerStatsEvents fire every 160 loops, then measure the residual.

**Architecture:** Replace EconomyTracker's tick-counting rate window with loop-counting. Add a Quarkus config property `starcraft.economy.rate-interval-loops` (default 160) following the WorkerPhysicsConfig pattern. Write a diagnostic test that confirms PlayerStatsEvent cadence and measures the MAPE improvement.

**Tech Stack:** Java 21, Quarkus (MicroProfile Config), JUnit 5, AssertJ

## Global Constraints

- EconomyTracker is package-private — all changes stay within `io.quarkmind.sc2.emulated`
- No `@QuarkusTest` — all tests are plain JUnit (EconomyTracker has no CDI deps)
- The diagnostic test is `@Tag("diagnostic")` — excluded from default surefire run
- LOOPS_PER_TICK = 22, GAME_LOOPS_PER_SECOND = 22.4 (from SC2Data)

---

## Batch 1: Loop-based rate windowing

### Task 1: EconomyTracker loop-based rate calculation

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java`

**Interfaces:**
- Consumes: `SC2Data.LOOPS_PER_TICK` (22), `SC2Data.GAME_LOOPS_PER_SECOND` (22.4)
- Produces: `EconomyTracker(int rateIntervalLoops)` constructor; `tickUpdate()` signature unchanged

- [ ] **Step 1: Write failing test — rate recalculation fires at loop threshold**

Add to `EconomyTrackerTest.java`:

```java
@Test
void rateRecalculationFiresAtLoopThreshold() {
    var tracker = new EconomyTracker(160);
    // 160 loops / 22 loops per tick = 7.27 ticks — rate should NOT fire after 7 ticks
    for (int i = 0; i < 7; i++) {
        tracker.recordIncome(10.0, 2.0);
        tracker.tickUpdate(0, 0, 15, 6, 12);
    }
    var after7 = tracker.currentStats(0, 0, 15, 6, 12);
    assertThat(after7.mineralsCollectionRate()).as("rate should not update after 7 ticks (154 loops < 160)").isEqualTo(0);

    // 8th tick pushes to 176 loops >= 160 — rate should fire
    tracker.recordIncome(10.0, 2.0);
    tracker.tickUpdate(0, 0, 15, 6, 12);
    var after8 = tracker.currentStats(0, 0, 15, 6, 12);
    assertThat(after8.mineralsCollectionRate()).as("rate should update after 8 ticks (176 loops >= 160)").isGreaterThan(0);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest#rateRecalculationFiresAtLoopThreshold -q`
Expected: FAIL — `EconomyTracker` has no `int` constructor

- [ ] **Step 3: Write failing test — loop-based rate formula uses actual elapsed loops**

Add to `EconomyTrackerTest.java`:

```java
@Test
void loopBasedRateUsesActualElapsedLoops() {
    var tracker = new EconomyTracker(160);
    // 8 ticks = 176 loops. Income: 10.0 minerals/tick over 8 ticks = 80.0 total
    // Rate = 80.0 / 176 loops * (60 * 22.4) loops/min = 80.0 / 176 * 1344 = 610.9 → 610
    for (int i = 0; i < 8; i++) {
        tracker.recordIncome(10.0, 2.0);
        tracker.tickUpdate(0, 0, 15, 6, 12);
    }
    var stats = tracker.currentStats(0, 0, 15, 6, 12);
    double loopsPerMinute = 60.0 * 22.4;
    int expectedMineralRate = (int) (80.0 / 176.0 * loopsPerMinute);
    int expectedVespeneRate = (int) (16.0 / 176.0 * loopsPerMinute);
    assertThat(stats.mineralsCollectionRate()).isEqualTo(expectedMineralRate);
    assertThat(stats.vespeneCollectionRate()).isEqualTo(expectedVespeneRate);
}
```

- [ ] **Step 4: Implement loop-based rate windowing in EconomyTracker**

Modify `EconomyTracker.java`:

1. Add constructor accepting `rateIntervalLoops`:

```java
private final int rateIntervalLoops;
private static final double LOOPS_PER_MINUTE = 60.0 * SC2Data.GAME_LOOPS_PER_SECOND;

EconomyTracker() {
    this(160);
}

EconomyTracker(int rateIntervalLoops) {
    this.rateIntervalLoops = rateIntervalLoops;
}
```

2. Replace `ticksSinceRateUpdate` with `loopsSinceRateUpdate`:

```java
private int loopsSinceRateUpdate;
```

3. Update `tickUpdate()`:

```java
void tickUpdate(double minerals, int vespene,
                int supply, double supplyUsed, int workerCount) {
    loopsSinceRateUpdate += SC2Data.LOOPS_PER_TICK;
    if (loopsSinceRateUpdate >= rateIntervalLoops) {
        double perLoopMineral = (grossMinerals - prevGrossMinerals) / loopsSinceRateUpdate;
        double perLoopVespene = (grossVespene - prevGrossVespene) / loopsSinceRateUpdate;
        mineralRate          = mineralRateOverride >= 0
                ? mineralRateOverride
                : (int) Math.max(0, perLoopMineral * LOOPS_PER_MINUTE);
        vespeneRate          = vespeneRateOverride >= 0
                ? vespeneRateOverride
                : (int) Math.max(0, perLoopVespene * LOOPS_PER_MINUTE);
        prevGrossMinerals    = grossMinerals;
        prevGrossVespene     = grossVespene;
        loopsSinceRateUpdate = 0;
    }
}
```

4. Update `reset()` — replace `ticksSinceRateUpdate = 0` with `loopsSinceRateUpdate = 0`.

5. Remove `RATE_INTERVAL_TICKS` and `TICKS_PER_MINUTE` constants (no longer needed).

- [ ] **Step 5: Run all EconomyTrackerTest tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest -q`
Expected: PASS — existing tests used default constructor (7 ticks → 154 loops), now default is 160 loops (8 ticks). Two existing tests need updating:

- `computesCollectionRateFromDelta` — uses 8 iterations, rate fires on tick 8 (176 loops >= 160). Should still pass as-is.
- `collectionRateIsConvertedToPerMinute` — uses 7 iterations. With 160-loop window, rate won't fire after 7 ticks. **Update this test to use 8 iterations** and adjust the expected value:

```java
@Test
void collectionRateIsConvertedToPerMinute() {
    var tracker = new EconomyTracker();
    // 8 ticks = 176 loops >= 160. Income: 10.0/tick * 8 = 80.0 total
    for (int i = 0; i < 8; i++) {
        tracker.recordIncome(10.0, 2.0);
        tracker.tickUpdate(0, 0, 15, 6, 12);
    }
    var stats = tracker.currentStats(0, 0, 15, 6, 12);
    double loopsPerMin = 60.0 * 22.4;
    assertThat(stats.mineralsCollectionRate()).isEqualTo((int) (80.0 / 176.0 * loopsPerMin));
    assertThat(stats.vespeneCollectionRate()).isEqualTo((int) (16.0 / 176.0 * loopsPerMin));
}
```

Also update `collectionRateReflectsGrossIncomeNotNetDelta` if it relies on rate firing at tick 7.

- [ ] **Step 6: Fix `resetClearsAllTracking` test if needed**

The `resetClearsAllTracking` test calls `tickUpdate` once then `reset()`. After the change, one tick (22 loops) won't trigger rate calculation (< 160), so `mineralsCollectionRate()` is already 0 before reset. The assertion `assertThat(stats.mineralsCollectionRate()).isEqualTo(0)` still holds. Verify by running it.

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java
git commit -m "feat(#590): loop-based rate windowing in EconomyTracker (default 160 loops)

Refs #590

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: EconomyConfig and EmulatedGame wiring

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfig.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfigProducer.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:129`

**Interfaces:**
- Consumes: `EconomyTracker(int rateIntervalLoops)` from Task 1
- Produces: `EconomyConfig` record with `rateIntervalLoops()`, `EconomyConfig.defaults()`, `EmulatedGame.setEconomyConfig(EconomyConfig)`

- [ ] **Step 1: Create EconomyConfig record**

Create `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfig.java`:

```java
package io.quarkmind.sc2.emulated;

public record EconomyConfig(int rateIntervalLoops) {

    public static final int DEFAULT_RATE_INTERVAL_LOOPS = 160;

    public static EconomyConfig defaults() {
        return new EconomyConfig(DEFAULT_RATE_INTERVAL_LOOPS);
    }
}
```

- [ ] **Step 2: Create EconomyConfigProducer**

Create `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfigProducer.java`:

```java
package io.quarkmind.sc2.emulated;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
class EconomyConfigProducer {

    @ConfigProperty(name = "starcraft.economy.rate-interval-loops",
                    defaultValue = "160")
    int rateIntervalLoops;

    @Produces
    @ApplicationScoped
    EconomyConfig produce() {
        return new EconomyConfig(rateIntervalLoops);
    }
}
```

- [ ] **Step 3: Wire EconomyConfig into EmulatedGame**

Modify `EmulatedGame.java` line 129. Change:

```java
private final EconomyTracker   economyTracker   = new EconomyTracker();
```

to:

```java
private EconomyTracker economyTracker = new EconomyTracker();
```

Add a setter method:

```java
public void setEconomyConfig(EconomyConfig config) {
    this.economyTracker = new EconomyTracker(config.rateIntervalLoops());
}
```

- [ ] **Step 4: Run existing test suite to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest,WorkerSimulationTest,EmulatedGameTest -q`
Expected: PASS — default constructor still uses 160, same as before Task 1

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfig.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyConfigProducer.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat(#590): add EconomyConfig with rate-interval-loops property

Refs #590

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Diagnostic and validation

### Task 3: RateCadenceDiagnosticTest

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/RateCadenceDiagnosticTest.java`

**Interfaces:**
- Consumes: `ReplaySimulatedGame`, `TrackerIntentExtractor`, `EmulatedGame`, `EconomyConfig`, `StandaloneEmulatedGameBaselineTest.runStandalone()`
- Produces: Console output — PlayerStatsEvent cadence stats + per-checkpoint MAPE comparison at 154 vs 160 loop windows

- [ ] **Step 1: Write the diagnostic test**

Create `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/RateCadenceDiagnosticTest.java`:

```java
package io.quarkmind.sc2.emulated;

import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelightapi.sc2.rep.model.trackerevents.IPlayerStatsEvent;
import io.quarkmind.domain.PlayerEconomyStats;
import io.quarkmind.domain.Race;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.sc2.intent.TimedIntent;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import io.quarkmind.sc2.replay.DivergenceReport;
import io.quarkmind.sc2.replay.StandaloneEmulatedGameBaselineTest;
import io.quarkmind.sc2.replay.TrackerIntentExtractor;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.EnumSet;
import java.util.List;

@Tag("diagnostic")
class RateCadenceDiagnosticTest {

    private static final Path ORACLE_DIR = Path.of(
        "../quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored");
    private static final int TICKS_PER_MINUTE =
        (int) (60 * SC2Data.GAME_LOOPS_PER_SECOND / SC2Data.LOOPS_PER_TICK);
    private static final int[] CHECKPOINTS_MIN = {1, 3, 5, 7};
    private static final int TICK_LIMIT = TICKS_PER_MINUTE * 8;

    static boolean oracleExists() {
        if (!Files.isDirectory(ORACLE_DIR)) return false;
        try (var stream = Files.list(ORACLE_DIR)) {
            return stream.anyMatch(p -> p.toString().endsWith(".SC2Replay"));
        } catch (Exception e) { return false; }
    }

    @Test
    @EnabledIf("oracleExists")
    void measurePlayerStatsEventCadence() throws IOException {
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        List<Integer> allDeltas = new ArrayList<>();
        int replayCount = 0;

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath,
                    EnumSet.of(RepContent.TRACKER_EVENTS, RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.trackerEvents == null) continue;

            List<Integer> playerStatsLoops = new ArrayList<>();
            for (var event : replay.trackerEvents.events) {
                if (event instanceof IPlayerStatsEvent && event.getPlayerId() != null) {
                    if (event.getPlayerId() == 1) {
                        playerStatsLoops.add(event.getLoop());
                    }
                }
            }

            for (int i = 1; i < playerStatsLoops.size(); i++) {
                allDeltas.add(playerStatsLoops.get(i) - playerStatsLoops.get(i - 1));
            }
            replayCount++;
        }

        allDeltas.sort(Integer::compareTo);
        double mean = allDeltas.stream().mapToInt(i -> i).average().orElse(0);
        int median = allDeltas.isEmpty() ? 0 : allDeltas.get(allDeltas.size() / 2);
        int min = allDeltas.isEmpty() ? 0 : allDeltas.getFirst();
        int max = allDeltas.isEmpty() ? 0 : allDeltas.getLast();

        System.out.println("=== PlayerStatsEvent Cadence ===\n");
        System.out.printf("Replays: %d, Intervals: %d%n", replayCount, allDeltas.size());
        System.out.printf("Mean:   %.2f loops (%.3f seconds)%n", mean, mean / SC2Data.GAME_LOOPS_PER_SECOND);
        System.out.printf("Median: %d loops%n", median);
        System.out.printf("Min:    %d loops%n", min);
        System.out.printf("Max:    %d loops%n", max);

        long exactly160 = allDeltas.stream().filter(d -> d == 160).count();
        System.out.printf("Exactly 160: %d / %d (%.1f%%)%n",
            exactly160, allDeltas.size(), exactly160 * 100.0 / allDeltas.size());
    }

    @Test
    @EnabledIf("oracleExists")
    void compareCadenceMapeContribution() throws IOException {
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        int[] windows = {154, 160, 176};
        double[][][] mapeAccum = new double[windows.length][CHECKPOINTS_MIN.length][3];
        // [windowIdx][cpIdx][0=gtSum, 1=emSum, 2=absErrSum] — for mineralsCollectionRate
        double[][][] gasAccum = new double[windows.length][CHECKPOINTS_MIN.length][3];

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;
            if (replay.details.getPlayerList().length < 2) continue;

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    Race race = TrackerIntentExtractor.resolvePlayerRace(replay, playerId);
                    List<TimedIntent> intents = TrackerIntentExtractor.extract(replayPath, playerId);

                    for (int w = 0; w < windows.length; w++) {
                        ReplaySimulatedGame gt = new ReplaySimulatedGame(replayPath, playerId);
                        List<DivergenceReport.TickSnapshot> snapshots =
                            runWithRateInterval(gt, intents, TICK_LIMIT, race, windows[w]);

                        for (int c = 0; c < CHECKPOINTS_MIN.length; c++) {
                            int tickIndex = CHECKPOINTS_MIN[c] * TICKS_PER_MINUTE - 1;
                            if (tickIndex >= snapshots.size()) continue;
                            var snap = snapshots.get(tickIndex);
                            if (snap.groundTruthEconomy() == null || snap.emulatedEconomy() == null) continue;
                            PlayerEconomyStats gtEcon = snap.groundTruthEconomy();
                            PlayerEconomyStats emEcon = snap.emulatedEconomy();

                            mapeAccum[w][c][0] += gtEcon.mineralsCollectionRate();
                            mapeAccum[w][c][1] += emEcon.mineralsCollectionRate();
                            mapeAccum[w][c][2] += Math.abs(
                                gtEcon.mineralsCollectionRate() - emEcon.mineralsCollectionRate());

                            gasAccum[w][c][0] += gtEcon.vespeneCollectionRate();
                            gasAccum[w][c][1] += emEcon.vespeneCollectionRate();
                            gasAccum[w][c][2] += Math.abs(
                                gtEcon.vespeneCollectionRate() - emEcon.vespeneCollectionRate());
                        }
                    }
                } catch (Exception e) { /* skip */ }
            }
        }

        System.out.println("\n=== Rate Cadence MAPE Comparison ===\n");
        System.out.println("| Checkpoint | Field | 154-loop MAPE | 160-loop MAPE | 176-loop MAPE | Delta (154→160) |");
        System.out.println("|---|---|---|---|---|---|");
        for (int c = 0; c < CHECKPOINTS_MIN.length; c++) {
            double[] mineralMapes = new double[windows.length];
            double[] gasMapes = new double[windows.length];
            for (int w = 0; w < windows.length; w++) {
                mineralMapes[w] = mapeAccum[w][c][0] != 0
                    ? mapeAccum[w][c][2] / Math.abs(mapeAccum[w][c][0]) * 100 : 0;
                gasMapes[w] = gasAccum[w][c][0] != 0
                    ? gasAccum[w][c][2] / Math.abs(gasAccum[w][c][0]) * 100 : 0;
            }
            System.out.printf("| %d-min | mineralsCollectionRate | %.1f%% | %.1f%% | %.1f%% | %+.1f%% |%n",
                CHECKPOINTS_MIN[c], mineralMapes[0], mineralMapes[1], mineralMapes[2],
                mineralMapes[1] - mineralMapes[0]);
            System.out.printf("| %d-min | vespeneCollectionRate | %.1f%% | %.1f%% | %.1f%% | %+.1f%% |%n",
                CHECKPOINTS_MIN[c], gasMapes[0], gasMapes[1], gasMapes[2],
                gasMapes[1] - gasMapes[0]);
        }
    }

    private static List<DivergenceReport.TickSnapshot> runWithRateInterval(
            ReplaySimulatedGame groundTruth, List<TimedIntent> intents,
            int tickLimit, Race race, int rateIntervalLoops) {

        EmulatedGame emulated = new EmulatedGame();
        emulated.setPlayerRaceModel(RaceModelFactory.forRace(race));
        emulated.setEconomyConfig(new EconomyConfig(rateIntervalLoops));
        groundTruth.reset();
        emulated.setMineralPatchResources(groundTruth.getMineralPatches());
        emulated.resetForHarness();

        int cursor = 0;
        List<DivergenceReport.TickSnapshot> snapshots = new ArrayList<>(Math.min(tickLimit, 10000));
        float gasPerWorkerPerMin = 54.0f;

        for (int tick = 0; tick < tickLimit && !groundTruth.isComplete(); tick++) {
            long windowEnd = (long) (tick + 1) * SC2Data.LOOPS_PER_TICK;

            emulated.tick();
            groundTruth.tick();

            var gt = groundTruth.snapshot();

            emulated.processDeferredTrainIntents();
            emulated.processDeferredResearchIntents();

            while (cursor < intents.size() && intents.get(cursor).loop() < windowEnd) {
                emulated.applyIntent(intents.get(cursor));
                cursor++;
            }

            int gtGasRate = gt.playerEconomy().vespeneCollectionRate();
            int estimatedGasWorkers = gtGasRate > 0
                ? Math.max(1, Math.round(gtGasRate / gasPerWorkerPerMin)) : 0;
            emulated.syncTotalGasWorkers(estimatedGasWorkers);

            var em = emulated.snapshot();

            var gtUnitsByType = new java.util.EnumMap<>(io.quarkmind.domain.UnitType.class);
            var emUnitsByType = new java.util.EnumMap<>(io.quarkmind.domain.UnitType.class);
            var gtBldgsByType = new java.util.EnumMap<>(io.quarkmind.domain.BuildingType.class);
            var emBldgsByType = new java.util.EnumMap<>(io.quarkmind.domain.BuildingType.class);

            snapshots.add(new DivergenceReport.TickSnapshot(
                tick,
                gt.myUnits().size(), em.myUnits().size(),
                gt.myBuildings().size(), em.myBuildings().size(),
                gt.minerals(), em.minerals(),
                gt.vespene(), em.vespene(),
                gtUnitsByType, emUnitsByType,
                gtBldgsByType, emBldgsByType,
                gt.playerUpgrades(), em.playerUpgrades(),
                gt.playerEconomy(), em.playerEconomy()));
        }
        return snapshots;
    }
}
```

- [ ] **Step 2: Run the diagnostic**

Run: `mvn test -pl quarkmind-sc2 -Dtest=RateCadenceDiagnosticTest -q`
Expected: PASS — prints cadence stats and MAPE comparison table

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/RateCadenceDiagnosticTest.java
git commit -m "feat(#590): diagnostic test — PlayerStatsEvent cadence + rate MAPE comparison

Refs #590

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Validation and docs update

**Files:**
- Modify: `docs/sc2-physics-constants.md`

**Interfaces:**
- Consumes: Diagnostic test output (MAPE numbers), StandaloneEmulatedGameBaselineTest output
- Produces: Updated physics constants doc

- [ ] **Step 1: Run the standalone baseline to get updated MAPE numbers**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StandaloneEmulatedGameBaselineTest -q`
Expected: PASS — prints baseline with updated rate MAPE

- [ ] **Step 2: Run DivergenceRegressionTest to confirm no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceRegressionTest -q`
Expected: PASS — all thresholds met

- [ ] **Step 3: Update docs/sc2-physics-constants.md**

Add to section 14 (Insights and Derived Physics), after the "Rate measurement cadence" subsection, update the constant value:

```markdown
### Rate measurement cadence

SC2 emits PlayerStatsEvents every 160 game loops (7.14 seconds). EconomyTracker
aligns its rate window to 160 loops via `starcraft.economy.rate-interval-loops`.
The window fires after 8 ticks (176 loops >= 160), using actual elapsed loops
in the rate denominator for accurate per-minute conversion.
```

Add `rate-interval-loops` to section 19 (Quarkus Config Properties):

```properties
starcraft.economy.rate-interval-loops=160
```

Update section 16 (Economy Baseline) with the new MAPE numbers from the baseline test output.

- [ ] **Step 4: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#590): update physics constants for rate cadence alignment

Refs #590

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-09-rate-cadence-alignment-design.md] — design spec this plan implements
- [EconomyTracker.java:43] — RATE_INTERVAL_TICKS=7 (replaced by loop-based window)
- [EconomyTrackerTest.java] — existing unit tests for rate calculation
- [WorkerPhysicsConfig.java] — config pattern followed for EconomyConfig
- [WorkerPhysicsConfigProducer.java] — CDI producer pattern followed
- [EmulatedGame.java:129] — EconomyTracker instantiation point
- [EmulatedGame.java:231-232] — tickUpdate call site
- [StandaloneEmulatedGameBaselineTest.java] — validation baseline
- [docs/sc2-physics-constants.md] — reference doc to update
- [GitHub #590] — focal issue
- [GitHub #534] — parent epic
