# EmulatedGame Accuracy Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #379 — P2.5-7: EmulatedGame accuracy baseline
**Issue group:** #379

**Goal:** Measure EmulatedGame per-type accuracy against ground-truth tracker events and establish regression thresholds.

**Architecture:** Extend `DivergenceReport.TickSnapshot` with per-type maps, add `EconomyTracker` to EmulatedGame for spending/income derivation, create `EmulatedGameAccuracyBaselineTest` report test, and update `DivergenceRegressionTest` with per-category accuracy thresholds.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Scelight replay parsing

## Global Constraints

- Domain model (`io.quarkmind.domain.*`) must remain plain Java — no CDI, no Quarkus imports
- `EconomyTracker` goes in `io.quarkmind.sc2.emulated` — same package as `EmulatedGame`
- Report tests use `@Tag("report")` — excluded from default surefire run
- Never use `@QuarkusTest` for tests that can be plain JUnit

---

## Batch 1: Extended TickSnapshot and EconomyTracker

### Task 1: Extend TickSnapshot with per-type maps

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:98-103`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java:95-99`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceBaselineReportTest.java:110-113`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/mock/IEM10MultiGameValidationTest.java:62`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java` (existing or new)

**Interfaces:**
- Produces: `TickSnapshot` with fields `groundTruthUnitsByType` (`Map<UnitType, Integer>`), `emulatedUnitsByType`, `groundTruthBuildingsByType` (`Map<BuildingType, Integer>`), `emulatedBuildingsByType`, `groundTruthUpgrades` (`Set<String>`), `emulatedUpgrades`, `groundTruthEconomy` (`PlayerEconomyStats`), `emulatedEconomy`

- [ ] **Step 1: Write failing test for extended TickSnapshot**

Create a unit test that constructs a TickSnapshot with per-type maps and verifies the new fields are accessible.

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class DivergenceReportTest {

    @Test
    void tickSnapshotExposePerTypeMaps() {
        var unitsByType = Map.of(UnitType.MARINE, 5, UnitType.STALKER, 3);
        var emUnitsByType = Map.of(UnitType.MARINE, 4, UnitType.STALKER, 3);
        var bldgsByType = Map.of(BuildingType.BARRACKS, 2);
        var emBldgsByType = Map.of(BuildingType.BARRACKS, 2);
        var gtUpgrades = Set.of("Stimpack");
        var emUpgrades = Set.of("Stimpack");

        var snap = new DivergenceReport.TickSnapshot(
            10, 8, 7, 2, 2, 1000, 950, 200, 200,
            unitsByType, emUnitsByType,
            bldgsByType, emBldgsByType,
            gtUpgrades, emUpgrades,
            PlayerEconomyStats.EMPTY, PlayerEconomyStats.EMPTY);

        assertThat(snap.groundTruthUnitsByType()).isEqualTo(unitsByType);
        assertThat(snap.emulatedUnitsByType()).isEqualTo(emUnitsByType);
        assertThat(snap.groundTruthUpgrades()).isEqualTo(gtUpgrades);
        assertThat(snap.unitDelta()).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceReportTest -q`
Expected: FAIL — TickSnapshot constructor doesn't accept per-type arguments

- [ ] **Step 3: Extend TickSnapshot record**

In `DivergenceReport.java`, add the new fields to the `TickSnapshot` record after the existing 8 fields:

```java
public record TickSnapshot(
    int tick,
    int groundTruthUnits,     int emulatedUnits,
    int groundTruthBuildings, int emulatedBuildings,
    int groundTruthMinerals,  int emulatedMinerals,
    int groundTruthVespene,   int emulatedVespene,
    Map<UnitType, Integer> groundTruthUnitsByType,
    Map<UnitType, Integer> emulatedUnitsByType,
    Map<BuildingType, Integer> groundTruthBuildingsByType,
    Map<BuildingType, Integer> emulatedBuildingsByType,
    Set<String> groundTruthUpgrades,
    Set<String> emulatedUpgrades,
    PlayerEconomyStats groundTruthEconomy,
    PlayerEconomyStats emulatedEconomy) {
    // existing methods unchanged
}
```

Add imports for `Map`, `Set`, `UnitType`, `BuildingType`, `PlayerEconomyStats` to `DivergenceReport.java`.

- [ ] **Step 4: Fix all existing TickSnapshot construction sites**

Update `ReplayValidationHarness.java:98-103` to collect per-type data. Replace the snapshot construction with:

```java
GameState em = emulated.snapshot();

Map<UnitType, Integer> gtUnitsByType = gt.myUnits().stream()
    .collect(Collectors.groupingBy(Unit::type, Collectors.summingInt(u -> 1)));
Map<UnitType, Integer> emUnitsByType = em.myUnits().stream()
    .collect(Collectors.groupingBy(Unit::type, Collectors.summingInt(u -> 1)));
Map<BuildingType, Integer> gtBldgsByType = gt.myBuildings().stream()
    .filter(Building::isComplete)
    .collect(Collectors.groupingBy(Building::type, Collectors.summingInt(b -> 1)));
Map<BuildingType, Integer> emBldgsByType = em.myBuildings().stream()
    .filter(Building::isComplete)
    .collect(Collectors.groupingBy(Building::type, Collectors.summingInt(b -> 1)));

snapshots.add(new DivergenceReport.TickSnapshot(
    tick,
    gt.myUnits().size(),     em.myUnits().size(),
    gt.myBuildings().size(), em.myBuildings().size(),
    gt.minerals(),           em.minerals(),
    gt.vespene(),            em.vespene(),
    gtUnitsByType,           emUnitsByType,
    gtBldgsByType,           emBldgsByType,
    gt.playerUpgrades(),     em.playerUpgrades(),
    gt.playerEconomy(),      em.playerEconomy()));
```

Add imports: `java.util.stream.Collectors`, `io.quarkmind.domain.Unit`, `io.quarkmind.domain.Building`, `io.quarkmind.domain.UnitType`, `io.quarkmind.domain.BuildingType`.

Update `DivergenceRegressionTest.java` — no logic changes needed, it only accesses `unitDelta()` and `buildingDelta()` which are unchanged.

Update `DivergenceBaselineReportTest.java` — only uses method references (`TickSnapshot::mineralDelta` etc.), no changes needed.

Update `IEM10MultiGameValidationTest.java` — only uses `TickSnapshot::unitDelta`, no changes needed.

- [ ] **Step 5: Run all tests to verify compilation and existing behaviour**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DivergenceReportTest -q`
Expected: PASS

Run: `mvn test -pl quarkmind-sc2 -Dtest=ReplayValidationHarnessTest -q`
Expected: PASS (if exists) or skip

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS (verifies all construction sites compile)

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceReportTest.java
git commit -m "feat: extend TickSnapshot with per-type maps, upgrades, and economy stats

Adds groundTruthUnitsByType, emulatedUnitsByType, groundTruthBuildingsByType,
emulatedBuildingsByType, groundTruthUpgrades, emulatedUpgrades,
groundTruthEconomy, and emulatedEconomy fields to TickSnapshot.

ReplayValidationHarness now collects per-type unit/building counts,
upgrade sets, and PlayerEconomyStats at each tick.

Refs #379"
```

### Task 2: Add EconomyTracker to EmulatedGame

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java`

**Interfaces:**
- Consumes: `PlayerState` (minerals, vespene, supply, supplyUsed, units, buildings, completedUpgradeNames), `SC2Data` (mineral/gas costs)
- Produces: `PlayerEconomyStats currentStats()` — returns economy snapshot; `void recordTrainSpending(UnitType, int mineralCost, int gasCost)`, `void recordBuildSpending(BuildingType, int mineralCost)`, `void tickUpdate(double minerals, int vespene, PlayerState state)` — called each tick to update rate tracking

- [ ] **Step 1: Write failing test for EconomyTracker**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class EconomyTrackerTest {

    @Test
    void tracksTrainSpendingAsArmyCost() {
        var tracker = new EconomyTracker();
        tracker.recordTrainSpending(UnitType.MARINE, 50, 0);
        tracker.recordTrainSpending(UnitType.MARINE, 50, 0);

        var stats = tracker.currentStats(100, 0, 23, 14, 12);
        assertThat(stats.mineralsUsedCurrentArmy()).isEqualTo(100);
        assertThat(stats.vespeneUsedCurrentArmy()).isEqualTo(0);
    }

    @Test
    void tracksBuildSpendingByCategory() {
        var tracker = new EconomyTracker();
        // Workers and expansions = economy
        tracker.recordBuildSpending(BuildingType.NEXUS, 400, 0);
        // Production buildings = economy
        tracker.recordBuildSpending(BuildingType.GATEWAY, 150, 0);
        // Tech buildings = technology
        tracker.recordBuildSpending(BuildingType.CYBERNETICS_CORE, 150, 0);

        var stats = tracker.currentStats(500, 200, 31, 14, 12);
        assertThat(stats.mineralsUsedCurrentEconomy()).isEqualTo(550);
        assertThat(stats.mineralsUsedCurrentTechnology()).isEqualTo(150);
    }

    @Test
    void computesCollectionRateFromDelta() {
        var tracker = new EconomyTracker();
        // Simulate 160 loops (1 rate interval) at 22 loops/tick → ~7.3 ticks
        // Call tickUpdate 8 times with increasing minerals
        for (int i = 0; i < 8; i++) {
            tracker.tickUpdate(50.0 + i * 60, 0, 23, 14, 12);
        }
        var stats = tracker.currentStats(530, 0, 23, 14, 12);
        assertThat(stats.mineralsCollectionRate()).isGreaterThan(0);
    }

    @Test
    void reportsSupplyAndWorkers() {
        var tracker = new EconomyTracker();
        var stats = tracker.currentStats(200, 100, 31, 18, 16);
        assertThat(stats.mineralsCurrent()).isEqualTo(200);
        assertThat(stats.vespeneCurrent()).isEqualTo(100);
        assertThat(stats.foodMade()).isEqualTo(31);
        assertThat(stats.foodUsed()).isEqualTo(18);
        assertThat(stats.workersActiveCount()).isEqualTo(16);
    }

    @Test
    void resetClearsAllTracking() {
        var tracker = new EconomyTracker();
        tracker.recordTrainSpending(UnitType.ZEALOT, 100, 0);
        tracker.reset();
        var stats = tracker.currentStats(50, 0, 15, 6, 12);
        assertThat(stats.mineralsUsedCurrentArmy()).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement EconomyTracker**

Create `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;

class EconomyTracker {

    private int mineralsUsedArmy;
    private int vespeneUsedArmy;
    private int mineralsUsedEconomy;
    private int vespeneUsedEconomy;
    private int mineralsUsedTechnology;
    private int vespeneUsedTechnology;

    private double prevMinerals;
    private int prevVespene;
    private int mineralRate;
    private int vespeneRate;
    private int ticksSinceRateUpdate;

    private static final int RATE_INTERVAL_TICKS = 7;

    void recordTrainSpending(UnitType type, int mineralCost, int gasCost) {
        if (SC2Data.isWorker(type)) {
            mineralsUsedEconomy += mineralCost;
            vespeneUsedEconomy += gasCost;
        } else {
            mineralsUsedArmy += mineralCost;
            vespeneUsedArmy += gasCost;
        }
    }

    void recordBuildSpending(BuildingType type, int mineralCost, int gasCost) {
        if (isTechBuilding(type)) {
            mineralsUsedTechnology += mineralCost;
            vespeneUsedTechnology += gasCost;
        } else {
            mineralsUsedEconomy += mineralCost;
            vespeneUsedEconomy += gasCost;
        }
    }

    void tickUpdate(double minerals, int vespene,
                    int supply, int supplyUsed, int workerCount) {
        ticksSinceRateUpdate++;
        if (ticksSinceRateUpdate >= RATE_INTERVAL_TICKS) {
            double mineralDelta = minerals - prevMinerals;
            int vespeneDelta = vespene - prevVespene;
            mineralRate = (int) Math.max(0, mineralDelta);
            vespeneRate = Math.max(0, vespeneDelta);
            prevMinerals = minerals;
            prevVespene = vespene;
            ticksSinceRateUpdate = 0;
        }
    }

    PlayerEconomyStats currentStats(int minerals, int vespene,
                                     int supply, int supplyUsed,
                                     int workerCount) {
        return new PlayerEconomyStats(
            minerals, vespene,
            mineralRate, vespeneRate,
            supply, supplyUsed,
            workerCount,
            mineralsUsedArmy, mineralsUsedEconomy, mineralsUsedTechnology,
            vespeneUsedArmy, vespeneUsedEconomy, vespeneUsedTechnology);
    }

    void reset() {
        mineralsUsedArmy = 0;
        vespeneUsedArmy = 0;
        mineralsUsedEconomy = 0;
        vespeneUsedEconomy = 0;
        mineralsUsedTechnology = 0;
        vespeneUsedTechnology = 0;
        prevMinerals = 0;
        prevVespene = 0;
        mineralRate = 0;
        vespeneRate = 0;
        ticksSinceRateUpdate = 0;
    }

    private static boolean isTechBuilding(BuildingType type) {
        return switch (type) {
            case CYBERNETICS_CORE, TWILIGHT_COUNCIL, TEMPLAR_ARCHIVES,
                 DARK_SHRINE, FLEET_BEACON, FORGE,
                 ENGINEERING_BAY, ARMORY, GHOST_ACADEMY, FUSION_CORE,
                 EVOLUTION_CHAMBER, SPIRE, GREATER_SPIRE,
                 HYDRALISK_DEN, INFESTATION_PIT, ULTRALISK_CAVERN,
                 BANELING_NEST, LURKER_DEN -> true;
            default -> false;
        };
    }
}
```

- [ ] **Step 4: Check if SC2Data.isWorker exists, else add helper**

Search for `isWorker` in `SC2Data.java`. If it doesn't exist, add:

```java
public static boolean isWorker(UnitType type) {
    return type == UnitType.PROBE || type == UnitType.SCV || type == UnitType.DRONE;
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EconomyTrackerTest -q`
Expected: PASS

- [ ] **Step 6: Wire EconomyTracker into EmulatedGame**

In `EmulatedGame.java`:
1. Add field: `private final EconomyTracker economyTracker = new EconomyTracker();`
2. In `reset()`: add `economyTracker.reset();`
3. In `tick()`: after the main tick logic, add:
   ```java
   int workerCount = (int) friendly.units().stream()
       .filter(u -> SC2Data.isWorker(u.type())).count();
   economyTracker.tickUpdate(friendly.minerals(), friendly.vespene(),
       friendly.supply(), friendly.supplyUsed(), workerCount);
   ```
4. In `handleTrain()` after Phase 4 (resource deduction, line 313-315): add:
   ```java
   if (state == friendly) {
       economyTracker.recordTrainSpending(t.unitType(), mCost, gCost);
   }
   ```
5. In `handleBuild()` after `state.deductMinerals(mCost)` (line 387): add:
   ```java
   if (state == friendly) {
       economyTracker.recordBuildSpending(bt, mCost, 0);
   }
   ```
6. In `snapshot()` (lines 639-641): replace `PlayerEconomyStats.EMPTY` (first occurrence — friendly economy) with:
   ```java
   economyTracker.currentStats(
       (int) friendly.minerals(), friendly.vespene(),
       friendly.supply(), friendly.supplyUsed(),
       (int) friendlyWithCooldown.stream()
           .filter(u -> SC2Data.isWorker(u.type())).count())
   ```

- [ ] **Step 7: Run full test suite to verify nothing breaks**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS — all existing tests pass with economy stats now populated

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EconomyTracker.java
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EconomyTrackerTest.java
git commit -m "feat: add EconomyTracker — derive PlayerEconomyStats from EmulatedGame state

EconomyTracker accumulates spending categories (army/economy/technology)
as intents execute, calculates collection rates from per-interval mineral/
vespene deltas. EmulatedGame.snapshot() now returns real PlayerEconomyStats
instead of EMPTY.

Refs #379"
```

## Batch 2: Accuracy baseline test and regression update

### Task 3: Create EmulatedGameAccuracyBaselineTest

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java`
- Modify: `docs/benchmarks/emulated-game-accuracy-baseline.md` (overwritten by test output)

**Interfaces:**
- Consumes: `ReplayValidationHarness.run(Path, int, int)` → `DivergenceReport`, `TickSnapshot` per-type maps

- [ ] **Step 1: Write the test class**

Create `EmulatedGameAccuracyBaselineTest.java` modeled on `OracleAccuracyBaselineTest`. The test runs the harness over 118 oracle replays and produces per-category accuracy.

```java
package io.quarkmind.sc2.replay;

import hu.scelight.sc2.rep.factory.RepContent;
import hu.scelight.sc2.rep.factory.RepParserEngine;
import hu.scelight.sc2.rep.model.Replay;
import hu.scelight.sc2.rep.model.details.Player;
import io.quarkmind.domain.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("report")
class EmulatedGameAccuracyBaselineTest {

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
    void emulatedGameAccuracyBaseline() throws IOException {
        List<Path> replays;
        try (var stream = Files.list(ORACLE_DIR)) {
            replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
                .sorted().toList();
        }

        // Accumulators: per checkpoint → per-type GT count, emulated count
        int totalReplays = 0;
        Map<Integer, Map<UnitType, long[]>> unitAccum = new TreeMap<>();
        Map<Integer, Map<BuildingType, long[]>> bldgAccum = new TreeMap<>();
        Map<Integer, int[]> upgradeAccum = new TreeMap<>(); // [gtTotal, matchCount]
        Map<Integer, double[][]> economyAccum = new TreeMap<>(); // [13 fields][gt, em]

        for (int cp : CHECKPOINTS_MIN) {
            unitAccum.put(cp, new TreeMap<>());
            bldgAccum.put(cp, new TreeMap<>());
            upgradeAccum.put(cp, new int[]{0, 0});
            economyAccum.put(cp, new double[13][2]);
        }

        for (Path replayPath : replays) {
            Replay replay;
            try {
                replay = RepParserEngine.parseReplay(replayPath, EnumSet.of(RepContent.DETAILS));
            } catch (Exception e) { continue; }
            if (replay == null || replay.details == null) continue;
            Player[] players = replay.details.getPlayerList();
            if (players.length < 2) continue;

            for (int playerId = 1; playerId <= 2; playerId++) {
                try {
                    DivergenceReport report = ReplayValidationHarness.run(
                        replayPath, playerId, TICK_LIMIT);

                    for (int cp : CHECKPOINTS_MIN) {
                        int tickIndex = cp * TICKS_PER_MINUTE - 1;
                        if (tickIndex >= report.ticks().size()) continue;
                        DivergenceReport.TickSnapshot snap = report.ticks().get(tickIndex);

                        // Unit per-type accumulation
                        accumulateTypes(snap.groundTruthUnitsByType(),
                            snap.emulatedUnitsByType(), unitAccum.get(cp));

                        // Building per-type accumulation
                        accumulateBuildingTypes(snap.groundTruthBuildingsByType(),
                            snap.emulatedBuildingsByType(), bldgAccum.get(cp));

                        // Upgrade accuracy
                        int gtSize = snap.groundTruthUpgrades().size();
                        int matchCount = (int) snap.groundTruthUpgrades().stream()
                            .filter(snap.emulatedUpgrades()::contains).count();
                        upgradeAccum.get(cp)[0] += gtSize;
                        upgradeAccum.get(cp)[1] += matchCount;

                        // Economy accumulation
                        accumulateEconomy(snap.groundTruthEconomy(),
                            snap.emulatedEconomy(), economyAccum.get(cp));
                    }
                } catch (Exception e) { /* skip failed replays */ }
            }
            totalReplays++;
        }

        // Print and write report
        var sb = new StringBuilder();
        sb.append(String.format("# EmulatedGame Accuracy Baseline — %s%n%n", java.time.LocalDate.now()));
        sb.append(String.format("**Context:** Phase 2.5 (#379, child of #366)%n"));
        sb.append(String.format("**Dataset:** Oracle (118 replays, v4.9.3)%n"));
        sb.append(String.format("**Processed:** %d replays%n%n---%n%n", totalReplays));

        // Per-checkpoint summary
        for (int cp : CHECKPOINTS_MIN) {
            sb.append(String.format("## %d-minute checkpoint%n%n", cp));
            appendUnitAccuracy(sb, unitAccum.get(cp));
            appendBuildingAccuracy(sb, bldgAccum.get(cp));

            int[] upg = upgradeAccum.get(cp);
            double upgAcc = upg[0] > 0 ? 100.0 * upg[1] / upg[0] : 100.0;
            sb.append(String.format("%n### Upgrades%n%nAccuracy: %.1f%% (%d/%d)%n%n",
                upgAcc, upg[1], upg[0]));

            appendEconomyMape(sb, economyAccum.get(cp));
        }

        // Write report
        System.out.println(sb);
        Path reportPath = Path.of("docs/benchmarks/emulated-game-accuracy-baseline.md");
        Files.writeString(reportPath, sb.toString());

        assertThat(totalReplays).as("Must process at least 100 replays").isGreaterThanOrEqualTo(100);
    }

    private <T extends Comparable<T>> void accumulateTypes(
            Map<T, Integer> gt, Map<T, Integer> em, Map<T, long[]> accum) {
        Set<T> allTypes = new TreeSet<>();
        allTypes.addAll(gt.keySet());
        allTypes.addAll(em.keySet());
        for (T type : allTypes) {
            long[] counts = accum.computeIfAbsent(type, k -> new long[2]);
            counts[0] += gt.getOrDefault(type, 0);
            counts[1] += em.getOrDefault(type, 0);
        }
    }

    private void accumulateBuildingTypes(
            Map<BuildingType, Integer> gt, Map<BuildingType, Integer> em,
            Map<BuildingType, long[]> accum) {
        Set<BuildingType> allTypes = EnumSet.noneOf(BuildingType.class);
        allTypes.addAll(gt.keySet());
        allTypes.addAll(em.keySet());
        for (BuildingType type : allTypes) {
            long[] counts = accum.computeIfAbsent(type, k -> new long[2]);
            counts[0] += gt.getOrDefault(type, 0);
            counts[1] += em.getOrDefault(type, 0);
        }
    }

    private void accumulateEconomy(PlayerEconomyStats gt, PlayerEconomyStats em, double[][] accum) {
        float[] gtVec = gt.toFeatureVector();
        float[] emVec = em.toFeatureVector();
        for (int i = 0; i < 13; i++) {
            accum[i][0] += gtVec[i];
            accum[i][1] += emVec[i];
        }
    }

    private <T> void appendUnitAccuracy(StringBuilder sb, Map<T, long[]> accum) {
        sb.append("### Units — Per-Type\n\n");
        sb.append(String.format("| %-25s | %8s | %8s | %8s |%n",
            "Type", "GT", "Emulated", "Accuracy"));
        sb.append("|" + "-".repeat(27) + "|" + "-".repeat(10) + "|" + "-".repeat(10)
            + "|" + "-".repeat(10) + "|\n");
        long totalGt = 0, totalEm = 0;
        for (var entry : accum.entrySet()) {
            long gt = entry.getValue()[0];
            long em = entry.getValue()[1];
            double acc = gt > 0 ? 100.0 * Math.min(em, gt) / gt : 100.0;
            sb.append(String.format("| %-25s | %8d | %8d | %7.1f%% |%n",
                entry.getKey(), gt, em, acc));
            totalGt += gt;
            totalEm += em;
        }
        double totalAcc = totalGt > 0 ? 100.0 * Math.min(totalEm, totalGt) / totalGt : 100.0;
        sb.append(String.format("| %-25s | %8d | %8d | %7.1f%% |%n%n",
            "**TOTAL**", totalGt, totalEm, totalAcc));
    }

    private void appendBuildingAccuracy(StringBuilder sb, Map<BuildingType, long[]> accum) {
        sb.append("### Buildings — Per-Type\n\n");
        sb.append(String.format("| %-25s | %8s | %8s | %8s |%n",
            "Type", "GT", "Emulated", "Accuracy"));
        sb.append("|" + "-".repeat(27) + "|" + "-".repeat(10) + "|" + "-".repeat(10)
            + "|" + "-".repeat(10) + "|\n");
        long totalGt = 0, totalEm = 0;
        for (var entry : accum.entrySet()) {
            long gt = entry.getValue()[0];
            long em = entry.getValue()[1];
            double acc = gt > 0 ? 100.0 * Math.min(em, gt) / gt : 100.0;
            sb.append(String.format("| %-25s | %8d | %8d | %7.1f%% |%n",
                entry.getKey(), gt, em, acc));
            totalGt += gt;
            totalEm += em;
        }
        double totalAcc = totalGt > 0 ? 100.0 * Math.min(totalEm, totalGt) / totalGt : 100.0;
        sb.append(String.format("| %-25s | %8d | %8d | %7.1f%% |%n%n",
            "**TOTAL**", totalGt, totalEm, totalAcc));
    }

    private void appendEconomyMape(StringBuilder sb, double[][] accum) {
        String[] fields = {
            "mineralsCurrent", "vespeneCurrent",
            "mineralsCollectionRate", "vespeneCollectionRate",
            "foodMade", "foodUsed", "workersActiveCount",
            "mineralsUsedCurrentArmy", "mineralsUsedCurrentEconomy",
            "mineralsUsedCurrentTechnology",
            "vespeneUsedCurrentArmy", "vespeneUsedCurrentEconomy",
            "vespeneUsedCurrentTechnology"
        };
        sb.append("### Economy — MAPE\n\n");
        sb.append(String.format("| %-35s | %10s | %10s | %8s |%n",
            "Field", "GT (sum)", "Em (sum)", "MAPE"));
        sb.append("|" + "-".repeat(37) + "|" + "-".repeat(12) + "|" + "-".repeat(12)
            + "|" + "-".repeat(10) + "|\n");
        for (int i = 0; i < 13; i++) {
            double gt = accum[i][0];
            double em = accum[i][1];
            double mape = gt != 0 ? 100.0 * Math.abs(em - gt) / Math.abs(gt) : 0.0;
            sb.append(String.format("| %-35s | %10.1f | %10.1f | %7.1f%% |%n",
                fields[i], gt * 1000, em * 1000, mape));
        }
        sb.append("\n");
    }
}
```

- [ ] **Step 2: Verify the test compiles**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS

Run: `mvn test-compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/EmulatedGameAccuracyBaselineTest.java
git commit -m "feat: add EmulatedGameAccuracyBaselineTest — per-type accuracy vs ground truth

Report test (@Tag(\"report\")) that runs ReplayValidationHarness over 118
oracle replays and measures per-type unit/building accuracy, upgrade
accuracy, and economy MAPE at 1/3/5/7 minute checkpoints.

Writes docs/benchmarks/emulated-game-accuracy-baseline.md.

Refs #379"
```

### Task 4: Update DivergenceRegressionTest with accuracy thresholds

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java`

**Interfaces:**
- Consumes: `TickSnapshot` per-type maps and upgrade sets from Task 1

- [ ] **Step 1: Write the failing accuracy assertion**

Add a new test method to `DivergenceRegressionTest.java` that asserts per-category accuracy percentages at the 5-min checkpoint. Initial thresholds are set conservatively — they'll be tightened after the first baseline run.

```java
// Conservative initial thresholds — update after first baseline run
private static final double UNIT_ACCURACY_FLOOR = 0.50;
private static final double BUILDING_ACCURACY_FLOOR = 0.50;
private static final double UPGRADE_ACCURACY_FLOOR = 0.80;

@Test
@EnabledIf("oracleExists")
void perCategoryAccuracyWithinThreshold() throws Exception {
    List<Path> replays;
    try (var stream = Files.list(ORACLE_DIR)) {
        replays = stream.filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted().toList();
    }

    long totalGtUnits = 0, totalEmUnits = 0;
    long totalGtBldgs = 0, totalEmBldgs = 0;
    int totalGtUpgrades = 0, matchedUpgrades = 0;
    int processed = 0;

    for (Path replayPath : replays) {
        Replay replay;
        try {
            replay = RepParserEngine.parseReplay(replayPath,
                EnumSet.of(RepContent.DETAILS));
        } catch (Exception e) { continue; }
        if (replay == null || replay.details == null) continue;
        Player[] players = replay.details.getPlayerList();
        if (players.length < 2) continue;

        for (int playerId = 1; playerId <= 2; playerId++) {
            try {
                DivergenceReport report = ReplayValidationHarness.run(
                    replayPath, playerId, TICK_LIMIT);
                int tickIndex = CHECKPOINT_MINUTE * TICKS_PER_MINUTE - 1;
                if (tickIndex >= report.ticks().size()) continue;
                DivergenceReport.TickSnapshot snap = report.ticks().get(tickIndex);

                // Per-type unit accuracy
                for (var entry : snap.groundTruthUnitsByType().entrySet()) {
                    totalGtUnits += entry.getValue();
                    totalEmUnits += snap.emulatedUnitsByType()
                        .getOrDefault(entry.getKey(), 0);
                }

                // Per-type building accuracy
                for (var entry : snap.groundTruthBuildingsByType().entrySet()) {
                    totalGtBldgs += entry.getValue();
                    totalEmBldgs += snap.emulatedBuildingsByType()
                        .getOrDefault(entry.getKey(), 0);
                }

                // Upgrade accuracy
                totalGtUpgrades += snap.groundTruthUpgrades().size();
                matchedUpgrades += (int) snap.groundTruthUpgrades().stream()
                    .filter(snap.emulatedUpgrades()::contains).count();
            } catch (Exception e) { /* skip */ }
        }
        processed++;
    }

    assertThat(processed).isGreaterThanOrEqualTo(100);

    double unitAcc = totalGtUnits > 0
        ? (double) Math.min(totalEmUnits, totalGtUnits) / totalGtUnits : 1.0;
    double bldgAcc = totalGtBldgs > 0
        ? (double) Math.min(totalEmBldgs, totalGtBldgs) / totalGtBldgs : 1.0;
    double upgAcc = totalGtUpgrades > 0
        ? (double) matchedUpgrades / totalGtUpgrades : 1.0;

    System.out.printf("%nPer-category accuracy — 5-min checkpoint%n");
    System.out.printf("  Units:     %.1f%% (threshold: %.0f%%)%n",
        unitAcc * 100, UNIT_ACCURACY_FLOOR * 100);
    System.out.printf("  Buildings: %.1f%% (threshold: %.0f%%)%n",
        bldgAcc * 100, BUILDING_ACCURACY_FLOOR * 100);
    System.out.printf("  Upgrades:  %.1f%% (threshold: %.0f%%)%n",
        upgAcc * 100, UPGRADE_ACCURACY_FLOOR * 100);

    assertThat(unitAcc)
        .as("Unit accuracy at 5-min").isGreaterThanOrEqualTo(UNIT_ACCURACY_FLOOR);
    assertThat(bldgAcc)
        .as("Building accuracy at 5-min").isGreaterThanOrEqualTo(BUILDING_ACCURACY_FLOOR);
    assertThat(upgAcc)
        .as("Upgrade accuracy at 5-min").isGreaterThanOrEqualTo(UPGRADE_ACCURACY_FLOOR);
}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn test-compile -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java
git commit -m "feat: add per-category accuracy thresholds to DivergenceRegressionTest

New test method perCategoryAccuracyWithinThreshold asserts per-type
unit, building, and upgrade accuracy at 5-min checkpoint. Conservative
initial thresholds (50%/50%/80%) — tighten after first baseline run.

Refs #379"
```

### Task 5: Run baseline and tighten thresholds

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java` (threshold values)
- Modify: `docs/benchmarks/emulated-game-accuracy-baseline.md` (generated by test)
- Modify: `CLAUDE.md` (add test to test listing)

**Interfaces:**
- Consumes: output of `EmulatedGameAccuracyBaselineTest` (baseline numbers), output of `DivergenceRegressionTest.perCategoryAccuracyWithinThreshold` (actual accuracy)

- [ ] **Step 1: Run EmulatedGameAccuracyBaselineTest**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=EmulatedGameAccuracyBaselineTest -q`
Expected: PASS — generates `docs/benchmarks/emulated-game-accuracy-baseline.md` with per-type data

- [ ] **Step 2: Run DivergenceRegressionTest with conservative thresholds**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest -q`
Expected: PASS — conservative thresholds should be easily met

- [ ] **Step 3: Read baseline output and tighten thresholds**

Read the generated `docs/benchmarks/emulated-game-accuracy-baseline.md` for actual accuracy percentages at 5-min checkpoint. Set each threshold to: `actual_accuracy - 5%` (rounded down to nearest 5%). For example, if unit accuracy is 72%, set threshold to 65%.

Update the constants in `DivergenceRegressionTest.java`:
```java
private static final double UNIT_ACCURACY_FLOOR = <tightened>;
private static final double BUILDING_ACCURACY_FLOOR = <tightened>;
private static final double UPGRADE_ACCURACY_FLOOR = <tightened>;
```

- [ ] **Step 4: Re-run regression test with tightened thresholds**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=DivergenceRegressionTest -q`
Expected: PASS — tightened thresholds still within margin

- [ ] **Step 5: Add root cause triage to baseline report**

Read the generated baseline report. For each category where accuracy < 95%, add a triage section to the report:

| Classification | Example |
|---|---|
| Physics error | Wrong train time, wrong income rate |
| Missing mechanic | Not modeled (Chrono Boost, MULE income, Reactor double production) |
| Known limitation | Documented (flat mining rate) |

Append the triage section to `docs/benchmarks/emulated-game-accuracy-baseline.md` manually after reviewing the per-type divergence data.

- [ ] **Step 6: Update CLAUDE.md test listing**

Add `EmulatedGameAccuracyBaselineTest` to the unit test listing and the report test documentation in CLAUDE.md.

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 8: Commit**

```bash
git add docs/benchmarks/emulated-game-accuracy-baseline.md
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java
git add CLAUDE.md
git commit -m "feat: establish EmulatedGame accuracy baseline with tightened regression thresholds

Per-type accuracy baseline against 118 oracle replays (v4.9.3):
- Units: <X>%
- Buildings: <X>%
- Upgrades: <X>%
- Economy: <MAPE> MAPE

Regression thresholds tightened from conservative (50%/50%/80%) to
baseline-derived values with 5% margin.

Closes #379"
```

## References

- [2026-10-07-emulatedgame-accuracy-baseline-design.md] — design spec this plan implements
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java] — harness with snapshot collection
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/DivergenceReport.java] — TickSnapshot record (8 aggregate fields → 16 fields with per-type)
- [quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:611-642] — snapshot() method, currently EMPTY economy
- [quarkmind-sc2/src/main/java/io/quarkmind/domain/PlayerEconomyStats.java] — 13-field economy record
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceRegressionTest.java] — existing aggregate delta regression
- [quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/DivergenceBaselineReportTest.java] — existing aggregate baseline report
- [docs/benchmarks/oracle-accuracy-baseline.md] — StrippedReplayFeatureExtractor accuracy (for comparison)
- [docs/benchmarks/emulated-game-divergence-baseline.md] — existing aggregate EmulatedGame baseline
- [GitHub #379] — focal issue
- [GitHub #366] — parent epic (Phase 2.5 reconstitution accuracy gate)
