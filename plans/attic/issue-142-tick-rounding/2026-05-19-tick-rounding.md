# Train Timing Precision — Sub-tick CmdEvent Offset Fix

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix systematic 1-tick-early unit completion in replay validation by propagating the sub-tick game-loop offset from `TimedIntent` through to `EmulatedGame.startTraining`.

**Architecture:** Three coordinated changes — (1) `SC2Data` gains exact loop-based train times and LOOPS_PER_TICK as domain constants; (2) `TimedIntent` moves from `sc2.replay` to `sc2.intent` so `EmulatedGame` can accept it without a layering violation; (3) `EmulatedGame` gains a `applyIntent(TimedIntent)` overload that uses the absolute game loop to compute the correct fractional-tick completion frame.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Maven (`mvn test -Dtest=ClassName -q`)

---

## File Map

| File | Change |
|------|--------|
| `src/main/java/io/quarkmind/domain/SC2Data.java` | Add `LOOPS_PER_TICK`, `GAME_LOOPS_PER_SECOND`, `trainTimeInLoops`; derive `trainTimeInTicks` |
| `src/test/java/io/quarkmind/domain/SC2DataTest.java` | Add 2 tests |
| `src/main/java/io/quarkmind/sc2/replay/TimedIntent.java` | Delete (moved) |
| `src/main/java/io/quarkmind/sc2/intent/TimedIntent.java` | Create (moved here via IntelliJ) |
| `src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java` | Add import for moved `TimedIntent` |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Add `applyIntent(TimedIntent)`; update `handleTrain` and `startTraining` to accept `absLoop` |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add 2 tests |
| `src/main/java/io/quarkmind/sc2/mock/Sc2ReplayShared.java` | Derive `LOOPS_PER_TICK` from `SC2Data` |
| `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` | Pass `TimedIntent` directly; remove local constant; use `SC2Data.LOOPS_PER_TICK` |
| `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java` | Tighten assertions to `firstUnitDivergenceTick == -1` and `maxUnitDelta == 0` |

---

## Task 1: SC2Data — domain constants and derived train timing

**Files:**
- Modify: `src/test/java/io/quarkmind/domain/SC2DataTest.java`
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

- [ ] **Step 1: Write the two failing tests**

Add to the bottom of `SC2DataTest` (inside the class, before the closing `}`):

```java
@Test
void trainTimeInLoopsDefinedForProtossUnits() {
    assertThat(SC2Data.trainTimeInLoops(UnitType.PROBE))    .isEqualTo(268.8);
    assertThat(SC2Data.trainTimeInLoops(UnitType.ZEALOT))   .isEqualTo(627.2);
    assertThat(SC2Data.trainTimeInLoops(UnitType.STALKER))  .isEqualTo(694.4);
    assertThat(SC2Data.trainTimeInLoops(UnitType.IMMORTAL)) .isEqualTo(896.0);
    assertThat(SC2Data.trainTimeInLoops(UnitType.OBSERVER)) .isEqualTo(492.8);
}

@Test
void trainTimeInLoopsConsistentWithTicks() {
    for (UnitType t : List.of(UnitType.PROBE, UnitType.ZEALOT,
                               UnitType.STALKER, UnitType.IMMORTAL, UnitType.OBSERVER)) {
        assertThat((int)(SC2Data.trainTimeInLoops(t) / SC2Data.LOOPS_PER_TICK))
            .as("trainTimeInLoops / LOOPS_PER_TICK must equal trainTimeInTicks for %s", t)
            .isEqualTo(SC2Data.trainTimeInTicks(t));
    }
}
```

Add `import java.util.List;` at the top of `SC2DataTest`.

- [ ] **Step 2: Run tests to confirm they fail**

```bash
mvn test -Dtest=SC2DataTest -q 2>&1 | tail -15
```

Expected: compilation failure — `trainTimeInLoops` not found.

- [ ] **Step 3: Implement the constants and methods in SC2Data**

In `SC2Data.java`, add the two public constants directly after the class opening (before `MINERALS_PER_PROBE_PER_TICK`):

```java
/** Game loops per outer tick at SC2 Faster speed. */
public static final int    LOOPS_PER_TICK        = 22;

/** Game loops per real-time second at SC2 Faster speed (16 loops/sec × 1.4 multiplier). */
public static final double GAME_LOOPS_PER_SECOND = 22.4;
```

Add the new method after `MINERALS_PER_PROBE_PER_TICK` (before `INITIAL_MINERALS`):

```java
/**
 * Exact train time in game loops at SC2 Faster speed.
 * Source of truth for all train timing — {@link #trainTimeInTicks} derives from this.
 */
public static double trainTimeInLoops(UnitType type) {
    return switch (type) {
        case PROBE    -> 12 * GAME_LOOPS_PER_SECOND;  // 268.8
        case ZEALOT   -> 28 * GAME_LOOPS_PER_SECOND;  // 627.2
        case STALKER  -> 31 * GAME_LOOPS_PER_SECOND;  // 694.4
        case IMMORTAL -> 40 * GAME_LOOPS_PER_SECOND;  // 896.0
        case OBSERVER -> 22 * GAME_LOOPS_PER_SECOND;  // 492.8
        default       -> 30 * GAME_LOOPS_PER_SECOND;  // 672.0
    };
}
```

Replace the existing `trainTimeInTicks` method body (lines 58–67) with a derived implementation:

```java
public static int trainTimeInTicks(UnitType type) {
    return (int)(trainTimeInLoops(type) / LOOPS_PER_TICK);
}
```

- [ ] **Step 4: Run the tests to confirm they pass**

```bash
mvn test -Dtest=SC2DataTest -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Run the full unit suite to confirm no regression**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkmind add \
    src/main/java/io/quarkmind/domain/SC2Data.java \
    src/test/java/io/quarkmind/domain/SC2DataTest.java
git -C /Users/mdproctor/claude/quarkmind commit -m "feat(#142): add LOOPS_PER_TICK, GAME_LOOPS_PER_SECOND, trainTimeInLoops to SC2Data; derive trainTimeInTicks"
```

---

## Task 2: Move TimedIntent to sc2/intent/

**Files:**
- Move: `src/main/java/io/quarkmind/sc2/replay/TimedIntent.java` → `src/main/java/io/quarkmind/sc2/intent/TimedIntent.java`
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java` (import)
- Modify: `src/main/java/io/quarkmind/sc2/replay/AbilityMapping.java` (import — IDE handles)
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` (import — IDE handles)

- [ ] **Step 1: Move the file using IntelliJ**

Use the IDE move-file tool to relocate `TimedIntent` — this updates all import sites automatically:

```
ide_move_file:
  file: "src/main/java/io/quarkmind/sc2/replay/TimedIntent.java"
  destination: "src/main/java/io/quarkmind/sc2/intent"
```

The moved file's package declaration becomes `package io.quarkmind.sc2.intent;` and its import `import io.quarkmind.sc2.intent.Intent;` becomes a sibling import (same package, no import needed).

- [ ] **Step 2: Verify the moved file is correct**

Read `src/main/java/io/quarkmind/sc2/intent/TimedIntent.java`. It should read:

```java
package io.quarkmind.sc2.intent;

/** An Intent extracted from replay GAME_EVENTS, tagged with its raw SC2 game loop. */
public record TimedIntent(long loop, Intent intent) {}
```

No import needed for `Intent` — same package.

- [ ] **Step 3: Add import to ReplayCommandStream**

`ReplayCommandStream` was in the same package as `TimedIntent` before, so had no import. Now add one. Open `src/main/java/io/quarkmind/sc2/replay/ReplayCommandStream.java` and add:

```java
import io.quarkmind.sc2.intent.TimedIntent;
```

The full file becomes:

```java
package io.quarkmind.sc2.replay;

import io.quarkmind.sc2.intent.TimedIntent;
import java.util.List;

/** Full extraction result from a replay's GAME_EVENTS for one player. */
public record ReplayCommandStream(
    List<UnitOrder>   movementOrders,
    List<TimedIntent> intents) {}
```

- [ ] **Step 4: Build to confirm all imports resolved**

```bash
mvn compile -q 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Run full unit suite**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkmind add -A
git -C /Users/mdproctor/claude/quarkmind commit -m "refactor(#142): move TimedIntent from sc2.replay to sc2.intent"
```

---

## Task 3: EmulatedGame — loop-aware applyIntent (TDD)

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Write the two failing tests**

Add to `EmulatedGameTest` (add `import io.quarkmind.sc2.intent.TimedIntent;` at the top if not present):

```java
@Test
void probeCompletesOnTimeWithZeroLoopOffset() {
    EmulatedGame game = new EmulatedGame();
    game.reset();
    game.tick(); // gameFrame = 1; applyIntent is called post-tick in harness
    int unitsBefore = game.snapshot().myUnits().size();

    game.applyIntent(new TimedIntent(0L, new TrainIntent("nexus-0", UnitType.PROBE)));

    for (int i = 0; i < 11; i++) game.tick(); // ticks 2–12, gameFrame = 12
    assertThat(game.snapshot().myUnits().size())
        .as("Probe should not have completed yet after 11 ticks")
        .isEqualTo(unitsBefore);

    game.tick(); // tick 13, gameFrame = 13 — completesAt = 1 + 12 = 13, fires here
    assertThat(game.snapshot().myUnits().size())
        .as("Probe should complete after 12 ticks with zero loop offset")
        .isEqualTo(unitsBefore + 1);
}

@Test
void probeCompletesOneLaterWithLateLoopOffset() {
    EmulatedGame game = new EmulatedGame();
    game.reset();
    game.tick(); // gameFrame = 1
    int unitsBefore = game.snapshot().myUnits().size();

    // loop=18: offset = 18 % 22 = 18; (18 + 268.8) / 22 = 286.8 / 22 = 13.03 → floor 13
    // completesAt = 1 + 13 = 14; fires at gameFrame=14, i.e., after 13 ticks
    game.applyIntent(new TimedIntent(18L, new TrainIntent("nexus-0", UnitType.PROBE)));

    for (int i = 0; i < 12; i++) game.tick(); // ticks 2–13, gameFrame = 13
    assertThat(game.snapshot().myUnits().size())
        .as("Probe should not complete after 12 ticks — late command pushes to tick 13")
        .isEqualTo(unitsBefore);

    game.tick(); // tick 14, gameFrame = 14 — completesAt = 14, fires here
    assertThat(game.snapshot().myUnits().size())
        .as("Probe should complete after 13 ticks with loop offset 18")
        .isEqualTo(unitsBefore + 1);
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
mvn test -Dtest=EmulatedGameTest#probeCompletesOnTimeWithZeroLoopOffset+probeCompletesOneLaterWithLateLoopOffset -q 2>&1 | tail -15
```

Expected: compilation failure — `applyIntent(TimedIntent)` not found.

- [ ] **Step 3: Add applyIntent(TimedIntent) to EmulatedGame**

Add this import at the top of `EmulatedGame.java`:

```java
import io.quarkmind.sc2.intent.TimedIntent;
```

Add this new public method after the existing `applyIntent(Intent intent)` method (after line 222):

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

- [ ] **Step 4: Add the loop-aware handleTrain overload**

After the existing `handleTrain(TrainIntent t, PlayerState state)` method (after line 283), add a delegate:

```java
private void handleTrain(TrainIntent t, PlayerState state) {
    handleTrain(t, state, 0L);
}
```

Then rename the existing `handleTrain` body to accept `absLoop` — replace the current method signature and the `startTraining` call inside it:

The existing method at line 244:
```java
private void handleTrain(TrainIntent t, PlayerState state) {
```
becomes:
```java
private void handleTrain(TrainIntent t, PlayerState state, long absLoop) {
```

And inside, replace the `startTraining` call (currently at line 278):
```java
startTraining(buildingTag, t.unitType(), state);
```
with:
```java
startTraining(buildingTag, t.unitType(), state, absLoop);
```

- [ ] **Step 5: Update startTraining to accept absLoop**

Replace the existing `startTraining` signature and first two lines (lines 285–287):

```java
// Before:
private void startTraining(String buildingTag, UnitType unitType, PlayerState state) {
    boolean isEnemy  = (state == enemy);
    long completesAt = gameFrame + SC2Data.trainTimeInTicks(unitType);

// After:
private void startTraining(String buildingTag, UnitType unitType, PlayerState state, long absLoop) {
    boolean isEnemy  = (state == enemy);
    int  loopOffset  = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
    long completesAt = gameFrame
        + (int)((loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK);
```

- [ ] **Step 6: Update drainBuildingQueues to pass 0L**

In `drainBuildingQueues` (line 312), change:

```java
startTraining(buildingTag, next, state);
```
to:
```java
startTraining(buildingTag, next, state, 0L);
```

- [ ] **Step 7: Run the two new tests**

```bash
mvn test -Dtest=EmulatedGameTest#probeCompletesOnTimeWithZeroLoopOffset+probeCompletesOneLaterWithLateLoopOffset -q 2>&1 | tail -15
```

Expected: both PASS.

- [ ] **Step 8: Run the full EmulatedGame test class**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS` — all existing tests still pass.

- [ ] **Step 9: Run full unit suite**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/quarkmind add \
    src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
    src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git -C /Users/mdproctor/claude/quarkmind commit -m "feat(#142): add applyIntent(TimedIntent) with sub-tick completesAt to EmulatedGame"
```

---

## Task 4: Wire harness — pass TimedIntent directly, consolidate LOOPS_PER_TICK

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/mock/Sc2ReplayShared.java`
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java`

- [ ] **Step 1: Update Sc2ReplayShared to derive LOOPS_PER_TICK from SC2Data**

In `Sc2ReplayShared.java`, change line 10:

```java
// Before:
static final int LOOPS_PER_TICK = 22;

// After:
static final int LOOPS_PER_TICK = SC2Data.LOOPS_PER_TICK;
```

`SC2Data` is already in scope via the existing `import io.quarkmind.domain.*;` on line 3.

- [ ] **Step 2: Update ReplayValidationHarness**

In `ReplayValidationHarness.java`:

**a)** Remove the local constant (line 31):
```java
// Delete this line:
private static final int LOOPS_PER_TICK = 22;
```

**b)** Add the import for the moved `TimedIntent` at the top:
```java
import io.quarkmind.sc2.intent.TimedIntent;
```

**c)** Add import for `SC2Data`:
```java
import io.quarkmind.domain.SC2Data;
```

**d)** Replace the two references to the removed local `LOOPS_PER_TICK` with `SC2Data.LOOPS_PER_TICK`:

Line 62 becomes:
```java
long windowEnd = (long) (tick + 1) * SC2Data.LOOPS_PER_TICK;
```

Line 70 becomes:
```java
emulated.setMiningProbes(countProbes(gtBefore) * SC2Data.LOOPS_PER_TICK);
```

**e)** Replace the intent application in the while loop (line 91):

```java
// Before:
emulated.applyIntent(intents.get(cursor).intent());

// After:
emulated.applyIntent(intents.get(cursor));
```

- [ ] **Step 3: Build to confirm everything compiles**

```bash
mvn compile -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run full unit suite**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/quarkmind add \
    src/main/java/io/quarkmind/sc2/mock/Sc2ReplayShared.java \
    src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java
git -C /Users/mdproctor/claude/quarkmind commit -m "refactor(#142): pass TimedIntent directly in harness; consolidate LOOPS_PER_TICK to SC2Data"
```

---

## Task 5: Update ReplayValidationTest to document sub-tick improvement

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java`

> **⚠️ Outcome note:** The original spec targeted `firstUnitDivergenceTick == -1` and `maxUnitDelta == 0`.
> This was not achievable with the sub-tick fix alone. The sub-tick fix is mathematically correct
> (verified by brute force across all unit types and loops 0–3000), but residual divergence at
> tick 86+ is caused by mineral-timing: EmulatedGame accumulates ~1800 more minerals than GT
> (flat model vs saturation model), enabling trains GT's player couldn't yet afford.
> The actual improvement: `firstUnitDivergenceTick` moved from 36 → 86.
> Mineral-timing gap tracked in issue #146.

- [ ] **Step 1: Update the unit-count test to document the actual improvement**

Replace the `unitCountWithinTwoOfGroundTruthForThreeMinutes` test method entirely:

```java
@Test
void unitCountWithinTwoOfGroundTruthForThreeMinutes() {
    DivergenceReport report = ReplayValidationHarness.run(REPLAY, 1, THREE_MINUTES_TICKS);

    assertThat(report.summary().firstUnitDivergenceTick())
        .as("Sub-tick fix (#142) must keep first divergence at or above tick 80 "
            + "(was 36 before the fix; now 86 — mineral-timing gap tracked in #146). "
            + "First divergence was at tick %d.\n%s",
            report.summary().firstUnitDivergenceTick(), report.renderReport())
        .isGreaterThanOrEqualTo(80);

    assertThat(report.summary().maxUnitDelta())
        .as("Unit count delta must stay <= 2 at every tick (flat mining model trains 1 tick "
            + "early when emulated minerals exceed GT; exact match requires #141 or #146). "
            + "Max was %d.\n%s",
            report.summary().maxUnitDelta(), report.renderReport())
        .isLessThanOrEqualTo(2);
}
```

- [ ] **Step 2: Run the replay validation test**

```bash
mvn test -Dtest=ReplayValidationTest -q 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` — both tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkmind add \
    src/test/java/io/quarkmind/sc2/replay/ReplayValidationTest.java
git -C /Users/mdproctor/claude/quarkmind commit -m "test(#142): document sub-tick improvement — firstUnitDivergenceTick >= 80, maxUnitDelta <= 2"
```

---

## Task 6: Run complete test suite and verify

- [ ] **Step 1: Run everything**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS` with all tests green.

- [ ] **Step 2: If any test fails, investigate before proceeding**

Do not mark this task complete if any test is failing. Check the failure output, identify whether it is a pre-existing failure or introduced by this change, and fix before committing.
