# Vespene Income Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #394 — EmulatedGame: model vespene income from gas buildings
**Issue group:** #394

**Goal:** Add vespene income accumulation to EmulatedGame so that workers on completed gas buildings generate gas per tick, mirroring the existing mineral income model.

**Architecture:** Tiered gas income rates in SC2Data (parallel to MINERAL_TIER_RATES_PER_TICK). EmulatedGame.tick() computes gas worker budget (3 × completed gas buildings, capped at total workers), deducts from mineral worker counts, and accumulates vespene. PlayerState gains a double-precision vespene field with addVespene(). Playbook doAssert() gains vespene bounds checking.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- Gas income rates are community-sourced (38/38/20 gas/min per worker tier), not replay-calibrated — follow-up per PP-20260522-572156
- Domain model (`domain/`) must remain plain Java — no CDI, no Quarkus
- Never use `@QuarkusTest` for tests that can be plain JUnit
- All gas building types: ASSIMILATOR, ASSIMILATOR_RICH, REFINERY, EXTRACTOR

---

## Batch 1: Foundation — SC2Data gas constants + PlayerState vespene precision

### Task 1: SC2Data gas income constants and methods

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/domain/SC2DataGasIncomeTest.java`

**Interfaces:**
- Produces:
  - `SC2Data.GAS_WORKERS_PER_BUILDING` → `int` (3)
  - `SC2Data.GAS_TIER_RATES_PER_TICK` → `double[]` (3 elements)
  - `SC2Data.gasIncomePerTick(int workerCount)` → `double`
  - `SC2Data.isGasBuilding(BuildingType type)` → `boolean`

- [ ] **Step 1: Write failing tests for gasIncomePerTick and isGasBuilding**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.within;

class SC2DataGasIncomeTest {

    @Test
    void gasIncomePerTick_zeroWorkers_returnsZero() {
        assertThat(SC2Data.gasIncomePerTick(0)).isEqualTo(0.0);
    }

    @Test
    void gasIncomePerTick_oneWorker_returnsTier0Only() {
        double expected = SC2Data.GAS_TIER_RATES_PER_TICK[0];
        assertThat(SC2Data.gasIncomePerTick(1)).isCloseTo(expected, within(0.0001));
    }

    @Test
    void gasIncomePerTick_twoWorkers_returnsTier0PlusTier1() {
        double expected = SC2Data.GAS_TIER_RATES_PER_TICK[0]
                        + SC2Data.GAS_TIER_RATES_PER_TICK[1];
        assertThat(SC2Data.gasIncomePerTick(2)).isCloseTo(expected, within(0.0001));
    }

    @Test
    void gasIncomePerTick_threeWorkers_returnsAllTiers() {
        double expected = SC2Data.GAS_TIER_RATES_PER_TICK[0]
                        + SC2Data.GAS_TIER_RATES_PER_TICK[1]
                        + SC2Data.GAS_TIER_RATES_PER_TICK[2];
        assertThat(SC2Data.gasIncomePerTick(3)).isCloseTo(expected, within(0.0001));
    }

    @Test
    void gasIncomePerTick_beyondThreeWorkers_capsAtThreeTiers() {
        assertThat(SC2Data.gasIncomePerTick(5))
            .isCloseTo(SC2Data.gasIncomePerTick(3), within(0.0001));
    }

    @Test
    void gasIncomePerTick_negativeWorkers_throws() {
        assertThatThrownBy(() -> SC2Data.gasIncomePerTick(-1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void isGasBuilding_allGasTypes_returnTrue() {
        assertThat(SC2Data.isGasBuilding(BuildingType.ASSIMILATOR)).isTrue();
        assertThat(SC2Data.isGasBuilding(BuildingType.ASSIMILATOR_RICH)).isTrue();
        assertThat(SC2Data.isGasBuilding(BuildingType.REFINERY)).isTrue();
        assertThat(SC2Data.isGasBuilding(BuildingType.EXTRACTOR)).isTrue();
    }

    @Test
    void isGasBuilding_nonGasTypes_returnFalse() {
        assertThat(SC2Data.isGasBuilding(BuildingType.NEXUS)).isFalse();
        assertThat(SC2Data.isGasBuilding(BuildingType.GATEWAY)).isFalse();
        assertThat(SC2Data.isGasBuilding(BuildingType.COMMAND_CENTER)).isFalse();
        assertThat(SC2Data.isGasBuilding(BuildingType.HATCHERY)).isFalse();
    }

    @Test
    void gasTierRates_hasThreeElements() {
        assertThat(SC2Data.GAS_TIER_RATES_PER_TICK).hasSize(3);
    }

    @Test
    void gasWorkersPerBuilding_isThree() {
        assertThat(SC2Data.GAS_WORKERS_PER_BUILDING).isEqualTo(3);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataGasIncomeTest -q`
Expected: compilation failure — `gasIncomePerTick`, `isGasBuilding`, `GAS_TIER_RATES_PER_TICK`, `GAS_WORKERS_PER_BUILDING` do not exist yet

- [ ] **Step 3: Implement gas constants and methods in SC2Data**

Add after `mineralIncomePerTick()` (after line 60 in SC2Data.java):

```java
public static final int GAS_WORKERS_PER_BUILDING = 3;

public static final double[] GAS_TIER_RATES_PER_TICK = {
    38.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,  // worker 1  (~0.622 gas/tick)
    38.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,  // worker 2  (~0.622 gas/tick)
    20.0 / 60.0 * LOOPS_PER_TICK / GAME_LOOPS_PER_SECOND,  // worker 3  (~0.327 gas/tick)
};

public static double gasIncomePerTick(final int workerCount) {
    if (workerCount < 0) throw new IllegalArgumentException("workerCount must be >= 0, got: " + workerCount);
    double income = 0;
    final int effectiveWorkers = Math.min(workerCount, GAS_TIER_RATES_PER_TICK.length);
    for (int i = 0; i < effectiveWorkers; i++) {
        income += GAS_TIER_RATES_PER_TICK[i];
    }
    return income;
}

public static boolean isGasBuilding(final BuildingType type) {
    return type == BuildingType.ASSIMILATOR || type == BuildingType.ASSIMILATOR_RICH
        || type == BuildingType.REFINERY || type == BuildingType.EXTRACTOR;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DataGasIncomeTest -q`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java quarkmind-sc2/src/test/java/io/quarkmind/domain/SC2DataGasIncomeTest.java
git commit -m "feat: add gas income constants and methods to SC2Data Refs #394"
```

---

### Task 2: PlayerState vespene field migration to double

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`

**Interfaces:**
- Consumes: none
- Produces:
  - `PlayerState.addVespene(double amount)` → `void`
  - `PlayerState.vespene()` → `int` (unchanged public API — casts internal double to int)

**Note:** `PlayerStateView.vespene()` returns `int` — no interface change needed. The internal field changes to `double` for accumulation precision, but the accessor truncates to `int`.

- [ ] **Step 1: Write failing test for addVespene**

Add to a new test file:

```java
package io.quarkmind.sc2.emulated;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class PlayerStateVespeneTest {

    @Test
    void addVespene_accumulatesFromZero() {
        var state = new PlayerState();
        state.addVespene(0.622);
        state.addVespene(0.622);
        state.addVespene(0.622);
        assertThat(state.vespene()).isEqualTo(1);
    }

    @Test
    void addVespene_accumulatesPrecisely() {
        var state = new PlayerState();
        for (int i = 0; i < 100; i++) {
            state.addVespene(0.622);
        }
        assertThat(state.vespene()).isEqualTo(62);
    }

    @Test
    void setVespene_overwritesAccumulated() {
        var state = new PlayerState();
        state.addVespene(50.0);
        state.setVespene(100);
        assertThat(state.vespene()).isEqualTo(100);
    }

    @Test
    void deductVespene_worksWithDoubleField() {
        var state = new PlayerState();
        state.setVespene(100);
        state.deductVespene(25);
        assertThat(state.vespene()).isEqualTo(75);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PlayerStateVespeneTest -q`
Expected: compilation failure — `addVespene(double)` does not exist

- [ ] **Step 3: Implement addVespene and change field to double**

In `PlayerState.java`, change line 28 from:
```java
private int    vespene;
```
to:
```java
private double vespene;
```

Change the vespene methods (lines 39-41) to:
```java
public void setVespene(int v)            { this.vespene = v; }
public void addVespene(double amount)    { this.vespene += amount; }
public void deductVespene(int cost)      { this.vespene -= cost; }
public int  vespene()                    { return (int) vespene; }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PlayerStateVespeneTest -q`
Expected: all 4 tests PASS

- [ ] **Step 5: Run full test suite to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all existing tests PASS — the `int` return on `vespene()` is unchanged, so no caller breaks

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/PlayerState.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/PlayerStateVespeneTest.java
git commit -m "feat: change PlayerState vespene to double, add addVespene() Refs #394"
```

---

## Batch 2: Core Feature — gas income in EmulatedGame + playbook assertions

### Task 3: Gas income accumulation in EmulatedGame.tick()

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameGasIncomeTest.java`

**Interfaces:**
- Consumes:
  - `SC2Data.gasIncomePerTick(int)` → `double`
  - `SC2Data.isGasBuilding(BuildingType)` → `boolean`
  - `SC2Data.GAS_WORKERS_PER_BUILDING` → `int` (3)
  - `PlayerState.addVespene(double)` → `void`
- Produces: gas income accumulation in `tick()` — no new public API

**Design detail — gas worker deduction from mineral counts:**

After `countWorkersPerBase` runs (assigns ALL workers to bases), deduct gas workers from the per-base counts. Remove from the largest base first (saturated bases are where workers move to gas in real SC2). Clamp each base to zero minimum.

**Gas income placement in tick():** immediately after the mineral income loop (line 124), before `tickPassive`. This ensures:
1. Gas income is computed after `countWorkersPerBase` (which runs at line 118)
2. Gas income is available before `tickPassive` (line 125) reads state
3. Consistent timing with mineral income

- [ ] **Step 1: Write failing test — gas building produces vespene**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.BuildIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class EmulatedGameGasIncomeTest {

    EmulatedGame game;

    @BeforeEach
    void setup() {
        game = new EmulatedGame(Race.PROTOSS);
    }

    @Test
    void completedAssimilator_producesVespeneIncome() {
        GameState before = game.snapshot();
        assertThat(before.vespene()).isEqualTo(0);

        game.applyIntent(new BuildIntent(BuildingType.ASSIMILATOR));
        int buildTime = SC2Data.buildTimeInLoops(BuildingType.ASSIMILATOR)
                      / SC2Data.LOOPS_PER_TICK + 1;
        for (int i = 0; i < buildTime; i++) game.tick();

        GameState afterBuild = game.snapshot();
        boolean assimilatorComplete = afterBuild.myBuildings().stream()
            .anyMatch(b -> b.type() == BuildingType.ASSIMILATOR && b.isComplete());
        assertThat(assimilatorComplete)
            .as("Assimilator must be complete after build time")
            .isTrue();

        int extraTicks = 50;
        for (int i = 0; i < extraTicks; i++) game.tick();

        GameState after = game.snapshot();
        assertThat(after.vespene())
            .as("Vespene should accumulate after gas building completes")
            .isGreaterThan(0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameGasIncomeTest -q`
Expected: FAIL — vespene stays at 0 because tick() has no gas income logic

- [ ] **Step 3: Write failing test — gas workers reduce mineral income**

Add to `EmulatedGameGasIncomeTest`:

```java
@Test
void gasBuilding_reducesMineralIncome() {
    // Run 100 ticks with no gas building — baseline mineral income
    for (int i = 0; i < 100; i++) game.tick();
    double mineralsWithoutGas = game.snapshot().minerals();

    // Reset and run 100 ticks with gas building present from start
    game = new EmulatedGame(Race.PROTOSS);
    game.applyIntent(new BuildIntent(BuildingType.ASSIMILATOR));
    // Fast-forward past build time
    int buildTime = SC2Data.buildTimeInLoops(BuildingType.ASSIMILATOR)
                  / SC2Data.LOOPS_PER_TICK + 1;
    for (int i = 0; i < buildTime; i++) game.tick();
    double mineralsAtBuildComplete = game.snapshot().minerals();

    // Now tick 100 more times with the gas building active
    for (int i = 0; i < 100; i++) game.tick();
    double mineralIncomeWithGas = game.snapshot().minerals() - mineralsAtBuildComplete;

    // Mineral income over 100 ticks should be lower with gas
    // (3 workers diverted from minerals to gas)
    assertThat(mineralIncomeWithGas)
        .as("Mineral income should decrease when workers are on gas")
        .isLessThan(mineralsWithoutGas);
}
```

- [ ] **Step 4: Write failing test — multiple gas buildings cap at total workers**

Add to `EmulatedGameGasIncomeTest`:

```java
@Test
void multipleGasBuildings_workerBudgetCapsAtTotalWorkers() {
    // Start with 12 probes (default). Build 5 gas buildings.
    // Gas worker budget = min(5 * 3, 12) = 12 — all workers on gas.
    // Mineral income should be ~zero.
    for (int i = 0; i < 5; i++) {
        game.applyIntent(new BuildIntent(BuildingType.ASSIMILATOR));
    }
    int buildTime = SC2Data.buildTimeInLoops(BuildingType.ASSIMILATOR)
                  / SC2Data.LOOPS_PER_TICK + 1;
    for (int i = 0; i < buildTime; i++) game.tick();

    double mineralsAfterBuild = game.snapshot().minerals();

    // Tick 100 more — all 12 workers should be on gas, 0 on minerals
    for (int i = 0; i < 100; i++) game.tick();

    double mineralIncome = game.snapshot().minerals() - mineralsAfterBuild;
    assertThat(mineralIncome)
        .as("No mineral income when all workers are on gas")
        .isCloseTo(0.0, org.assertj.core.api.Assertions.within(1.0));

    assertThat(game.snapshot().vespene())
        .as("Vespene income should accumulate from multiple gas buildings")
        .isGreaterThan(0);
}
```

- [ ] **Step 5: Implement gas income in EmulatedGame.tick()**

In `EmulatedGame.java`, modify the `tick()` method. After the existing mineral income loop (line 122-124), add gas income logic:

```java
public void tick() {
    if (!miningProbesOverridden) {
        miningProbesPerBase = countWorkersPerBase(playerRaceModel, friendly.buildings(), friendly.units());
    }
    miningProbesOverridden = false;
    gameFrame++;

    // Count completed gas buildings and compute gas worker budget
    final long completedGasBuildings = friendly.buildings().stream()
        .filter(b -> SC2Data.isGasBuilding(b.type()) && b.isComplete())
        .count();
    final int totalWorkers = (int) friendly.units().stream()
        .filter(u -> u.type() == playerRaceModel.workerType())
        .count();
    final int gasWorkerBudget = Math.min(
        (int) completedGasBuildings * SC2Data.GAS_WORKERS_PER_BUILDING,
        totalWorkers);

    // Deduct gas workers from mineral counts (remove from largest base first)
    int gasWorkersToDeduct = gasWorkerBudget;
    if (gasWorkersToDeduct > 0 && miningProbesPerBase.length > 0) {
        int[] sorted = java.util.stream.IntStream.range(0, miningProbesPerBase.length)
            .boxed()
            .sorted((a, b) -> Integer.compare(miningProbesPerBase[b], miningProbesPerBase[a]))
            .mapToInt(Integer::intValue)
            .toArray();
        for (int idx : sorted) {
            if (gasWorkersToDeduct <= 0) break;
            int deduct = Math.min(miningProbesPerBase[idx], gasWorkersToDeduct);
            miningProbesPerBase[idx] -= deduct;
            gasWorkersToDeduct -= deduct;
        }
    }

    // Mineral income (existing — now uses reduced per-base counts)
    for (final int workersAtBase : miningProbesPerBase) {
        friendly.addMinerals(SC2Data.mineralIncomePerTick(workersAtBase));
    }

    // Gas income
    int gasWorkersRemaining = gasWorkerBudget;
    for (final Building b : friendly.buildings()) {
        if (gasWorkersRemaining <= 0) break;
        if (!SC2Data.isGasBuilding(b.type()) || !b.isComplete()) continue;
        int workersOnThisGeyser = Math.min(SC2Data.GAS_WORKERS_PER_BUILDING, gasWorkersRemaining);
        friendly.addVespene(SC2Data.gasIncomePerTick(workersOnThisGeyser));
        gasWorkersRemaining -= workersOnThisGeyser;
    }

    playerRaceModel.tickPassive(friendly, gameFrame * (long) SC2Data.LOOPS_PER_TICK);
    // ... rest of tick() unchanged
```

Key points:
- Gas worker budget deduction happens BEFORE mineral income loop
- Gas income accumulation happens AFTER mineral income, BEFORE tickPassive
- Deduction from largest base first handles the saturated-base-first heuristic
- Each base count is clamped to zero via `Math.min(miningProbesPerBase[idx], gasWorkersToDeduct)`

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameGasIncomeTest -q`
Expected: all 3 tests PASS

- [ ] **Step 7: Run full test suite to verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all existing tests PASS — no existing test builds gas buildings, so behaviour is unchanged

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/EmulatedGameGasIncomeTest.java
git commit -m "feat: add vespene income accumulation to EmulatedGame.tick() Refs #394"
```

---

### Task 4: Playbook vespene assertions in doAssert()

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java`
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerVespeneAssertTest.java`

**Interfaces:**
- Consumes: `GameState.vespene()` → `int`
- Produces: `doAssert()` supports `vespene: {min: N, max: N}` in expect map

- [ ] **Step 1: Write failing test — vespene min assertion**

```java
package io.quarkmind.sc2.playbook;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.intent.BuildIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Map;

class SC2DeliveryHandlerVespeneAssertTest {

    EmulatedGame game;
    SC2DeliveryHandler handler;

    @BeforeEach
    void setup() {
        game = new EmulatedGame(Race.PROTOSS);
        handler = new SC2DeliveryHandler(game);
    }

    @Test
    void assertVespene_minBound_passes() {
        game.setVespene(100);
        var step = Map.of(
            "action", "assert",
            "expect", Map.<String, Object>of(
                "vespene", Map.of("min", 50)
            )
        );
        var result = handler.handle("check-gas", step);
        assertThat(result.ok()).isTrue();
    }

    @Test
    void assertVespene_minBound_fails() {
        game.setVespene(10);
        var step = Map.of(
            "action", "assert",
            "expect", Map.<String, Object>of(
                "vespene", Map.of("min", 50)
            )
        );
        assertThatThrownBy(() -> handler.handle("check-gas", step))
            .isInstanceOf(AssertionError.class)
            .hasMessageContaining("vespene expected >=50");
    }

    @Test
    void assertVespene_maxBound_passes() {
        game.setVespene(30);
        var step = Map.of(
            "action", "assert",
            "expect", Map.<String, Object>of(
                "vespene", Map.of("max", 100)
            )
        );
        var result = handler.handle("check-gas", step);
        assertThat(result.ok()).isTrue();
    }

    @Test
    void assertVespene_maxBound_fails() {
        game.setVespene(200);
        var step = Map.of(
            "action", "assert",
            "expect", Map.<String, Object>of(
                "vespene", Map.of("max", 100)
            )
        );
        assertThatThrownBy(() -> handler.handle("check-gas", step))
            .isInstanceOf(AssertionError.class)
            .hasMessageContaining("vespene expected <=100");
    }
}
```

**Note:** This test requires `SC2DeliveryHandler` to have a public constructor accepting `EmulatedGame` and a `handle(String, Map)` method that dispatches to `doAssert`. If the current API differs (e.g. `doAssert` is private and called from a playbook runner), adjust the test to use the playbook runner or make `doAssert` package-visible for testing. Check the actual `SC2DeliveryHandler` constructor and `handle` method signatures before writing.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerVespeneAssertTest -q`
Expected: FAIL — vespene key is not handled in doAssert

- [ ] **Step 3: Implement vespene bounds in doAssert()**

In `SC2DeliveryHandler.java`, in the `doAssert` method (after the minerals check, around line 149), add:

```java
if (expect.containsKey("vespene")) {
    @SuppressWarnings("unchecked")
    var vespeneBounds = (Map<String, Integer>) expect.get("vespene");
    int actual = state.vespene();
    if (vespeneBounds.containsKey("min") && actual < vespeneBounds.get("min")) {
        errors.add("vespene expected >=" + vespeneBounds.get("min") + " but was " + actual);
    }
    if (vespeneBounds.containsKey("max") && actual > vespeneBounds.get("max")) {
        errors.add("vespene expected <=" + vespeneBounds.get("max") + " but was " + actual);
    }
}
```

Also update the log line (around line 120-124) to include vespene:

```java
System.out.println("[PLAYBOOK] " + stepName
    + " units=" + state.myUnits().size()
    + " buildings=" + state.myBuildings().size()
    + " minerals=" + (int) state.minerals()
    + " vespene=" + state.vespene()
    + " supply=" + (int) state.supplyUsed() + "/" + state.supply());
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2DeliveryHandlerVespeneAssertTest -q`
Expected: all 4 tests PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/playbook/SC2DeliveryHandler.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/SC2DeliveryHandlerVespeneAssertTest.java
git commit -m "feat: add vespene bounds to playbook doAssert() Refs #394"
```

---

## References

- [2026-10-09-vespene-income-design.md] — design spec this plan implements
- [SC2Data.java:33-60] — MINERAL_TIER_RATES_PER_TICK pattern (template for gas rates)
- [EmulatedGame.java:116-124] — current tick() mineral income loop
- [EmulatedGame.java:944-968] — countWorkersPerBase
- [PlayerState.java:27-41] — current economy fields
- [PlayerStateView.java:15] — vespene() returns int (interface unchanged)
- [SC2DeliveryHandler.java:116-155] — current doAssert
- [PP-20260522-572156] — SC2Data calibration protocol
- [GitHub #394] — focal issue
