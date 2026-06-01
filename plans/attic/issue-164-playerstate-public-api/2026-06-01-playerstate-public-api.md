# PlayerState Public API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split `PlayerState` into public game-state API (for external race plugins) and package-private physics machinery, enabling external `RaceModel` implementations per issue #164.

**Architecture:** Three-way concern split: `PlayerState` (public, private fields, typed mutation API) holds game state; new `PhysicsState` (package-private) holds EmulatedGame's physics machinery; `EnemyBehavior.stagingArea` holds the enemy retreat buffer. `RaceModel`, `ProductionResult`, and `RaceModelFactory` promoted to public. Race model implementations migrated to the new public API to validate it.

**Tech Stack:** Java 21, plain JUnit 5 + AssertJ (no CDI/Quarkus), Maven (`mvn test -Dtest=ClassName -q`).

---

## File Map

| File | Action | What changes |
|---|---|---|
| `src/main/java/io/quarkmind/sc2/emulated/PhysicsState.java` | **Create** | New package-private class holding physics fields and `PendingCompletion` |
| `src/test/java/io/quarkmind/sc2/emulated/PhysicsStateTest.java` | **Create** | Tests for `fireCompletions` and `clear` |
| `src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java` | **Modify** | Add `stagingArea` field; add `stagingArea.clear()` to `reset()` |
| `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java` | **Rewrite** | Public class, private fields, full typed public+package-private API |
| `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java` | **Modify** | `interface` → `public interface` |
| `src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java` | **Modify** | `enum` → `public enum` |
| `src/main/java/io/quarkmind/sc2/emulated/RaceModelFactory.java` | **Modify** | `class` → `public class` |
| `src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java` | **Modify** | Direct field access → public API calls |
| `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java` | **Modify** | Direct field access → public API calls |
| `src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java` | **Modify** | Direct field access → public API calls |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | **Modify** | Add `PhysicsState` fields; update all internal methods to pass PhysicsState; migrate game-state and physics-field accesses |
| `src/test/java/io/quarkmind/sc2/emulated/PlayerStateTest.java` | **Modify** | Remove physics tests (moved to PhysicsStateTest); update game-state tests to use new API |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | **Modify** | Update 6 `unitTargets` sites, 2 `stagingArea` sites, 1 `minerals =` site, fog-of-war staging test |

---

## Task 1: Create `PhysicsState` and `PhysicsStateTest`

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/PhysicsState.java`
- Create: `src/test/java/io/quarkmind/sc2/emulated/PhysicsStateTest.java`

- [ ] **Step 1.1: Write failing tests**

```java
// src/test/java/io/quarkmind/sc2/emulated/PhysicsStateTest.java
package io.quarkmind.sc2.emulated;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class PhysicsStateTest {

    @Test
    void fireCompletions_runsActionAtCorrectTick() {
        PhysicsState s = new PhysicsState();
        boolean[] fired = {false};
        s.pendingCompletions.add(new PhysicsState.PendingCompletion(5L, () -> fired[0] = true));
        s.fireCompletions(4L);
        assertThat(fired[0]).isFalse();
        s.fireCompletions(5L);
        assertThat(fired[0]).isTrue();
        assertThat(s.pendingCompletions).isEmpty();
    }

    @Test
    void fireCompletions_doesNotFireFutureCompletions() {
        PhysicsState s = new PhysicsState();
        boolean[] fired = {false};
        s.pendingCompletions.add(new PhysicsState.PendingCompletion(10L, () -> fired[0] = true));
        s.fireCompletions(5L);
        assertThat(fired[0]).isFalse();
        assertThat(s.pendingCompletions).hasSize(1);
    }

    @Test
    void fireCompletions_firesMultipleCompletionsAtSameTick() {
        PhysicsState s = new PhysicsState();
        int[] count = {0};
        s.pendingCompletions.add(new PhysicsState.PendingCompletion(3L, () -> count[0]++));
        s.pendingCompletions.add(new PhysicsState.PendingCompletion(3L, () -> count[0]++));
        s.fireCompletions(3L);
        assertThat(count[0]).isEqualTo(2);
        assertThat(s.pendingCompletions).isEmpty();
    }

    @Test
    void clear_resetsAllFields() {
        PhysicsState s = new PhysicsState();
        s.unitTargets.put("u1", new io.quarkmind.domain.Point2d(1, 1));
        s.unitCooldowns.put("u1", 3);
        s.blinkCooldowns.put("u1", 2);
        s.buildingQueues.put("b1", new java.util.ArrayDeque<>());
        s.buildingTrainingUntil.put("b1", 10L);
        s.buildingCompletionAtLoop.put("b1", 220L);
        s.pendingCompletions.add(new PhysicsState.PendingCompletion(999L, () -> {}));
        s.clear();
        assertThat(s.unitTargets).isEmpty();
        assertThat(s.unitCooldowns).isEmpty();
        assertThat(s.blinkCooldowns).isEmpty();
        assertThat(s.buildingQueues).isEmpty();
        assertThat(s.buildingTrainingUntil).isEmpty();
        assertThat(s.buildingCompletionAtLoop).isEmpty();
        assertThat(s.pendingCompletions).isEmpty();
    }
}
```

- [ ] **Step 1.2: Run tests — expect compilation failure** (`PhysicsState` does not exist yet)

```
mvn test -Dtest=PhysicsStateTest -q 2>&1 | head -20
```

Expected: compilation error mentioning `PhysicsState`.

- [ ] **Step 1.3: Implement `PhysicsState`**

```java
// src/main/java/io/quarkmind/sc2/emulated/PhysicsState.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;

import java.util.*;

/**
 * Internal simulation state not exposed to external observers: production queues,
 * movement targets, and combat cooldowns. Contains both production scheduling state
 * (buildingQueues, pendingCompletions, buildingTrainingUntil, buildingCompletionAtLoop)
 * and movement/combat state (unitTargets, unitCooldowns, blinkCooldowns) — grouped here
 * because both categories are EmulatedGame internals that race model plugins never touch.
 *
 * <p>A PendingCompletion lambda captures the PlayerState and PhysicsState pair that was
 * in scope when it was registered. Never transfer a PendingCompletion between physics
 * objects — it references a specific player's pair.
 */
class PhysicsState {

    // Movement and combat
    final Map<String, Point2d> unitTargets    = new HashMap<>();
    final Map<String, Integer> unitCooldowns  = new HashMap<>();
    final Map<String, Integer> blinkCooldowns = new HashMap<>();

    // Production machinery
    final Map<String, Deque<UnitType>> buildingQueues           = new HashMap<>();
    final Map<String, Long>            buildingTrainingUntil    = new HashMap<>();
    final Map<String, Long>            buildingCompletionAtLoop = new HashMap<>();

    // In-flight completion callbacks
    final List<PendingCompletion> pendingCompletions = new ArrayList<>();

    void fireCompletions(long currentTick) {
        pendingCompletions.removeIf(item -> {
            if (item.completesAtTick() > currentTick) return false;
            item.action().run();
            return true;
        });
    }

    void clear() {
        unitTargets.clear();
        unitCooldowns.clear();
        blinkCooldowns.clear();
        buildingQueues.clear();
        buildingTrainingUntil.clear();
        buildingCompletionAtLoop.clear();
        pendingCompletions.clear();
    }

    record PendingCompletion(long completesAtTick, Runnable action) {}
}
```

- [ ] **Step 1.4: Run tests — expect pass**

```
mvn test -Dtest=PhysicsStateTest -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 1.5: Compile check — full build passes**

```
mvn compile -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 1.6: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/PhysicsState.java src/test/java/io/quarkmind/sc2/emulated/PhysicsStateTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): add PhysicsState — package-private physics machinery container"
```

---

## Task 2: Add `stagingArea` to `EnemyBehavior` and update `reset()`

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EnemyBehaviorTest.java`

- [ ] **Step 2.1: Write failing test for `reset_clearsStagingArea`**

Add this test to `EnemyBehaviorTest` after the existing `setUp` method. The test accesses `behavior.stagingArea` which does not exist yet — it will fail to compile.

```java
@Test
void reset_clearsStagingArea() {
    int hp = SC2Data.maxHealth(UnitType.ZEALOT);
    behavior.stagingArea.add(new Unit("z1", UnitType.ZEALOT, new Point2d(5, 5),
        hp, hp, SC2Data.maxShields(UnitType.ZEALOT), SC2Data.maxShields(UnitType.ZEALOT), 0, 0));
    assertThat(behavior.stagingArea).hasSize(1);
    behavior.reset(zealotSpam());
    assertThat(behavior.stagingArea).isEmpty();
}
```

- [ ] **Step 2.2: Run test — expect compilation failure**

```
mvn test -Dtest=EnemyBehaviorTest -q 2>&1 | head -10
```

Expected: compilation error — `stagingArea` not found on `EnemyBehavior`.

- [ ] **Step 2.3: Add `stagingArea` field to `EnemyBehavior` and clear it in `reset()`**

Open `EnemyBehavior.java`. Add the field in the "Retreat state" section (alongside the existing `retreating` field). Find the `reset()` method and add `stagingArea.clear()`:

Field (add after the `retreating` field declaration):
```java
// Staging area — units that have retreated; re-deployed each wave. Package-private for EmulatedGame.
final List<Unit> stagingArea = new ArrayList<>();
```

In `reset(EnemyStrategy strategy)` (find the method and add one line at the end of its body):
```java
stagingArea.clear();
```

- [ ] **Step 2.4: Run test — expect pass**

```
mvn test -Dtest=EnemyBehaviorTest -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 2.5: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/EnemyBehavior.java src/test/java/io/quarkmind/sc2/emulated/EnemyBehaviorTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): add EnemyBehavior.stagingArea field; clear in reset()"
```

---

## Task 3: Add typed public API to `PlayerState`

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/PlayerStateTest.java`

At this stage, PlayerState gains new public+package-private methods. The existing package-private fields stay — nothing breaks yet.

- [ ] **Step 3.1: Write failing tests for new public API**

Add these tests to `PlayerStateTest` (the existing tests remain unchanged — the fields are still accessible):

```java
// --- New public API tests ---

@Test
void addUnit_appendsAndUnits_returnsUnmodifiableView() {
    PlayerState s = new PlayerState();
    Unit u = new Unit("u1", UnitType.ZEALOT, new Point2d(1,1), 100,100,50,50,0,0);
    s.addUnit(u);
    assertThat(s.units()).containsExactly(u);
    assertThatThrownBy(() -> s.units().add(u))
        .isInstanceOf(UnsupportedOperationException.class);
}

@Test
void removeUnit_byTag_removesCorrectUnit() {
    PlayerState s = new PlayerState();
    s.addUnit(new Unit("u1", UnitType.ZEALOT, new Point2d(1,1), 100,100,50,50,0,0));
    s.addUnit(new Unit("u2", UnitType.ZEALOT, new Point2d(2,2), 100,100,50,50,0,0));
    s.removeUnit("u1");
    assertThat(s.units()).extracting(Unit::tag).containsExactly("u2");
}

@Test
void addBuilding_appendsAndBuildings_returnsUnmodifiableView() {
    PlayerState s = new PlayerState();
    Building b = new Building("b1", BuildingType.NEXUS, new Point2d(8,8), 900,900,true);
    s.addBuilding(b);
    assertThat(s.buildings()).containsExactly(b);
    assertThatThrownBy(() -> s.buildings().add(b))
        .isInstanceOf(UnsupportedOperationException.class);
}

@Test
void setMinerals_and_minerals() {
    PlayerState s = new PlayerState();
    s.setMinerals(150.0);
    assertThat(s.minerals()).isEqualTo(150.0);
}

@Test
void addMinerals_accumulates() {
    PlayerState s = new PlayerState();
    s.setMinerals(100.0);
    s.addMinerals(25.5);
    assertThat(s.minerals()).isEqualTo(125.5);
}

@Test
void deductMinerals_reduces() {
    PlayerState s = new PlayerState();
    s.setMinerals(100.0);
    s.deductMinerals(50.0);
    assertThat(s.minerals()).isEqualTo(50.0);
}

@Test
void setVespene_and_deductVespene() {
    PlayerState s = new PlayerState();
    s.setVespene(200);
    s.deductVespene(75);
    assertThat(s.vespene()).isEqualTo(125);
}

@Test
void setSupply_and_addSupply() {
    PlayerState s = new PlayerState();
    s.setSupply(15);
    s.addSupply(8);
    assertThat(s.supply()).isEqualTo(23);
}

@Test
void setSupplyUsed_and_addSupplyUsed() {
    PlayerState s = new PlayerState();
    s.setSupplyUsed(10);
    s.addSupplyUsed(2);
    assertThat(s.supplyUsed()).isEqualTo(12);
}

@Test
void removeUnitsWhere_returnsRemovedUnitsAndRemovesThem() {
    PlayerState s = new PlayerState();
    Unit dead = new Unit("d1", UnitType.ZEALOT, new Point2d(1,1), 0,100,50,50,0,0);
    Unit alive = new Unit("a1", UnitType.ZEALOT, new Point2d(2,2), 100,100,50,50,0,0);
    s.addUnit(dead);
    s.addUnit(alive);
    List<Unit> removed = s.removeUnitsWhere(u -> u.health() <= 0);
    assertThat(removed).containsExactly(dead);
    assertThat(s.units()).containsExactly(alive);
}

@Test
void replaceAllUnits_updatesUnitsInPlace() {
    PlayerState s = new PlayerState();
    s.addUnit(new Unit("u1", UnitType.ZEALOT, new Point2d(1,1), 100,100,50,50,0,0));
    s.replaceAllUnits(u -> new Unit(u.tag(), u.type(), new Point2d(5,5),
        u.health(), u.maxHealth(), u.shields(), u.maxShields(), 0, 0));
    assertThat(s.units().get(0).position()).isEqualTo(new Point2d(5,5));
}

@Test
void replaceAllBuildings_updatesBuildings() {
    PlayerState s = new PlayerState();
    s.addBuilding(new Building("b1", BuildingType.NEXUS, new Point2d(8,8), 900,900,false));
    s.replaceAllBuildings(b -> b.tag().equals("b1")
        ? new Building(b.tag(), b.type(), b.position(), b.health(), b.maxHealth(), true)
        : b);
    assertThat(s.buildings().get(0).isComplete()).isTrue();
}
```

- [ ] **Step 3.2: Run tests — expect compilation failure** (methods don't exist yet)

```
mvn test -Dtest=PlayerStateTest -q 2>&1 | head -20
```

Expected: compilation errors on `addUnit`, `removeUnit`, `units()`, etc.

- [ ] **Step 3.3: Add public API methods to `PlayerState`**

Replace the entire contents of `PlayerState.java` with this. The existing package-private fields stay to avoid breaking other callers — they are removed in Task 6.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;

import java.util.*;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;

/**
 * Per-player mutable game state: units, buildings, and resources. Public so that
 * external {@link RaceModel} implementations (loaded from outside this package) can
 * read and mutate the state they need.
 *
 * <p>Public methods are semantic operations meaningful to a race plugin.
 * Package-private methods ({@code replaceAllUnits}, {@code removeUnitsWhere},
 * {@code replaceAllBuildings}, {@code clear}) are bulk structural operations used
 * only by EmulatedGame's physics engine.
 *
 * <p>Not thread-safe — all access is from the single game-tick scheduler thread.
 */
public class PlayerState {

    // --- Private game-state fields (package-private temporarily during migration) ---
    // These become private in Task 6 once all callers use the typed API.
    final List<Unit>               units               = new ArrayList<>();
    final List<Building>           buildings           = new ArrayList<>();
    double minerals;
    int    vespene;
    int    supply;
    int    supplyUsed;

    // --- Physics fields: TEMPORARY — moved to PhysicsState in Task 6 ---
    final List<Unit>               stagingArea              = new ArrayList<>();
    final List<PendingCompletion>  pendingCompletions       = new ArrayList<>();
    final Map<String, Deque<UnitType>> buildingQueues       = new HashMap<>();
    final Map<String, Long>            buildingTrainingUntil = new HashMap<>();
    final Map<String, Long>            buildingCompletionAtLoop = new HashMap<>();
    final Map<String, Point2d>         unitTargets    = new HashMap<>();
    final Map<String, Integer>         unitCooldowns  = new HashMap<>();
    final Map<String, Integer>         blinkCooldowns = new HashMap<>();

    // --- Public typed API ---

    public void setMinerals(double m)        { this.minerals = m; }
    public void addMinerals(double amount)   { this.minerals += amount; }
    public void deductMinerals(double cost)  { this.minerals -= cost; }
    public double minerals()                 { return minerals; }

    public void setVespene(int v)            { this.vespene = v; }
    public void deductVespene(int cost)      { this.vespene -= cost; }
    public int  vespene()                    { return vespene; }

    public void setSupply(int s)             { this.supply = s; }
    public void addSupply(int amount)        { this.supply += amount; }
    public int  supply()                     { return supply; }

    public void setSupplyUsed(int s)         { this.supplyUsed = s; }
    public void addSupplyUsed(int cost)      { this.supplyUsed += cost; }
    public int  supplyUsed()                 { return supplyUsed; }

    public void addUnit(Unit unit)           { units.add(unit); }
    public void removeUnit(String tag)       { units.removeIf(u -> u.tag().equals(tag)); }
    public List<Unit> units()               { return Collections.unmodifiableList(units); }

    public void addBuilding(Building b)      { buildings.add(b); }
    public List<Building> buildings()       { return Collections.unmodifiableList(buildings); }

    // --- Package-private bulk ops for EmulatedGame physics ---

    List<Unit> removeUnitsWhere(Predicate<Unit> pred) {
        List<Unit> removed = new ArrayList<>();
        units.removeIf(u -> {
            if (pred.test(u)) { removed.add(u); return true; }
            return false;
        });
        return List.copyOf(removed);
    }

    void replaceAllUnits(UnaryOperator<Unit> op)     { units.replaceAll(op); }
    void replaceAllBuildings(UnaryOperator<Building> op) { buildings.replaceAll(op); }

    void clear() {
        units.clear();
        buildings.clear();
        stagingArea.clear();
        pendingCompletions.clear();
        buildingQueues.clear();
        buildingTrainingUntil.clear();
        buildingCompletionAtLoop.clear();
        unitTargets.clear();
        unitCooldowns.clear();
        blinkCooldowns.clear();
        minerals   = 0;
        vespene    = 0;
        supply     = 0;
        supplyUsed = 0;
    }

    // --- Temporary: fireCompletions stays here during transition, moves to PhysicsState in Task 5 ---
    void fireCompletions(long currentTick) {
        pendingCompletions.removeIf(item -> {
            if (item.completesAtTick() > currentTick) return false;
            item.action().run();
            return true;
        });
    }

    record PendingCompletion(long completesAtTick, Runnable action) {}
}
```

- [ ] **Step 3.4: Run all PlayerState tests — expect pass**

```
mvn test -Dtest=PlayerStateTest -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3.5: Run full suite to verify no regressions**

```
mvn test -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3.6: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/PlayerState.java src/test/java/io/quarkmind/sc2/emulated/PlayerStateTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): add typed public API to PlayerState (fields remain accessible during migration)"
```

---

## Task 4: Migrate race model implementations to `PlayerState` public API

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java`

No new tests — existing race model tests (`TerranEmulatedGameTest`, `ZergEmulatedGameTest`, `EmulatedGameTest`) provide coverage. The goal is to switch all direct field access to public API calls so the implementations work from any package.

- [ ] **Step 4.1: Migrate `ProtossRaceModel`**

Replace every direct field access with the public API. The full updated class:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import java.util.List;
import java.util.Set;
import java.util.function.Supplier;

class ProtossRaceModel implements RaceModel {

    @Override
    public void seedInitialState(final PlayerState state, final List<Resource> geysers) {
        state.setMinerals(SC2Data.INITIAL_MINERALS);
        state.setVespene(SC2Data.INITIAL_VESPENE);
        state.setSupply(SC2Data.INITIAL_SUPPLY);
        state.setSupplyUsed(SC2Data.INITIAL_SUPPLY_USED);

        for (int i = 0; i < SC2Data.INITIAL_PROBES; i++) {
            final int hp = SC2Data.maxHealth(UnitType.PROBE);
            state.addUnit(new Unit("probe-" + i, UnitType.PROBE,
                new Point2d(9 + i * 0.5f, 9),
                hp, hp, SC2Data.maxShields(UnitType.PROBE), SC2Data.maxShields(UnitType.PROBE), 0, 0));
        }
        state.addBuilding(new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8),
            SC2Data.maxBuildingHealth(BuildingType.NEXUS),
            SC2Data.maxBuildingHealth(BuildingType.NEXUS),
            true));

        geysers.add(new Resource("geyser-0", new Point2d(5, 11), 2250));
        geysers.add(new Resource("geyser-1", new Point2d(11, 5), 2250));
    }

    @Override
    public void tickPassive(final PlayerState state, final long gameLoop) {}

    @Override
    public ProductionResult canProduce(final PlayerState state, final String buildingTag,
                                       final UnitType unitType) {
        return ProductionResult.PROCEED;
    }

    @Override
    public void onProductionCommitted(final PlayerState state, final String buildingTag,
                                      final UnitType unitType, final Supplier<String> tagSupplier) {}

    @Override
    public void onUnitSpawned(final PlayerState state, final UnitType type,
                              final String unitTag, final String buildingTag) {}

    private static final Set<BuildingType> TOWN_HALLS = Set.of(BuildingType.NEXUS);

    @Override public UnitType workerType()          { return UnitType.PROBE; }
    @Override public Set<BuildingType> townHallTypes() { return TOWN_HALLS; }
}
```

- [ ] **Step 4.2: Migrate `TerranRaceModel`**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import java.util.*;
import java.util.function.Supplier;

class TerranRaceModel implements RaceModel {

    static final int INITIAL_WORKERS = 12;

    private final Map<String, Long> muleExpiresAtLoop = new HashMap<>();
    private long currentGameLoop;

    @Override
    public void seedInitialState(final PlayerState state, final List<Resource> geysers) {
        muleExpiresAtLoop.clear();
        currentGameLoop = 0;

        state.setMinerals(SC2Data.INITIAL_MINERALS);
        state.setVespene(SC2Data.INITIAL_VESPENE);
        state.setSupply(SC2Data.INITIAL_SUPPLY);
        state.setSupplyUsed(SC2Data.INITIAL_SUPPLY_USED);

        for (int i = 0; i < INITIAL_WORKERS; i++) {
            final int hp = SC2Data.maxHealth(UnitType.SCV);
            state.addUnit(new Unit("scv-" + i, UnitType.SCV,
                new Point2d(9 + i * 0.5f, 9), hp, hp, 0, 0, 0, 0));
        }
        state.addBuilding(new Building("cc-0", BuildingType.COMMAND_CENTER,
            new Point2d(8, 8),
            SC2Data.maxBuildingHealth(BuildingType.COMMAND_CENTER),
            SC2Data.maxBuildingHealth(BuildingType.COMMAND_CENTER),
            true));

        geysers.add(new Resource("geyser-0", new Point2d(5, 11), 2250));
        geysers.add(new Resource("geyser-1", new Point2d(11, 5), 2250));
    }

    @Override
    public void tickPassive(final PlayerState state, final long gameLoop) {
        currentGameLoop = gameLoop;

        final List<String> expired = new ArrayList<>();
        muleExpiresAtLoop.forEach((tag, expiresAt) -> {
            if (gameLoop >= expiresAt) expired.add(tag);
        });
        expired.forEach(tag -> {
            muleExpiresAtLoop.remove(tag);
            state.removeUnit(tag);
        });

        if (!muleExpiresAtLoop.isEmpty()) {
            state.addMinerals(muleExpiresAtLoop.size() * SC2Data.muleIncomePerTick());
        }
    }

    @Override
    public ProductionResult canProduce(final PlayerState state, final String buildingTag,
                                       final UnitType unitType) {
        if (unitType == UnitType.MULE) {
            final Building oc = state.buildings().stream()
                .filter(b -> b.tag().equals(buildingTag) && b.isComplete())
                .findFirst().orElse(null);
            if (oc == null) return ProductionResult.BLOCKED;

            final String muleTag = "mule-" + buildingTag + "-" + currentGameLoop;
            final int hp = SC2Data.maxHealth(UnitType.MULE);
            state.addUnit(new Unit(muleTag, UnitType.MULE, oc.position(),
                hp, hp, 0, 0, 0, 0));
            muleExpiresAtLoop.put(muleTag, currentGameLoop + SC2Data.MULE_LIFETIME_LOOPS);
            return ProductionResult.HANDLED;
        }
        return ProductionResult.PROCEED;
    }

    @Override
    public void onProductionCommitted(final PlayerState state, final String buildingTag,
                                      final UnitType unitType, final Supplier<String> tagSupplier) {}

    @Override
    public void onUnitSpawned(final PlayerState state, final UnitType type,
                              final String unitTag, final String buildingTag) {}

    @Override
    public int trainCount(final UnitType type) { return 1; }

    private static final Set<BuildingType> TOWN_HALLS =
        Set.of(BuildingType.COMMAND_CENTER, BuildingType.ORBITAL_COMMAND,
               BuildingType.PLANETARY_FORTRESS);

    @Override public UnitType workerType()          { return UnitType.SCV; }
    @Override public Set<BuildingType> townHallTypes() { return TOWN_HALLS; }

    int activeMuleCount() { return muleExpiresAtLoop.size(); }
}
```

- [ ] **Step 4.3: Migrate `ZergRaceModel`**

The key changes: `state.minerals += x` → `state.addMinerals(x)`, `state.units.add(u)` → `state.addUnit(u)`, `state.units.removeIf(...)` → `state.removeUnit(tag)`, `state.supply += x` → `state.addSupply(x)`, all `state.buildings` reads → `state.buildings()`, all `state.units` reads → `state.units()`.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import java.util.*;
import java.util.function.Supplier;

class ZergRaceModel implements RaceModel {

    static final int INITIAL_WORKERS   = 12;
    static final int INITIAL_SUPPLY    = 14;
    static final int MAX_LARVA         = 3;
    static final int MAX_INJECT_LARVA  = 19;
    static final long LARVA_REGEN_LOOPS = 245L;
    static final double INJECT_COST_ENERGY = 25.0;
    static final int    INJECT_LARVA_COUNT = 4;

    private final Map<String, Integer> hatcheryLarvaCount    = new HashMap<>();
    private final Map<String, Long>    hatcheryNextLarvaLoop = new HashMap<>();
    private final Map<String, Deque<String>> eggTagByBuilding = new HashMap<>();
    private final Map<String, Double>  queenEnergyMap        = new HashMap<>();
    private long currentGameLoop;

    @Override
    public void seedInitialState(final PlayerState state, final List<Resource> geysers) {
        hatcheryLarvaCount.clear();
        hatcheryNextLarvaLoop.clear();
        eggTagByBuilding.clear();
        queenEnergyMap.clear();
        currentGameLoop = 0;

        state.setMinerals(SC2Data.INITIAL_MINERALS);
        state.setVespene(SC2Data.INITIAL_VESPENE);
        state.setSupply(INITIAL_SUPPLY);
        state.setSupplyUsed(SC2Data.INITIAL_SUPPLY_USED);

        for (int i = 0; i < INITIAL_WORKERS; i++) {
            final int hp = SC2Data.maxHealth(UnitType.DRONE);
            state.addUnit(new Unit("drone-" + i, UnitType.DRONE,
                new Point2d(9 + i * 0.5f, 9), hp, hp, 0, 0, 0, 0));
        }

        final String hatcheryTag = "hatchery-0";
        state.addBuilding(new Building(hatcheryTag, BuildingType.HATCHERY,
            new Point2d(8, 8),
            SC2Data.maxBuildingHealth(BuildingType.HATCHERY),
            SC2Data.maxBuildingHealth(BuildingType.HATCHERY),
            true));
        hatcheryLarvaCount.put(hatcheryTag, MAX_LARVA);

        final int overlordHp = SC2Data.maxHealth(UnitType.OVERLORD);
        state.addUnit(new Unit("overlord-0", UnitType.OVERLORD,
            new Point2d(14, 14), overlordHp, overlordHp, 0, 0, 0, 0));

        geysers.add(new Resource("geyser-0", new Point2d(5, 11), 2250));
        geysers.add(new Resource("geyser-1", new Point2d(11, 5), 2250));
    }

    @Override
    public void tickPassive(final PlayerState state, final long gameLoop) {
        currentGameLoop = gameLoop;

        for (final Building b : state.buildings()) {
            if (!townHallTypes().contains(b.type()) || !b.isComplete()) continue;
            final String tag = b.tag();
            final int count = hatcheryLarvaCount.getOrDefault(tag, 0);
            if (count >= MAX_LARVA) continue;
            final long nextLoop = hatcheryNextLarvaLoop.getOrDefault(tag, 0L);
            if (gameLoop >= nextLoop) {
                hatcheryLarvaCount.put(tag, count + 1);
                hatcheryNextLarvaLoop.put(tag, gameLoop + LARVA_REGEN_LOOPS);
            }
        }

        for (final Unit u : state.units()) {
            if (u.type() != UnitType.QUEEN) continue;
            final double energy = queenEnergyMap.getOrDefault(u.tag(), INJECT_COST_ENERGY);
            queenEnergyMap.put(u.tag(), Math.min(200.0,
                energy + SC2Data.QUEEN_ENERGY_REGEN_PER_LOOP * SC2Data.LOOPS_PER_TICK));
        }

        for (final Unit queen : state.units()) {
            if (queen.type() != UnitType.QUEEN) continue;
            final double energy = queenEnergyMap.getOrDefault(queen.tag(), 0.0);
            if (energy < INJECT_COST_ENERGY) continue;
            final Building nearest = nearestTownHall(queen.position(), state.buildings());
            if (nearest == null) continue;
            final String tag = nearest.tag();
            final int current = hatcheryLarvaCount.getOrDefault(tag, 0);
            hatcheryLarvaCount.put(tag, Math.min(MAX_INJECT_LARVA, current + INJECT_LARVA_COUNT));
            queenEnergyMap.put(queen.tag(), energy - INJECT_COST_ENERGY);
        }
    }

    private Building nearestTownHall(final Point2d pos, final List<Building> buildings) {
        Building nearest = null;
        double minDist = Double.MAX_VALUE;
        for (final Building b : buildings) {
            if (!townHallTypes().contains(b.type()) || !b.isComplete()) continue;
            final double dx = pos.x() - b.position().x();
            final double dy = pos.y() - b.position().y();
            final double d = dx * dx + dy * dy;
            if (d < minDist) { minDist = d; nearest = b; }
        }
        return nearest;
    }

    @Override
    public ProductionResult canProduce(final PlayerState state, final String buildingTag,
                                       final UnitType unitType) {
        if (hatcheryLarvaCount.getOrDefault(buildingTag, 0) > 0) return ProductionResult.PROCEED;
        return ProductionResult.BLOCKED;
    }

    @Override
    public void onProductionCommitted(final PlayerState state, final String buildingTag,
                                      final UnitType unitType, final Supplier<String> tagSupplier) {
        hatcheryLarvaCount.merge(buildingTag, -1, Integer::sum);

        final Building hatchery = state.buildings().stream()
            .filter(b -> b.tag().equals(buildingTag))
            .findFirst().orElse(null);
        final Point2d eggPos = (hatchery != null) ? hatchery.position() : new Point2d(9, 9);
        final String eggTag = tagSupplier.get();
        final int eggHp = SC2Data.maxHealth(UnitType.EGG);
        state.addUnit(new Unit(eggTag, UnitType.EGG, eggPos, eggHp, eggHp, 0, 0, 0, 0));
        eggTagByBuilding.computeIfAbsent(buildingTag, k -> new ArrayDeque<>()).add(eggTag);
    }

    @Override
    public void onUnitSpawned(final PlayerState state, final UnitType type,
                              final String unitTag, final String buildingTag) {
        final Deque<String> eggs = eggTagByBuilding.get(buildingTag);
        if (eggs != null) {
            final String eggTag = eggs.poll();
            if (eggTag != null) state.removeUnit(eggTag);
            if (eggs.isEmpty()) eggTagByBuilding.remove(buildingTag);
        }
        if (type == UnitType.OVERLORD) state.addSupply(8);
        if (type == UnitType.QUEEN)    queenEnergyMap.put(unitTag, INJECT_COST_ENERGY);
    }

    @Override
    public int trainCount(final UnitType type) { return type == UnitType.ZERGLING ? 2 : 1; }

    private static final Set<BuildingType> TOWN_HALLS =
        Set.of(BuildingType.HATCHERY, BuildingType.LAIR, BuildingType.HIVE);

    @Override public UnitType workerType()          { return UnitType.DRONE; }
    @Override public Set<BuildingType> townHallTypes() { return TOWN_HALLS; }

    int larvaCount(final String hatcheryTag)  { return hatcheryLarvaCount.getOrDefault(hatcheryTag, 0); }
    double queenEnergy(final String queenTag) { return queenEnergyMap.getOrDefault(queenTag, 0.0); }
    void setQueenEnergyForTesting(final String queenTag, final double energy) { queenEnergyMap.put(queenTag, energy); }
    boolean eggPendingForBuilding(final String buildingTag) {
        final Deque<String> q = eggTagByBuilding.get(buildingTag);
        return q != null && !q.isEmpty();
    }
}
```

- [ ] **Step 4.4: Run all race model tests**

```
mvn test -Dtest="TerranEmulatedGameTest+ZergEmulatedGameTest+EmulatedGameTest" -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4.5: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/ProtossRaceModel.java src/main/java/io/quarkmind/sc2/emulated/TerranRaceModel.java src/main/java/io/quarkmind/sc2/emulated/ZergRaceModel.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): migrate race model implementations to PlayerState public API"
```

---

## Task 5: Migrate `EmulatedGame` — add `PhysicsState` fields, update all internal methods

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

This is the largest single change. Work through the file top to bottom, updating each section. No new tests — all existing EmulatedGame tests cover this.

- [ ] **Step 5.1: Add `PhysicsState` fields and update field visibility**

In the field declarations section, add `friendlyPhysics` and `enemyPhysics`, and make `enemyBehavior` package-private (remove `private`):

```java
// Per-player state
final PlayerState  friendly        = new PlayerState();
final PlayerState  enemy           = new PlayerState();
final PhysicsState friendlyPhysics = new PhysicsState();
final PhysicsState enemyPhysics    = new PhysicsState();

// Enemy AI
EnemyBehavior enemyBehavior;   // package-private for test access (was private)
```

- [ ] **Step 5.2: Update `reset()`**

```java
public void reset() {
    friendly.clear();
    friendlyPhysics.clear();
    enemy.clear();
    enemyPhysics.clear();
    nextTag     = 200;
    gameFrame   = 0;
    miningProbesOverridden = false;
    movementStrategy.reset();
    visibility.reset();
    geysers.clear();

    playerRaceModel.seedInitialState(friendly, geysers);
    miningProbesPerBase = countWorkersPerBase(playerRaceModel, friendly.buildings(), friendly.units());

    enemy.setSupply(200);
    enemy.setVespene(9999);

    if (enemyBehavior != null) enemyBehavior.reset(enemyBehavior.currentStrategy());
}
```

- [ ] **Step 5.3: Update `tick()`**

```java
public void tick() {
    if (!miningProbesOverridden) {
        miningProbesPerBase = countWorkersPerBase(playerRaceModel, friendly.buildings(), friendly.units());
    }
    miningProbesOverridden = false;
    gameFrame++;
    for (final int workersAtBase : miningProbesPerBase) {
        friendly.addMinerals(SC2Data.mineralIncomePerTick(workersAtBase));
    }
    playerRaceModel.tickPassive(friendly, gameFrame * (long) SC2Data.LOOPS_PER_TICK);
    moveFriendlyUnits();
    visibility.recompute(friendly.units(), friendly.buildings(), terrainGrid);
    moveEnemyUnits();
    resolveCombat();
    tickEnemyRetreatTransfer();
    friendlyPhysics.fireCompletions(gameFrame);
    enemyPhysics.fireCompletions(gameFrame);
    drainBuildingQueues(friendly, friendlyPhysics);
    drainBuildingQueues(enemy, enemyPhysics);
    spawnEnemyWaves();
    if (enemyBehavior != null) {
        GameState enemyPov = snapshotForEnemy();
        enemyBehavior.tick(enemyPov, enemyIntentQueue);
        enemyIntentQueue.drainAll().forEach(intent -> applyIntent(intent, enemy, enemyPhysics));
    }
}
```

- [ ] **Step 5.4: Update `enforceWall()` — use `buildings()` readers**

Replace `friendly.buildings` and `enemy.buildings` iteration with `friendly.buildings()` and `enemy.buildings()`. The wall collision loops become:

```java
for (var bldg : friendly.buildings()) {
    ...
}
for (var bldg : enemy.buildings()) {
    ...
}
```

- [ ] **Step 5.5: Update `moveFriendlyUnits()` and `moveEnemyUnits()`**

```java
private void moveFriendlyUnits() {
    friendly.replaceAllUnits(u -> {
        Point2d target = friendlyPhysics.unitTargets.get(u.tag());
        if (target == null) return u;
        Point2d newPos = enforceWall(u.tag(),
            movementStrategy.advance(u.tag(), u.position(), target, unitSpeed),
            u.position());
        if (distance(newPos, target) < 0.2) friendlyPhysics.unitTargets.remove(u.tag());
        return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                        u.shields(), u.maxShields(), 0, 0);
    });
}

private void moveEnemyUnits() {
    Set<String> retreating = enemyBehavior != null ? enemyBehavior.retreatingUnits() : Set.of();
    enemy.replaceAllUnits(u -> {
        Point2d target = enemyPhysics.unitTargets.getOrDefault(u.tag(), EnemyBehavior.NEXUS_POS);
        Point2d newPos = enforceWall(u.tag(),
            movementStrategy.advance(u.tag(), u.position(), target, unitSpeed),
            u.position());
        float prevY = u.position().y();
        float nextY = newPos.y();
        if ((prevY > 18f && nextY <= 18f) || (prevY >= 18f && nextY < 18f)) {
            log.infof("[WALL-CROSS] %s crossed y=18: from (%.2f,%.2f) to (%.2f,%.2f) — x=%.2f %s",
                u.tag(), u.position().x(), prevY, newPos.x(), nextY, newPos.x(),
                (newPos.x() >= 11f && newPos.x() <= 14f) ? "IN GAP ✓" : "THROUGH WALL ✗");
        }
        return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                        u.shields(), u.maxShields(), 0, 0);
    });
}
```

- [ ] **Step 5.6: Update `tickEnemyRetreatTransfer()`**

```java
private void tickEnemyRetreatTransfer() {
    if (enemyBehavior == null) return;
    Set<String> retreating = enemyBehavior.retreatingUnits();
    enemy.removeUnitsWhere(u -> {
        if (!retreating.contains(u.tag())) return false;
        if (distance(u.position(), EnemyBehavior.STAGING_POS) >= 0.1) return false;
        enemyBehavior.clearRetreating(u.tag());
        enemyPhysics.unitTargets.remove(u.tag());
        movementStrategy.clearUnit(u.tag());
        enemyBehavior.stagingArea.add(u);
        log.debugf("[EMULATED] Unit %s arrived at staging (hp=%d shields=%d)",
            u.tag(), u.health(), u.shields());
        return true;
    });
}
```

Note: `removeUnitsWhere` here returns a list but we don't use it — the physics cleanup is done inside the predicate (which is safe since predicate runs during the remove iteration, in the same thread, with full access to `enemyPhysics`).

- [ ] **Step 5.7: Update `spawnEnemyWaves()`**

```java
private void spawnEnemyWaves() {
    pendingWaves.removeIf(wave -> {
        if (wave.spawnFrame() > gameFrame) return false;
        for (int i = 0; i < wave.unitTypes().size(); i++) {
            UnitType type = wave.unitTypes().get(i);
            Point2d pos = new Point2d(wave.spawnPosition().x() + i * 0.5f,
                                      wave.spawnPosition().y());
            String tag = "enemy-" + nextTag++;
            int hp = SC2Data.maxHealth(type);
            enemy.addUnit(new Unit(tag, type, pos, hp, hp,
                SC2Data.maxShields(type), SC2Data.maxShields(type), 0, 0));
            enemyPhysics.unitTargets.put(tag, wave.targetPosition());
        }
        log.infof("[EMULATED] Enemy wave spawned: %dx%s at frame %d",
            wave.unitTypes().size(), wave.unitTypes().get(0), gameFrame);
        return true;
    });
}
```

- [ ] **Step 5.8: Update `applyIntent()` (public and internal overloads) and `applyIntent(TimedIntent)`**

```java
public void applyIntent(Intent intent) {
    applyIntent(intent, friendly, friendlyPhysics);
}

public void applyIntent(TimedIntent ti) {
    Runnable action = switch (ti.intent()) {
        case TrainIntent  t -> () -> handleTrain(t, friendly, friendlyPhysics, ti.loop());
        case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), friendly, friendlyPhysics);
        case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), friendly, friendlyPhysics);
        case BuildIntent  b -> () -> handleBuild(b, friendly, friendlyPhysics, ti.loop());
        case BlinkIntent  b -> () -> executeBlink(b.unitTag(), friendly, friendlyPhysics);
    };
    action.run();
}

void applyIntent(Intent intent, PlayerState state, PhysicsState physics) {
    Runnable action = switch (intent) {
        case MoveIntent   m -> () -> setTarget(m.unitTag(), m.targetLocation(), state, physics);
        case AttackIntent a -> () -> setTarget(a.unitTag(), a.targetLocation(), state, physics);
        case TrainIntent  t -> () -> handleTrain(t, state, physics);
        case BuildIntent  b -> () -> handleBuild(b, state, physics, gameFrame * SC2Data.LOOPS_PER_TICK);
        case BlinkIntent  b -> () -> executeBlink(b.unitTag(), state, physics);
    };
    action.run();
}
```

- [ ] **Step 5.9: Update `setTarget()`**

```java
private void setTarget(String tag, Point2d target, PlayerState state, PhysicsState physics) {
    if (state.units().stream().anyMatch(u -> u.tag().equals(tag))) {
        physics.unitTargets.put(tag, target);
        log.debugf("[EMULATED] %s → (%.1f,%.1f)", tag, target.x(), target.y());
    }
}
```

- [ ] **Step 5.10: Update `handleTrain()` overloads**

```java
private void handleTrain(TrainIntent t, PlayerState state, PhysicsState physics) {
    handleTrain(t, state, physics, 0L);
}

private void handleTrain(final TrainIntent t, final PlayerState state,
                         final PhysicsState physics, final long absLoop) {
    final String buildingTag = t.buildingTag();
    final Building building = state.buildings().stream()
        .filter(b -> b.tag().equals(buildingTag) && b.isComplete())
        .findFirst().orElse(null);
    if (building == null) {
        log.debugf("[EMULATED] Train rejected — building %s not ready", buildingTag);
        return;
    }
    final BuildingType required = SC2Data.trainedBy(t.unitType());
    if (required != BuildingType.UNKNOWN && building.type() != required) {
        log.debugf("[EMULATED] Train rejected — %s cannot train %s (needs %s)",
            building.type(), t.unitType(), required);
        return;
    }

    final RaceModel model = (state == friendly) ? playerRaceModel : null;
    if (model != null) {
        final ProductionResult pr = model.canProduce(state, buildingTag, t.unitType());
        if (pr == ProductionResult.BLOCKED) {
            log.debugf("[EMULATED] Train rejected — production resource unavailable for %s", t.unitType());
            return;
        }
        if (pr == ProductionResult.HANDLED) {
            log.debugf("[EMULATED] Train handled by race model (MULE) for %s", t.unitType());
            return;
        }
    }

    final int mCost = SC2Data.mineralCost(t.unitType());
    final int gCost = SC2Data.gasCost(t.unitType());
    final int sCost = SC2Data.supplyCost(t.unitType());
    if ((int) state.minerals() < mCost || state.vespene() < gCost
            || state.supplyUsed() + sCost > state.supply()) {
        log.debugf("[EMULATED] Cannot train %s — insufficient resources", t.unitType());
        return;
    }

    final boolean isBusy = physics.buildingTrainingUntil.containsKey(buildingTag);
    final Deque<UnitType> existingQueue = physics.buildingQueues.get(buildingTag);
    final int total = (isBusy ? 1 : 0) + (existingQueue != null ? existingQueue.size() : 0);
    if (total >= 5) {
        log.debugf("[EMULATED] Train rejected — building %s queue full", buildingTag);
        return;
    }

    state.addSupplyUsed(sCost);
    state.deductMinerals(mCost);
    state.deductVespene(gCost);

    if (model != null) {
        model.onProductionCommitted(state, buildingTag, t.unitType(), this::nextTagString);
    }

    if (!isBusy) {
        startTraining(buildingTag, t.unitType(), state, physics, absLoop);
    } else {
        physics.buildingQueues.computeIfAbsent(buildingTag, k -> new ArrayDeque<>())
            .add(t.unitType());
    }
}
```

- [ ] **Step 5.11: Update `startTraining()`**

```java
private void startTraining(final String buildingTag, final UnitType unitType,
                            final PlayerState state, final PhysicsState physics,
                            final long absLoop) {
    final boolean isEnemy  = (state == enemy);
    final RaceModel model  = (state == friendly) ? playerRaceModel : null;
    final int  loopOffset  = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
    final long completesAt = gameFrame
        + (loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK;
    physics.buildingTrainingUntil.put(buildingTag, completesAt);
    physics.buildingCompletionAtLoop.put(buildingTag,
        absLoop + SC2Data.trainTimeInLoops(unitType));
    final int spawnCount = (model != null) ? model.trainCount(unitType) : 1;
    physics.pendingCompletions.add(new PhysicsState.PendingCompletion(completesAt, () -> {
        physics.buildingTrainingUntil.remove(buildingTag);
        if (!physics.buildingQueues.containsKey(buildingTag)) {
            physics.buildingCompletionAtLoop.remove(buildingTag);
        }
        for (int i = 0; i < spawnCount; i++) {
            final String tag = nextTagString();
            final int hp = SC2Data.maxHealth(unitType);
            state.addUnit(new Unit(tag, unitType,
                new Point2d(9 + i * 0.5f, 9), hp, hp,
                SC2Data.maxShields(unitType), SC2Data.maxShields(unitType), 0, 0));
            log.debugf("[EMULATED] Trained %s (tag=%s)", unitType, tag);
            if (model != null) model.onUnitSpawned(state, unitType, tag, buildingTag);
            if (isEnemy && enemyBehavior != null) enemyBehavior.notifyUnitTrained();
        }
    }));
}
```

- [ ] **Step 5.12: Update `drainBuildingQueues()`**

```java
private void drainBuildingQueues(PlayerState state, PhysicsState physics) {
    for (String buildingTag : new ArrayList<>(physics.buildingQueues.keySet())) {
        if (physics.buildingTrainingUntil.containsKey(buildingTag)) continue;
        Deque<UnitType> queue = physics.buildingQueues.get(buildingTag);
        if (queue == null || queue.isEmpty()) {
            physics.buildingQueues.remove(buildingTag);
            physics.buildingCompletionAtLoop.remove(buildingTag);
            continue;
        }
        UnitType next = queue.poll();
        if (queue.isEmpty()) physics.buildingQueues.remove(buildingTag);
        long completionLoop = physics.buildingCompletionAtLoop.getOrDefault(buildingTag, 0L);
        physics.buildingCompletionAtLoop.remove(buildingTag);
        startTraining(buildingTag, next, state, physics, completionLoop);
    }
}
```

- [ ] **Step 5.13: Update `handleBuild()`**

```java
private void handleBuild(BuildIntent b, PlayerState state, PhysicsState physics, long absLoop) {
    int mCost = SC2Data.mineralCost(b.buildingType());
    if ((int) state.minerals() < mCost) {
        log.debugf("[EMULATED] Cannot build %s — insufficient minerals", b.buildingType());
        return;
    }
    state.deductMinerals(mCost);
    final String tag = "bldg-" + nextTag++;
    final BuildingType bt = b.buildingType();
    state.addBuilding(new Building(tag, bt, b.location(),
        SC2Data.maxBuildingHealth(bt), SC2Data.maxBuildingHealth(bt), false));
    final int loopOffset = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
    final long completesAt = gameFrame
        + (loopOffset + SC2Data.buildTimeInLoops(bt)) / SC2Data.LOOPS_PER_TICK;
    physics.pendingCompletions.add(new PhysicsState.PendingCompletion(completesAt, () -> {
        markBuildingComplete(tag, state);
        state.addSupply(SC2Data.supplyBonus(bt));
        log.debugf("[EMULATED] Completed %s (tag=%s)", bt, tag);
    }));
}
```

- [ ] **Step 5.14: Update `markBuildingComplete()`**

```java
private void markBuildingComplete(String tag, PlayerState state) {
    state.replaceAllBuildings(b -> b.tag().equals(tag)
        ? new Building(b.tag(), b.type(), b.position(), b.health(), b.maxHealth(), true)
        : b);
}
```

- [ ] **Step 5.15: Update `resolveCombat()`**

```java
private void resolveCombat() {
    friendlyPhysics.unitCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
    enemyPhysics.unitCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
    friendlyPhysics.blinkCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
    enemyPhysics.blinkCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));

    Map<String, Integer> pending       = new HashMap<>();
    Set<String>          firedFriendly = new HashSet<>();
    Set<String>          firedEnemy    = new HashSet<>();

    for (Unit attacker : friendly.units()) {
        if (friendlyPhysics.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
        nearestInRange(attacker.position(), enemy.units(), SC2Data.attackRange(attacker.type()))
            .ifPresent(target -> {
                if (!missesHighGround(attacker.position(), target.position(), attacker.type())) {
                    pending.merge(target.tag(),
                        damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
                }
                firedFriendly.add(attacker.tag());
            });
    }
    for (Unit attacker : enemy.units()) {
        if (enemyPhysics.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
        nearestInRange(attacker.position(), friendly.units(), SC2Data.attackRange(attacker.type()))
            .ifPresent(target -> {
                if (!missesHighGround(attacker.position(), target.position(), attacker.type())) {
                    pending.merge(target.tag(),
                        damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
                }
                firedEnemy.add(attacker.tag());
            });
    }

    friendly.replaceAllUnits(u -> applyDamage(u, pending.getOrDefault(u.tag(), 0)));
    List<Unit> deadFriendly = friendly.removeUnitsWhere(u -> u.health() <= 0);
    deadFriendly.forEach(u -> {
        friendlyPhysics.unitTargets.remove(u.tag());
        friendlyPhysics.unitCooldowns.remove(u.tag());
        friendlyPhysics.blinkCooldowns.remove(u.tag());
        movementStrategy.clearUnit(u.tag());
    });

    enemy.replaceAllUnits(u -> applyDamage(u, pending.getOrDefault(u.tag(), 0)));
    List<Unit> deadEnemy = enemy.removeUnitsWhere(u -> u.health() <= 0);
    deadEnemy.forEach(u -> {
        enemyPhysics.unitTargets.remove(u.tag());
        enemyPhysics.unitCooldowns.remove(u.tag());
        enemyPhysics.blinkCooldowns.remove(u.tag());
        movementStrategy.clearUnit(u.tag());
        if (enemyBehavior != null) enemyBehavior.clearRetreating(u.tag());
    });

    for (Unit u : friendly.units()) {
        if (firedFriendly.contains(u.tag()))
            friendlyPhysics.unitCooldowns.put(u.tag(), SC2Data.attackCooldownInTicks(u.type()));
    }
    for (Unit u : enemy.units()) {
        if (firedEnemy.contains(u.tag()))
            enemyPhysics.unitCooldowns.put(u.tag(), SC2Data.attackCooldownInTicks(u.type()));
    }
}
```

- [ ] **Step 5.16: Update `blinkRetreatTarget()` and `executeBlink()`**

```java
private Point2d blinkRetreatTarget(Unit unit, PlayerState opponents) {
    Unit nearest = opponents.units().stream()
        .min(Comparator.comparingDouble(e -> distance(unit.position(), e.position())))
        .orElse(null);
    if (nearest == null) return unit.position();
    double dx = unit.position().x() - nearest.position().x();
    double dy = unit.position().y() - nearest.position().y();
    double len = Math.sqrt(dx * dx + dy * dy);
    if (len < 0.001) return unit.position();
    float step = SC2Data.blinkRange(unit.type());
    double baseAngle = Math.atan2(dy, dx);
    for (int i = 0; i <= 4; i++) {
        for (int sign : new int[]{1, -1}) {
            double angle = baseAngle + sign * i * Math.PI / 4;
            Point2d candidate = new Point2d(
                (float)(unit.position().x() + Math.cos(angle) * step),
                (float)(unit.position().y() + Math.sin(angle) * step));
            if (terrainGrid == null || terrainGrid.isWalkable((int) candidate.x(), (int) candidate.y()))
                return candidate;
            if (i == 0) break;
        }
    }
    return unit.position();
}

private void executeBlink(String tag, PlayerState state, PhysicsState physics) {
    PlayerState opponents = (state == friendly) ? enemy : friendly;
    state.replaceAllUnits(u -> {
        if (!u.tag().equals(tag)) return u;
        Point2d dest = blinkRetreatTarget(u, opponents);
        int restored = Math.min(u.shields() + SC2Data.blinkShieldRestore(u.type()), u.maxShields());
        physics.unitTargets.put(tag, dest);
        physics.blinkCooldowns.put(tag, SC2Data.blinkCooldownInTicks(u.type()));
        return new Unit(u.tag(), u.type(), dest,
                        u.health(), u.maxHealth(), restored, u.maxShields(), 0, 0);
    });
}
```

- [ ] **Step 5.17: Update `snapshot()` — three migrations**

```java
public GameState snapshot() {
    List<Unit> friendlyWithCooldown = friendly.units().stream()
        .map(u -> new Unit(u.tag(), u.type(), u.position(),
                           u.health(), u.maxHealth(), u.shields(), u.maxShields(),
                           friendlyPhysics.unitCooldowns.getOrDefault(u.tag(), 0),
                           friendlyPhysics.blinkCooldowns.getOrDefault(u.tag(), 0)))
        .toList();

    List<Unit> stagingArea = enemyBehavior != null
        ? enemyBehavior.stagingArea
        : List.of();

    if (terrainGrid != null) {
        List<Unit> visibleEnemies = enemy.units().stream()
            .filter(u -> visibility.isVisible(u.position()))
            .toList();
        List<Unit> visibleStaging = stagingArea.stream()
            .filter(u -> visibility.isVisible(u.position()))
            .toList();
        return new GameState(
            (int) friendly.minerals(),
            friendly.vespene(), friendly.supply(), friendly.supplyUsed(),
            friendlyWithCooldown, List.copyOf(friendly.buildings()),
            visibleEnemies,
            List.copyOf(enemy.buildings()),
            visibleStaging,
            List.copyOf(geysers),
            List.of(),
            gameFrame);
    }
    return new GameState(
        (int) friendly.minerals(),
        friendly.vespene(), friendly.supply(), friendly.supplyUsed(),
        friendlyWithCooldown, List.copyOf(friendly.buildings()),
        List.copyOf(enemy.units()),
        List.copyOf(enemy.buildings()),
        List.copyOf(stagingArea),
        List.copyOf(geysers),
        List.of(),
        gameFrame);
}
```

- [ ] **Step 5.18: Update harness methods**

```java
// Replace the existing implementations:
void setMineralsForTesting(int amount)    { friendly.setMinerals(amount); }
int  enemyMinerals()                      { return (int) enemy.minerals(); }
int  enemyStagingSize()                   { return enemyBehavior != null ? enemyBehavior.stagingArea.size() : 0; }

void addStagedUnitForTesting(UnitType type, Point2d position) {
    // Requires enemyBehavior != null — call setEnemyStrategy() first in tests
    String tag = "test-staging-" + nextTag++;
    int hp = SC2Data.maxHealth(type);
    enemyBehavior.stagingArea.add(new Unit(tag, type, position, hp, hp,
        SC2Data.maxShields(type), SC2Data.maxShields(type), 0, 0));
}

void spawnFriendlyUnitForTesting(UnitType type, Point2d position) {
    int hp  = SC2Data.maxHealth(type);
    int sh  = SC2Data.maxShields(type);
    String tag = "test-friendly-" + nextTag++;
    friendly.addUnit(new Unit(tag, type, position, hp, hp, sh, sh, 0, 0));
}

public void setSupplyCapForHarness(int supply)   { friendly.setSupply(supply); }
public void setVespeneForHarness(int vespene)    { friendly.setVespene(vespene); }

public void injectReplayBuilding(Building building) {
    friendly.addBuilding(building);
}

public void injectReplayBuildingWithCost(Building building) {
    int mCost = SC2Data.mineralCost(building.type());
    friendly.deductMinerals(mCost);
    friendly.addBuilding(building);
}

public void markReplayBuildingComplete(String tag) {
    friendly.replaceAllBuildings(b -> b.tag().equals(tag)
        ? new Building(b.tag(), b.type(), b.position(), b.health(), b.maxHealth(), true)
        : b);
}
```

- [ ] **Step 5.19: Run full test suite**

```
mvn test -q
```

Expected: `BUILD SUCCESS`. If there are compile errors, use `mvn compile -q 2>&1 | head -40` to find remaining references to old field access patterns and fix them.

- [ ] **Step 5.20: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): migrate EmulatedGame to PhysicsState fields and PlayerState public API"
```

---

## Task 6: Finalize `PlayerState` — make fields private, remove physics fields; promote visibilities

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/PlayerState.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/RaceModel.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/RaceModelFactory.java`

- [ ] **Step 6.1: Rewrite `PlayerState` with private fields and no physics fields**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;

import java.util.*;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;

/**
 * Per-player mutable game state: units, buildings, and resources. Public so that
 * external {@link RaceModel} implementations (loaded from outside this package) can
 * read and mutate the state they need.
 *
 * <p>Public methods are semantic operations meaningful to a race plugin.
 * Package-private methods ({@code replaceAllUnits}, {@code removeUnitsWhere},
 * {@code replaceAllBuildings}, {@code clear}) are bulk structural operations used
 * only by EmulatedGame's physics engine.
 *
 * <p>Not thread-safe — all access is from the single game-tick scheduler thread.
 */
public class PlayerState {

    private final List<Unit>     units     = new ArrayList<>();
    private final List<Building> buildings = new ArrayList<>();
    private double minerals;
    private int    vespene;
    private int    supply;
    private int    supplyUsed;

    // --- Public typed API ---

    public void setMinerals(double m)        { this.minerals = m; }
    public void addMinerals(double amount)   { this.minerals += amount; }
    public void deductMinerals(double cost)  { this.minerals -= cost; }
    public double minerals()                 { return minerals; }

    public void setVespene(int v)            { this.vespene = v; }
    public void deductVespene(int cost)      { this.vespene -= cost; }
    public int  vespene()                    { return vespene; }

    public void setSupply(int s)             { this.supply = s; }
    public void addSupply(int amount)        { this.supply += amount; }
    public int  supply()                     { return supply; }

    public void setSupplyUsed(int s)         { this.supplyUsed = s; }
    public void addSupplyUsed(int cost)      { this.supplyUsed += cost; }
    public int  supplyUsed()                 { return supplyUsed; }

    public void addUnit(Unit unit)           { units.add(unit); }
    public void removeUnit(String tag)       { units.removeIf(u -> u.tag().equals(tag)); }
    public List<Unit> units()               { return Collections.unmodifiableList(units); }

    public void addBuilding(Building b)      { buildings.add(b); }
    public List<Building> buildings()       { return Collections.unmodifiableList(buildings); }

    // --- Package-private bulk ops for EmulatedGame physics ---

    List<Unit> removeUnitsWhere(Predicate<Unit> pred) {
        List<Unit> removed = new ArrayList<>();
        units.removeIf(u -> {
            if (pred.test(u)) { removed.add(u); return true; }
            return false;
        });
        return List.copyOf(removed);
    }

    void replaceAllUnits(UnaryOperator<Unit> op)         { units.replaceAll(op); }
    void replaceAllBuildings(UnaryOperator<Building> op) { buildings.replaceAll(op); }

    void clear() {
        units.clear();
        buildings.clear();
        minerals   = 0;
        vespene    = 0;
        supply     = 0;
        supplyUsed = 0;
    }
}
```

- [ ] **Step 6.2: Compile check**

```
mvn compile -q 2>&1 | head -30
```

Expected: `BUILD SUCCESS`. If there are any remaining references to old fields (`s.minerals`, `state.stagingArea`, `state.pendingCompletions`, etc.), find and fix them using the IDE's "find usages" on each deleted field.

- [ ] **Step 6.3: Promote visibility of `RaceModel`, `ProductionResult`, `RaceModelFactory`**

In `RaceModel.java`: change `interface RaceModel` to `public interface RaceModel`.

In `ProductionResult.java`: change `enum ProductionResult` to `public enum ProductionResult`.

In `RaceModelFactory.java`: change `class RaceModelFactory` to `public class RaceModelFactory`.

- [ ] **Step 6.4: Run full test suite**

```
mvn test -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6.5: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/sc2/emulated/PlayerState.java src/main/java/io/quarkmind/sc2/emulated/RaceModel.java src/main/java/io/quarkmind/sc2/emulated/ProductionResult.java src/main/java/io/quarkmind/sc2/emulated/RaceModelFactory.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#164): finalize PlayerState (private fields, no physics), promote RaceModel/ProductionResult/RaceModelFactory to public"
```

---

## Task 7: Update test files

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/PlayerStateTest.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 7.1: Rewrite `PlayerStateTest` — game-state tests only, using the new API**

All physics tests (`pendingCompletions`, `unitTargets`) were already covered by `PhysicsStateTest` in Task 1. The `stagingArea` field is gone from PlayerState. Remove those tests; update the remaining tests to use the new API.

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class PlayerStateTest {

    @Test
    void initialState_hasNoUnitsOrBuildings() {
        PlayerState s = new PlayerState();
        assertThat(s.units()).isEmpty();
        assertThat(s.buildings()).isEmpty();
    }

    @Test
    void minerals_defaultsToZero() {
        assertThat(new PlayerState().minerals()).isZero();
    }

    @Test
    void clear_resetsGameState() {
        PlayerState s = new PlayerState();
        s.setMinerals(500);
        s.setVespene(100);
        s.setSupply(22);
        s.setSupplyUsed(10);
        s.addUnit(new Unit("u1", UnitType.ZEALOT, new Point2d(1,1), 100,100,50,50,0,0));
        s.addBuilding(new Building("b1", BuildingType.GATEWAY, new Point2d(2,2), 500,500,true));
        s.clear();
        assertThat(s.minerals()).isZero();
        assertThat(s.vespene()).isZero();
        assertThat(s.supply()).isZero();
        assertThat(s.supplyUsed()).isZero();
        assertThat(s.units()).isEmpty();
        assertThat(s.buildings()).isEmpty();
    }
}
```

- [ ] **Step 7.2: Run `PlayerStateTest`**

```
mvn test -Dtest=PlayerStateTest -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7.3: Update `EmulatedGameTest` — field access patterns**

Search `EmulatedGameTest.java` for the following patterns and apply each substitution.

**Pattern A — `unitTargets.put` (6 sites: lines 404, 988, 1217, 1218, 1247, 1477):**
```java
// Before
game.enemy.unitTargets.put(zealotTag, new Point2d(3f, 9f));
// After
game.enemyPhysics.unitTargets.put(zealotTag, new Point2d(3f, 9f));
```
Apply to all 6 occurrences.

**Pattern B — `enemy.stagingArea.stream()` (2 sites: approx lines 1525, 1679):**
```java
// Before
game.enemy.stagingArea.stream().anyMatch(u -> u.type() == UnitType.ZEALOT)
// After
game.enemyBehavior.stagingArea.stream().anyMatch(u -> u.type() == UnitType.ZEALOT)
```

**Pattern C — `enemy.minerals` assignment (1 site: approx lines 1520, 1674):**
```java
// Before
game.enemy.minerals = 200;
// After
game.enemy.setMinerals(200);
```

**Pattern D — `enemy.minerals` read (approx line 1429, 1431):**
```java
// Before
double before = game.enemy.minerals;
assertThat(game.enemy.minerals).isGreaterThan(before);
// After
double before = game.enemy.minerals();
assertThat(game.enemy.minerals()).isGreaterThan(before);
```

**Pattern E — `enemy.buildings.add(...)` (approx lines 1447, 1497):**
```java
// Before
game.enemy.buildings.add(new Building(...));
// After
game.enemy.addBuilding(new Building(...));
```

**Pattern F — `enemy.units.add(...)` (approx line 1475):**
```java
// Before
game.enemy.units.add(new Unit(...));
// After
game.enemy.addUnit(new Unit(...));
```

**Pattern G — fog-of-war staging test (the test that calls `addStagedUnitForTesting` without `setEnemyStrategy`):**

Find the test `stagingAreaEnemiesAreFilteredByVisibility` (or similar name). Add `game.setEnemyStrategy(game.defaultStrategy())` before the `game.reset()` call (or use `game.setEnemyBehavior(new EnemyBehavior(...))` — check how other tests set up enemy behavior). The simplest approach is to use a `FixedBuildOrderStrategy` with `List.of()` as the build order:

```java
// Add before game.reset() in the fog-of-war staging test:
game.setEnemyStrategy(new FixedBuildOrderStrategy("FOG_TEST", Race.PROTOSS,
    List.of(), 0, new EnemyAttackConfig(99, 200, 0, 0)));
game.reset();
game.addStagedUnitForTesting(UnitType.ZEALOT, new Point2d(26, 26));
```

Check the test file for the exact method name that sets the enemy strategy — it may be `setEnemyStrategy`, `configureEnemy`, or similar.

- [ ] **Step 7.4: Run `EmulatedGameTest`**

```
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7.5: Run full suite — final verification**

```
mvn test -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7.6: Commit**

```
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/sc2/emulated/PlayerStateTest.java src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(#164): update PlayerStateTest and EmulatedGameTest for split PlayerState/PhysicsState"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] `PhysicsState` created with all 7 fields + `fireCompletions` + `clear` + `PendingCompletion` — Task 1
- [x] `EnemyBehavior.stagingArea` + `reset()` — Task 2
- [x] `PlayerState` public API (all methods in spec) — Task 3
- [x] Race model migration (all 3 implementations) — Task 4
- [x] `EmulatedGame` refactor (all internal methods updated) — Task 5
- [x] `resolveCombat()` two-step cleanup with `removeUnitsWhere` returning `List<Unit>` — Task 5 step 5.15
- [x] `tickEnemyRetreatTransfer()` using `enemyPhysics.unitTargets` and `enemyBehavior.stagingArea` — Task 5 step 5.6
- [x] `snapshot()` three migrations (unitCooldowns, blinkCooldowns, stagingArea null-guard) — Task 5 step 5.17
- [x] Harness methods (9 methods) — Task 5 step 5.18
- [x] `addStagedUnitForTesting` pre-condition (requires `enemyBehavior != null`) — Task 5 step 5.18
- [x] `RaceModel`, `ProductionResult`, `RaceModelFactory` promoted to public — Task 6
- [x] `PlayerState` fields made private — Task 6
- [x] `PlayerStateTest` game-state tests updated; physics tests removed — Task 7
- [x] `EmulatedGameTest` all 6 `unitTargets` sites, 2 `stagingArea` sites, mineral sites — Task 7
- [x] Fog-of-war staging test gets `setEnemyStrategy` before `addStagedUnitForTesting` — Task 7
- [x] `PhysicsStateTest` created — Task 1

**Type consistency:** `PhysicsState.PendingCompletion` used in Tasks 5.11 and 5.13; `removeUnitsWhere` returns `List<Unit>` in Tasks 3 and 5.15 — consistent throughout.

**Placeholder scan:** No TBD, no "similar to Task N", no vague "add validation" — all code is complete.
