# Multi-base Mining + Queued-unit Loop Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix replay validation mineral drift past the 3-minute window (#143) and eliminate ±1 tick queued-unit imprecision (#147).

**Architecture:** Keep `SC2Data.mineralIncomePerTick(int)` as the per-base function. Replace `EmulatedGame.miningProbes` (scalar) with `miningProbesPerBase` (array); tick sums income across bases. Harness computes per-base probe counts from GT unit/building positions. For #147, propagate the absolute completion loop from the finishing unit to the next queued unit via a new `buildingCompletionAtLoop` map on `PlayerState`.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Quarkus (CDI not involved — all plain unit tests)

**Spec:** `docs/superpowers/specs/2026-05-22-multi-base-mining-design.md`

---

## File Map

| File | Change |
|------|--------|
| `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java` | Add `buildingCompletionAtLoop` map; clear in `clear()` |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Replace `miningProbes` with `miningProbesPerBase`; update `tick()`, `reset()`, API. Store completion loop in `startTraining`; propagate in `drainBuildingQueues`. |
| `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` | Replace `countProbes` with `countProbesPerBase`; update caller. |
| `src/main/java/io/quarkmind/domain/SC2Data.java` | Remove TODO comment for #143 (work is done). |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add queued-unit sub-tick test; update `setMiningProbes(0)` → `setMiningProbesPerBase(0)`; add multi-base mining test. |

---

## Task 1: #147 — Queued-unit sub-tick precision

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Add `buildingCompletionAtLoop` to PlayerState**

In `PlayerState.java`, add the new map alongside the existing `buildingTrainingUntil`:

```java
final Map<String, Long> buildingCompletionAtLoop = new HashMap<>();
```

In `clear()`, add:

```java
buildingCompletionAtLoop.clear();
```

- [ ] **Step 2: Write the failing test**

In `EmulatedGameTest.java`, after the existing `queueDrainsSequentially` test (~line 1483), add:

```java
@Test
void queuedUnitPreservesSubTickPrecision() {
    EmulatedGame game = new EmulatedGame();
    game.reset();
    game.setMineralsForTesting(500);
    game.tick(); // gameFrame = 1
    int unitsBefore = game.snapshot().myUnits().size();

    // Train Probe A at loop 10: offset=10, (10+272)/22 = 282/22 = 12
    // completesAt = 1 + 12 = 13
    game.applyIntent(new TimedIntent(10L, new TrainIntent("nexus-0", UnitType.PROBE)));
    // Queue Probe B behind Probe A
    game.applyIntent(new TrainIntent("nexus-0", UnitType.PROBE));

    // Tick to frame 13: Probe A completes, drain starts Probe B
    for (int i = 0; i < 12; i++) game.tick();
    assertThat(game.snapshot().myUnits()).hasSize(unitsBefore + 1);

    // Tick 12 more: frames 14–25
    // Probe A's completion loop = 10 + 272 = 282; offset = 282 % 22 = 18
    // With propagation: (18 + 272) / 22 = 290 / 22 = 13; completesAt = 13 + 13 = 26
    // Without propagation (0L): (0 + 272) / 22 = 12; completesAt = 13 + 12 = 25
    for (int i = 0; i < 12; i++) game.tick();
    assertThat(game.snapshot().myUnits())
        .as("Queued unit should NOT complete yet — sub-tick offset pushes it 1 tick later")
        .hasSize(unitsBefore + 1);

    game.tick(); // frame 26
    assertThat(game.snapshot().myUnits())
        .as("Queued unit completes at frame 26 with propagated sub-tick offset")
        .hasSize(unitsBefore + 2);
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -Dtest=EmulatedGameTest#queuedUnitPreservesSubTickPrecision -q`

Expected: FAIL at "should NOT complete yet" — Probe B completes at frame 25 (offset=0) instead of 26.

- [ ] **Step 4: Store completion loop in `startTraining`**

In `EmulatedGame.java`, in `startTraining` (line ~301), after the `state.buildingTrainingUntil.put(...)` line, add:

```java
state.buildingCompletionAtLoop.put(buildingTag,
    absLoop + SC2Data.trainTimeInLoops(unitType));
```

- [ ] **Step 5: Propagate completion loop in `drainBuildingQueues`**

In `EmulatedGame.java`, in `drainBuildingQueues` (line ~330), replace:

```java
startTraining(buildingTag, next, state, 0L);
```

with:

```java
long completionLoop = state.buildingCompletionAtLoop.getOrDefault(buildingTag, 0L);
state.buildingCompletionAtLoop.remove(buildingTag);
startTraining(buildingTag, next, state, completionLoop);
```

Remove the old trailing comment (`// queued units carry no original loop...`).

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -Dtest=EmulatedGameTest#queuedUnitPreservesSubTickPrecision -q`

Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `mvn test -q`

Expected: all tests pass. Existing `queueDrainsSequentially` still passes — the tick count changes by at most 1 but that test uses 12-tick windows which still encompass the ±1 shift.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/PlayerState.java \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "fix(#147): propagate sub-tick completion loop to queued units — Closes #147"
```

---

## Task 2: #143 — Per-base mining API on EmulatedGame

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Write the failing test for multi-base mining**

In `EmulatedGameTest.java`, in the E1 tests section (after `zeroProbesYieldsNoMineralGain`), add:

```java
@Test
void multiBaseMiningDistributesIncomeAcrossBases() {
    // 12 probes on each of 2 bases should earn more than 24 probes on 1 base
    // because 24 probes on 1 base hits the diminishing tier3 (probes 17–24),
    // while 12 probes per base each stay in the higher tier1+tier2 range.
    game.setMineralsForTesting(0);
    game.setMiningProbesPerBase(12, 12);
    game.tick();

    double twoBaseIncome = 2 * SC2Data.mineralIncomePerTick(12);
    double oneBaseIncome = SC2Data.mineralIncomePerTick(24);
    // Verify the test premise: two-base is actually higher
    assertThat(twoBaseIncome).isGreaterThan(oneBaseIncome);

    assertThat(game.snapshot().minerals())
        .as("Multi-base income should sum per-base rates, not single-base curve")
        .isCloseTo((int) twoBaseIncome, within(1));
}
```

- [ ] **Step 2: Replace `miningProbes` with `miningProbesPerBase`**

In `EmulatedGame.java`:

Replace the field (line ~30):
```java
// Before:
private int  miningProbes;

// After:
private int[] miningProbesPerBase;
```

Replace in `reset()` (line ~49):
```java
// Before:
miningProbes = SC2Data.INITIAL_PROBES;

// After:
miningProbesPerBase = new int[]{SC2Data.INITIAL_PROBES};
```

Replace in `tick()` (line ~86):
```java
// Before:
friendly.minerals += SC2Data.mineralIncomePerTick(miningProbes);

// After:
for (int probesAtBase : miningProbesPerBase) {
    friendly.minerals += SC2Data.mineralIncomePerTick(probesAtBase);
}
```

Replace the setter (line ~596):
```java
// Before:
public void setMiningProbes(int count) { this.miningProbes = count; }

// After:
public void setMiningProbesPerBase(int... probesPerBase) {
    this.miningProbesPerBase = probesPerBase;
}
```

- [ ] **Step 3: Fix the existing test that uses the old API**

In `EmulatedGameTest.java`, update `zeroProbesYieldsNoMineralGain` (line ~59):

```java
// Before:
game.setMiningProbes(0);

// After:
game.setMiningProbesPerBase(0);
```

- [ ] **Step 4: Run tests to verify multi-base test passes**

Run: `mvn test -Dtest=EmulatedGameTest -q`

Expected: all EmulatedGameTest tests pass, including `multiBaseMiningDistributesIncomeAcrossBases`.

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#143): per-base mining API on EmulatedGame — Refs #143"
```

---

## Task 3: #143 — Per-base probe counting in ReplayValidationHarness

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`

- [ ] **Step 1: Replace `countProbes` with `countProbesPerBase`**

In `ReplayValidationHarness.java`, replace the `countProbes` method (lines ~130–134) with:

```java
private static int[] countProbesPerBase(GameState state) {
    List<Building> nexuses = state.myBuildings().stream()
        .filter(b -> b.type() == BuildingType.NEXUS && b.isComplete())
        .toList();
    if (nexuses.isEmpty()) return new int[0];

    int[] counts = new int[nexuses.size()];
    for (Unit u : state.myUnits()) {
        if (u.type() != UnitType.PROBE) continue;
        int nearest = 0;
        double minDist = Double.MAX_VALUE;
        for (int i = 0; i < nexuses.size(); i++) {
            double dx = u.position().x() - nexuses.get(i).position().x();
            double dy = u.position().y() - nexuses.get(i).position().y();
            double d = Math.sqrt(dx * dx + dy * dy);
            if (d < minDist) { minDist = d; nearest = i; }
        }
        counts[nearest]++;
    }
    return counts;
}
```

Add the necessary import at the top (if not already present):
```java
import java.util.List;
```

- [ ] **Step 2: Update the caller**

In `ReplayValidationHarness.run()` (line ~69), replace:

```java
emulated.setMiningProbes(countProbes(gtBefore));
```

with:

```java
emulated.setMiningProbesPerBase(countProbesPerBase(gtBefore));
```

- [ ] **Step 3: Run full test suite**

Run: `mvn test -q`

Expected: all tests pass. `ReplayValidationTest` may show improved mineral divergence for replays where the player expands.

- [ ] **Step 4: Run replay divergence report**

Run: `mvn test -Preport -q`

Review mineral divergence numbers in the report output. Note any improvement for reference.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#143): per-base probe counting in ReplayValidationHarness — Closes #143"
```

---

## Task 4: Clean up TODO and remove stale reference

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

- [ ] **Step 1: Remove the #143 TODO from `mineralIncomePerTick` Javadoc**

In `SC2Data.java` (lines ~37–39), remove:

```java
 * <p>Single-base only — assumes all probes mine one Nexus. When expansion is modelled,
 * this will need a base-count parameter to distribute probes across bases correctly.
 * TODO(multi-base #143): extend to accept base count once expansion is simulated.
```

Replace with:

```java
 * <p>Per-base income — callers with multiple bases sum this across each base's
 * probe count. See {@code EmulatedGame.tick()} for multi-base usage.
```

- [ ] **Step 2: Run full test suite**

Run: `mvn test -q`

Expected: all tests pass (Javadoc-only change).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/domain/SC2Data.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(#143): update mineralIncomePerTick Javadoc — per-base usage documented"
```
