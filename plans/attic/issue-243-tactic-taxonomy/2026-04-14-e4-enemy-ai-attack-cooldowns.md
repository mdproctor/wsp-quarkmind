# E4: Enemy Active AI + Attack Cooldowns — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the enemy an economy (mine minerals, follow a configurable build order, stage units, attack on threshold or timer), and replace flat per-tick damage with per-unit attack cooldowns.

**Architecture:** `EnemyStrategy`/`EnemyBuildStep`/`EnemyAttackConfig` are pure domain records (no framework deps) with Jackson annotations for JSON/YAML serde. `EmulatedConfig` holds the active strategy (defaulted, file-loadable, REST-overridable). `EmulatedGame` executes it each tick via `tickEnemyStrategy()`. Attack cooldowns live in two `Map<String, Integer>` fields in `EmulatedGame`; `SC2Data.damagePerTick` is deleted and replaced with `damagePerAttack` + `attackCooldownInTicks`.

**Tech Stack:** Java 21 records, Jackson (already on classpath via Quarkus REST), RestAssured + AssertJ (tests), Playwright (E2E).

**Test discipline:** Do NOT proceed to the next task until all tests for the current task pass. Every task ends with a green `mvn test` run covering its scope.

---

## File Map

| Action | File |
|---|---|
| Create | `src/main/java/io/quarkmind/domain/EnemyBuildStep.java` |
| Create | `src/main/java/io/quarkmind/domain/EnemyAttackConfig.java` |
| Create | `src/main/java/io/quarkmind/domain/EnemyStrategy.java` |
| Create | `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java` |
| Create | `src/test/java/io/quarkmind/domain/SC2DataTest.java` |
| Create | `src/main/resources/enemy-strategy.yaml` |
| Modify | `src/main/java/io/quarkmind/domain/SC2Data.java` |
| Modify | `src/main/java/io/quarkmind/domain/GameState.java` |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java` |
| Modify | `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java` |
| Modify | `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` |
| Modify | `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java` |
| Modify | `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java` |
| Modify | `src/main/java/io/quarkmind/qa/EmulatedConfig.java` |
| Modify | `src/main/java/io/quarkmind/qa/EmulatedConfigResource.java` |
| Modify | `src/main/resources/META-INF/resources/visualizer.js` |
| Modify | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` |
| Modify | `src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java` |
| Modify | `src/test/java/io/quarkmind/qa/GameStateWebSocketTest.java` |
| Modify | `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java` |

---

## Task 1: Domain Records — EnemyBuildStep, EnemyAttackConfig, EnemyStrategy

**Files:**
- Create: `src/main/java/io/quarkmind/domain/EnemyBuildStep.java`
- Create: `src/main/java/io/quarkmind/domain/EnemyAttackConfig.java`
- Create: `src/main/java/io/quarkmind/domain/EnemyStrategy.java`
- Create: `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`
- Create: `src/main/resources/enemy-strategy.yaml`

- [ ] **Step 1: Write the failing tests**

Create `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`:

```java
package io.quarkmind.domain;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class EnemyStrategyTest {

    @Test
    void buildStepHoldsUnitType() {
        EnemyBuildStep step = new EnemyBuildStep(UnitType.STALKER);
        assertThat(step.unitType()).isEqualTo(UnitType.STALKER);
    }

    @Test
    void attackConfigHoldsBothTriggers() {
        EnemyAttackConfig cfg = new EnemyAttackConfig(3, 200);
        assertThat(cfg.armyThreshold()).isEqualTo(3);
        assertThat(cfg.attackIntervalFrames()).isEqualTo(200);
    }

    @Test
    void defaultProtossHasThreeZealots() {
        EnemyStrategy s = EnemyStrategy.defaultProtoss();
        assertThat(s.buildOrder()).hasSize(3);
        assertThat(s.buildOrder()).allMatch(step -> step.unitType() == UnitType.ZEALOT);
    }

    @Test
    void defaultProtossConfigIsCorrect() {
        EnemyStrategy s = EnemyStrategy.defaultProtoss();
        assertThat(s.loop()).isTrue();
        assertThat(s.mineralsPerTick()).isEqualTo(2);
        assertThat(s.attackConfig().armyThreshold()).isEqualTo(3);
        assertThat(s.attackConfig().attackIntervalFrames()).isEqualTo(200);
    }

    @Test
    void strategyRoundTripsViaJson() throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        EnemyStrategy original = EnemyStrategy.defaultProtoss();
        String json = mapper.writeValueAsString(original);

        assertThat(json).contains("\"buildOrder\"");
        assertThat(json).contains("\"ZEALOT\"");
        assertThat(json).contains("\"loop\"");
        assertThat(json).contains("\"mineralsPerTick\"");
        assertThat(json).contains("\"attackConfig\"");

        EnemyStrategy restored = mapper.readValue(json, EnemyStrategy.class);
        assertThat(restored.buildOrder()).hasSize(3);
        assertThat(restored.loop()).isTrue();
        assertThat(restored.mineralsPerTick()).isEqualTo(2);
        assertThat(restored.attackConfig().armyThreshold()).isEqualTo(3);
    }

    @Test
    void buildStepRoundTripsViaJson() throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        EnemyBuildStep step = new EnemyBuildStep(UnitType.STALKER);
        String json = mapper.writeValueAsString(step);
        EnemyBuildStep restored = mapper.readValue(json, EnemyBuildStep.class);
        assertThat(restored.unitType()).isEqualTo(UnitType.STALKER);
    }

    @Test
    void customStrategyCanBeConstructedProgrammatically() {
        EnemyStrategy s = new EnemyStrategy(
            List.of(new EnemyBuildStep(UnitType.STALKER), new EnemyBuildStep(UnitType.IMMORTAL)),
            false, 5,
            new EnemyAttackConfig(2, 100));
        assertThat(s.buildOrder()).hasSize(2);
        assertThat(s.buildOrder().get(0).unitType()).isEqualTo(UnitType.STALKER);
        assertThat(s.loop()).isFalse();
    }
}
```

- [ ] **Step 2: Run to confirm compilation failure**

```bash
mvn test -Dtest=EnemyStrategyTest -q 2>&1 | head -20
```
Expected: `COMPILATION ERROR` — classes don't exist yet.

- [ ] **Step 3: Create EnemyBuildStep**

`src/main/java/io/quarkmind/domain/EnemyBuildStep.java`:
```java
package io.quarkmind.domain;

import com.fasterxml.jackson.annotation.JsonProperty;

public record EnemyBuildStep(
    @JsonProperty("unitType") UnitType unitType
) {}
```

- [ ] **Step 4: Create EnemyAttackConfig**

`src/main/java/io/quarkmind/domain/EnemyAttackConfig.java`:
```java
package io.quarkmind.domain;

import com.fasterxml.jackson.annotation.JsonProperty;

public record EnemyAttackConfig(
    @JsonProperty("armyThreshold")       int armyThreshold,
    @JsonProperty("attackIntervalFrames") int attackIntervalFrames
) {}
```

- [ ] **Step 5: Create EnemyStrategy**

`src/main/java/io/quarkmind/domain/EnemyStrategy.java`:
```java
package io.quarkmind.domain;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

public record EnemyStrategy(
    @JsonProperty("buildOrder")      List<EnemyBuildStep>  buildOrder,
    @JsonProperty("loop")            boolean               loop,
    @JsonProperty("mineralsPerTick") int                   mineralsPerTick,
    @JsonProperty("attackConfig")    EnemyAttackConfig     attackConfig
) {
    public static EnemyStrategy defaultProtoss() {
        return new EnemyStrategy(
            List.of(
                new EnemyBuildStep(UnitType.ZEALOT),
                new EnemyBuildStep(UnitType.ZEALOT),
                new EnemyBuildStep(UnitType.ZEALOT)
            ),
            true, 2,
            new EnemyAttackConfig(3, 200)
        );
    }
}
```

- [ ] **Step 6: Create sample enemy-strategy.yaml**

`src/main/resources/enemy-strategy.yaml`:
```yaml
# Sample enemy build order — copy to a path and set emulated.enemy.strategy-file=<path>
buildOrder:
  - unitType: ZEALOT
  - unitType: ZEALOT
  - unitType: STALKER
loop: true
mineralsPerTick: 2
attackConfig:
  armyThreshold: 3
  attackIntervalFrames: 200
```

- [ ] **Step 7: Run tests and confirm green**

```bash
mvn test -Dtest=EnemyStrategyTest -q
```
Expected: `BUILD SUCCESS`, 7 tests pass.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/domain/EnemyBuildStep.java \
        src/main/java/io/quarkmind/domain/EnemyAttackConfig.java \
        src/main/java/io/quarkmind/domain/EnemyStrategy.java \
        src/test/java/io/quarkmind/domain/EnemyStrategyTest.java \
        src/main/resources/enemy-strategy.yaml
git commit -m "feat(e4): add EnemyStrategy domain records with Jackson serde"
```

---

## Task 2: SC2Data — Add damagePerAttack and attackCooldownInTicks

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`
- Create: `src/test/java/io/quarkmind/domain/SC2DataTest.java`

> **Note:** Do NOT delete `damagePerTick` yet — that happens in Task 4 alongside the `resolveCombat()` rewrite. Deleting it now would break compilation.

- [ ] **Step 1: Write failing tests**

Create `src/test/java/io/quarkmind/domain/SC2DataTest.java`:

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SC2DataTest {

    @Test
    void damagePerAttackDefinedForProtossUnits() {
        assertThat(SC2Data.damagePerAttack(UnitType.PROBE)).isEqualTo(5);
        assertThat(SC2Data.damagePerAttack(UnitType.ZEALOT)).isEqualTo(8);
        assertThat(SC2Data.damagePerAttack(UnitType.STALKER)).isEqualTo(13);
        assertThat(SC2Data.damagePerAttack(UnitType.IMMORTAL)).isEqualTo(20);
    }

    @Test
    void damagePerAttackDefinedForTerranAndZergUnits() {
        assertThat(SC2Data.damagePerAttack(UnitType.MARINE)).isEqualTo(6);
        assertThat(SC2Data.damagePerAttack(UnitType.MARAUDER)).isEqualTo(10);
        assertThat(SC2Data.damagePerAttack(UnitType.ROACH)).isEqualTo(9);
        assertThat(SC2Data.damagePerAttack(UnitType.HYDRALISK)).isEqualTo(12);
    }

    @Test
    void attackCooldownInTicksDefinedForAllCombatUnits() {
        assertThat(SC2Data.attackCooldownInTicks(UnitType.MARINE)).isEqualTo(1);
        assertThat(SC2Data.attackCooldownInTicks(UnitType.HYDRALISK)).isEqualTo(1);
        assertThat(SC2Data.attackCooldownInTicks(UnitType.PROBE)).isEqualTo(2);
        assertThat(SC2Data.attackCooldownInTicks(UnitType.ZEALOT)).isEqualTo(2);
        assertThat(SC2Data.attackCooldownInTicks(UnitType.STALKER)).isEqualTo(3);
    }

    @Test
    void defaultCooldownAppliesForUnknownType() {
        assertThat(SC2Data.attackCooldownInTicks(UnitType.UNKNOWN)).isEqualTo(2);
    }

    @Test
    void defaultDamageAppliesForUnknownType() {
        assertThat(SC2Data.damagePerAttack(UnitType.UNKNOWN)).isEqualTo(5);
    }
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
mvn test -Dtest=SC2DataTest -q 2>&1 | head -10
```
Expected: `COMPILATION ERROR` — methods don't exist yet.

- [ ] **Step 3: Add new methods to SC2Data**

In `src/main/java/io/quarkmind/domain/SC2Data.java`, add after `damagePerTick`:

```java
/**
 * Damage dealt per attack event (replaces damagePerTick from E3).
 * Phase E4: units fire at cooldown intervals, not every tick.
 */
public static int damagePerAttack(UnitType type) {
    return switch (type) {
        case PROBE     ->  5;
        case ZEALOT    ->  8;
        case STALKER   -> 13;
        case IMMORTAL  -> 20;
        case MARINE    ->  6;
        case MARAUDER  -> 10;
        case ROACH     ->  9;
        case HYDRALISK -> 12;
        default        ->  5;
    };
}

/** Ticks between attacks (cooldown reset after firing). 1 tick = 500ms at Faster speed. */
public static int attackCooldownInTicks(UnitType type) {
    return switch (type) {
        case MARINE, HYDRALISK                         -> 1;
        case PROBE, ZEALOT, IMMORTAL, MARAUDER, ROACH  -> 2;
        case STALKER                                   -> 3;
        default                                        -> 2;
    };
}
```

- [ ] **Step 4: Run tests and confirm green**

```bash
mvn test -Dtest=SC2DataTest -q
```
Expected: `BUILD SUCCESS`, 5 tests pass.

- [ ] **Step 5: Run full test suite to confirm nothing broken**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS` — `damagePerTick` still present, nothing broken.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/test/java/io/quarkmind/domain/SC2DataTest.java
git commit -m "feat(e4): add damagePerAttack and attackCooldownInTicks to SC2Data"
```

---

## Task 3: GameState Breaking Change + SimulatedGame Staging Helper

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/GameState.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`
- Modify: `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` (line 53)
- Modify: `src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java` (line 93)
- Modify: `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java` (line 157)
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (line 272)

- [ ] **Step 1: Write a failing GameState test**

Add to `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java` (or a new file — add to the existing domain test class for brevity):

```java
// Add this import at top: import io.quarkmind.domain.*;
// Add this test to EnemyStrategyTest:

@Test
void gameStateIncludesEnemyStagingArea() {
    Unit staged = new Unit("s-1", UnitType.ZEALOT, new Point2d(26, 26),
        100, 100, 50, 50);
    GameState state = new GameState(50, 0, 15, 12,
        List.of(), List.of(), List.of(),
        List.of(staged),   // enemyStagingArea
        List.of(), 0L);
    assertThat(state.enemyStagingArea()).hasSize(1);
    assertThat(state.enemyStagingArea().get(0).tag()).isEqualTo("s-1");
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
mvn test -Dtest=EnemyStrategyTest#gameStateIncludesEnemyStagingArea -q 2>&1 | head -15
```
Expected: `COMPILATION ERROR` — `GameState` constructor doesn't accept 10 args yet.

- [ ] **Step 3: Update GameState record**

Replace `src/main/java/io/quarkmind/domain/GameState.java` entirely:

```java
package io.quarkmind.domain;

import java.util.List;

public record GameState(
    int minerals,
    int vespene,
    int supply,
    int supplyUsed,
    List<Unit> myUnits,
    List<Building> myBuildings,
    List<Unit> enemyUnits,
    List<Unit> enemyStagingArea,
    List<Resource> geysers,
    long gameFrame
) {
    public GameState {
        myUnits          = List.copyOf(myUnits);
        myBuildings      = List.copyOf(myBuildings);
        enemyUnits       = List.copyOf(enemyUnits);
        enemyStagingArea = List.copyOf(enemyStagingArea);
        geysers          = List.copyOf(geysers);
    }
}
```

- [ ] **Step 4: Fix all construction sites — run compiler to find them**

```bash
mvn compile 2>&1 | grep "error:" | head -20
```
Expected: ~5 errors at the known sites.

- [ ] **Step 5: Fix SimulatedGame.java line ~90**

Change the `snapshot()` method — add `List.of()` for `enemyStagingArea` before `geysers`:

```java
public synchronized GameState snapshot() {
    return new GameState(minerals, vespene, supply, supplyUsed,
        List.copyOf(myUnits), List.copyOf(myBuildings), List.copyOf(enemyUnits),
        List.copyOf(testStagingArea),   // enemyStagingArea — populated by test helpers
        List.copyOf(geysers), gameFrame.get());
}
```

Also add the staging test helpers and field to `SimulatedGame` (after the `geysers` field declaration):

```java
// E4: staging test helper — lets VisualizerRenderTest inject staged units
private final List<Unit> testStagingArea = new CopyOnWriteArrayList<>();

/** Test helper: adds a unit to the staging area returned by snapshot(). */
public synchronized void addStagedUnitForTesting(UnitType type, Point2d position) {
    testStagingArea.add(new Unit("staging-" + nextTag++, type, position,
        SC2Data.maxHealth(type), SC2Data.maxHealth(type),
        SC2Data.maxShields(type), SC2Data.maxShields(type)));
}

/** Test helper: clears the staging area. Call in @AfterEach or let reset() handle it. */
public synchronized void clearStagedUnitsForTesting() {
    testStagingArea.clear();
}
```

Also in `reset()`, add `testStagingArea.clear();` after `pendingCompletions.clear();`.

- [ ] **Step 6: Fix ObservationTranslator.java**

At the `new GameState(` call (around line 53), add `List.of()` for `enemyStagingArea` before the geysers `List.of()`:

```java
return new GameState(
    obs.getMinerals(),
    obs.getVespene(),
    obs.getFoodCap(),
    obs.getFoodUsed(),
    myUnits,
    myBuildings,
    enemies,
    List.of(),   // enemyStagingArea — not applicable for real SC2
    List.of(),   // geysers: neutral unit detection deferred to Phase 3+
    obs.getGameLoop()
);
```

- [ ] **Step 7: Fix ReplayEngine.java and RealSC2Engine.java**

Both have `return new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), List.of(), 0L);`

Change to:
```java
return new GameState(0, 0, 0, 0, List.of(), List.of(), List.of(), List.of(), List.of(), 0L);
```
(one extra `List.of()` for `enemyStagingArea`).

- [ ] **Step 8: Fix EmulatedGame.java snapshot() — temporarily**

At `EmulatedGame.java` line ~272, update to pass `List.of()` for now (Task 5 will pass the real staging area):

```java
public GameState snapshot() {
    return new GameState(
        (int) mineralAccumulator,
        vespene, supply, supplyUsed,
        List.copyOf(myUnits), List.copyOf(myBuildings),
        List.copyOf(enemyUnits),
        List.of(),                  // enemyStagingArea — wired in Task 5
        List.copyOf(geysers),
        gameFrame);
}
```

- [ ] **Step 9: Compile clean**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS` — zero errors.

- [ ] **Step 10: Run full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS` — all existing tests still pass.

- [ ] **Step 11: Run the new GameState test**

```bash
mvn test -Dtest=EnemyStrategyTest -q
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 12: Commit**

```bash
git add src/main/java/io/quarkmind/domain/GameState.java \
        src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java \
        src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java \
        src/main/java/io/quarkmind/sc2/replay/ReplayEngine.java \
        src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/domain/EnemyStrategyTest.java
git commit -m "feat(e4): add enemyStagingArea to GameState + SimulatedGame staging test helpers"
```

---

## Task 4: EmulatedGame — Attack Cooldowns + Cancel Path + resolveCombat() Rewrite

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java` (delete `damagePerTick`)
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Write new cooldown tests in EmulatedGameTest**

Add these tests to `EmulatedGameTest` (after the existing E3 section):

```java
// ---- E4: attack cooldowns ----

@Test
void firstAttackFiresImmediately() {
    // Initial cooldown = 0 (absent from map) — attack fires on first tick
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick();

    Unit zealot = game.snapshot().enemyUnits().get(0);
    assertThat(zealot.shields())
        .isLessThan(SC2Data.maxShields(UnitType.ZEALOT))
        .isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - SC2Data.damagePerAttack(UnitType.PROBE));
    // 50 - 5 = 45
}

@Test
void attackCooldownPreventsRepeatOnNextTick() {
    // After attack fires, probe cooldown resets to 2 — no damage on tick 2
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick(); // tick 1: probe fires, cooldown → 2
    int shieldsAfterTick1 = game.snapshot().enemyUnits().get(0).shields(); // 45

    game.tick(); // tick 2: cooldown = 1, probe does NOT fire
    int shieldsAfterTick2 = game.snapshot().enemyUnits().get(0).shields();

    assertThat(shieldsAfterTick2).isEqualTo(shieldsAfterTick1);
}

@Test
void cooldownExpiresAndAttackFiresAgain() {
    // PROBE cooldown = 2: fires tick 1, skips tick 2, fires tick 3
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick(); // tick 1: attack (shields 50→45)
    game.tick(); // tick 2: cooldown 1 — no attack
    game.tick(); // tick 3: cooldown 0 — attack fires again (45→40)

    Unit zealot = game.snapshot().enemyUnits().get(0);
    assertThat(zealot.shields())
        .isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - 2 * SC2Data.damagePerAttack(UnitType.PROBE));
    // 50 - 2*5 = 40
}

@Test
void moveIntentCancelsAutoAttack() {
    // AttackIntent adds to attackingUnits; MoveIntent removes it immediately
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.applyIntent(new MoveIntent("probe-0", new Point2d(9.3f, 9))); // cancel

    // Run enough ticks for cooldown to cycle — probe should never attack
    for (int i = 0; i < 5; i++) game.tick();

    Unit zealot = game.snapshot().enemyUnits().get(0);
    // Probe-0 never attacked — Zealot shields untouched by probe
    assertThat(zealot.shields()).isEqualTo(SC2Data.maxShields(UnitType.ZEALOT));
}

@Test
void enemyAlwaysAttacksWithCooldown() {
    // Enemy Zealot (cooldown=2) attacks probe every 2 ticks without AttackIntent
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick(); // tick 1: Zealot fires (cooldown 0→2), probe shields: 20→12
    int shieldsAfterTick1 = game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals("probe-0")).findFirst().orElseThrow().shields();
    assertThat(shieldsAfterTick1)
        .isEqualTo(SC2Data.maxShields(UnitType.PROBE) - SC2Data.damagePerAttack(UnitType.ZEALOT));
    // 20 - 8 = 12

    game.tick(); // tick 2: Zealot cooldown 1 — no attack. shields unchanged.
    int shieldsAfterTick2 = game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals("probe-0")).findFirst().orElseThrow().shields();
    assertThat(shieldsAfterTick2).isEqualTo(shieldsAfterTick1);
}
```

- [ ] **Step 2: Update broken E3 tests in EmulatedGameTest**

The E3 test `damageOverflowsFromShieldsToHp` assumed `damagePerTick(ZEALOT)=5`. With `damagePerAttack(ZEALOT)=8`:
- probe has 3 shields: 3 absorbed, 5 overflow to HP → HP = 45 - 5 = 40

```java
// FIND this assertion in damageOverflowsFromShieldsToHp:
assertThat(probe.health()).isEqualTo(SC2Data.maxHealth(UnitType.PROBE) - 2); // 45 - 2 = 43

// REPLACE with:
assertThat(probe.health()).isEqualTo(SC2Data.maxHealth(UnitType.PROBE) - 5); // 45 - 5 = 40
// Zealot damagePerAttack=8: 3 shields absorb, 5 overflow to HP
```

Also update `unitInsideAttackRangeReceivesDamage` which references `damagePerTick`:
```java
// FIND:
assertThat(probe.shields()).isEqualTo(
    SC2Data.maxShields(UnitType.PROBE) - SC2Data.damagePerTick(UnitType.STALKER));

// REPLACE with:
assertThat(probe.shields()).isEqualTo(
    SC2Data.maxShields(UnitType.PROBE) - SC2Data.damagePerAttack(UnitType.STALKER));
// 20 - 13 = 7
```

- [ ] **Step 3: Run tests to confirm 5 new tests fail + 2 updated tests pass**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | tail -20
```
Expected: New cooldown tests fail (feature not implemented), updated E3 tests compile and pass/fail as appropriate.

- [ ] **Step 4: Delete damagePerTick from SC2Data.java**

Remove the entire `damagePerTick` method block from `SC2Data.java`:
```java
// DELETE this block:
/**
 * Damage dealt per 500ms scheduler tick (SC2 DPS at Faster speed × 0.5).
 * Phase E3: flat per-tick damage. Phase E4 will replace with per-attack events.
 */
public static int damagePerTick(UnitType type) {
    return switch (type) { ... };
}
```

- [ ] **Step 5: Add cooldown fields and helpers to EmulatedGame**

In `EmulatedGame`, after the `attackingUnits` field declaration, add:

```java
// E4: per-unit attack cooldowns (absent key = 0 = can attack immediately)
private final Map<String, Integer> unitCooldowns  = new HashMap<>();
private final Map<String, Integer> enemyCooldowns = new HashMap<>();
```

In `reset()`, add after `attackingUnits.clear()`:
```java
unitCooldowns.clear();
enemyCooldowns.clear();
```

- [ ] **Step 6: Rewrite resolveCombat() in EmulatedGame**

Replace the entire `resolveCombat()` method:

```java
private void resolveCombat() {
    // Step 1: decrement all cooldowns (floor 0)
    unitCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
    enemyCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));

    Map<String, Integer> pending       = new HashMap<>();
    Set<String>          firedFriendly = new HashSet<>();
    Set<String>          firedEnemy    = new HashSet<>();

    // Step 2: collect damage from units where cooldown == 0
    for (Unit attacker : myUnits) {
        if (!attackingUnits.contains(attacker.tag())) continue;
        if (unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
        nearestInRange(attacker.position(), enemyUnits, SC2Data.attackRange(attacker.type()))
            .ifPresent(target -> {
                pending.merge(target.tag(), SC2Data.damagePerAttack(attacker.type()), Integer::sum);
                firedFriendly.add(attacker.tag());
            });
    }
    for (Unit attacker : enemyUnits) {
        if (enemyCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
        nearestInRange(attacker.position(), myUnits, SC2Data.attackRange(attacker.type()))
            .ifPresent(target -> {
                pending.merge(target.tag(), SC2Data.damagePerAttack(attacker.type()), Integer::sum);
                firedEnemy.add(attacker.tag());
            });
    }

    // Step 3: apply damage — two-pass (collect all, then apply)
    myUnits.replaceAll(u -> applyDamage(u, pending.getOrDefault(u.tag(), 0)));
    myUnits.removeIf(u -> {
        if (u.health() <= 0) {
            unitTargets.remove(u.tag());
            attackingUnits.remove(u.tag());
            unitCooldowns.remove(u.tag());
            return true;
        }
        return false;
    });
    enemyUnits.replaceAll(u -> applyDamage(u, pending.getOrDefault(u.tag(), 0)));
    enemyUnits.removeIf(u -> {
        if (u.health() <= 0) {
            enemyTargets.remove(u.tag());
            enemyCooldowns.remove(u.tag());
            return true;
        }
        return false;
    });

    // Step 4: reset cooldown for units that fired
    for (Unit u : myUnits) {
        if (firedFriendly.contains(u.tag()))
            unitCooldowns.put(u.tag(), SC2Data.attackCooldownInTicks(u.type()));
    }
    for (Unit u : enemyUnits) {
        if (firedEnemy.contains(u.tag()))
            enemyCooldowns.put(u.tag(), SC2Data.attackCooldownInTicks(u.type()));
    }
}
```

- [ ] **Step 7: Fix setTarget() cancel path**

Replace the `setTarget` method:

```java
private void setTarget(String tag, Point2d target, boolean isAttack) {
    if (myUnits.stream().anyMatch(u -> u.tag().equals(tag))) {
        unitTargets.put(tag, target);
        if (isAttack) attackingUnits.add(tag);
        else          attackingUnits.remove(tag);  // MoveIntent cancels auto-attack
        log.debugf("[EMULATED] %s → (%.1f,%.1f) attack=%b", tag, target.x(), target.y(), isAttack);
    }
}
```

- [ ] **Step 8: Compile**

```bash
mvn compile -q
```
Expected: `BUILD SUCCESS` — `damagePerTick` calls gone, new cooldown code compiles.

- [ ] **Step 9: Run EmulatedGameTest — all must pass**

```bash
mvn test -Dtest=EmulatedGameTest -q
```
Expected: `BUILD SUCCESS`. All E1, E2, E3, and new E4 cooldown tests pass.

- [ ] **Step 10: Run full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 11: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e4): attack cooldowns — replace damagePerTick with per-unit cooldown maps"
```

---

## Task 5: EmulatedGame — Enemy Economy + tickEnemyStrategy()

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Write all economy tests in EmulatedGameTest**

Add this section after the E4 cooldown tests:

```java
// ---- E4: enemy economy ----

@Test
void enemyStrategyNullIsNoop() {
    game.setEnemyStrategy(null);
    game.tick(); // must not throw NullPointerException
    assertThat(game.snapshot().enemyStagingArea()).isEmpty();
}

@Test
void enemyAccumulatesMineralsEachTick() {
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(), false, 5, new EnemyAttackConfig(10, 9999)));

    game.tick();
    assertThat(game.enemyMinerals()).isEqualTo(5);

    game.tick();
    assertThat(game.enemyMinerals()).isEqualTo(10);
}

@Test
void enemyTrainsUnitWhenMineralsAfford() {
    // 20 minerals/tick, Zealot costs 100 → trains after 5 ticks
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        false, 20,
        new EnemyAttackConfig(10, 9999)));

    for (int i = 0; i < 5; i++) game.tick();

    assertThat(game.snapshot().enemyStagingArea()).hasSize(1);
    assertThat(game.snapshot().enemyStagingArea().get(0).type()).isEqualTo(UnitType.ZEALOT);
}

@Test
void enemyDoesNotTrainWhenInsufficientMinerals() {
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        false, 0,   // no mineral income
        new EnemyAttackConfig(10, 9999)));

    for (int i = 0; i < 10; i++) game.tick();

    assertThat(game.snapshot().enemyStagingArea()).isEmpty();
}

@Test
void enemySendsAttackWhenArmyThresholdMet() {
    // threshold=1, 20 minerals/tick → after 5 ticks: 1 Zealot trained → attack fires
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        true, 20,
        new EnemyAttackConfig(1, 9999)));

    for (int i = 0; i < 5; i++) game.tick();

    assertThat(game.snapshot().enemyStagingArea()).isEmpty();  // cleared — attack sent
    assertThat(game.snapshot().enemyUnits()).isNotEmpty();
    assertThat(game.snapshot().enemyUnits().get(0).type()).isEqualTo(UnitType.ZEALOT);
}

@Test
void enemySendsAttackWhenTimerFires() {
    // threshold=100 (never), timer=5 frames. 10 minerals/tick, Zealot(100) trains at tick 10.
    // At tick 5: staging empty → no attack (timer fires but guard prevents it).
    // At tick 10: Zealot trained, framesSinceAttack=10 >= 5 → timer fires → attack.
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        true, 10,
        new EnemyAttackConfig(100, 5)));

    for (int i = 0; i < 5; i++) game.tick();
    assertThat(game.snapshot().enemyUnits()).isEmpty(); // timer fired but staging empty → no attack

    for (int i = 0; i < 5; i++) game.tick(); // tick 10: Zealot trained, timer fires → attack
    assertThat(game.snapshot().enemyStagingArea()).isEmpty();
    assertThat(game.snapshot().enemyUnits()).hasSize(1);
}

@Test
void timerFiresBeforeArmyThreshold() {
    // threshold=3, timer=5. 25 minerals/tick, Zealot(100) trains at tick 4.
    // At tick 4: 1 unit in staging, threshold not met (3), timer not fired (4 < 5).
    // At tick 5: minerals=25 (accumulated but can't afford 2nd Zealot yet),
    //   framesSinceAttack=5 >= 5 → timer fires with 1 unit in staging → attack.
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        true, 25,
        new EnemyAttackConfig(3, 5)));

    for (int i = 0; i < 4; i++) game.tick();
    assertThat(game.snapshot().enemyStagingArea()).hasSize(1); // training done, attack not yet

    game.tick(); // frame 5: timer fires → attack sent
    assertThat(game.snapshot().enemyStagingArea()).isEmpty();
    assertThat(game.snapshot().enemyUnits()).hasSize(1);
}

@Test
void enemyStagingClearedAfterAttack() {
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        true, 20,
        new EnemyAttackConfig(1, 9999)));

    for (int i = 0; i < 5; i++) game.tick(); // trains 1 Zealot → threshold=1 → attack fires

    assertThat(game.snapshot().enemyStagingArea()).isEmpty();
}

@Test
void enemyBuildOrderLoops() {
    // 2-step order [Zealot, Stalker] with loop=true, 125 minerals/tick
    // tick 1: +125 → train Zealot(100). minerals=25. staging=[Z].
    // tick 2: +125=150 → train Stalker(125). minerals=25. staging=[Z,S].
    // tick 3: +125=150 → loop: train Zealot(100). minerals=50. staging=[Z,S,Z].
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT), new EnemyBuildStep(UnitType.STALKER)),
        true, 125,
        new EnemyAttackConfig(10, 9999)));

    game.tick();
    assertThat(game.snapshot().enemyStagingArea()).hasSize(1);
    assertThat(game.snapshot().enemyStagingArea().get(0).type()).isEqualTo(UnitType.ZEALOT);

    game.tick();
    assertThat(game.snapshot().enemyStagingArea()).hasSize(2);
    assertThat(game.snapshot().enemyStagingArea().get(1).type()).isEqualTo(UnitType.STALKER);

    game.tick();
    assertThat(game.snapshot().enemyStagingArea()).hasSize(3);
    assertThat(game.snapshot().enemyStagingArea().get(2).type()).isEqualTo(UnitType.ZEALOT); // looped
}

@Test
void enemyBuildOrderStopsWhenExhausted() {
    // loop=false, 2-step order, 100 minerals/tick → exactly 2 units, then stops
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT), new EnemyBuildStep(UnitType.ZEALOT)),
        false, 100,
        new EnemyAttackConfig(10, 9999)));

    for (int i = 0; i < 10; i++) game.tick();

    assertThat(game.snapshot().enemyStagingArea()).hasSize(2); // exactly 2, not more
}

@Test
void enemyUnitsStayAtSpawnUntilAttack() {
    // threshold=5 (won't fire with 3 units), timer=9999 — no attack
    game.setEnemyStrategy(new EnemyStrategy(
        List.of(new EnemyBuildStep(UnitType.ZEALOT)),
        true, 100,
        new EnemyAttackConfig(5, 9999)));

    for (int i = 0; i < 3; i++) game.tick();

    game.snapshot().enemyStagingArea().forEach(u ->
        assertThat(u.position()).isEqualTo(new Point2d(26, 26)));
    assertThat(game.snapshot().enemyUnits()).isEmpty();
}
```

Note: these tests need `import io.quarkmind.domain.EnemyStrategy;`, `import io.quarkmind.domain.EnemyBuildStep;`, `import io.quarkmind.domain.EnemyAttackConfig;`, and `import java.util.List;` at the top of `EmulatedGameTest`.

- [ ] **Step 2: Run to confirm all economy tests fail**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | grep -E "FAIL|PASS|ERROR" | tail -20
```
Expected: ~12 new tests fail, all existing tests still pass.

- [ ] **Step 3: Add economy fields to EmulatedGame**

After the `enemyCooldowns` field, add:

```java
// E4: enemy economy
private EnemyStrategy enemyStrategy;
private double        enemyMineralAccumulator;
private int           enemyBuildIndex;
private long          framesSinceLastAttack;
private final List<Unit> enemyStagingArea = new ArrayList<>();
```

- [ ] **Step 4: Update reset() to clear economy state**

Add after `enemyCooldowns.clear()` in `reset()`:

```java
enemyMineralAccumulator = 0;
enemyBuildIndex         = 0;
framesSinceLastAttack   = 0;
enemyStagingArea.clear();
```

- [ ] **Step 5: Add tickEnemyStrategy() to EmulatedGame**

Add this new method after `spawnEnemyWaves()`:

```java
private void tickEnemyStrategy() {
    if (enemyStrategy == null) return;

    // 1. Accumulate enemy minerals
    enemyMineralAccumulator += enemyStrategy.mineralsPerTick();

    // 2. Execute next build step if affordable and order not exhausted
    List<EnemyBuildStep> order = enemyStrategy.buildOrder();
    if (!order.isEmpty()) {
        boolean canAdvance = enemyStrategy.loop() || enemyBuildIndex < order.size();
        if (canAdvance) {
            EnemyBuildStep step = order.get(enemyBuildIndex % order.size());
            int cost = SC2Data.mineralCost(step.unitType());
            if ((int) enemyMineralAccumulator >= cost) {
                enemyMineralAccumulator -= cost;
                String tag = "enemy-" + nextTag++;
                int hp = SC2Data.maxHealth(step.unitType());
                enemyStagingArea.add(new Unit(tag, step.unitType(),
                    new Point2d(26, 26), hp, hp,
                    SC2Data.maxShields(step.unitType()), SC2Data.maxShields(step.unitType())));
                if (enemyStrategy.loop() || enemyBuildIndex < order.size() - 1)
                    enemyBuildIndex++;
                else
                    enemyBuildIndex = order.size(); // mark exhausted
                log.debugf("[EMULATED] Enemy trained %s (staging size=%d)",
                    step.unitType(), enemyStagingArea.size());
            }
        }
    }

    // 3. Check attack triggers (army threshold OR frame timer)
    framesSinceLastAttack++;
    EnemyAttackConfig atk = enemyStrategy.attackConfig();
    boolean thresholdMet = enemyStagingArea.size() >= atk.armyThreshold();
    boolean timerFired   = framesSinceLastAttack >= atk.attackIntervalFrames();
    if ((thresholdMet || timerFired) && !enemyStagingArea.isEmpty()) {
        for (Unit u : enemyStagingArea) {
            enemyUnits.add(u);
            enemyTargets.put(u.tag(), NEXUS_POS);
        }
        log.infof("[EMULATED] Enemy attack launched: %d units (threshold=%b timer=%b)",
            enemyStagingArea.size(), thresholdMet, timerFired);
        enemyStagingArea.clear();
        framesSinceLastAttack = 0;
    }
}
```

- [ ] **Step 6: Call tickEnemyStrategy() from tick()**

In the `tick()` method, add after `spawnEnemyWaves()`:

```java
public void tick() {
    gameFrame++;
    mineralAccumulator += miningProbes * SC2Data.MINERALS_PER_PROBE_PER_TICK;
    moveFriendlyUnits();
    moveEnemyUnits();
    resolveCombat();
    fireCompletions();
    spawnEnemyWaves();
    tickEnemyStrategy();   // new E4
}
```

- [ ] **Step 7: Update snapshot() to expose staging area**

Replace the temporary `List.of()` in `snapshot()`:

```java
public GameState snapshot() {
    return new GameState(
        (int) mineralAccumulator,
        vespene, supply, supplyUsed,
        List.copyOf(myUnits), List.copyOf(myBuildings),
        List.copyOf(enemyUnits),
        List.copyOf(enemyStagingArea),   // real staging area
        List.copyOf(geysers),
        gameFrame);
}
```

- [ ] **Step 8: Add package-private helpers for tests**

```java
void setEnemyStrategy(EnemyStrategy s) { this.enemyStrategy = s; }
int  enemyMinerals()                   { return (int) enemyMineralAccumulator; }
int  enemyStagingSize()                { return enemyStagingArea.size(); }
```

- [ ] **Step 9: Run economy tests — all must pass**

```bash
mvn test -Dtest=EmulatedGameTest -q
```
Expected: `BUILD SUCCESS`, all tests pass including all new economy tests.

- [ ] **Step 10: Run full suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 11: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e4): enemy economy — build order, staging area, attack triggers"
```

---

## Task 6: EmulatedConfig + REST Endpoint + Integration Tests

**Files:**
- Modify: `src/main/java/io/quarkmind/qa/EmulatedConfig.java`
- Modify: `src/main/java/io/quarkmind/qa/EmulatedConfigResource.java`
- Modify: `src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java`
- Modify: `src/test/java/io/quarkmind/qa/GameStateWebSocketTest.java`

- [ ] **Step 1: Write integration tests for the new REST endpoint**

Add to `EmulatedConfigResourceTest.java` (after existing tests):

```java
// ---- E4: enemy strategy REST ----

@Test
void getEnemyStrategyReturnsDefault() {
    given()
        .when().get("/qa/emulated/config/enemy-strategy")
        .then()
        .statusCode(200)
        .body("buildOrder.size()", equalTo(3))
        .body("buildOrder[0].unitType", equalTo("ZEALOT"))
        .body("loop", equalTo(true))
        .body("mineralsPerTick", equalTo(2))
        .body("attackConfig.armyThreshold", equalTo(3))
        .body("attackConfig.attackIntervalFrames", equalTo(200));
}

@Test
void putEnemyStrategyUpdatesIt() {
    String body = """
        {
          "buildOrder": [{"unitType":"STALKER"},{"unitType":"IMMORTAL"}],
          "loop": false,
          "mineralsPerTick": 5,
          "attackConfig": {"armyThreshold": 2, "attackIntervalFrames": 100}
        }
        """;

    given()
        .contentType("application/json")
        .body(body)
        .when().put("/qa/emulated/config/enemy-strategy")
        .then()
        .statusCode(200);

    given()
        .when().get("/qa/emulated/config/enemy-strategy")
        .then()
        .statusCode(200)
        .body("buildOrder.size()", equalTo(2))
        .body("buildOrder[0].unitType", equalTo("STALKER"))
        .body("loop", equalTo(false))
        .body("mineralsPerTick", equalTo(5));
}

@Test
void putEnemyStrategyWithSingleStep() {
    String body = """
        {
          "buildOrder": [{"unitType":"ZEALOT"}],
          "loop": true,
          "mineralsPerTick": 10,
          "attackConfig": {"armyThreshold": 1, "attackIntervalFrames": 50}
        }
        """;

    given()
        .contentType("application/json")
        .body(body)
        .when().put("/qa/emulated/config/enemy-strategy")
        .then()
        .statusCode(200)
        .body("buildOrder.size()", equalTo(1))
        .body("attackConfig.armyThreshold", equalTo(1));
}
```

Add `@AfterEach` cleanup to reset the strategy after each test (add to existing `resetConfig()`):
```java
@AfterEach
void resetConfig() {
    config.setWaveSpawnFrame(200);
    config.setWaveUnitCount(4);
    config.setWaveUnitType("ZEALOT");
    config.setUnitSpeed(0.5);
    config.setEnemyStrategy(EnemyStrategy.defaultProtoss()); // reset strategy
}
```

- [ ] **Step 2: Write WebSocket staging area test**

Add to `GameStateWebSocketTest.java` (after existing tests):

```java
@Test
void gameStateJsonContainsEnemyStagingArea() throws Exception {
    var received = new LinkedBlockingQueue<String>(10);
    WebSocket ws = connect(received);
    ws.request(1);

    engine.observe();
    String json = poll(received);
    assertThat(json).isNotNull();
    // Field must be present even when empty — visualizer reads it on every frame
    assertThat(json).contains("\"enemyStagingArea\"");

    ws.abort();
}
```

- [ ] **Step 3: Run integration tests to confirm they fail**

```bash
mvn test -Dtest=EmulatedConfigResourceTest,GameStateWebSocketTest -q 2>&1 | tail -15
```
Expected: New tests fail (endpoints don't exist, `enemyStagingArea` field not in JSON).

- [ ] **Step 4: Update EmulatedConfig**

Add to `EmulatedConfig.java`:

```java
// Existing imports — add:
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.EnemyStrategy;
import jakarta.inject.Inject;
import java.nio.file.Path;

// New field alongside existing fields:
@ConfigProperty(name = "emulated.enemy.strategy-file", defaultValue = "")
String strategyFile;

private volatile EnemyStrategy enemyStrategy = EnemyStrategy.defaultProtoss();

@Inject
ObjectMapper objectMapper;
```

Update `init()` to load from file if configured:

```java
@PostConstruct
void init() {
    waveSpawnFrame = defaultWaveSpawnFrame;
    waveUnitCount  = defaultWaveUnitCount;
    waveUnitType   = defaultWaveUnitType;
    unitSpeed      = defaultUnitSpeed;
    // Load enemy strategy from file if configured
    if (!strategyFile.isBlank()) {
        try {
            enemyStrategy = objectMapper.readValue(
                Path.of(strategyFile).toFile(), EnemyStrategy.class);
            log.infof("[CONFIG] Loaded enemy strategy from %s", strategyFile);
        } catch (Exception e) {
            log.warnf("[CONFIG] Could not load strategy file %s — using default. Error: %s",
                strategyFile, e.getMessage());
        }
    }
}
```

Add getter and setter:

```java
public EnemyStrategy getEnemyStrategy()        { return enemyStrategy; }
public void setEnemyStrategy(EnemyStrategy s)  { this.enemyStrategy = s; }
```

Update `Snapshot` record and `snapshot()` to include strategy (for GET response):

```java
public record Snapshot(int waveSpawnFrame, int waveUnitCount,
                       String waveUnitType, double unitSpeed) {}
// Leave Snapshot as-is — strategy has its own endpoint
```

Add `import org.jboss.logging.Logger;` and `private static final Logger log = Logger.getLogger(EmulatedConfig.class);` if not already present.

- [ ] **Step 5: Update EmulatedConfigResource**

Add two new endpoints to `EmulatedConfigResource.java`:

```java
// Add imports:
import io.quarkmind.domain.EnemyStrategy;

// Add new endpoints after existing PUT:
@GET
@Path("/enemy-strategy")
public EnemyStrategy getEnemyStrategy() {
    return config.getEnemyStrategy();
}

@PUT
@Path("/enemy-strategy")
@Consumes(MediaType.APPLICATION_JSON)
public Response setEnemyStrategy(EnemyStrategy strategy) {
    config.setEnemyStrategy(strategy);
    return Response.ok(strategy).build();
}
```

- [ ] **Step 6: Run integration tests — all must pass**

```bash
mvn test -Dtest=EmulatedConfigResourceTest,GameStateWebSocketTest -q
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 7: Run full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/qa/EmulatedConfig.java \
        src/main/java/io/quarkmind/qa/EmulatedConfigResource.java \
        src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java \
        src/test/java/io/quarkmind/qa/GameStateWebSocketTest.java
git commit -m "feat(e4): EmulatedConfig + REST endpoint for EnemyStrategy with JSON serde"
```

---

## Task 7: EmulatedEngine — Wire Strategy Each Tick

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java`

- [ ] **Step 1: Add strategy wiring to EmulatedEngine.tick()**

In `EmulatedEngine.java`, update the `tick()` method:

```java
@Override
public void tick() {
    game.setUnitSpeed(config.getUnitSpeed());           // existing
    game.setEnemyStrategy(config.getEnemyStrategy());   // new E4: live strategy updates
    game.tick();
}
```

- [ ] **Step 2: Compile and run full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java
git commit -m "feat(e4): wire EnemyStrategy from EmulatedConfig into EmulatedGame each tick"
```

---

## Task 8: Visualizer Staging Layer + E2E Tests

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.js`
- Modify: `src/test/java/io/quarkmind/qa/VisualizerRenderTest.java`

- [ ] **Step 1: Write E2E tests**

Add to `VisualizerRenderTest.java` (after existing tests, before the `parseMinerals` helper):

```java
/**
 * Staging layer test: units in enemyStagingArea must render as blue-tinted sprites.
 * Uses SimulatedGame.addStagedUnitForTesting() to inject a staged enemy without
 * needing the %emulated profile — same pattern as setUnitHealth for combat tests.
 */
@Test
void enemyStagedUnitsRenderAtSpawn() {
    Page page = openPage();

    simulatedGame.addStagedUnitForTesting(UnitType.ZEALOT, new Point2d(26, 26));
    engine.observe();

    page.waitForFunction(
        "() => window.__test.spriteCount('staging') >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate(
        "() => window.__test.spriteCount('staging')")).intValue();
    assertThat(count).as("staging layer must have 1 sprite").isEqualTo(1);

    page.close();
}

/**
 * Staging count matches game state: two different unit types staged → two sprites.
 */
@Test
void stagedUnitCountMatchesGameState() {
    Page page = openPage();

    simulatedGame.addStagedUnitForTesting(UnitType.ZEALOT,  new Point2d(26,   26));
    simulatedGame.addStagedUnitForTesting(UnitType.STALKER, new Point2d(26.5f, 26));
    engine.observe();

    page.waitForFunction(
        "() => window.__test.spriteCount('staging') >= 2",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate(
        "() => window.__test.spriteCount('staging')")).intValue();
    assertThat(count).as("two staged units → two staging sprites").isEqualTo(2);

    page.close();
}

/**
 * Staging sprites disappear when the game state clears the staging area.
 * Simulates an attack being sent (staging → enemy).
 */
@Test
void stagedUnitsDisappearWhenStagingClears() {
    Page page = openPage();

    simulatedGame.addStagedUnitForTesting(UnitType.ZEALOT, new Point2d(26, 26));
    engine.observe();

    page.waitForFunction(
        "() => window.__test.spriteCount('staging') >= 1",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    simulatedGame.clearStagedUnitsForTesting(); // simulates attack sent
    engine.observe();

    page.waitForFunction(
        "() => window.__test.spriteCount('staging') === 0",
        null, new Page.WaitForFunctionOptions().setTimeout(5_000));

    int count = ((Number) page.evaluate(
        "() => window.__test.spriteCount('staging')")).intValue();
    assertThat(count).as("staging layer must be empty after clear").isEqualTo(0);

    page.close();
}
```

Also add `import io.quarkmind.domain.Point2d;` to `VisualizerRenderTest` if not already present.

- [ ] **Step 2: Run E2E tests — confirm they fail (staging layer not in JS yet)**

```bash
mvn test -Dtest=VisualizerRenderTest#enemyStagedUnitsRenderAtSpawn -q 2>&1 | tail -10
```
Expected: test fails — `spriteCount('staging')` returns 0 (layer not rendered).

- [ ] **Step 3: Add staging layer to visualizer.js**

In `init()`, after `const enemy = new PIXI.Container();` and before `const hud`, add:

```javascript
const staging = new PIXI.Container();
```

Update `app.stage.addChild(...)` to include staging (between enemy and hud):
```javascript
app.stage.addChild(background, resource, building, unit, enemy, staging, hud);
```

Update `window._layers` to include staging:
```javascript
window._layers = { resource, building, unit, enemy, staging };
```

In `updateScene()`, add the staging layer call after the enemy `syncLayer`:

```javascript
syncLayer(
    window._layers.staging,
    state.enemyStagingArea ?? [],
    'staging',
    e => makeUnitSprite(UNIT_ALIAS[e.type] ?? null, UNIT_RADIUS, 0x4488ff)
);
```

- [ ] **Step 4: Run E2E tests — all must pass**

```bash
mvn test -Dtest=VisualizerRenderTest -q
```
Expected: `BUILD SUCCESS` — all existing tests plus 3 new staging tests pass (or skip if Chromium not installed).

- [ ] **Step 5: Run full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.js \
        src/test/java/io/quarkmind/qa/VisualizerRenderTest.java
git commit -m "feat(e4): visualizer staging layer + E2E render tests for staged enemy units"
```

---

## Final Verification

- [ ] **Run full test suite one last time**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Run benchmark to record post-E4 baseline**

```bash
mvn test -Pbenchmark
```
Save output to `docs/benchmarks/2026-04-14-post-e4-baseline.md`.

- [ ] **Count total new tests**

```bash
mvn test -q 2>&1 | grep "Tests run:" | tail -5
```
Expected: ~240+ tests total (236 pre-E4 + ~17 unit + ~5 integration + ~3 E2E).

---

## Self-Review Against Spec

| Spec requirement | Task |
|---|---|
| `EnemyBuildStep`, `EnemyAttackConfig`, `EnemyStrategy` domain records | Task 1 |
| Jackson JSON serde + `defaultProtoss()` | Task 1 |
| `damagePerAttack` + `attackCooldownInTicks` in SC2Data | Task 2 |
| `damagePerTick` deleted | Task 4 |
| `GameState.enemyStagingArea` field | Task 3 |
| All GameState construction sites updated | Task 3 |
| Per-unit cooldown maps in EmulatedGame | Task 4 |
| `resolveCombat()` rewritten with cooldown logic | Task 4 |
| `MoveIntent` cancel path for `attackingUnits` | Task 4 |
| `tickEnemyStrategy()` — mineral accumulation | Task 5 |
| Build order execution with loop/stop | Task 5 |
| Staging area — units wait until attack fires | Task 5 |
| Attack trigger: army threshold OR frame timer | Task 5 |
| `EmulatedConfig` strategy field + file loading | Task 6 |
| REST `GET/PUT /qa/emulated/config/enemy-strategy` | Task 6 |
| `EmulatedEngine` passes strategy each tick | Task 7 |
| Visualizer staging layer (blue tint) | Task 8 |
| Unit tests: cooldowns (5) + economy (11) + updated E3 (2) | Tasks 4, 5 |
| Integration tests: REST strategy + WebSocket staging | Task 6 |
| E2E tests: staging render + disappear | Task 8 |
| `SimulatedGame` staging test helpers | Task 3 |
