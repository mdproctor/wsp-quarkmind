# E5: Damage Types, Armour & Hardened Shield — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace raw `damagePerAttack` with a SC2-accurate damage model that applies armour reduction, attribute-based bonus damage, and the Immortal's Hardened Shield — all behind a testable `DamageCalculator` class — and correct wrong HP values in `SC2Data`.

**Architecture:** `UnitAttribute` enum in `domain/` provides the attribute vocabulary. Four new `SC2Data` methods wire attributes, armour, bonus damage, and Hardened Shield to unit types. `DamageCalculator` (package-private, `sc2/emulated/`) computes effective-damage-per-hit. `EmulatedGame.resolveCombat()` swaps one line to call the calculator instead of raw damage.

**Tech Stack:** Pure Java records/enums, JUnit 5, AssertJ, Maven Surefire.

---

## File Map

| Action | Path | Purpose |
|---|---|---|
| Create | `src/main/java/io/quarkmind/domain/UnitAttribute.java` | Attribute vocabulary enum |
| Modify | `src/main/java/io/quarkmind/domain/SC2Data.java` | 4 new methods + HP corrections |
| Create | `src/main/java/io/quarkmind/sc2/emulated/DamageCalculator.java` | Per-hit damage computation |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | `resolveCombat` + 2 test helpers |
| Create | `src/test/java/io/quarkmind/domain/SC2DataTest.java` | All SC2Data assertions |
| Create | `src/test/java/io/quarkmind/sc2/emulated/DamageCalculatorTest.java` | All damage formula assertions |
| Modify | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | New E5 tests + 2 broken E4 test fixes |

---

## Task 1: Create GitHub Epic and Issues

**Files:** none (GitHub only)

- [ ] **Step 1: Create the E5 milestone**

```bash
MILESTONE=$(gh api repos/mdproctor/quarkmind/milestones \
  -f title="E5: Damage Types, Armour & Hardened Shield" \
  -f state=open \
  --jq '.number')
echo "Milestone: $MILESTONE"
```

- [ ] **Step 2: Create the five child issues**

```bash
gh issue create --repo mdproctor/quarkmind \
  --title "E5-1: UnitAttribute enum in domain/" \
  --body "Add UnitAttribute enum (LIGHT, ARMORED, BIOLOGICAL, MECHANICAL, PSIONIC, MASSIVE, STRUCTURE) to domain/. Pure Java, no framework deps." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E5-2: SC2Data — unitAttributes, armour, bonusDamageVs, hasHardenedShield + HP corrections" \
  --body "Four new static methods on SC2Data. Fix maxHealth for Immortal(200), Marine(45), Marauder(125), Roach(145), Hydralisk(90)." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E5-3: DamageCalculator in sc2/emulated/" \
  --body "Package-private class. computeEffective(UnitType, Unit) applies armour, bonus damage vs attributes, and Immortal Hardened Shield cap." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E5-4: EmulatedGame.resolveCombat() — use DamageCalculator" \
  --body "Replace raw damagePerAttack with damageCalculator.computeEffective(attacker.type(), target) in both combat loops. Add spawnFriendlyForTesting and setEnemyShieldsForTesting helpers." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E5-5: Tests — SC2DataTest, DamageCalculatorTest, EmulatedGameTest additions" \
  --body "New test classes for SC2Data and DamageCalculator. New EmulatedGameTest tests for armour, Hardened Shield, stat corrections. Fix 2 existing tests broken by armour introduction." \
  --milestone "$MILESTONE"
```

- [ ] **Step 3: Note the issue numbers**

Record the five issue numbers — you'll need them for commit messages (`Refs #N`). The issues were created in order so they'll be sequential.

---

## Task 2: UnitAttribute Enum

**Files:**
- Create: `src/main/java/io/quarkmind/domain/UnitAttribute.java`

This enum is a pure vocabulary type — no logic, no test needed. SC2DataTest (Task 3) will be the first to use it.

- [ ] **Step 1: Create the enum**

```java
// src/main/java/io/quarkmind/domain/UnitAttribute.java
package io.quarkmind.domain;

public enum UnitAttribute {
    LIGHT, ARMORED, BIOLOGICAL, MECHANICAL, PSIONIC, MASSIVE, STRUCTURE
}
```

- [ ] **Step 2: Verify it compiles**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/domain/UnitAttribute.java
git commit -m "feat(e5): UnitAttribute enum in domain/

Refs #<E5-1 issue number>"
```

---

## Task 3: SC2Data — New Methods and HP Corrections

**Files:**
- Create: `src/test/java/io/quarkmind/domain/SC2DataTest.java`
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

- [ ] **Step 1: Write the failing test class**

```java
// src/test/java/io/quarkmind/domain/SC2DataTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static io.quarkmind.domain.UnitAttribute.*;

class SC2DataTest {

    // ---- unitAttributes ----

    @Test
    void zealotAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.ZEALOT))
            .containsExactlyInAnyOrder(LIGHT, BIOLOGICAL);
    }

    @Test
    void stalkerAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.STALKER))
            .containsExactlyInAnyOrder(ARMORED, MECHANICAL);
    }

    @Test
    void immortalAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.IMMORTAL))
            .containsExactlyInAnyOrder(ARMORED, MECHANICAL, MASSIVE);
    }

    @Test
    void marineAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.MARINE))
            .containsExactlyInAnyOrder(LIGHT, BIOLOGICAL);
    }

    @Test
    void roachAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.ROACH))
            .containsExactlyInAnyOrder(ARMORED, BIOLOGICAL);
    }

    @Test
    void hydralisksAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.HYDRALISK))
            .containsExactlyInAnyOrder(LIGHT, BIOLOGICAL);
    }

    @Test
    void probeAttributes() {
        assertThat(SC2Data.unitAttributes(UnitType.PROBE))
            .containsExactlyInAnyOrder(LIGHT, MECHANICAL);
    }

    // ---- hasHardenedShield ----

    @Test
    void immortalHasHardenedShield() {
        assertThat(SC2Data.hasHardenedShield(UnitType.IMMORTAL)).isTrue();
    }

    @Test
    void stalkerHasNoHardenedShield() {
        assertThat(SC2Data.hasHardenedShield(UnitType.STALKER)).isFalse();
    }

    @Test
    void zealotHasNoHardenedShield() {
        assertThat(SC2Data.hasHardenedShield(UnitType.ZEALOT)).isFalse();
    }

    // ---- armour ----

    @Test
    void stalkerArmour() {
        assertThat(SC2Data.armour(UnitType.STALKER)).isEqualTo(1);
    }

    @Test
    void zealotArmour() {
        assertThat(SC2Data.armour(UnitType.ZEALOT)).isEqualTo(1);
    }

    @Test
    void immortalArmour() {
        assertThat(SC2Data.armour(UnitType.IMMORTAL)).isEqualTo(1);
    }

    @Test
    void marauderArmour() {
        assertThat(SC2Data.armour(UnitType.MARAUDER)).isEqualTo(1);
    }

    @Test
    void roachArmour() {
        assertThat(SC2Data.armour(UnitType.ROACH)).isEqualTo(1);
    }

    @Test
    void marineArmour() {
        assertThat(SC2Data.armour(UnitType.MARINE)).isEqualTo(0);
    }

    @Test
    void probeArmour() {
        assertThat(SC2Data.armour(UnitType.PROBE)).isEqualTo(0);
    }

    @Test
    void hydralisksArmour() {
        assertThat(SC2Data.armour(UnitType.HYDRALISK)).isEqualTo(0);
    }

    // ---- bonusDamageVs ----

    @Test
    void stalkerBonusVsArmored() {
        assertThat(SC2Data.bonusDamageVs(UnitType.STALKER, ARMORED)).isEqualTo(4);
    }

    @Test
    void stalkerBonusVsLight() {
        assertThat(SC2Data.bonusDamageVs(UnitType.STALKER, LIGHT)).isEqualTo(0);
    }

    @Test
    void marauderBonusVsArmored() {
        assertThat(SC2Data.bonusDamageVs(UnitType.MARAUDER, ARMORED)).isEqualTo(10);
    }

    @Test
    void immortalBonusVsArmored() {
        assertThat(SC2Data.bonusDamageVs(UnitType.IMMORTAL, ARMORED)).isEqualTo(3);
    }

    @Test
    void probeBonusVsArmored() {
        assertThat(SC2Data.bonusDamageVs(UnitType.PROBE, ARMORED)).isEqualTo(0);
    }

    @Test
    void zealotBonusVsArmored() {
        assertThat(SC2Data.bonusDamageVs(UnitType.ZEALOT, ARMORED)).isEqualTo(0);
    }

    // ---- corrected HP values ----

    @Test
    void correctedHp_immortal() {
        assertThat(SC2Data.maxHealth(UnitType.IMMORTAL)).isEqualTo(200);
    }

    @Test
    void correctedHp_marine() {
        assertThat(SC2Data.maxHealth(UnitType.MARINE)).isEqualTo(45);
    }

    @Test
    void correctedHp_marauder() {
        assertThat(SC2Data.maxHealth(UnitType.MARAUDER)).isEqualTo(125);
    }

    @Test
    void correctedHp_roach() {
        assertThat(SC2Data.maxHealth(UnitType.ROACH)).isEqualTo(145);
    }

    @Test
    void correctedHp_hydralisk() {
        assertThat(SC2Data.maxHealth(UnitType.HYDRALISK)).isEqualTo(90);
    }
}
```

- [ ] **Step 2: Run — confirm all tests fail**

```bash
mvn test -Dtest=SC2DataTest -q
```

Expected: FAIL — methods `unitAttributes`, `armour`, `bonusDamageVs`, `hasHardenedShield` don't exist yet; HP values wrong.

- [ ] **Step 3: Implement in SC2Data.java**

Add `import java.util.Set;` and `import static io.quarkmind.domain.UnitAttribute.*;` to the top of `SC2Data.java`.

Add these four methods and update `maxHealth`:

```java
// ---- E5: unit attributes ----

public static Set<UnitAttribute> unitAttributes(UnitType type) {
    return switch (type) {
        case PROBE     -> Set.of(LIGHT, MECHANICAL);
        case ZEALOT    -> Set.of(LIGHT, BIOLOGICAL);
        case STALKER   -> Set.of(ARMORED, MECHANICAL);
        case IMMORTAL  -> Set.of(ARMORED, MECHANICAL, MASSIVE);
        case OBSERVER  -> Set.of(ARMORED, MECHANICAL);
        case MARINE    -> Set.of(LIGHT, BIOLOGICAL);
        case MARAUDER  -> Set.of(BIOLOGICAL, ARMORED);
        case ROACH     -> Set.of(ARMORED, BIOLOGICAL);
        case HYDRALISK -> Set.of(LIGHT, BIOLOGICAL);
        default        -> Set.of();
    };
}

public static int armour(UnitType type) {
    return switch (type) {
        case ZEALOT, STALKER, IMMORTAL, MARAUDER, ROACH -> 1;
        default -> 0;  // Probe, Marine, Hydralisk, Observer
    };
}

public static int bonusDamageVs(UnitType attackerType, UnitAttribute targetAttribute) {
    return switch (attackerType) {
        case STALKER  -> targetAttribute == ARMORED ? 4  : 0;
        case IMMORTAL -> targetAttribute == ARMORED ? 3  : 0;
        case MARAUDER -> targetAttribute == ARMORED ? 10 : 0;
        default       -> 0;
    };
}

public static boolean hasHardenedShield(UnitType type) {
    return type == UnitType.IMMORTAL;
}
```

Update `maxHealth` — replace the existing switch with:

```java
public static int maxHealth(UnitType type) {
    return switch (type) {
        case PROBE     ->  45;
        case ZEALOT    -> 100;
        case STALKER   ->  80;
        case IMMORTAL  -> 200;
        case MARINE    ->  45;
        case MARAUDER  -> 125;
        case ROACH     -> 145;
        case HYDRALISK ->  90;
        default        -> 100;
    };
}
```

- [ ] **Step 4: Run — confirm all tests pass**

```bash
mvn test -Dtest=SC2DataTest -q
```

Expected: BUILD SUCCESS, 29 tests passing.

- [ ] **Step 5: Run full suite — confirm no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS (the maxHealth changes only affect units that were previously defaulting to 100 — no existing test asserts exact HP values for those unit types).

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/test/java/io/quarkmind/domain/SC2DataTest.java
git commit -m "feat(e5): SC2Data — unitAttributes, armour, bonusDamageVs, hasHardenedShield + HP corrections

Refs #<E5-2 issue number>"
```

---

## Task 4: DamageCalculator

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/emulated/DamageCalculatorTest.java`
- Create: `src/main/java/io/quarkmind/sc2/emulated/DamageCalculator.java`

Both are in package `io.quarkmind.sc2.emulated` — the test accesses the package-private class directly.

- [ ] **Step 1: Write the failing test class**

```java
// src/test/java/io/quarkmind/sc2/emulated/DamageCalculatorTest.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DamageCalculatorTest {

    private final DamageCalculator calc = new DamageCalculator();

    /** Helper: unit of given type at full HP and shields. */
    private Unit full(UnitType type) {
        int hp = SC2Data.maxHealth(type);
        int sh = SC2Data.maxShields(type);
        return new Unit("t", type, new Point2d(0, 0), hp, hp, sh, sh);
    }

    /** Helper: unit with custom shield value (HP at max). */
    private Unit withShields(UnitType type, int shields) {
        int hp = SC2Data.maxHealth(type);
        return new Unit("t", type, new Point2d(0, 0), hp, hp, shields, SC2Data.maxShields(type));
    }

    // ---- no bonus damage ----

    @Test
    void stalkerVsMarine() {
        // 13 + 0 (no bonus vs Light) − 0 (Marine armour) = 13
        assertThat(calc.computeEffective(UnitType.STALKER, full(UnitType.MARINE))).isEqualTo(13);
    }

    @Test
    void zealotVsMarine() {
        // 8 + 0 − 0 = 8
        assertThat(calc.computeEffective(UnitType.ZEALOT, full(UnitType.MARINE))).isEqualTo(8);
    }

    @Test
    void marauderVsMarine() {
        // 10 + 0 (no bonus vs Light) − 0 = 10
        assertThat(calc.computeEffective(UnitType.MARAUDER, full(UnitType.MARINE))).isEqualTo(10);
    }

    // ---- armour reduction ----

    @Test
    void zealotVsStalker() {
        // 8 + 0 (no bonus vs Armored) − 1 (Stalker armour) = 7
        assertThat(calc.computeEffective(UnitType.ZEALOT, full(UnitType.STALKER))).isEqualTo(7);
    }

    @Test
    void zealotVsZealot() {
        // 8 + 0 − 1 (Zealot armour) = 7
        assertThat(calc.computeEffective(UnitType.ZEALOT, full(UnitType.ZEALOT))).isEqualTo(7);
    }

    // ---- bonus damage vs Armored ----

    @Test
    void stalkerVsMarauder() {
        // 13 + 4 (vs Armored) − 1 (Marauder armour) = 16
        assertThat(calc.computeEffective(UnitType.STALKER, full(UnitType.MARAUDER))).isEqualTo(16);
    }

    @Test
    void stalkerVsRoach() {
        // 13 + 4 (vs Armored) − 1 (Roach armour) = 16
        assertThat(calc.computeEffective(UnitType.STALKER, full(UnitType.ROACH))).isEqualTo(16);
    }

    @Test
    void marauderVsStalker() {
        // 10 + 10 (vs Armored) − 1 (Stalker armour) = 19
        assertThat(calc.computeEffective(UnitType.MARAUDER, full(UnitType.STALKER))).isEqualTo(19);
    }

    @Test
    void immortalVsRoach() {
        // 20 + 3 (vs Armored) − 1 (Roach armour) = 22
        assertThat(calc.computeEffective(UnitType.IMMORTAL, full(UnitType.ROACH))).isEqualTo(22);
    }

    // ---- Hardened Shield (Immortal with shields > 0) ----

    @Test
    void hardenedShield_largeHit() {
        // Stalker vs shielded Immortal: 13+4−1=16, Hardened Shield caps to 10
        assertThat(calc.computeEffective(UnitType.STALKER, full(UnitType.IMMORTAL))).isEqualTo(10);
    }

    @Test
    void hardenedShield_smallHit() {
        // Probe vs shielded Immortal: 5+0−1=4, under the 10 cap — no clamping
        assertThat(calc.computeEffective(UnitType.PROBE, full(UnitType.IMMORTAL))).isEqualTo(4);
    }

    @Test
    void hardenedShield_unshielded_takesFullDamage() {
        // Stalker vs Immortal with 0 shields: 13+4−1=16, no Hardened Shield active
        assertThat(calc.computeEffective(UnitType.STALKER, withShields(UnitType.IMMORTAL, 0)))
            .isEqualTo(16);
    }

    @Test
    void hardenedShield_doesNotActivateOnNonImmortal() {
        // Zealot has no Hardened Shield — no cap even if it did
        assertThat(calc.computeEffective(UnitType.STALKER, full(UnitType.ZEALOT))).isEqualTo(12);
        // 13 + 0 − 1 = 12
    }
}
```

- [ ] **Step 2: Run — confirm all tests fail**

```bash
mvn test -Dtest=DamageCalculatorTest -q
```

Expected: FAIL — `DamageCalculator` class does not exist.

- [ ] **Step 3: Implement DamageCalculator**

```java
// src/main/java/io/quarkmind/sc2/emulated/DamageCalculator.java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.Unit;
import io.quarkmind.domain.UnitType;

/**
 * Computes effective damage per hit for the emulated combat engine.
 *
 * <p>Formula: {@code max(1, rawDamage + bonusVsAttributes − armour)},
 * then capped at 10 if the target is an Immortal with shields remaining
 * (Hardened Shield passive).
 */
class DamageCalculator {

    int computeEffective(UnitType attackerType, Unit target) {
        int raw = SC2Data.damagePerAttack(attackerType)
                + bonusVsTarget(attackerType, target.type());

        int afterArmour = Math.max(1, raw - SC2Data.armour(target.type()));

        if (SC2Data.hasHardenedShield(target.type()) && target.shields() > 0) {
            return Math.min(10, afterArmour);
        }
        return afterArmour;
    }

    private int bonusVsTarget(UnitType attackerType, UnitType targetType) {
        return SC2Data.unitAttributes(targetType).stream()
            .mapToInt(attr -> SC2Data.bonusDamageVs(attackerType, attr))
            .max()
            .orElse(0);
    }
}
```

- [ ] **Step 4: Run — confirm all tests pass**

```bash
mvn test -Dtest=DamageCalculatorTest -q
```

Expected: BUILD SUCCESS, 13 tests passing.

- [ ] **Step 5: Run full suite — confirm no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/DamageCalculator.java \
        src/test/java/io/quarkmind/sc2/emulated/DamageCalculatorTest.java
git commit -m "feat(e5): DamageCalculator — armour, attribute bonuses, Hardened Shield

Refs #<E5-3 issue number>"
```

---

## Task 5: Wire DamageCalculator into EmulatedGame + Tests

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

This task has three sub-steps: add test helpers, write failing tests, wire `resolveCombat`, fix two pre-existing tests that break under the new model.

- [ ] **Step 1: Add test helpers to EmulatedGame**

Add two package-private helpers to `EmulatedGame` (alongside the existing `spawnEnemyForTesting`, `setHealthForTesting`, `setShieldsForTesting`):

```java
/** Spawns a friendly unit at a specific position for combat tests. Returns the unit's tag. */
String spawnFriendlyForTesting(UnitType type, Point2d position) {
    String tag = "test-unit-" + nextTag++;
    int hp = SC2Data.maxHealth(type);
    myUnits.add(new Unit(tag, type, position, hp, hp,
        SC2Data.maxShields(type), SC2Data.maxShields(type)));
    return tag;
}

/** Sets an enemy unit's shields directly — for testing Hardened Shield at 0 shields. */
void setEnemyShieldsForTesting(String tag, int shields) {
    enemyUnits.replaceAll(u -> u.tag().equals(tag)
        ? new Unit(u.tag(), u.type(), u.position(), u.health(), u.maxHealth(),
                   shields, u.maxShields())
        : u);
}
```

- [ ] **Step 2: Write the new E5 tests (they will fail until Step 4)**

Add this section to the end of `EmulatedGameTest`:

```java
// ---- E5: damage types, armour, Hardened Shield ----

@Test
void stalkerDealsCorrectDamageVsArmored() {
    // Stalker (friendly) attacks Roach (Armored, 1 armour):
    // effective = 13 + 4 (vs Armored) − 1 = 16, not raw 13
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(3, 3));
    game.spawnEnemyForTesting(UnitType.ROACH, new Point2d(3, 5)); // distance=2.0 ≤ Stalker range 5.0
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(3, 5)));

    game.tick();

    Unit roach = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ROACH)
        .findFirst().orElseThrow();
    assertThat(roach.health()).isEqualTo(SC2Data.maxHealth(UnitType.ROACH) - 16); // 145 − 16 = 129
}

@Test
void armourReducesIncomingDamage() {
    // Stalker (friendly) attacks Zealot (LIGHT, 1 armour):
    // effective = 13 + 0 (no bonus vs Light) − 1 = 12, not raw 13
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(3, 3));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(3, 5)); // distance=2.0
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(3, 5)));

    game.tick();

    Unit zealot = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    // Without armour: shields = 50 − 13 = 37. With armour: 50 − 12 = 38.
    assertThat(zealot.shields()).isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - 12); // 38
}

@Test
void immortalShieldedCapsDamageAt10() {
    // Stalker (friendly) attacks shielded Immortal:
    // effective before cap = 13+4−1=16, Hardened Shield → min(10,16) = 10
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(3, 3));
    game.spawnEnemyForTesting(UnitType.IMMORTAL, new Point2d(3, 5)); // within Stalker range
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(3, 5)));

    game.tick();

    Unit immortal = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.IMMORTAL)
        .findFirst().orElseThrow();
    assertThat(immortal.shields()).isEqualTo(SC2Data.maxShields(UnitType.IMMORTAL) - 10); // 90
    assertThat(immortal.health()).isEqualTo(SC2Data.maxHealth(UnitType.IMMORTAL));          // 200 untouched
}

@Test
void immortalUnshieldedTakesFullDamage() {
    // Stalker vs Immortal with 0 shields: no Hardened Shield → full 16 damage to HP
    String stalkerTag = game.spawnFriendlyForTesting(UnitType.STALKER, new Point2d(3, 3));
    game.spawnEnemyForTesting(UnitType.IMMORTAL, new Point2d(3, 5));
    String immortalTag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyShieldsForTesting(immortalTag, 0);
    game.applyIntent(new AttackIntent(stalkerTag, new Point2d(3, 5)));

    game.tick();

    Unit immortal = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.IMMORTAL)
        .findFirst().orElseThrow();
    assertThat(immortal.health()).isEqualTo(SC2Data.maxHealth(UnitType.IMMORTAL) - 16); // 184
}

@Test
void spawnedMarineHasCorrectHp() {
    game.spawnEnemyForTesting(UnitType.MARINE, new Point2d(50, 50)); // far from Probes — no combat
    Unit marine = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.MARINE)
        .findFirst().orElseThrow();
    assertThat(marine.health()).isEqualTo(45);
}

@Test
void spawnedImmortalHasCorrectHp() {
    game.spawnEnemyForTesting(UnitType.IMMORTAL, new Point2d(50, 50));
    Unit immortal = game.snapshot().enemyUnits().stream()
        .filter(u -> u.type() == UnitType.IMMORTAL)
        .findFirst().orElseThrow();
    assertThat(immortal.health()).isEqualTo(200);
}
```

- [ ] **Step 3: Run new tests — confirm they fail**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: FAIL on all 6 new tests (calculator not wired yet). Existing tests pass.

- [ ] **Step 4: Wire DamageCalculator into EmulatedGame.resolveCombat()**

Add one field at the top of the class (after the existing field declarations):

```java
private final DamageCalculator damageCalculator = new DamageCalculator();
```

In `resolveCombat()`, locate the two `pending.merge(...)` calls (inside the friendly-attacks-enemy loop and enemy-attacks-friendly loop) and replace raw damage with `computeEffective`:

**Before (friendly loop):**
```java
nearestInRange(attacker.position(), enemyUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(), SC2Data.damagePerAttack(attacker.type()), Integer::sum);
        firedFriendly.add(attacker.tag());
    });
```

**After (friendly loop):**
```java
nearestInRange(attacker.position(), enemyUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(),
            damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        firedFriendly.add(attacker.tag());
    });
```

**Before (enemy loop):**
```java
nearestInRange(attacker.position(), myUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(), SC2Data.damagePerAttack(attacker.type()), Integer::sum);
        firedEnemy.add(attacker.tag());
    });
```

**After (enemy loop):**
```java
nearestInRange(attacker.position(), myUnits, SC2Data.attackRange(attacker.type()))
    .ifPresent(target -> {
        pending.merge(target.tag(),
            damageCalculator.computeEffective(attacker.type(), target), Integer::sum);
        firedEnemy.add(attacker.tag());
    });
```

- [ ] **Step 5: Run — new E5 tests should pass; find which existing tests break**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: 6 new E5 tests pass. Two existing E4 tests now fail:
- `firstAttackFiresImmediately` — asserts `shields == 50 − 5 = 45`; actual is `50 − 4 = 46` (Probe vs Zealot, 1 armour)
- `cooldownExpiresAndAttackFiresAgain` — asserts `shields == 50 − 2*5 = 40`; actual is `50 − 2*4 = 42`

- [ ] **Step 6: Fix the two broken tests**

In `EmulatedGameTest`, update `firstAttackFiresImmediately`:

```java
@Test
void firstAttackFiresImmediately() {
    // Initial cooldown = 0 (absent from map) — attack fires on first tick.
    // Probe vs Zealot (1 armour): effective = 5 − 1 = 4. Shields: 50 → 46.
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick();

    int effective = SC2Data.damagePerAttack(UnitType.PROBE) - SC2Data.armour(UnitType.ZEALOT); // 4
    Unit zealot = game.snapshot().enemyUnits().get(0);
    assertThat(zealot.shields())
        .isLessThan(SC2Data.maxShields(UnitType.ZEALOT))
        .isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - effective); // 46
}
```

Update `cooldownExpiresAndAttackFiresAgain`:

```java
@Test
void cooldownExpiresAndAttackFiresAgain() {
    // PROBE cooldown = 2: fires tick 1, skips tick 2, fires tick 3.
    // Probe vs Zealot (1 armour): effective = 5 − 1 = 4 per hit. Shields: 50 → 46 → 42.
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));

    game.tick(); // tick 1: attack (shields 50 → 46)
    game.tick(); // tick 2: cooldown 1 — no attack
    game.tick(); // tick 3: cooldown 0 — attack fires again (46 → 42)

    int effective = SC2Data.damagePerAttack(UnitType.PROBE) - SC2Data.armour(UnitType.ZEALOT); // 4
    Unit zealot = game.snapshot().enemyUnits().get(0);
    assertThat(zealot.shields())
        .isEqualTo(SC2Data.maxShields(UnitType.ZEALOT) - 2 * effective); // 42
}
```

- [ ] **Step 7: Run EmulatedGameTest — all tests pass**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 8: Run full suite — no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS. If any non-EmulatedGameTest tests fail, investigate before committing.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e5): wire DamageCalculator into resolveCombat + full test coverage

Fix firstAttackFiresImmediately and cooldownExpiresAndAttackFiresAgain to account
for armour reduction (Probe vs Zealot: effective 4, not raw 5).

Closes #<E5-4 issue number>
Closes #<E5-5 issue number>"
```

---

## Self-Check Before Calling Done

Run the full test suite one final time:

```bash
mvn test
```

Verify:
- `SC2DataTest`: all 35 tests pass
- `DamageCalculatorTest`: all 14 tests pass
- `EmulatedGameTest`: all tests pass (including the 6 new E5 tests and the 2 fixed E4 tests)
- All other test classes: unchanged, still passing

Then confirm all five GitHub issues are linked in commits and close the milestone once the final commit is pushed.
