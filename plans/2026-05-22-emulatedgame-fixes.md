# EmulatedGame Fixes (#144, #147, #148) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three small gaps in EmulatedGame accuracy and test coverage: pin the `trainTimeInLoops` default branch (#144), convert switch statements to exhaustiveness-checked expressions (#147), and sync vespene from replay ground truth so gas-unit train commands are no longer rejected (#148).

**Architecture:** All changes are contained to `EmulatedGame`, `SC2DataTest`, `EmulatedGameTest`, `ReplayValidationHarness`, and `ReplayValidationTest`. No new classes or interfaces. #148 follows the established harness sync pattern already used for probe count and supply cap.

**Tech Stack:** Java 21, JUnit 5 + AssertJ, Maven (`mvn test -Dtest=ClassName -q`)

---

## File Map

| File | Change |
|------|--------|
| `src/test/java/io/quarkmind/domain/SC2DataTest.java` | +2 assertions (#144) |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Convert 2 switch statements → expressions (#147); enhance comment (#147); add `setVespeneForHarness` (#148) |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | +1 test for `setVespeneForHarness` (#148) |
| `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` | +1 sync call (#148) |
| `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java` | Tighten unit-count assertion (#148) |

---

## Task 1: #144 — Pin SC2DataTest default branch

**Files:**
- Modify: `src/test/java/io/quarkmind/domain/SC2DataTest.java`

- [ ] **Step 1: Add assertions to the two existing test methods**

In `trainTimeInLoopsDefinedForProtossUnits` (currently ends at line ~188), append:
```java
assertThat(SC2Data.trainTimeInLoops(UnitType.UNKNOWN)).isEqualTo(672);  // default: 30s × 22.4 = 672.0 (exact)
```

In `trainTimeInTicksDefinedForProtossUnits` (currently ends at line ~197), append:
```java
assertThat(SC2Data.trainTimeInTicks(UnitType.UNKNOWN)).isEqualTo(30);   // 672 / 22 = 30 (integer division)
```

- [ ] **Step 2: Run the tests**

```bash
mvn test -Dtest=SC2DataTest -q
```

Expected: `BUILD SUCCESS`. Both new assertions pass immediately — `SC2Data.trainTimeInLoops` already has the `default -> 672` arm. If either assertion fails, the default value has drifted; fix the expected value to match the actual default.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/domain/SC2DataTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "test(sc2data): pin trainTimeInLoops default branch — Closes #144"
```

---

## Task 2: #147 — Switch expressions + queued-unit comment

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

**Background:** `EmulatedGame` has two void dispatch methods over the sealed `Intent` interface. Java 21 switch *statements* don't enforce exhaustiveness at compile time even on sealed types; switch *expressions* do. Converting to the `Runnable` dispatch pattern makes each method a switch expression without changing any behavior. The `drainBuildingQueues` comment is also sharpened to document the accuracy impact of the missing loop offset.

- [ ] **Step 1: Run existing tests to confirm baseline**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`. Establish this as the baseline before the refactor.

- [ ] **Step 2: Convert `applyIntent(TimedIntent)` to switch expression**

Find the current method near line 225:
```java
public void applyIntent(TimedIntent ti) {
    switch (ti.intent()) {
        case TrainIntent  t -> handleTrain(t, friendly, ti.loop());
        case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation(), false, friendly);
        case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation(), true,  friendly);
        case BuildIntent  b -> handleBuild(b, friendly);
        case BlinkIntent  b -> executeBlink(b.unitTag(), friendly);
    }
}
```

Replace with:
```java
public void applyIntent(TimedIntent ti) {
    Runnable action = switch (ti.intent()) {
        case TrainIntent  t -> () -> handleTrain(t, friendly, ti.loop());
        case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), false, friendly);
        case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), true,  friendly);
        case BuildIntent  b -> () -> handleBuild(b, friendly);
        case BlinkIntent  b -> () -> executeBlink(b.unitTag(), friendly);
    };
    action.run();
}
```

- [ ] **Step 3: Convert `applyIntent(Intent, PlayerState)` to switch expression**

Find the current method near line 235:
```java
void applyIntent(Intent intent, PlayerState state) {
    switch (intent) {
        case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation(), false, state);
        case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation(), true,  state);
        case TrainIntent  t -> handleTrain(t, state);
        case BuildIntent  b -> handleBuild(b, state);
        case BlinkIntent  b -> executeBlink(b.unitTag(), state);
    }
}
```

Replace with:
```java
void applyIntent(Intent intent, PlayerState state) {
    Runnable action = switch (intent) {
        case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), false, state);
        case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), true,  state);
        case TrainIntent  t -> () -> handleTrain(t, state);
        case BuildIntent  b -> () -> handleBuild(b, state);
        case BlinkIntent  b -> () -> executeBlink(b.unitTag(), state);
    };
    action.run();
}
```

- [ ] **Step 4: Sharpen the queued-unit comment in `drainBuildingQueues`**

Find near line 328:
```java
startTraining(buildingTag, next, state, 0L); // queued units have no loop context — 0L uses integer-tick precision
```

Replace with:
```java
startTraining(buildingTag, next, state, 0L); // queued units carry no original loop — sub-tick offset is lost; unit can complete ±1 tick vs replay
```

- [ ] **Step 5: Verify refactor compiles and all tests pass**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`. If a compile error appears saying the switch is not exhaustive, a new `Intent` subtype has been added since this plan was written — add the missing arm.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "refactor(emulated): switch expressions for compile-time Intent exhaustiveness — Refs #147"
```

---

## Task 3: #148 Part A — `setVespeneForHarness` + unit test

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

**Background:** `EmulatedGame` starts with 0 vespene and earns none during `tick()`. This causes `handleTrain` to reject Stalker (50 gas) and Immortal (100 gas) training. The fix mirrors the existing `setSupplyCapForHarness` pattern: a public method that writes directly into `friendly` state, called by the harness. The test is in the same package as `EmulatedGame` and follows existing `EmulatedGameTest` conventions.

- [ ] **Step 1: Write the failing test first**

Add to `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`:

```java
@Test
void setVespeneForHarness_enablesGasUnitTraining() {
    game.reset();

    // Inject a complete Gateway — Stalkers train here
    Building gateway = new Building(
        "gw-vespene-test", BuildingType.GATEWAY,
        new Point2d(12, 12),
        SC2Data.maxBuildingHealth(BuildingType.GATEWAY),
        SC2Data.maxBuildingHealth(BuildingType.GATEWAY),
        true);
    game.injectReplayBuilding(gateway);

    // Tick 30 times to accumulate ≥125 minerals (Stalker costs 125 minerals + 50 gas)
    for (int i = 0; i < 30; i++) game.tick();

    // Attempt 1: vespene == 0 — TrainIntent rejected at gas check
    game.applyIntent(new TrainIntent("gw-vespene-test", UnitType.STALKER));
    for (int i = 0; i < SC2Data.trainTimeInTicks(UnitType.STALKER) + 1; i++) game.tick();
    assertThat(game.snapshot().myUnits())
        .as("Stalker train must be rejected when vespene == 0")
        .noneMatch(u -> u.type() == UnitType.STALKER);

    // Attempt 2: set vespene — TrainIntent accepted
    game.setVespeneForHarness(50);
    game.applyIntent(new TrainIntent("gw-vespene-test", UnitType.STALKER));
    for (int i = 0; i < SC2Data.trainTimeInTicks(UnitType.STALKER) + 1; i++) game.tick();
    assertThat(game.snapshot().myUnits())
        .as("Stalker must be trained after setVespeneForHarness(50)")
        .anyMatch(u -> u.type() == UnitType.STALKER);
}
```

Verify the imports at the top of the file already include `Building`, `BuildingType`, `UnitType`, `SC2Data`, `Point2d`, `TrainIntent`. If any are missing, add them.

- [ ] **Step 2: Run to confirm compile failure**

```bash
mvn test -Dtest=EmulatedGameTest#setVespeneForHarness_enablesGasUnitTraining -q 2>&1 | head -20
```

Expected: compile error — `cannot find symbol: method setVespeneForHarness(int)`. If it passes instead, `setVespeneForHarness` already exists somewhere — check before proceeding.

- [ ] **Step 3: Implement `setVespeneForHarness` in `EmulatedGame`**

Add directly after `setSupplyCapForHarness` (near line 732):

```java
/**
 * Sets the friendly vespene pool from replay ground truth.
 * Mirrors the vespene the real player had available at this tick,
 * so gas-unit TrainIntents are not rejected by the resource check.
 * Public: called from ReplayValidationHarness in a different package.
 */
public void setVespeneForHarness(int vespene) {
    friendly.vespene = vespene;
}
```

- [ ] **Step 4: Run the new test**

```bash
mvn test -Dtest=EmulatedGameTest#setVespeneForHarness_enablesGasUnitTraining -q
```

Expected: `BUILD SUCCESS`. If the first assertion fails (Stalker IS trained with 0 vespene), the vespene check in `handleTrain` is not firing — check line 276. If the second assertion fails (Stalker NOT trained after setVespene), run with debug logging or check that the gateway is not busy from a prior training queue.

- [ ] **Step 5: Run full EmulatedGameTest to check for regressions**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(emulated): setVespeneForHarness — enables gas-unit training in replay harness — Refs #148"
```

---

## Task 4: #148 Part B — Harness sync + tighten ReplayValidationTest

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`
- Modify: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java`

**Background:** The harness already syncs probe count from `gtBefore` (pre-tick ground truth) so mineral income uses realistic inputs. Vespene must be synced from the same `gtBefore` snapshot — pre-tick represents what the real player had *before* issuing train commands, so the subsequent TrainIntent deduction correctly mirrors the real expenditure without double-counting. After this sync, gas units are never rejected, and `ReplayValidationTest.unitCountWithinToleranceForThreeMinutes` can assert zero divergence instead of the current ≤15 tolerance.

- [ ] **Step 1: Temporarily tighten the test assertion to confirm it currently fails**

In `ReplayValidationTest.unitCountWithinToleranceForThreeMinutes`, find the two existing assertions:
```java
assertThat(report.summary().firstUnitDivergenceTick())
    ...
    .isGreaterThanOrEqualTo(145);

assertThat(report.summary().maxUnitDelta())
    ...
    .isLessThanOrEqualTo(15);
```

Replace both with the target assertion:
```java
assertThat(report.summary().economicallyAccurate())
    .as("No unit or building divergence expected after vespene sync (#148).\n%s",
        report.renderReport())
    .isTrue();
```

- [ ] **Step 2: Run to confirm the test currently fails (pre-harness-fix)**

```bash
mvn test -Dtest=ReplayValidationTest#unitCountWithinToleranceForThreeMinutes -q 2>&1 | tail -20
```

Expected: `FAIL` — `economicallyAccurate()` is `false` because gas units are still being rejected (harness not yet syncing vespene). If this passes already, the implementation in Task 3 accidentally fixed the harness path — investigate before continuing.

- [ ] **Step 3: Add vespene sync to `ReplayValidationHarness`**

In `ReplayValidationHarness.run()`, find the existing probe sync block (around line 66):
```java
GameState gtBefore = replayGame.snapshot();
emulated.setMiningProbes(countProbes(gtBefore));
```

Add one line immediately after:
```java
GameState gtBefore = replayGame.snapshot();
emulated.setMiningProbes(countProbes(gtBefore));
emulated.setVespeneForHarness(gtBefore.vespene());
```

Also update the Javadoc comment at line 24 (currently says "Resources (minerals, vespene) are NOT synced"). Change:
```
 *   <li>Resources (minerals, vespene) are NOT synced — divergence here is expected given
 *       EmulatedGame's flat mining rate vs SC2's saturation-based model.</li>
```
to:
```
 *   <li>Minerals are NOT synced — divergence is expected given EmulatedGame's flat mining
 *       rate vs SC2's saturation-based model.</li>
 *   <li>Vespene IS synced from pre-tick ground truth so gas-unit TrainIntents are not
 *       rejected by the resource check. See #148.</li>
```

- [ ] **Step 4: Run the target test**

```bash
mvn test -Dtest=ReplayValidationTest#unitCountWithinToleranceForThreeMinutes -q
```

Expected: `BUILD SUCCESS` — `economicallyAccurate()` is now `true`. If it still fails, the report output embedded in the assertion message will show which ticks diverge — check whether those ticks involve gas units (Stalker/Immortal) or something else (building divergence or a new cause).

- [ ] **Step 5: Also update the class-level Javadoc on `ReplayValidationTest`**

The test's Javadoc currently says:
```
 * <p>Unit count divergence has one remaining cause:
 * <ul>
 *   <li><b>Gas units</b>: EmulatedGame starts with 0 vespene and has no gas income model.
 *       Train commands for Stalkers (50 gas) and Immortals (100 gas) are rejected, creating a
 *       growing divergence proportional to the number of gas units trained. See #148.</li>
 * </ul>
```

Replace with:
```
 * <p>Unit count divergence: zero within the 3-minute window. Vespene is synced from pre-tick
 * ground truth (#148), so gas-unit train commands are never rejected.
```

Also update the final sentence referencing #148:
```
 * Refs #137, #141, #142, #148, #149
```
(keep this line as-is — #148 is now closed by this commit).

- [ ] **Step 6: Run the full ReplayValidationTest suite**

```bash
mvn test -Dtest=ReplayValidationTest -q
```

Expected: `BUILD SUCCESS`. Both `unitCountWithinToleranceForThreeMinutes` and `mineralDeltaWithinToleranceForThreeMinutes` pass. The mineral delta test is unchanged — mineral sync is still not done.

- [ ] **Step 7: Run the full test suite**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`. If `SC2TrainTimeCalibrationTest` or `ReplayValidationReportTest` fail, check that they are excluded from the default surefire run (`excludedGroups=benchmark,browser,report`). Those profiles run separately.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java \
  src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m \
  "feat(harness): sync vespene from GT; gas units no longer rejected — Closes #148"
```

---

## Summary

| Task | Issue | Files | Commits |
|------|-------|-------|---------|
| 1 | #144 | SC2DataTest.java | 1 |
| 2 | #147 | EmulatedGame.java | 1 |
| 3 | #148A | EmulatedGame.java, EmulatedGameTest.java | 1 |
| 4 | #148B | ReplayValidationHarness.java, ReplayValidationTest.java | 1 |
