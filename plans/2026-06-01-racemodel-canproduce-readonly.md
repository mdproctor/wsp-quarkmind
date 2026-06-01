# RaceModel.canProduce() Read-Only Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce the read-only contract of `RaceModel.canProduce()` at compile time via a `PlayerStateView` interface and a `MuleCalldownIntent` intent type, replacing the ad-hoc `HANDLED` escape hatch with a correct semantic model.

**Architecture:** Three new types (`PlayerStateView`, `ProductionDecision` enum, `MuleCalldownIntent`) are added first. `MuleCalldownIntent` is wired into the sealed `Intent` hierarchy atomically. `RaceModel.canProduce` then takes `PlayerStateView` and returns `ProductionDecision` — with `TerranRaceModel` losing all MULE logic from `canProduce` and gaining `onCalldown`. `ProductionResult` is deleted.

**Tech Stack:** Java 17+, JUnit 5, AssertJ. Maven: `mvn test -Dtest=<TestClass> -q` for single-class runs, `mvn test -q` for full suite. All files in `io.quarkmind.*`. Tests are plain JUnit (no `@QuarkusTest`).

---

## File Map

**Create:**
- `src/main/java/io/quarkmind/sc2/emulated/PlayerStateView.java` — public read-only interface
- `src/main/java/io/quarkmind/sc2/emulated/ProductionDecision.java` — two-value enum replacing `ProductionResult`
- `src/main/java/io/quarkmind/sc2/intent/MuleCalldownIntent.java` — new Intent record

**Modify:**
- `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java` — add `implements PlayerStateView`
- `src/main/java/io/quarkmind/sc2/intent/Intent.java` — add `MuleCalldownIntent` to permits clause
- `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java` — change `canProduce` signature, add `onCalldown` default, fix stale Javadoc
- `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java` — trivialise `canProduce`, add `onCalldown`, remove `currentGameLoop` field
- `src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java` — update `canProduce` parameter type
- `src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java` — update `canProduce` parameter type
- `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — two Intent switches, new `handleMuleCalldown`, simplify `handleTrain`
- `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java` — add `MuleCalldownIntent` case (stub)
- `src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java` — update 2 existing MULE tests, add 4 new tests
- `src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java` — add 2 new view-enforcement tests

**Delete:**
- `src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java`

---

## Task 1: Create `PlayerStateView` + `ProductionDecision` + update `PlayerState`

These are purely additive — no existing code changes except one `implements` clause. All existing tests continue to pass.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/PlayerStateView.java`
- Create: `src/main/java/io/quarkmind/sc2/emulated/ProductionDecision.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`

- [ ] **Step 1: Create `PlayerStateView`**

```java
// src/main/java/io/quarkmind/sc2/emulated/PlayerStateView.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Building;
import io.quarkmind.domain.Unit;

import java.util.List;

public interface PlayerStateView {
    double minerals();
    int vespene();
    int supply();
    int supplyUsed();
    List<Unit> units();
    List<Building> buildings();
}
```

- [ ] **Step 2: Create `ProductionDecision`**

```java
// src/main/java/io/quarkmind/sc2/emulated/ProductionDecision.java
package io.quarkmind.sc2.emulated;

public enum ProductionDecision { PROCEED, BLOCKED }
```

- [ ] **Step 3: Add `implements PlayerStateView` to `PlayerState`**

Open `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`. Change the class declaration from:
```java
public class PlayerState {
```
to:
```java
public class PlayerState implements PlayerStateView {
```

No other changes. All six methods (`minerals()`, `vespene()`, `supply()`, `supplyUsed()`, `units()`, `buildings()`) already exist with the exact signatures defined in `PlayerStateView`.

- [ ] **Step 4: Compile and confirm clean**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS, zero errors.

- [ ] **Step 5: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass. No behavioral change.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/PlayerStateView.java \
  src/main/java/io/quarkmind/sc2/emulated/ProductionDecision.java \
  src/main/java/io/quarkmind/sc2/emulated/PlayerState.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#165): add PlayerStateView interface and ProductionDecision enum"
```

---

## Task 2: Wire `MuleCalldownIntent` into the sealed `Intent` hierarchy (atomic)

**Convention (GE-20260418-9b272f):** The `Intent` permits clause and every switch over `Intent` must update in the same commit. Breaking this atomicity causes compile failures across the codebase.

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/intent/MuleCalldownIntent.java`
- Modify: `src/main/java/io/quarkmind/sc2/intent/Intent.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java`

- [ ] **Step 1: Create `MuleCalldownIntent`**

```java
// src/main/java/io/quarkmind/sc2/intent/MuleCalldownIntent.java
package io.quarkmind.sc2.intent;

public record MuleCalldownIntent(String buildingTag) implements Intent {}
```

- [ ] **Step 2: Add `MuleCalldownIntent` to `Intent` permits clause**

Open `src/main/java/io/quarkmind/sc2/intent/Intent.java`. Replace:
```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent {
}
```
with:
```java
public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent, MuleCalldownIntent {
}
```

- [ ] **Step 3: Add `MuleCalldownIntent` to the timed-intent switch in `EmulatedGame`**

Open `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`. Add an import at the top:
```java
import io.quarkmind.sc2.intent.MuleCalldownIntent;
```

Find the `applyIntent(TimedIntent ti)` method (currently around line 228). Its switch is:
```java
Runnable action = switch (ti.intent()) {
    case TrainIntent  t -> () -> handleTrain(t, friendly, friendlyPhysics, ti.loop());
    case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), friendly, friendlyPhysics);
    case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), friendly, friendlyPhysics);
    case BuildIntent  b -> () -> handleBuild(b, friendly, friendlyPhysics, ti.loop());
    case BlinkIntent  b -> () -> executeBlink(b.unitTag(), friendly, friendlyPhysics);
};
```

Add the new case:
```java
Runnable action = switch (ti.intent()) {
    case TrainIntent        t -> () -> handleTrain(t, friendly, friendlyPhysics, ti.loop());
    case MoveIntent         m -> () -> setTarget(m.unitTag(), m.targetLocation(), friendly, friendlyPhysics);
    case AttackIntent       a -> () -> setTarget(a.unitTag(), a.targetLocation(), friendly, friendlyPhysics);
    case BuildIntent        b -> () -> handleBuild(b, friendly, friendlyPhysics, ti.loop());
    case BlinkIntent        b -> () -> executeBlink(b.unitTag(), friendly, friendlyPhysics);
    case MuleCalldownIntent m -> () -> handleMuleCalldown(m, friendly, friendlyPhysics, ti.loop());
};
```

- [ ] **Step 4: Add `MuleCalldownIntent` to the immediate-intent switch in `EmulatedGame`**

Find the `applyIntent(Intent intent, PlayerState state, PhysicsState physics)` method (around line 239). Its switch is:
```java
Runnable action = switch (intent) {
    case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), state, physics);
    case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), state, physics);
    case TrainIntent  t -> () -> handleTrain(t, state, physics);
    case BuildIntent  b -> () -> handleBuild(b, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
    case BlinkIntent  b -> () -> executeBlink(b.unitTag(), state, physics);
};
```

Add the new case:
```java
Runnable action = switch (intent) {
    case MoveIntent         m -> () -> setTarget(m.unitTag(), m.targetLocation(), state, physics);
    case AttackIntent       a -> () -> setTarget(a.unitTag(), a.targetLocation(), state, physics);
    case TrainIntent        t -> () -> handleTrain(t, state, physics);
    case BuildIntent        b -> () -> handleBuild(b, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
    case BlinkIntent        b -> () -> executeBlink(b.unitTag(), state, physics);
    case MuleCalldownIntent m -> () -> handleMuleCalldown(m, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
};
```

- [ ] **Step 5: Add stub `handleMuleCalldown` to `EmulatedGame`**

Add this private method to `EmulatedGame` (alongside the other `handle*` methods):
```java
private void handleMuleCalldown(MuleCalldownIntent m, PlayerState state,
                                 PhysicsState physics, long absLoop) {
    // implemented in next task
}
```

- [ ] **Step 6: Add `MuleCalldownIntent` case to `ActionTranslator`**

Open `src/main/java/io/quarkmind/sc2/real/ActionTranslator.java`. Add import:
```java
import io.quarkmind.sc2.intent.MuleCalldownIntent;
```

The switch inside `translate()` is:
```java
ResolvedCommand cmd = switch (intent) {
    case BuildIntent  b -> build(b);
    case TrainIntent  t -> train(t);
    case AttackIntent a -> attack(a);
    case MoveIntent   m -> move(m);
    case BlinkIntent  b -> blink(b);
};
```

Add the case (returns null; the `if (cmd != null)` guard above already handles null):
```java
ResolvedCommand cmd = switch (intent) {
    case BuildIntent        b -> build(b);
    case TrainIntent        t -> train(t);
    case AttackIntent       a -> attack(a);
    case MoveIntent         m -> move(m);
    case BlinkIntent        b -> blink(b);
    case MuleCalldownIntent m -> null; // TODO: wire OC calldown ability for real SC2
};
```

- [ ] **Step 7: Compile and confirm clean**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS. The exhaustive switches must compile without a `default` clause.

- [ ] **Step 8: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass. No behavioral change — `handleMuleCalldown` is a no-op stub.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/intent/MuleCalldownIntent.java \
  src/main/java/io/quarkmind/sc2/intent/Intent.java \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/main/java/io/quarkmind/sc2/real/ActionTranslator.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#165): add MuleCalldownIntent — wire into Intent sealed type and all switches"
```

---

## Task 3: Implement `handleMuleCalldown` + `RaceModel.onCalldown` + `TerranRaceModel.onCalldown` (TDD)

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Write failing integration tests in `TerranEmulatedGameTest`**

Add these two tests to `TerranEmulatedGameTest` after the existing MULE section (`// --- MULE ---`). Add import `import io.quarkmind.sc2.intent.MuleCalldownIntent;` at the top.

```java
@Test
void muleCalldown_ocPresent_spawnsMuleImmediately() {
    final Building oc = game.spawnBuildingForTesting(BuildingType.ORBITAL_COMMAND, new Point2d(12, 8));

    game.applyIntent(new MuleCalldownIntent(oc.tag()));

    final long muleCount = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.MULE).count();
    assertThat(muleCount).isEqualTo(1);
}

@Test
void muleCalldown_tagNotAnOc_noUnitAdded() {
    // CC tag — building exists but type is COMMAND_CENTER, not ORBITAL_COMMAND
    final Building cc = game.snapshot().myBuildings().stream()
        .filter(b -> b.type() == BuildingType.COMMAND_CENTER)
        .findFirst().orElseThrow();

    final int unitsBefore = game.snapshot().myUnits().size();
    game.applyIntent(new MuleCalldownIntent(cc.tag()));

    assertThat(game.snapshot().myUnits()).hasSize(unitsBefore);
}
```

- [ ] **Step 2: Run the new tests to confirm they fail**

```bash
mvn test -Dtest=TerranEmulatedGameTest#muleCalldown_ocPresent_spawnsMuleImmediately+muleCalldown_tagNotAnOc_noUnitAdded -q
```

Expected: both FAIL — `handleMuleCalldown` is a no-op.

- [ ] **Step 3: Add `onCalldown` default method to `RaceModel`**

Open `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`. Add this default method after `onUnitSpawned`:

```java
/**
 * Handle a direct calldown ability for this race (e.g. MULE calldown from Orbital Command).
 * Called by EmulatedGame after OC building validation succeeds.
 * May call addUnit/removeUnit on state and update model-internal state.
 * Must NOT manipulate resource fields (minerals, vespene, supply, supplyUsed).
 * Default: no-op.
 *
 * @param absLoop absolute game loop (gameFrame × LOOPS_PER_TICK) when calldown was issued
 */
default void onCalldown(PlayerState state, String buildingTag, long absLoop) {}
```

- [ ] **Step 4: Implement `onCalldown` in `TerranRaceModel`**

Open `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`. Add this method (after `onUnitSpawned` and before `trainCount`):

```java
@Override
public void onCalldown(final PlayerState state, final String buildingTag, final long absLoop) {
    final Building oc = state.buildings().stream()
        .filter(b -> b.tag().equals(buildingTag) && b.isComplete())
        .findFirst().orElse(null);
    if (oc == null) return;
    final String muleTag = "mule-" + buildingTag + "-" + absLoop;
    final int hp = SC2Data.maxHealth(UnitType.MULE);
    state.addUnit(new Unit(muleTag, UnitType.MULE, oc.position(), hp, hp, 0, 0, 0, 0));
    muleExpiresAtLoop.put(muleTag, absLoop + SC2Data.MULE_LIFETIME_LOOPS);
}
```

- [ ] **Step 5: Implement `handleMuleCalldown` in `EmulatedGame`**

Replace the stub `handleMuleCalldown` with the full implementation:

```java
private void handleMuleCalldown(final MuleCalldownIntent m, final PlayerState state,
                                 final PhysicsState physics, final long absLoop) {
    final boolean ocPresent = state.buildings().stream()
        .anyMatch(b -> b.tag().equals(m.buildingTag()) && b.isComplete()
                  && b.type() == BuildingType.ORBITAL_COMMAND);
    if (!ocPresent) {
        log.debugf("[EMULATED] MULE calldown rejected — OC %s not ready", m.buildingTag());
        return;
    }
    final RaceModel model = (state == friendly) ? playerRaceModel : null;
    if (model != null) model.onCalldown(state, m.buildingTag(), absLoop);
}
```

- [ ] **Step 6: Run the new tests to confirm they pass**

```bash
mvn test -Dtest=TerranEmulatedGameTest#muleCalldown_ocPresent_spawnsMuleImmediately+muleCalldown_tagNotAnOc_noUnitAdded -q
```

Expected: both PASS.

- [ ] **Step 7: Run full Terran test class**

```bash
mvn test -Dtest=TerranEmulatedGameTest -q
```

Expected: all tests pass.

- [ ] **Step 8: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/RaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#165): implement handleMuleCalldown and TerranRaceModel.onCalldown"
```

---

## Task 4: Update `RaceModel.canProduce` signature + all implementations + delete `ProductionResult` (atomic)

This task changes the public interface. All implementations must update in the same commit. The two existing MULE tests that dispatch `TrainIntent(oc, MULE)` must also change here — once `TerranRaceModel.canProduce` returns `PROCEED` for MULE, the `TrainIntent(MULE)` path fails (no queue exists for MULE, and the building type check passes but behavior differs).

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Delete: `src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java`

- [ ] **Step 1: Update `RaceModel.canProduce` signature and fix stale Javadoc**

Open `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`. Replace the entire class-level Javadoc and `canProduce` declaration:

Replace the stale class Javadoc paragraph:
```java
 * <p>Lives in {@code sc2.emulated} alongside {@link PlayerState} (package-private) so
 * implementations have direct field access. When #74 (pluggable races) arrives, PlayerState
 * will need a public mutator API and implementations can move to external modules.
```
with:
```java
 * <p>Lives in {@code sc2.emulated} alongside {@link PlayerState}. {@code PlayerState} is
 * public (#164) — implementations outside this package may read and mutate it.
 * When #74 (pluggable races) arrives, implementations can move to external modules.
```

Replace the `canProduce` method signature and Javadoc:
```java
    /**
     * Query whether production can proceed for the given unit from the given building.
     * Called after building validation but BEFORE resource deduction.
     *
     * <p>The view is read-only by construction — structural enforcement replaces the
     * prior doc-only constraint. Return BLOCKED when a race-specific resource is
     * unavailable (e.g. no larva). Calldown abilities (e.g. MULE) route through
     * {@link #onCalldown} via {@code MuleCalldownIntent} — never through this method.
     *
     * @return PROCEED if resources are available (or not applicable for this race/unit),
     *         BLOCKED if a race-specific resource is unavailable
     */
    ProductionDecision canProduce(PlayerStateView view, String buildingTag, UnitType unitType);
```

- [ ] **Step 2: Update `TerranRaceModel.canProduce`**

Open `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`.

Replace the existing `canProduce` method (the entire block from `@Override` through the closing `}`):
```java
@Override
public ProductionDecision canProduce(final PlayerStateView view, final String buildingTag,
                                     final UnitType unitType) {
    return ProductionDecision.PROCEED;
}
```

Also remove the `currentGameLoop` field declaration and its assignment in `tickPassive` — it is no longer used:

Remove the field:
```java
private long currentGameLoop;
```

Remove this line from `tickPassive`:
```java
currentGameLoop = gameLoop;
```

Remove any imports of `ProductionResult` if present.

- [ ] **Step 3: Update `ZergRaceModel.canProduce`**

Open `src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java`.

Replace the `canProduce` method:
```java
@Override
public ProductionDecision canProduce(final PlayerStateView view, final String buildingTag,
                                     final UnitType unitType) {
    if (hatcheryLarvaCount.getOrDefault(buildingTag, 0) > 0) return ProductionDecision.PROCEED;
    return ProductionDecision.BLOCKED;
}
```

Remove any import of `ProductionResult`.

- [ ] **Step 4: Update `ProtossRaceModel.canProduce`**

Open `src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java`.

Replace the `canProduce` method:
```java
@Override
public ProductionDecision canProduce(final PlayerStateView view, final String buildingTag,
                                     final UnitType unitType) {
    return ProductionDecision.PROCEED;
}
```

Remove any import of `ProductionResult`.

- [ ] **Step 5: Update `EmulatedGame.handleTrain` to use new signature and enum**

Open `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`. Find the `handleTrain` race model check (currently around line 278–290). Replace:

```java
        final ProductionResult pr = model.canProduce(state, buildingTag, t.unitType());
        if (pr == ProductionResult.BLOCKED) {
            log.debugf("[EMULATED] Train rejected — production resource unavailable for %s", t.unitType());
            return;
        }
        if (pr == ProductionResult.HANDLED) {
            log.debugf("[EMULATED] Train handled by race model (MULE) for %s", t.unitType());
            return;
        }
```

with:
```java
        if (model.canProduce(state, buildingTag, t.unitType()) == ProductionDecision.BLOCKED) {
            log.debugf("[EMULATED] Train rejected — production resource unavailable for %s", t.unitType());
            return;
        }
```

Remove any import of `ProductionResult` from `EmulatedGame`.

- [ ] **Step 6: Delete `ProductionResult.java`**

```bash
rm /Users/mdproctor/claude/casehub/quarkmind/src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java
```

- [ ] **Step 7: Update the two existing MULE tests in `TerranEmulatedGameTest`**

The two tests that dispatch `new io.quarkmind.sc2.intent.TrainIntent(oc.tag(), UnitType.MULE)` must change to `new MuleCalldownIntent(oc.tag())`. The import `import io.quarkmind.sc2.intent.MuleCalldownIntent;` was already added in Task 3.

In `muleSpawn_appearsInSnapshot_andAddsIncomePerTick`, replace:
```java
game.applyIntent(new io.quarkmind.sc2.intent.TrainIntent(oc.tag(), UnitType.MULE));
```
with:
```java
game.applyIntent(new MuleCalldownIntent(oc.tag()));
```

In `muleExpires_afterLifetime_unitGoneAndNoMoreMuleIncome`, replace:
```java
game.applyIntent(new io.quarkmind.sc2.intent.TrainIntent(oc.tag(), UnitType.MULE));
```
with:
```java
game.applyIntent(new MuleCalldownIntent(oc.tag()));
```

- [ ] **Step 8: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS. If any `ProductionResult` references remain, the compiler will flag them — fix any stragglers.

- [ ] **Step 9: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/sc2/emulated/RaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java \
  src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
  src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind rm \
  src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#165): enforce canProduce read-only via PlayerStateView + ProductionDecision enum, delete ProductionResult"
```

---

## Task 5: Add isolation unit tests

Direct unit tests on the model classes and the view interface. These test the structural enforcement specifically — not just end-to-end through `EmulatedGame`.

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java`

- [ ] **Step 1: Write `TerranRaceModel` isolation tests**

Add these tests to `TerranEmulatedGameTest` (they use package-private access since the test is in `io.quarkmind.sc2.emulated`). Add import `import io.quarkmind.domain.Building;` if not already present.

```java
// --- RaceModel isolation tests ---

@Test
void canProduce_alwaysReturnsProceed_forAnyUnitType() {
    final TerranRaceModel model = new TerranRaceModel();
    final PlayerState state = new PlayerState();
    // PlayerState implements PlayerStateView — passes as view
    assertThat(model.canProduce(state, "any-tag", UnitType.SCV))
        .isEqualTo(ProductionDecision.PROCEED);
    assertThat(model.canProduce(state, "any-tag", UnitType.MARINE))
        .isEqualTo(ProductionDecision.PROCEED);
    assertThat(model.canProduce(state, "any-tag", UnitType.MULE))
        .isEqualTo(ProductionDecision.PROCEED);
}

@Test
void onCalldown_spawnsAndRegistersExpiry() {
    final TerranRaceModel model = new TerranRaceModel();
    final PlayerState state = new PlayerState();
    state.addBuilding(new Building("oc-0", BuildingType.ORBITAL_COMMAND,
        new Point2d(8, 8), 1500, 1500, true));

    model.onCalldown(state, "oc-0", 100L);

    assertThat(state.units()).hasSize(1);
    assertThat(state.units().get(0).type()).isEqualTo(UnitType.MULE);
    assertThat(model.activeMuleCount()).isEqualTo(1);
}

@Test
void onCalldown_buildingTagMissing_isNoOp() {
    final TerranRaceModel model = new TerranRaceModel();
    final PlayerState state = new PlayerState();
    // No buildings added — tag won't be found

    model.onCalldown(state, "oc-0", 100L);

    assertThat(state.units()).isEmpty();
    assertThat(model.activeMuleCount()).isEqualTo(0);
}
```

- [ ] **Step 2: Write `ZergRaceModel` view-enforcement tests**

Add these tests to `ZergEmulatedGameTest`. Add import `import java.util.ArrayList;` if not present.

```java
// --- canProduce view-enforcement tests ---

@Test
void canProduce_withView_noLarva_returnsBlocked() {
    final ZergRaceModel model = new ZergRaceModel();
    final PlayerState state = new PlayerState();
    // hatcheryLarvaCount is empty — BLOCKED regardless of PlayerStateView content
    assertThat(model.canProduce(state, "hatch-0", UnitType.DRONE))
        .isEqualTo(ProductionDecision.BLOCKED);
}

@Test
void canProduce_withView_larvaAvailable_returnsProceed() {
    final ZergRaceModel model = new ZergRaceModel();
    final PlayerState state = new PlayerState();
    model.seedInitialState(state, new ArrayList<>());
    // seedInitialState adds "hatchery-0" with 3 larva
    final String hatchTag = state.buildings().get(0).tag();

    assertThat(model.canProduce(state, hatchTag, UnitType.DRONE))
        .isEqualTo(ProductionDecision.PROCEED);
}
```

- [ ] **Step 3: Run new tests to confirm they pass**

```bash
mvn test -Dtest=TerranEmulatedGameTest -q
mvn test -Dtest=ZergEmulatedGameTest -q
```

Expected: all tests pass.

- [ ] **Step 4: Run full test suite**

```bash
mvn test -q
```

Expected: all tests pass. Note the count — it should be higher than before this task.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java \
  src/test/java/io/quarkmind/sc2/emulated/ZergEmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#165): add isolation tests for PlayerStateView enforcement and onCalldown"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered in |
|---|---|
| `PlayerStateView` public interface with 6 accessors | Task 1 |
| `PlayerState implements PlayerStateView` | Task 1 |
| `ProductionDecision` enum `{ PROCEED, BLOCKED }` | Task 1 |
| `MuleCalldownIntent(buildingTag)` record in `sc2.intent` | Task 2 |
| `Intent` permits clause updated atomically | Task 2 |
| All EmulatedGame Intent switches updated | Task 2 |
| `ActionTranslator` MuleCalldownIntent case (stub) | Task 2 |
| `RaceModel.onCalldown` default no-op | Task 3 |
| `TerranRaceModel.onCalldown` spawns MULE + registers expiry | Task 3 |
| `EmulatedGame.handleMuleCalldown` OC validation + dispatch | Task 3 |
| `RaceModel.canProduce` takes `PlayerStateView`, returns `ProductionDecision` | Task 4 |
| All three implementations updated | Task 4 |
| `handleTrain` simplified to equality check | Task 4 |
| `currentGameLoop` field removed from `TerranRaceModel` | Task 4 |
| Stale RaceModel Javadoc corrected | Task 4 |
| `ProductionResult` deleted | Task 4 |
| Existing MULE tests updated to `MuleCalldownIntent` | Task 4 |
| New integration tests: calldown OC present + absent | Task 3 |
| New isolation tests: canProduce trivial Terran, Zerg BLOCKED/PROCEED | Task 5 |
| New isolation tests: onCalldown spawn + no-op | Task 5 |
| PP-20260601-5fa812 compliance | All public types: Tasks 1, 2 |
| GE-20260418-9b272f atomicity | Task 2 |

All spec requirements covered. No gaps.
