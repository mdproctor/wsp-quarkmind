# E10 Tactics: Kiting, Focus-Fire, Range Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend DroolsTacticsTask with per-unit attack range, focus-fire (all units target lowest-HP enemy), and kiting (ranged units step backward during weapon cooldown), following the full Drools → GOAP → dispatch pipeline.

**Architecture:** `weaponCooldownTicks` is added to the `Unit` record and stamped from EmulatedGame's `unitCooldowns` map at snapshot time, matching the SC2 protobuf `weapon_cooldown` field. Drools classifies on-cooldown in-range units into a `"kiting"` group; the GOAP `KITE` action dispatches a `MoveIntent` stepping backward from the nearest enemy. Focus-fire selects the lowest-HP enemy in pure Java before Drools fires; all attacking units receive `AttackIntent` at that shared target.

**Tech Stack:** Java 21 records, Quarkus 3.x, Drools Rule Units, A\* GOAP planner, AssertJ, JUnit 5, `@QuarkusTest`.

**Issue linking:** Record the issue numbers created in Task 0. Replace `#EPIC`, `#UNIT-REC`, `#RANGE`, `#FOCUS`, `#KITE` with the actual numbers in every commit message below.

---

## File Map

| File | Change |
|---|---|
| `src/main/java/io/quarkmind/domain/Unit.java` | Add `int weaponCooldownTicks` field (last) |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Stamp cooldown in `snapshot()`; all internal `new Unit(...)` pass `0` |
| `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java` | All `new Unit(...)` pass `0` |
| `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java` | All `new Unit(...)` pass `0` |
| `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` | Map `weapon_cooldown` from SC2 protobuf |
| `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` | Range fix; `selectFocusTarget`; `computeOnCooldownTags`; `kiteRetreatTarget`; `KITE` action; updated `buildWorldState`/`goalConditionKey`/`dispatch`/`execute` |
| `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java` | Add `onCooldownTags` DataStore |
| `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl` | Tighten in-range rule; add kiting group + Phase 2 kite action |
| `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java` | **Create** — unit tests for pure-Java static methods |
| `src/test/java/io/quarkmind/plugin/DroolsTacticsRuleUnitTest.java` | Add kiting Drools rule tests |
| `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskIT.java` | Add focus-fire and kiting integration tests |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add focus-fire and kiting E2E physics tests |
| All other test files with `new Unit(...)` | Append `, 0` to constructor — see Task 1 |

---

## Task 0: Create GitHub Epic and Child Issues

**Files:** none (GitHub only)

- [ ] **Step 1: Create the E10 epic issue**

```bash
gh issue create \
  --title "E10: TacticsTask extension — kiting, focus-fire, per-unit range" \
  --body "$(cat <<'EOF'
## E10 Epic

Extends DroolsTacticsTask with three coordinated improvements:

1. **Range fix** — replace hardcoded STALKER_RANGE=6.0 with SC2Data.attackRange(unit.type()) per unit
2. **Focus-fire** — all attacking units target same lowest-HP enemy
3. **Kiting** — ranged units on weapon cooldown step backward, matching SC2 weapon_cooldown field

Design spec: docs/superpowers/specs/2026-04-17-e10-tactics-kiting-focusfire-design.md
EOF
)" \
  --label "epic"
```

Record the issue number as `#EPIC`.

- [ ] **Step 2: Create child issue — Unit record**

```bash
gh issue create \
  --title "feat(e10): expose weaponCooldownTicks on Unit snapshot" \
  --body "Add int weaponCooldownTicks to Unit record. EmulatedGame.snapshot() stamps from unitCooldowns map. ObservationTranslator maps SC2 weapon_cooldown. Refs #EPIC" \
  --label "enhancement"
```

Record as `#UNIT-REC`.

- [ ] **Step 3: Create child issue — range fix**

```bash
gh issue create \
  --title "fix(e10): use per-unit attack range in computeInRangeTags" \
  --body "Replace hardcoded STALKER_RANGE=6.0 with SC2Data.attackRange(unit.type()). Stalker range is 5.0, not 6.0. Refs #EPIC" \
  --label "bug"
```

Record as `#RANGE`.

- [ ] **Step 4: Create child issue — focus-fire**

```bash
gh issue create \
  --title "feat(e10): focus-fire — all attacking units target lowest-HP enemy" \
  --body "selectFocusTarget() picks enemy with min(health+shields). dispatch(ATTACK) uses focus target for all units in group. Refs #EPIC" \
  --label "enhancement"
```

Record as `#FOCUS`.

- [ ] **Step 5: Create child issue — kiting**

```bash
gh issue create \
  --title "feat(e10): kiting — Drools classification and GOAP KITE action" \
  --body "onCooldownTags in TacticsRuleUnit; kiting DRL group; KITE GoapAction; kiteRetreatTarget geometry; dispatch emits MoveIntent backward. Refs #EPIC" \
  --label "enhancement"
```

Record as `#KITE`.

---

## Task 1: Add `weaponCooldownTicks` to `Unit` record and stamp in snapshot

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/Unit.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`
- Modify: `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` (add tests)
- Modify: all test files listed in Task 1 step 5

- [ ] **Step 1: Write two failing tests in EmulatedGameTest**

Add inside `class EmulatedGameTest` (before the closing brace), in the `// E10` section:

```java
// ---- E10: weapon cooldown snapshot ----

@Test
void firingUnit_hasCooldownInSnapshot() {
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(10, 10));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10, 12)); // within Stalker range 5.0
    game.applyIntent(new AttackIntent(tag, new Point2d(10, 12)));
    game.tick(); // Stalker fires; cooldown = SC2Data.attackCooldownInTicks(STALKER) = 3
    Unit stalker = game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals(tag)).findFirst().orElseThrow();
    assertThat(stalker.weaponCooldownTicks())
        .isEqualTo(SC2Data.attackCooldownInTicks(UnitType.STALKER));
}

@Test
void freshUnit_hasCooldownZeroInSnapshot() {
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(10, 10));
    Unit stalker = game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals(tag)).findFirst().orElseThrow();
    assertThat(stalker.weaponCooldownTicks()).isEqualTo(0);
}
```

- [ ] **Step 2: Run tests — verify compile error**

```bash
mvn test -Dtest="EmulatedGameTest#firingUnit_hasCooldownInSnapshot+freshUnit_hasCooldownZeroInSnapshot" -q 2>&1 | grep -E "ERROR|cannot find"
```

Expected: compile error — `weaponCooldownTicks()` does not exist.

- [ ] **Step 3: Add field to `Unit.java`**

Replace the entire file:

```java
package io.quarkmind.domain;

public record Unit(String tag, UnitType type, Point2d position,
                   int health, int maxHealth,
                   int shields, int maxShields,
                   int weaponCooldownTicks) {}
```

- [ ] **Step 4: Find all broken constructor call sites**

```bash
mvn compile -q 2>&1 | grep "error:" | grep -o '"[^"]*\.java":[0-9]*' | sort -u
```

This will list every file and line that needs `, 0` (or the appropriate cooldown value) appended.

- [ ] **Step 5: Fix all `new Unit(...)` call sites — append `, 0` (or preserve existing field)**

**Rule:** Internal/test constructions that create new units → append `, 0`. Constructions that copy an existing unit's fields → preserve `u.weaponCooldownTicks()`.

Apply these changes:

**`EmulatedGame.java` — every `new Unit(...)` call:**

*reset() probes (line ~94):*
```java
myUnits.add(new Unit("probe-" + i, UnitType.PROBE,
    new Point2d(9 + i * 0.5f, 9),
    SC2Data.maxHealth(UnitType.PROBE), SC2Data.maxHealth(UnitType.PROBE),
    SC2Data.maxShields(UnitType.PROBE), SC2Data.maxShields(UnitType.PROBE), 0));
```

*moveFriendlyUnits (line ~149) — preserve:*
```java
return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                u.shields(), u.maxShields(), 0);
```
Note: internal units always carry `0` — cooldown is tracked in `unitCooldowns` map, stamped at snapshot time only.

*moveEnemyUnits (line ~174) — preserve:*
```java
return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth(),
                u.shields(), u.maxShields(), 0);
```

*spawnEnemyWaves (line ~196):*
```java
enemyUnits.add(new Unit(tag, type, pos, hp, hp,
    SC2Data.maxShields(type), SC2Data.maxShields(type), 0));
```

*tickEnemyStrategy staging (line ~273):*
```java
enemyStagingArea.add(new Unit(tag, step.unitType(),
    STAGING_POS, hp, hp,
    SC2Data.maxShields(step.unitType()), SC2Data.maxShields(step.unitType()), 0));
```

*handleTrain (line ~342):*
```java
myUnits.add(new Unit(tag, t.unitType(), new Point2d(9, 9), hp, hp,
    SC2Data.maxShields(t.unitType()), SC2Data.maxShields(t.unitType()), 0));
```

*applyDamage (line ~468) — preserve:*
```java
return new Unit(u.tag(), u.type(), u.position(), hpLeft, u.maxHealth(),
                shieldsLeft, u.maxShields(), 0);
```

*spawnEnemyForTesting (line ~556):*
```java
enemyUnits.add(new Unit(tag, type, position, hp, hp,
    SC2Data.maxShields(type), SC2Data.maxShields(type), 0));
```

*setHealthForTesting (line ~564):*
```java
? new Unit(u.tag(), u.type(), u.position(), health, u.maxHealth(), u.shields(), u.maxShields(), 0)
```

*setShieldsForTesting (line ~571):*
```java
? new Unit(u.tag(), u.type(), u.position(), u.health(), u.maxHealth(), shields, u.maxShields(), 0)
```

*spawnFriendlyForTesting (line ~579):*
```java
myUnits.add(new Unit(tag, type, position, hp, hp,
    SC2Data.maxShields(type), SC2Data.maxShields(type), 0));
```

*setEnemyShieldsForTesting (line ~587):*
```java
? new Unit(u.tag(), u.type(), u.position(), u.health(), u.maxHealth(),
           shields, u.maxShields(), 0)
```

*setEnemyHealthForTesting (line ~601):*
```java
? new Unit(u.tag(), u.type(), u.position(), health, u.maxHealth(),
           u.shields(), u.maxShields(), 0)
```

*addStagedUnitForTesting (line ~617):*
```java
enemyStagingArea.add(new Unit(tag, type, position, hp, hp,
    SC2Data.maxShields(type), SC2Data.maxShields(type), 0));
```

**`SimulatedGame.java` — all `new Unit(...)` calls:** append `, 0` to each.

**`ReplaySimulatedGame.java` — all `new Unit(...)` calls:** append `, 0` to each.

**`ObservationTranslator.java` — map SC2 weapon_cooldown:**
```java
private static Unit toUnit(UnitInPool uip) {
    var u = uip.unit();
    var pos = u.getPosition();
    int cooldownTicks = u.getWeaponCooldown()
        .map(f -> (int)(f / 0.5f))   // SC2 seconds → ticks at 500ms/tick
        .orElse(0);
    return new Unit(
        String.valueOf(u.getTag().getValue()),
        mapUnitType(toUnitsEnum(uip)),
        new Point2d(pos.getX(), pos.getY()),
        u.getHealth().map(Float::intValue).orElse(0),
        u.getHealthMax().map(Float::intValue).orElse(0),
        u.getShield().map(Float::intValue).orElse(0),
        u.getShieldMax().map(Float::intValue).orElse(0),
        cooldownTicks
    );
}
```

**All test files — append `, 0` to every `new Unit(...)` call:**

Files to update (34 call sites across these files — use `mvn compile` output to find each):
- `BasicEconomicsTaskTest.java`, `BasicTacticsTaskTest.java`, `BasicScoutingTaskTest.java`
- `BasicStrategyTaskTest.java`, `ScoutingSessionManagerTest.java`
- `DroolsTacticsTaskIT.java`, `DroolsScoutingTaskIT.java`
- `DroolsStrategyTaskTest.java`, `EconomicsDecisionServiceTest.java`, `EconomicsFlowTest.java`
- `DroolsTacticsRuleUnitTest.java`, `GameStateBroadcasterTest.java`
- `GameStateTranslatorTest.java`, `VisibilityGridTest.java`, `DamageCalculatorTest.java`
- `GameStateTest.java`, `EnemyStrategyTest.java`, `DroolsScoutingTaskTest.java`

Also update helper methods in these test files that construct `Unit`:
- `DroolsTacticsTaskIT.stalker(...)` → append `, 0`
- `DroolsTacticsTaskIT.enemy(...)` → append `, 0`
- `DroolsTacticsRuleUnitTest.stalker(...)` → append `, 0`
- `DroolsTacticsRuleUnitTest.enemy()` → append `, 0`

- [ ] **Step 6: Stamp cooldown in `EmulatedGame.snapshot()`**

In `snapshot()`, replace `List.copyOf(myUnits)` (the friendly units line) with a stream that reads `unitCooldowns`:

```java
List<Unit> friendlyWithCooldown = myUnits.stream()
    .map(u -> new Unit(u.tag(), u.type(), u.position(),
                       u.health(), u.maxHealth(), u.shields(), u.maxShields(),
                       unitCooldowns.getOrDefault(u.tag(), 0)))
    .toList();
```

Then use `friendlyWithCooldown` in both branches of `snapshot()` (terrain-filtered and plain):

```java
// terrain-filtered branch:
return new GameState(
    (int) mineralAccumulator,
    vespene, supply, supplyUsed,
    friendlyWithCooldown, List.copyOf(myBuildings),
    visibleEnemies, visibleStaging,
    List.copyOf(geysers), gameFrame);

// plain branch:
return new GameState(
    (int) mineralAccumulator,
    vespene, supply, supplyUsed,
    friendlyWithCooldown, List.copyOf(myBuildings),
    List.copyOf(enemyUnits), List.copyOf(enemyStagingArea),
    List.copyOf(geysers), gameFrame);
```

- [ ] **Step 7: Run the two new tests — verify they pass**

```bash
mvn test -Dtest="EmulatedGameTest#firingUnit_hasCooldownInSnapshot+freshUnit_hasCooldownZeroInSnapshot" -q
```

Expected: `BUILD SUCCESS`, 2 tests pass.

- [ ] **Step 8: Run full test suite — verify no regressions**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/domain/Unit.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java \
        src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java \
        src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java \
        src/test/java/  # all updated test files

git commit -m "feat(e10): add weaponCooldownTicks to Unit record; stamp from EmulatedGame snapshot

Matches SC2 protobuf weapon_cooldown field. SimulatedGame/ReplaySimulatedGame always 0.
ObservationTranslator maps SC2 seconds → ticks at 500ms/tick.

Refs #UNIT-REC, #EPIC"
```

---

## Task 2: Per-unit attack range fix in `computeInRangeTags`

**Files:**
- Create: `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java`
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`

- [ ] **Step 1: Create `DroolsTacticsTaskTest.java` with failing range tests**

```java
package io.quarkmind.plugin;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.Unit;
import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class DroolsTacticsTaskTest {

    // ---- Helpers ----

    private static Unit unit(String tag, UnitType type, Point2d pos) {
        return new Unit(tag, type, pos, 80, 80, 80, 80, 0);
    }

    private static Unit unit(String tag, UnitType type, Point2d pos, int cooldown) {
        return new Unit(tag, type, pos, 80, 80, 80, 80, cooldown);
    }

    private static Unit enemy(Point2d pos) {
        return new Unit("e-0", UnitType.ZEALOT, pos, 100, 100, 50, 50, 0);
    }

    // ---- computeInRangeTags: per-unit range ----

    @Test
    void stalkerAt4_5_isInRange() {
        // Stalker range = 5.0; distance 4.5 → in range
        Unit s = unit("s-0", UnitType.STALKER, new Point2d(10, 10));
        Unit e = enemy(new Point2d(14.5f, 10)); // distance exactly 4.5
        Set<String> result = DroolsTacticsTask.computeInRangeTags(List.of(s), List.of(e));
        assertThat(result).contains("s-0");
    }

    @Test
    void stalkerAt5_5_isNotInRange() {
        // Stalker range = 5.0; distance 5.5 → out of range
        Unit s = unit("s-0", UnitType.STALKER, new Point2d(10, 10));
        Unit e = enemy(new Point2d(15.5f, 10)); // distance exactly 5.5
        Set<String> result = DroolsTacticsTask.computeInRangeTags(List.of(s), List.of(e));
        assertThat(result).doesNotContain("s-0");
    }

    @Test
    void zealotAt0_4_isInRange() {
        // Zealot range = 0.5; distance 0.4 → in range
        Unit z = unit("z-0", UnitType.ZEALOT, new Point2d(10, 10));
        Unit e = enemy(new Point2d(10.4f, 10));
        Set<String> result = DroolsTacticsTask.computeInRangeTags(List.of(z), List.of(e));
        assertThat(result).contains("z-0");
    }

    @Test
    void zealotAt0_6_isNotInRange() {
        // Zealot range = 0.5; distance 0.6 → out of range
        Unit z = unit("z-0", UnitType.ZEALOT, new Point2d(10, 10));
        Unit e = enemy(new Point2d(10.6f, 10));
        Set<String> result = DroolsTacticsTask.computeInRangeTags(List.of(z), List.of(e));
        assertThat(result).doesNotContain("z-0");
    }
}
```

- [ ] **Step 2: Run — verify compile failure (method is private)**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q 2>&1 | grep -E "error:|has private access"
```

Expected: compile error — `computeInRangeTags` has private access.

- [ ] **Step 3: Make `computeInRangeTags` and `distance` static package-private; fix range**

In `DroolsTacticsTask.java`:

Delete the constant:
```java
// DELETE this line:
static final Point2d MAP_CENTER   = new Point2d(64, 64);
static final double  STALKER_RANGE = 6.0;  // ← DELETE THIS LINE ONLY
```

Change `computeInRangeTags` from `private` to `static` (package-private):
```java
static Set<String> computeInRangeTags(List<Unit> army, List<Unit> enemies) {
    Set<String> result = new HashSet<>();
    for (Unit unit : army) {
        for (Unit enemy : enemies) {
            if (distance(unit.position(), enemy.position()) <= SC2Data.attackRange(unit.type())) {
                result.add(unit.tag());
                break;
            }
        }
    }
    return result;
}
```

Change `distance` from `private` to `static` (package-private):
```java
static double distance(Point2d a, Point2d b) {
    double dx = a.x() - b.x();
    double dy = a.y() - b.y();
    return Math.sqrt(dx * dx + dy * dy);
}
```

Update call site in `execute()`:
```java
Set<String> inRangeSet = computeInRangeTags(army, enemies);
```

- [ ] **Step 4: Run range tests — verify they pass**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q
```

Expected: 4 tests pass, `BUILD SUCCESS`.

- [ ] **Step 5: Run full suite — verify no regressions**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
        src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java

git commit -m "fix(e10): use SC2Data.attackRange(unit.type()) in computeInRangeTags

Removes hardcoded STALKER_RANGE=6.0 (wrong; Stalker range is 5.0).
Per-unit range now correct for mixed-type armies.

Closes #RANGE, Refs #EPIC"
```

---

## Task 3: Focus-fire — `selectFocusTarget`

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java` (add tests)
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` (add method)

- [ ] **Step 1: Add failing focus-fire unit tests to `DroolsTacticsTaskTest`**

Add inside the class after the range tests:

```java
// ---- selectFocusTarget ----

@Test
void selectFocusTarget_returnsLowestCombinedHpAndShields() {
    Unit strong = new Unit("e-strong", UnitType.ZEALOT, new Point2d(10,10), 100, 100, 50, 50, 0); // 150
    Unit weak   = new Unit("e-weak",   UnitType.ZEALOT, new Point2d(10,10),  20, 100,  0, 50, 0); //  20
    Unit mid    = new Unit("e-mid",    UnitType.ZEALOT, new Point2d(10,10),  60, 100, 30, 50, 0); //  90
    var result = DroolsTacticsTask.selectFocusTarget(List.of(strong, weak, mid));
    assertThat(result).isPresent();
    assertThat(result.get().tag()).isEqualTo("e-weak");
}

@Test
void selectFocusTarget_shieldsCountTowardTotal() {
    // e-1: 80 hp + 0 shields = 80 total
    // e-2: 10 hp + 50 shields = 60 total → lower, should be chosen
    Unit e1 = new Unit("e-1", UnitType.ZEALOT, new Point2d(10,10), 80, 100,  0, 50, 0);
    Unit e2 = new Unit("e-2", UnitType.ZEALOT, new Point2d(10,10), 10, 100, 50, 50, 0);
    var result = DroolsTacticsTask.selectFocusTarget(List.of(e1, e2));
    assertThat(result).isPresent();
    assertThat(result.get().tag()).isEqualTo("e-2");
}

@Test
void selectFocusTarget_emptyList_returnsEmpty() {
    assertThat(DroolsTacticsTask.selectFocusTarget(List.of())).isEmpty();
}
```

- [ ] **Step 2: Run — verify compile failure**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q 2>&1 | grep "error:"
```

Expected: `selectFocusTarget` does not exist.

- [ ] **Step 3: Add `selectFocusTarget` to `DroolsTacticsTask`**

Add the following imports at the top of `DroolsTacticsTask.java`:
```java
import java.util.Comparator;
import java.util.Optional;
import java.util.stream.Collectors;
```

Add the static method (package-private):
```java
static Optional<Unit> selectFocusTarget(List<Unit> enemies) {
    return enemies.stream()
        .min(Comparator.comparingInt(e -> e.health() + e.shields()));
}
```

- [ ] **Step 4: Run focus-fire tests — verify they pass**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q
```

Expected: 7 tests pass (4 range + 3 focus), `BUILD SUCCESS`.

- [ ] **Step 5: Run full suite**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
        src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java

git commit -m "feat(e10): add selectFocusTarget — pick lowest-HP enemy for focus-fire

Not yet wired into dispatch; pure-Java static method with unit tests.

Refs #FOCUS, #EPIC"
```

---

## Task 4: Drools kiting classification — `onCooldownTags` + DRL rules

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java`
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl`
- Modify: `src/test/java/io/quarkmind/plugin/DroolsTacticsRuleUnitTest.java`

- [ ] **Step 1: Add failing kiting Drools tests to `DroolsTacticsRuleUnitTest`**

Add new helper methods and tests inside the class:

```java
// ---- Phase 1: Kiting classification ----

@Test
void inRangeUnitOnCooldown_classifiedAsKiting() {
    TacticsRuleUnit data = attackWithCooldown(
        List.of(stalker("s-0", 80, 80, 3)),   // healthy, on cooldown
        List.of(enemy()),
        List.of("s-0"),                         // in range
        List.of("s-0")                          // on cooldown
    );
    fire(data);
    assertThat(data.getGroupDecisions())
        .anyMatch(g -> g.startsWith("kiting:KITING:s-0"));
}

@Test
void inRangeUnitOffCooldown_classifiedAsInRange_notKiting() {
    TacticsRuleUnit data = attackWithCooldown(
        List.of(stalker("s-0", 80, 80, 0)),   // healthy, off cooldown
        List.of(enemy()),
        List.of("s-0"),                         // in range
        List.of()                               // NOT on cooldown
    );
    fire(data);
    assertThat(data.getGroupDecisions())
        .anyMatch(g -> g.startsWith("in-range:ENEMY_ELIMINATED:s-0"))
        .noneMatch(g -> g.startsWith("kiting:"));
}

@Test
void lowHealthUnitOnCooldown_classifiedAsLowHealth_notKiting() {
    // Low-health guard (< 30%) takes priority over kiting classification
    TacticsRuleUnit data = attackWithCooldown(
        List.of(stalker("s-0", 20, 100, 3)),  // low health + on cooldown
        List.of(enemy()),
        List.of("s-0"),                         // in range
        List.of("s-0")                          // on cooldown
    );
    fire(data);
    assertThat(data.getGroupDecisions())
        .anyMatch(g -> g.startsWith("low-health:UNIT_SAFE:s-0"))
        .noneMatch(g -> g.startsWith("kiting:"));
}

// ---- Phase 2: Kite action emission ----

@Test
void kitingGroupEmitsKiteAction() {
    TacticsRuleUnit data = attackWithCooldown(
        List.of(stalker("s-0", 80, 80, 3)),
        List.of(enemy()),
        List.of("s-0"),
        List.of("s-0")
    );
    fire(data);
    assertThat(data.getActionDecisions()).contains("KITE:1");
}

// ---- Helpers ----

private TacticsRuleUnit attackWithCooldown(List<Unit> army, List<Unit> enemies,
                                            List<String> inRangeTags,
                                            List<String> onCooldownTags) {
    TacticsRuleUnit data = new TacticsRuleUnit();
    data.setStrategyGoal("ATTACK");
    army.forEach(data.getArmy()::add);
    enemies.forEach(data.getEnemies()::add);
    inRangeTags.forEach(data.getInRangeTags()::add);
    onCooldownTags.forEach(data.getOnCooldownTags()::add);
    return data;
}

// Overload with cooldown parameter
private Unit stalker(String tag, int health, int maxHealth, int cooldown) {
    return new Unit(tag, UnitType.STALKER, new Point2d(10, 10), health, maxHealth, 80, 80, cooldown);
}
```

- [ ] **Step 2: Run — verify failure**

```bash
mvn test -Dtest="DroolsTacticsRuleUnitTest" -q 2>&1 | grep "error:"
```

Expected: `getOnCooldownTags()` does not exist on `TacticsRuleUnit`.

- [ ] **Step 3: Add `onCooldownTags` to `TacticsRuleUnit`**

In `TacticsRuleUnit.java`, add field and getter after `inRangeTags`:

```java
private final DataStore<String> onCooldownTags  = DataSource.createStore();
```

Add getter after `getInRangeTags()`:

```java
public DataStore<String> getOnCooldownTags()    { return onCooldownTags; }
```

- [ ] **Step 4: Update `StarCraftTactics.drl` — tighten in-range rule, add kiting**

Replace the entire DRL file with:

```
package io.quarkmind.plugin.drools;
unit TacticsRuleUnit;

// ---------------------------------------------------------------------------
// Phase 1 — Group Classification (salience 200+)
//
// Groups are mutually exclusive:
//   low-health (< 30% hp) → UNIT_SAFE goal
//   kiting (healthy + in-range + on cooldown) → KITING goal
//   in-range (healthy + in-range + off cooldown) → ENEMY_ELIMINATED goal
//   out-of-range (healthy + not in-range) → ENEMY_ELIMINATED goal
//
// Low-health guard (health/maxHealth >= 0.3) on kiting and in-range rules
// ensures a low-health unit is never also classified as kiting or in-range.
// ---------------------------------------------------------------------------

rule "Group: low health"
    salience 210
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() < 0.3 ]
then
    groupDecisions.add("low-health:UNIT_SAFE:" + $u.tag());
    activeGroups.add("low-health");
end

rule "Group: kiting"
    salience 200
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() >= 0.3 ]
    /inRangeTags[ this == $u.tag() ]
    /onCooldownTags[ this == $u.tag() ]
then
    groupDecisions.add("kiting:KITING:" + $u.tag());
    activeGroups.add("kiting");
end

rule "Group: in range"
    salience 200
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() >= 0.3 ]
    /inRangeTags[ this == $u.tag() ]
    not /onCooldownTags[ this == $u.tag() ]
then
    groupDecisions.add("in-range:ENEMY_ELIMINATED:" + $u.tag());
    activeGroups.add("in-range");
end

rule "Group: out of range"
    salience 200
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() >= 0.3 ]
    not /inRangeTags[ this == $u.tag() ]
    /enemies
then
    groupDecisions.add("out-of-range:ENEMY_ELIMINATED:" + $u.tag());
    activeGroups.add("out-of-range");
end

// ---------------------------------------------------------------------------
// Phase 2 — Action Emission (salience 100+)
// ---------------------------------------------------------------------------

rule "Action: Retreat available"
    salience 110
when
    /activeGroups[ this == "low-health" ]
then
    actionDecisions.add("RETREAT:1");
end

rule "Action: Kite available"
    salience 100
when
    /activeGroups[ this == "kiting" ]
then
    actionDecisions.add("KITE:1");
end

rule "Action: Attack available"
    salience 100
when
    /activeGroups[ this == "in-range" ]
then
    actionDecisions.add("ATTACK:2");
end

rule "Action: MoveToEngage available"
    salience 100
when
    /activeGroups[ this == "out-of-range" ]
then
    actionDecisions.add("MOVE_TO_ENGAGE:1");
end
```

- [ ] **Step 5: Run kiting Drools tests — verify they pass**

```bash
mvn test -Dtest="DroolsTacticsRuleUnitTest" -q
```

Expected: all tests pass (existing + 4 new), `BUILD SUCCESS`.

- [ ] **Step 6: Run full suite**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java \
        src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl \
        src/test/java/io/quarkmind/plugin/DroolsTacticsRuleUnitTest.java

git commit -m "feat(e10): add kiting Drools classification — onCooldownTags + kiting group rules

TacticsRuleUnit gains onCooldownTags DataStore. DRL: kiting group fires when
healthy + in-range + on-cooldown. In-range rule tightened with not onCooldown guard.
Phase 2 emits KITE:1 action for kiting group.

Refs #KITE, #EPIC"
```

---

## Task 5: Kite geometry — `computeOnCooldownTags` and `kiteRetreatTarget`

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java` (add tests)
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` (add methods)

- [ ] **Step 1: Add failing kite geometry tests to `DroolsTacticsTaskTest`**

Add inside the class after the focus-fire tests:

```java
// ---- computeOnCooldownTags ----

@Test
void computeOnCooldownTags_includesOnlyUnitsWithCooldown() {
    Unit ready  = unit("s-ready",  UnitType.STALKER, new Point2d(10, 10), 0);
    Unit kiting = unit("s-kiting", UnitType.STALKER, new Point2d(10, 10), 3);
    Set<String> result = DroolsTacticsTask.computeOnCooldownTags(List.of(ready, kiting));
    assertThat(result).containsOnly("s-kiting");
}

@Test
void computeOnCooldownTags_allReady_returnsEmpty() {
    Unit r0 = unit("s-0", UnitType.STALKER, new Point2d(10,10), 0);
    Unit r1 = unit("s-1", UnitType.STALKER, new Point2d(10,10), 0);
    assertThat(DroolsTacticsTask.computeOnCooldownTags(List.of(r0, r1))).isEmpty();
}

// ---- kiteRetreatTarget ----

@Test
void kiteRetreatTarget_movesAwayOnYAxis() {
    // Unit at (10,10), enemy at (10,15): retreat direction is (0,-1), step 1.0 → (10,9)
    Point2d unitPos = new Point2d(10, 10);
    Unit e = enemy(new Point2d(10, 15));
    Point2d retreat = DroolsTacticsTask.kiteRetreatTarget(unitPos, List.of(e));
    assertThat(retreat.x()).isCloseTo(10f, within(0.01f));
    assertThat(retreat.y()).isCloseTo(9f,  within(0.01f));
}

@Test
void kiteRetreatTarget_stepLengthEqualsKiteStep_diagonal() {
    // Any direction: step vector must have magnitude == KITE_STEP
    Point2d unitPos = new Point2d(10, 10);
    Unit e = enemy(new Point2d(13, 14)); // diagonal enemy
    Point2d retreat = DroolsTacticsTask.kiteRetreatTarget(unitPos, List.of(e));
    double stepLen = Math.sqrt(
        Math.pow(retreat.x() - unitPos.x(), 2) +
        Math.pow(retreat.y() - unitPos.y(), 2));
    assertThat(stepLen).isCloseTo(DroolsTacticsTask.KITE_STEP, within(0.01));
}

@Test
void kiteRetreatTarget_degenerate_unitOnEnemy_returnsUnitPos() {
    Point2d unitPos = new Point2d(10, 10);
    Unit e = enemy(new Point2d(10, 10)); // exactly overlapping
    Point2d retreat = DroolsTacticsTask.kiteRetreatTarget(unitPos, List.of(e));
    assertThat(retreat).isEqualTo(unitPos);
}

@Test
void kiteRetreatTarget_usesNearestEnemy() {
    // Retreat should be away from the closest enemy, not the farther one
    Point2d unitPos = new Point2d(10, 10);
    Unit near = enemy(new Point2d(10, 12)); // 2 tiles away
    Unit far  = enemy(new Point2d(10, 20)); // 10 tiles away
    Point2d retreat = DroolsTacticsTask.kiteRetreatTarget(unitPos, List.of(near, far));
    // Away from near (10,12): direction (0,-1), step 1.0 → (10,9)
    assertThat(retreat.y()).isCloseTo(9f, within(0.01f));
}
```

- [ ] **Step 2: Run — verify failure**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q 2>&1 | grep "error:"
```

Expected: `computeOnCooldownTags` and `kiteRetreatTarget` do not exist.

- [ ] **Step 3: Add both methods to `DroolsTacticsTask`**

Add the constant and two static methods (package-private):

```java
static final double KITE_STEP = 1.0; // tiles per kite step

static Set<String> computeOnCooldownTags(List<Unit> army) {
    return army.stream()
        .filter(u -> u.weaponCooldownTicks() > 0)
        .map(Unit::tag)
        .collect(Collectors.toSet());
}

static Point2d kiteRetreatTarget(Point2d unitPos, List<Unit> enemies) {
    Unit nearest = enemies.stream()
        .min(Comparator.comparingDouble(e -> distance(unitPos, e.position())))
        .orElseThrow();
    double dx = unitPos.x() - nearest.position().x();
    double dy = unitPos.y() - nearest.position().y();
    double len = Math.sqrt(dx * dx + dy * dy);
    if (len < 0.001) return unitPos; // degenerate: unit on top of enemy
    return new Point2d(
        (float)(unitPos.x() + dx / len * KITE_STEP),
        (float)(unitPos.y() + dy / len * KITE_STEP));
}
```

- [ ] **Step 4: Run kite geometry tests — verify they pass**

```bash
mvn test -Dtest="DroolsTacticsTaskTest" -q
```

Expected: all 13 tests pass, `BUILD SUCCESS`.

- [ ] **Step 5: Run full suite**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
        src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java

git commit -m "feat(e10): add computeOnCooldownTags and kiteRetreatTarget with unit tests

kiteRetreatTarget steps KITE_STEP=1.0 tile away from nearest enemy (normalised).
Degenerate case (unit on enemy) returns current position unchanged.

Refs #KITE, #EPIC"
```

---

## Task 6: Wire GOAP KITE action + updated `execute()` and `dispatch()`

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`

- [ ] **Step 1: Add `KITE` to `ACTION_TEMPLATES`**

In `DroolsTacticsTask.java`, update `ACTION_TEMPLATES`:

```java
private static final Map<String, GoapAction> ACTION_TEMPLATES = Map.of(
    "RETREAT",        new GoapAction("RETREAT",
        Map.of("lowHealth", true),
        Map.of("unitSafe", true), 1),
    "ATTACK",         new GoapAction("ATTACK",
        Map.of("inRange", true, "enemyVisible", true),
        Map.of("enemyEliminated", true), 2),
    "MOVE_TO_ENGAGE", new GoapAction("MOVE_TO_ENGAGE",
        Map.of("enemyVisible", true, "inRange", false),
        Map.of("inRange", true), 1),
    "KITE",           new GoapAction("KITE",
        Map.of("inRange", true, "onCooldown", true, "enemyVisible", true),
        Map.of("onCooldown", false), 1)
);
```

- [ ] **Step 2: Add `"kiting"` case to `buildWorldState`**

In `buildWorldState`, add after the `"out-of-range"` case:

```java
case "kiting" -> new WorldState(Map.of(
    "lowHealth",       false,
    "enemyVisible",    true,
    "inRange",         true,
    "onCooldown",      true,
    "unitSafe",        false,
    "enemyEliminated", false))
```

- [ ] **Step 3: Add `"KITING"` to `goalConditionKey`**

```java
private String goalConditionKey(String goalName) {
    return switch (goalName) {
        case "UNIT_SAFE"        -> "unitSafe";
        case "ENEMY_ELIMINATED" -> "enemyEliminated";
        case "KITING"           -> "unitSafe";
        default                 -> goalName.toLowerCase();
    };
}
```

- [ ] **Step 4: Update `buildRuleUnit` to populate `onCooldownTags`**

Change method signature and body:

```java
private TacticsRuleUnit buildRuleUnit(List<Unit> army, List<Unit> enemies,
                                       Set<String> inRangeTags, Set<String> onCooldownTags,
                                       String strategy) {
    TacticsRuleUnit data = new TacticsRuleUnit();
    data.setStrategyGoal(strategy);
    army.forEach(data.getArmy()::add);
    enemies.forEach(data.getEnemies()::add);
    inRangeTags.forEach(data.getInRangeTags()::add);
    onCooldownTags.forEach(data.getOnCooldownTags()::add);
    return data;
}
```

- [ ] **Step 5: Update `execute()` to compute onCooldownTags and focusTarget**

In `execute()`, replace:
```java
Set<String> inRangeSet = computeInRangeTags(army, enemies);
TacticsRuleUnit data = buildRuleUnit(army, enemies, inRangeSet, strategy);
```

With:
```java
Set<String> inRangeSet    = computeInRangeTags(army, enemies);
Set<String> onCooldownSet = computeOnCooldownTags(army);
TacticsRuleUnit data = buildRuleUnit(army, enemies, inRangeSet, onCooldownSet, strategy);
```

And replace the dispatch call:
```java
dispatch(plan.get(0), groupInfo.unitTags(), threat, bld);
```

With (add `Optional<Unit> focusTarget = selectFocusTarget(enemies);` before the groups loop):
```java
Optional<Unit> focusTarget = selectFocusTarget(enemies);
// ... inside the loop:
dispatch(plan.get(0), groupInfo.unitTags(), army, enemies, threat, bld, focusTarget);
```

- [ ] **Step 6: Replace `dispatch()` with updated signature and KITE case**

```java
private void dispatch(GoapAction action, List<String> unitTags,
                      List<Unit> army, List<Unit> enemies,
                      Point2d threat, List<Building> buildings,
                      Optional<Unit> focusTarget) {
    switch (action.name()) {
        case "ATTACK" -> {
            Point2d target = focusTarget
                .map(Unit::position)
                .orElse(threat != null ? threat : MAP_CENTER);
            unitTags.forEach(tag -> intentQueue.add(new AttackIntent(tag, target)));
        }
        case "MOVE_TO_ENGAGE" -> {
            Point2d target = threat != null ? threat : MAP_CENTER;
            unitTags.forEach(tag -> intentQueue.add(new MoveIntent(tag, target)));
        }
        case "RETREAT" -> {
            Point2d rally = buildings.stream()
                .filter(b -> b.type() == BuildingType.NEXUS)
                .findFirst()
                .map(Building::position)
                .orElse(MAP_CENTER);
            unitTags.forEach(tag -> intentQueue.add(new MoveIntent(tag, rally)));
        }
        case "KITE" -> {
            unitTags.forEach(tag ->
                army.stream()
                    .filter(u -> u.tag().equals(tag))
                    .findFirst()
                    .ifPresent(unit -> {
                        Point2d retreatTarget = kiteRetreatTarget(unit.position(), enemies);
                        intentQueue.add(new MoveIntent(tag, retreatTarget));
                    })
            );
        }
    }
}
```

- [ ] **Step 7: Run full suite — verify no regressions**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java

git commit -m "feat(e10): wire GOAP KITE action, focus-fire, and onCooldownTags into execute()

KITE GoapAction added to ACTION_TEMPLATES. buildWorldState gains kiting case.
dispatch() uses focusTarget for ATTACK, kiteRetreatTarget for KITE.
execute() computes onCooldownTags and focusTarget before Drools fires.

Refs #FOCUS, #KITE, #EPIC"
```

---

## Task 7: Integration tests — `DroolsTacticsTaskIT`

**Files:**
- Modify: `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskIT.java`

- [ ] **Step 1: Add 5-arg `stalker` overload and `distance` helper**

In the helpers section of `DroolsTacticsTaskIT`:

```java
private Unit stalker(String tag, Point2d pos, int health, int maxHealth, int cooldown) {
    return new Unit(tag, UnitType.STALKER, pos, health, maxHealth, 80, 80, cooldown);
}

private double distance(Point2d a, Point2d b) {
    return Math.sqrt(Math.pow(a.x() - b.x(), 2) + Math.pow(a.y() - b.y(), 2));
}
```

Also update existing `enemy` helper to include cooldown 0:
```java
private Unit enemy(Point2d pos) {
    return new Unit("e-0", UnitType.ZEALOT, pos, 100, 100, 50, 50, 0);
}
```

- [ ] **Step 2: Add focus-fire integration test**

```java
@Test
void focusFire_allInRangeUnitsTargetWeakestEnemy() {
    // Arrange: two Stalkers in range, two enemies at different HP
    Unit weakEnemy   = new Unit("e-weak",   UnitType.ZEALOT, new Point2d(14, 10), 10, 100,  0, 50, 0);
    Unit strongEnemy = new Unit("e-strong", UnitType.ZEALOT, new Point2d(14, 11), 100, 100, 50, 50, 0);
    var cf = caseFile("ATTACK",
        List.of(stalker("s-0", new Point2d(10, 10), 80, 80, 0),
                stalker("s-1", new Point2d(10, 10), 80, 80, 0)),
        List.of(weakEnemy, strongEnemy),
        weakEnemy.position());
    // Act
    tacticsTask.execute(cf);
    // Assert: both Stalkers attack the weak enemy's position
    assertThat(intentQueue.pending()).hasSize(2)
        .allMatch(i -> i instanceof AttackIntent ai &&
                       ai.targetLocation().equals(weakEnemy.position()));
}
```

- [ ] **Step 3: Run this test — verify it passes**

```bash
mvn test -Dtest="DroolsTacticsTaskIT#focusFire_allInRangeUnitsTargetWeakestEnemy" -q
```

Expected: PASS.

- [ ] **Step 4: Add kiting integration test — on-cooldown unit gets MoveIntent backward**

```java
@Test
void kitingUnit_onCooldown_getsMoveIntentAwayFromEnemy() {
    Point2d stalkerPos = new Point2d(10, 10);
    Point2d enemyPos   = new Point2d(10, 14); // enemy is "above" (higher y)
    var cf = caseFile("ATTACK",
        List.of(stalker("s-kite", stalkerPos, 80, 80, 3)), // cooldown = 3
        List.of(new Unit("e-0", UnitType.ZEALOT, enemyPos, 100, 100, 50, 50, 0)),
        enemyPos);
    tacticsTask.execute(cf);
    assertThat(intentQueue.pending()).hasSize(1);
    assertThat(intentQueue.pending().get(0)).isInstanceOf(MoveIntent.class);
    MoveIntent move = (MoveIntent) intentQueue.pending().get(0);
    // Retreat target must be farther from enemy than current position
    assertThat(distance(move.targetLocation(), enemyPos))
        .isGreaterThan(distance(stalkerPos, enemyPos));
}
```

- [ ] **Step 5: Run this test — verify it passes**

```bash
mvn test -Dtest="DroolsTacticsTaskIT#kitingUnit_onCooldown_getsMoveIntentAwayFromEnemy" -q
```

Expected: PASS.

- [ ] **Step 6: Add mixed-cooldown integration test**

```java
@Test
void mixedCooldown_correctIntentPerUnit() {
    // s-ready: off cooldown → in-range → AttackIntent
    // s-kiting: on cooldown → kiting → MoveIntent backward
    Point2d enemyPos = new Point2d(14, 10);
    var cf = caseFile("ATTACK",
        List.of(stalker("s-ready",  new Point2d(10, 10), 80, 80, 0),
                stalker("s-kiting", new Point2d(10, 10), 80, 80, 3)),
        List.of(new Unit("e-0", UnitType.ZEALOT, enemyPos, 100, 100, 50, 50, 0)),
        enemyPos);
    tacticsTask.execute(cf);
    assertThat(intentQueue.pending()).hasSize(2);
    assertThat(intentQueue.pending())
        .anyMatch(i -> i instanceof AttackIntent ai && ai.unitTag().equals("s-ready"));
    assertThat(intentQueue.pending())
        .anyMatch(i -> i instanceof MoveIntent mi && mi.unitTag().equals("s-kiting"));
}
```

- [ ] **Step 7: Add happy path — full attack cycle focus-fires weakest**

```java
@Test
void happyPath_attackCycle_focusFiresWeakestEnemy() {
    // Three Stalkers vs two enemies; all in range, all off cooldown
    // All three should target the weakest enemy
    Unit weak   = new Unit("e-weak",   UnitType.ZEALOT, new Point2d(14, 10),  5, 100,  0, 50, 0);
    Unit strong = new Unit("e-strong", UnitType.ZEALOT, new Point2d(14, 11), 100, 100, 50, 50, 0);
    var cf = caseFile("ATTACK",
        List.of(stalker("s-0", new Point2d(10, 10), 80, 80, 0),
                stalker("s-1", new Point2d(10, 10), 80, 80, 0),
                stalker("s-2", new Point2d(10, 10), 80, 80, 0)),
        List.of(weak, strong),
        weak.position());
    tacticsTask.execute(cf);
    long focusCount = intentQueue.pending().stream()
        .filter(i -> i instanceof AttackIntent ai &&
                     ai.targetLocation().equals(weak.position()))
        .count();
    assertThat(focusCount).isEqualTo(3);
}
```

- [ ] **Step 8: Run all new IT tests**

```bash
mvn test -Dtest="DroolsTacticsTaskIT" -q
```

Expected: all tests pass (existing + 4 new), `BUILD SUCCESS`.

- [ ] **Step 9: Run full suite**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 10: Commit**

```bash
git add src/test/java/io/quarkmind/plugin/DroolsTacticsTaskIT.java

git commit -m "test(e10): integration tests for focus-fire, kiting, and mixed cooldown

Verifies full Drools→GOAP→dispatch pipeline for all three E10 features.
Happy path confirms three Stalkers all target weakest enemy.

Refs #FOCUS, #KITE, #EPIC"
```

---

## Task 8: E2E physics tests — `EmulatedGameTest`

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

These tests run the real EmulatedGame physics loop and verify that correct intents produce measurably better combat outcomes — not just correct intent emission.

- [ ] **Step 1: Add focus-fire E2E test**

Add inside `EmulatedGameTest` (new `// E10` section):

```java
// ---- E10: Focus-fire physics ----

@Test
void focusFire_eliminatesFirstEnemyBeforeSpreadFire() {
    // Focus-fire: both Stalkers attack the same Zealot → it dies faster
    // Spread-fire: each Stalker attacks a different Zealot
    // Assert: focus-fire kills first enemy in fewer ticks

    int focusTicks = runFocusFireScenario(true);
    int spreadTicks = runFocusFireScenario(false);
    assertThat(focusTicks).isLessThan(spreadTicks);
}

private int runFocusFireScenario(boolean focusFire) {
    game.reset();
    String s0 = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(10, 10));
    String s1 = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(10, 11));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(12, 10));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(12, 11));
    int initialEnemyCount = game.snapshot().enemyUnits().size();

    for (int tick = 1; tick <= 60; tick++) {
        List<Unit> enemies = game.snapshot().enemyUnits();
        if (enemies.isEmpty()) return tick;
        if (focusFire) {
            // Both Stalkers target same (first) enemy
            game.applyIntent(new AttackIntent(s0, enemies.get(0).position()));
            game.applyIntent(new AttackIntent(s1, enemies.get(0).position()));
        } else {
            // Each Stalker targets a different enemy (spread fire)
            game.applyIntent(new AttackIntent(s0, enemies.get(0).position()));
            game.applyIntent(new AttackIntent(s1, enemies.size() > 1
                ? enemies.get(1).position() : enemies.get(0).position()));
        }
        game.tick();
        if (game.snapshot().enemyUnits().size() < initialEnemyCount) return tick;
    }
    return 60;
}
```

- [ ] **Step 2: Run focus-fire E2E test — verify it passes**

```bash
mvn test -Dtest="EmulatedGameTest#focusFire_eliminatesFirstEnemyBeforeSpreadFire" -q
```

Expected: PASS.

- [ ] **Step 3: Add kiting E2E test**

```java
// ---- E10: Kiting physics ----

@Test
void kiting_stallsEnemyContact_stalkerRetainsMoreHp() {
    // Kiting Stalker reads weaponCooldownTicks from snapshot and retreats during cooldown.
    // Standing Stalker moves toward enemy every tick.
    // After 25 ticks, kiting Stalker should have more combined HP+shields.

    int standingHp = runStandingScenario();
    int kitingHp   = runKitingScenario();
    assertThat(kitingHp).isGreaterThan(standingHp);
}

private int runStandingScenario() {
    game.reset();
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(8, 12));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(8, 16)); // approaching Nexus at (8,8)
    for (int tick = 0; tick < 25; tick++) {
        List<Unit> enemies = game.snapshot().enemyUnits();
        if (!enemies.isEmpty()) {
            // Standing: always move toward (and attack) enemy position
            game.applyIntent(new AttackIntent(tag, enemies.get(0).position()));
        }
        game.tick();
    }
    return game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals(tag))
        .mapToInt(u -> u.health() + u.shields()).findFirst().orElse(0);
}

private int runKitingScenario() {
    game.reset();
    String tag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(8, 12));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(8, 16));
    for (int tick = 0; tick < 25; tick++) {
        GameState state = game.snapshot();
        Unit stalker = state.myUnits().stream()
            .filter(u -> u.tag().equals(tag)).findFirst().orElse(null);
        List<Unit> enemies = state.enemyUnits();
        if (stalker == null || enemies.isEmpty()) break;
        if (stalker.weaponCooldownTicks() > 0) {
            // On cooldown: kite backward (away from nearest enemy)
            Unit nearest = enemies.stream()
                .min(Comparator.comparingDouble(e ->
                    Math.sqrt(Math.pow(e.position().x() - stalker.position().x(), 2) +
                              Math.pow(e.position().y() - stalker.position().y(), 2))))
                .orElseThrow();
            double dx = stalker.position().x() - nearest.position().x();
            double dy = stalker.position().y() - nearest.position().y();
            double len = Math.sqrt(dx * dx + dy * dy);
            if (len > 0.001) {
                Point2d retreat = new Point2d(
                    (float)(stalker.position().x() + dx / len),
                    (float)(stalker.position().y() + dy / len));
                game.applyIntent(new MoveIntent(tag, retreat));
            }
        } else {
            // Off cooldown: attack nearest enemy
            game.applyIntent(new AttackIntent(tag, enemies.get(0).position()));
        }
        game.tick();
    }
    return game.snapshot().myUnits().stream()
        .filter(u -> u.tag().equals(tag))
        .mapToInt(u -> u.health() + u.shields()).findFirst().orElse(0);
}
```

Add `import java.util.Comparator;` to the test file imports if not already present.

- [ ] **Step 4: Run kiting E2E test — verify it passes**

```bash
mvn test -Dtest="EmulatedGameTest#kiting_stallsEnemyContact_stalkerRetainsMoreHp" -q
```

Expected: PASS.

- [ ] **Step 5: Run full E10 test suite**

```bash
mvn test -Dtest="EmulatedGameTest,DroolsTacticsTaskTest,DroolsTacticsRuleUnitTest,DroolsTacticsTaskIT" -q
```

Expected: all tests pass, `BUILD SUCCESS`.

- [ ] **Step 6: Run complete suite**

```bash
mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7: Commit**

```bash
git add src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java

git commit -m "test(e10): E2E physics tests — focus-fire kills faster, kiting retains HP

focusFire_eliminatesFirstEnemyBeforeSpreadFire: focus-fire always kills first
enemy sooner. kiting_stallsEnemyContact_stalkerRetainsMoreHp: kiting Stalker
avoids Zealot melee contact, retaining more HP after 25 ticks.

Closes #FOCUS, Closes #KITE, Closes #UNIT-REC, Refs #EPIC"
```

- [ ] **Step 8: Close the epic**

```bash
gh issue close #EPIC --comment "All child issues closed. E10 complete: weaponCooldownTicks, per-unit range, focus-fire, kiting."
```
