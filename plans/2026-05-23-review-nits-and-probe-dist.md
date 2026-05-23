# #153/#152 Review Nits + Probe Distribution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three code review nits from #143/#147 and implement auto-compute probe distribution for AI expansion in EmulatedGame.

**Architecture:** #153 fixes sqrt, defensive copy, and adds a zero-nexus test. #152 extracts the probe-to-nexus algorithm into EmulatedGame, adds per-tick auto-compute with a one-shot override flag for the replay harness, and adds expansion-aware tests.

**Tech Stack:** Java 21, JUnit 5, AssertJ

---

### Task 1: #153 nit 1 — sqrt → squared distance in countProbesPerBase

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:132-152`

- [ ] **Step 1: Replace sqrt with squared distance**

In `ReplayValidationHarness.countProbesPerBase`, change line 146 from:
```java
double d = Math.sqrt(dx * dx + dy * dy);
if (d < minDist) { minDist = d; nearest = i; }
```
to:
```java
double dSq = dx * dx + dy * dy;
if (dSq < minDist) { minDist = dSq; nearest = i; }
```

`minDist` is initialized to `Double.MAX_VALUE` — works with squared values.

- [ ] **Step 2: Run existing tests to verify no regression**

Run: `mvn test -Dtest=ReplayValidationTest -q`
Expected: PASS — ordering is identical with squared distance.

- [ ] **Step 3: Commit**

```
fix(#153): replace sqrt with squared distance in countProbesPerBase — Refs #153
```

---

### Task 2: #153 nit 2 — defensive copy in setMiningProbesPerBase

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:606-608`

- [ ] **Step 1: Add defensive copy**

Change `setMiningProbesPerBase` from:
```java
public void setMiningProbesPerBase(int... probesPerBase) {
    this.miningProbesPerBase = probesPerBase;
}
```
to:
```java
public void setMiningProbesPerBase(int... probesPerBase) {
    this.miningProbesPerBase = probesPerBase.clone();
}
```

- [ ] **Step 2: Run existing mining tests to verify no regression**

Run: `mvn test -Dtest=EmulatedGameTest#mineralAccumulatesWithMiningProbes+zeroProbesYieldsNoMineralGain+multiBaseMiningDistributesIncomeAcrossBases -q`
Expected: PASS

- [ ] **Step 3: Commit**

```
fix(#153): defensive copy in setMiningProbesPerBase — Refs #153
```

---

### Task 3: #153 nit 3 — zero-nexus test for countProbesPerBase

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java:132` (visibility change)
- Create: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java`

- [ ] **Step 1: Make countProbesPerBase package-private**

In `ReplayValidationHarness.java`, change `private static int[] countProbesPerBase` to `static int[] countProbesPerBase` (drop `private`).

- [ ] **Step 2: Write the zero-nexus test**

Create `ReplayValidationHarnessTest.java`:

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ReplayValidationHarnessTest {

    @Test
    void countProbesPerBase_zeroCompleteNexuses_returnsEmptyArray() {
        Building incompleteNexus = new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8), 750, 1500, false);
        List<Unit> probes = List.of(
            probe("p-0", 9, 9),
            probe("p-1", 10, 9));

        GameState state = gameState(probes, List.of(incompleteNexus));
        int[] result = ReplayValidationHarness.countProbesPerBase(state);

        assertThat(result).isEmpty();
    }

    @Test
    void countProbesPerBase_singleNexus_allProbesAssigned() {
        Building nexus = new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8), 1500, 1500, true);
        List<Unit> probes = List.of(
            probe("p-0", 9, 9),
            probe("p-1", 10, 9),
            probe("p-2", 7, 8));

        GameState state = gameState(probes, List.of(nexus));
        int[] result = ReplayValidationHarness.countProbesPerBase(state);

        assertThat(result).containsExactly(3);
    }

    @Test
    void countProbesPerBase_twoNexuses_probesAssignedByProximity() {
        Building nexus0 = new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8), 1500, 1500, true);
        Building nexus1 = new Building("nexus-1", BuildingType.NEXUS,
            new Point2d(30, 30), 1500, 1500, true);
        List<Unit> probes = List.of(
            probe("p-0", 9, 9),      // near nexus-0
            probe("p-1", 10, 8),     // near nexus-0
            probe("p-2", 29, 30));   // near nexus-1

        GameState state = gameState(probes, List.of(nexus0, nexus1));
        int[] result = ReplayValidationHarness.countProbesPerBase(state);

        assertThat(result).containsExactly(2, 1);
    }

    @Test
    void countProbesPerBase_nonProbeUnitsIgnored() {
        Building nexus = new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8), 1500, 1500, true);
        List<Unit> units = List.of(
            probe("p-0", 9, 9),
            new Unit("z-0", UnitType.ZEALOT, new Point2d(10, 10),
                100, 100, 50, 50, 0, 0));

        GameState state = gameState(units, List.of(nexus));
        int[] result = ReplayValidationHarness.countProbesPerBase(state);

        assertThat(result).containsExactly(1);
    }

    private static Unit probe(String tag, float x, float y) {
        return new Unit(tag, UnitType.PROBE, new Point2d(x, y),
            SC2Data.maxHealth(UnitType.PROBE), SC2Data.maxHealth(UnitType.PROBE),
            SC2Data.maxShields(UnitType.PROBE), SC2Data.maxShields(UnitType.PROBE), 0, 0);
    }

    private static GameState gameState(List<Unit> units, List<Building> buildings) {
        return new GameState(0, 0, 0, 0, units, buildings,
            List.of(), List.of(), List.of(), List.of(), List.of(), 0);
    }
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `mvn test -Dtest=ReplayValidationHarnessTest -q`
Expected: PASS (4 tests)

- [ ] **Step 4: Commit**

```
test(#153): add zero-nexus and multi-nexus tests for countProbesPerBase — Closes #153
```

---

### Task 4: #152 — extract countProbesPerBase into EmulatedGame

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (add static method)
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` (delegate)
- Modify: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationHarnessTest.java` (update calls)

- [ ] **Step 1: Add static countProbesPerBase to EmulatedGame**

Add a package-private static method after `setMiningProbesPerBase`:

```java
static int[] countProbesPerBase(List<Building> buildings, List<Unit> units) {
    List<Building> nexuses = buildings.stream()
        .filter(b -> b.type() == BuildingType.NEXUS && b.isComplete())
        .toList();
    if (nexuses.isEmpty()) return new int[0];

    int[] counts = new int[nexuses.size()];
    for (Unit u : units) {
        if (u.type() != UnitType.PROBE) continue;
        int nearest = 0;
        double minDistSq = Double.MAX_VALUE;
        for (int i = 0; i < nexuses.size(); i++) {
            double dx = u.position().x() - nexuses.get(i).position().x();
            double dy = u.position().y() - nexuses.get(i).position().y();
            double dSq = dx * dx + dy * dy;
            if (dSq < minDistSq) { minDistSq = dSq; nearest = i; }
        }
        counts[nearest]++;
    }
    return counts;
}
```

- [ ] **Step 2: Delegate from ReplayValidationHarness**

Replace the body of `ReplayValidationHarness.countProbesPerBase(GameState)`:

```java
static int[] countProbesPerBase(GameState state) {
    return EmulatedGame.countProbesPerBase(state.myBuildings(), state.myUnits());
}
```

- [ ] **Step 3: Update ReplayValidationHarnessTest to also test EmulatedGame.countProbesPerBase directly**

Add a parallel test that calls the `EmulatedGame` method directly to confirm the extraction:

```java
@Test
void emulatedGame_countProbesPerBase_twoNexuses() {
    Building nexus0 = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(8, 8), 1500, 1500, true);
    Building nexus1 = new Building("nexus-1", BuildingType.NEXUS,
        new Point2d(30, 30), 1500, 1500, true);
    List<Unit> probes = List.of(
        probe("p-0", 9, 9),
        probe("p-1", 29, 30));

    int[] result = EmulatedGame.countProbesPerBase(
        List.of(nexus0, nexus1), probes);

    assertThat(result).containsExactly(1, 1);
}
```

Note: `ReplayValidationHarnessTest` is in `io.quarkmind.sc2.replay` package. `EmulatedGame.countProbesPerBase` is package-private in `io.quarkmind.sc2.emulated`. The test needs to move or the method needs wider visibility. Since both packages are internal and `EmulatedGame` is the canonical owner, make the method **package-private in `emulated`** and add the direct-call test to `EmulatedGameTest` instead:

Add to `EmulatedGameTest.java`:

```java
@Test
void countProbesPerBase_twoNexuses_distributesByProximity() {
    Building nexus0 = new Building("nexus-0", BuildingType.NEXUS,
        new Point2d(8, 8), 1500, 1500, true);
    Building nexus1 = new Building("nexus-1", BuildingType.NEXUS,
        new Point2d(30, 30), 1500, 1500, true);
    List<Unit> probes = List.of(
        probe("p-0", 9, 9),
        probe("p-1", 29, 30));

    int[] result = EmulatedGame.countProbesPerBase(
        List.of(nexus0, nexus1), probes);

    assertThat(result).containsExactly(1, 1);
}

private static Unit probe(String tag, float x, float y) {
    return new Unit(tag, UnitType.PROBE, new Point2d(x, y),
        SC2Data.maxHealth(UnitType.PROBE), SC2Data.maxHealth(UnitType.PROBE),
        SC2Data.maxShields(UnitType.PROBE), SC2Data.maxShields(UnitType.PROBE), 0, 0);
}
```

- [ ] **Step 4: Run all tests**

Run: `mvn test -Dtest=ReplayValidationHarnessTest+EmulatedGameTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
refactor(#152): extract countProbesPerBase into EmulatedGame — Refs #152
```

---

### Task 5: #152 — auto-compute miningProbesPerBase in tick() with override flag

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (add flag, modify tick, modify setter)

- [ ] **Step 1: Write failing test — tick auto-computes after second nexus completes**

Add to `EmulatedGameTest.java`:

```java
@Test
void tick_autoComputesProbeDistribution_afterSecondNexusCompletes() {
    game.setMineralsForTesting(5000);
    game.setMiningProbesPerBase(12);

    // Build a second nexus at a distant location
    game.applyIntent(new BuildIntent("probe-0", BuildingType.NEXUS, new Point2d(30, 30)));

    // Tick through build time (NEXUS default = 40 ticks)
    int buildTicks = SC2Data.buildTimeInTicks(BuildingType.NEXUS);
    for (int i = 0; i < buildTicks; i++) game.tick();

    // After completion, the 12 probes are all near nexus-0 at (8,8).
    // Auto-compute should assign all 12 to base 0, 0 to base 1.
    GameState state = game.snapshot();
    long nexusCount = state.myBuildings().stream()
        .filter(b -> b.type() == BuildingType.NEXUS && b.isComplete())
        .count();
    assertThat(nexusCount).as("Two nexuses should be complete").isEqualTo(2);
}
```

- [ ] **Step 2: Run test to verify it passes (nexus builds, baseline)**

Run: `mvn test -Dtest=EmulatedGameTest#tick_autoComputesProbeDistribution_afterSecondNexusCompletes -q`
Expected: PASS (this just checks nexus completion, not probe distribution yet)

- [ ] **Step 3: Write failing test — auto-compute changes mineral income**

Add to `EmulatedGameTest.java`:

```java
@Test
void tick_autoComputeProbeDistribution_singleBase_matchesManualSet() {
    // With auto-compute, default 12 probes near the single nexus at (8,8)
    // should produce the same income as manually setting {12}
    game.setMineralsForTesting(0);

    // Do NOT call setMiningProbesPerBase — let auto-compute handle it
    game.tick();

    double expectedIncome = SC2Data.mineralIncomePerTick(SC2Data.INITIAL_PROBES);
    assertThat(game.snapshot().minerals())
        .as("Auto-compute should assign all probes to single nexus")
        .isCloseTo((int) expectedIncome, within(1));
}
```

- [ ] **Step 4: Add the override flag and auto-compute to EmulatedGame**

Add field:
```java
private boolean miningProbesOverridden;
```

Modify `setMiningProbesPerBase`:
```java
public void setMiningProbesPerBase(int... probesPerBase) {
    this.miningProbesPerBase = probesPerBase.clone();
    this.miningProbesOverridden = true;
}
```

Modify `reset()` — add after `miningProbesPerBase = new int[]{SC2Data.INITIAL_PROBES};`:
```java
miningProbesOverridden = false;
```

Modify `tick()` — add at the very start, before the mineral income loop:
```java
if (!miningProbesOverridden) {
    miningProbesPerBase = countProbesPerBase(friendly.buildings, friendly.units);
}
miningProbesOverridden = false;
```

- [ ] **Step 5: Run the auto-compute tests**

Run: `mvn test -Dtest=EmulatedGameTest#tick_autoComputeProbeDistribution_singleBase_matchesManualSet+tick_autoComputesProbeDistribution_afterSecondNexusCompletes -q`
Expected: PASS

- [ ] **Step 6: Write test — setMiningProbesPerBase override is consumed then cleared**

Add to `EmulatedGameTest.java`:

```java
@Test
void setMiningProbesPerBase_overrideConsumedThenAutoComputeResumes() {
    game.setMineralsForTesting(0);

    // Override to zero probes — should produce zero income this tick
    game.setMiningProbesPerBase(0);
    game.tick();
    assertThat(game.snapshot().minerals())
        .as("Override to 0 probes should yield no income")
        .isEqualTo(0);

    // Next tick: no override set, auto-compute resumes.
    // 12 probes near nexus-0 → full income.
    game.setMineralsForTesting(0);
    game.tick();
    double expectedIncome = SC2Data.mineralIncomePerTick(SC2Data.INITIAL_PROBES);
    assertThat(game.snapshot().minerals())
        .as("Auto-compute should resume after override consumed")
        .isCloseTo((int) expectedIncome, within(1));
}
```

- [ ] **Step 7: Run test**

Run: `mvn test -Dtest=EmulatedGameTest#setMiningProbesPerBase_overrideConsumedThenAutoComputeResumes -q`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `mvn test -q`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```
feat(#152): auto-compute miningProbesPerBase in tick() with one-shot override — Closes #152
```

---

### Task 6: Verify full suite and final commit

- [ ] **Step 1: Run full test suite**

Run: `mvn test -q`
Expected: All tests PASS

- [ ] **Step 2: Verify no other callers of setMiningProbesPerBase are broken**

Check callers via IntelliJ `ide_find_references` on `setMiningProbesPerBase`. Known callers:
- `ReplayValidationHarness.run()` — calls override before each tick (correct)
- `EmulatedGameTest.zeroProbesYieldsNoMineralGain` — sets override, ticks immediately (correct)
- `EmulatedGameTest.multiBaseMiningDistributesIncomeAcrossBases` — sets override, ticks immediately (correct)

All callers set the override and tick within the same test — the one-shot clear does not affect them.
