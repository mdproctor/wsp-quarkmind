# E11: Strategy Provider Pattern + Tactics Extensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce CDI-based `KiteStrategy` / `FocusFireStrategy` strategy interfaces, extract existing behaviours into named implementations, then deliver terrain-aware kiting, multi-target focus-fire, a Stalker timing fix, and (conditionally) Stalker Blink micro.

**Architecture:** `@Named` CDI beans implement each strategy interface; `DroolsTacticsTask` selects by name via `@ConfigProperty` + `Instance<T>.select(NamedLiteral.of(...))` in `@PostConstruct`. Terrain is routed through a `TerrainProvider` `@ApplicationScoped` bean rather than through `GameState`, keeping the domain record unchanged. Existing static methods in `DroolsTacticsTask` (`kiteRetreatTarget`, `selectFocusTarget`) are extracted into `@Named("direct")` / `@Named("lowest-hp")` implementations first — no behaviour change until config is switched.

**Tech stack:** Jakarta CDI (`Instance<T>`, `NamedLiteral`, `@PostConstruct`, `@Named`), Quarkus `@ConfigProperty`, Drools Rule Units, GOAP planner (`plugin/tactics/`)

---

## File Map

### New
- `src/main/java/io/quarkmind/plugin/tactics/KiteStrategy.java`
- `src/main/java/io/quarkmind/plugin/tactics/FocusFireStrategy.java`
- `src/main/java/io/quarkmind/plugin/tactics/DirectKiteStrategy.java`
- `src/main/java/io/quarkmind/plugin/tactics/TerrainAwareKiteStrategy.java`
- `src/main/java/io/quarkmind/plugin/tactics/LowestHpFocusFireStrategy.java`
- `src/main/java/io/quarkmind/plugin/tactics/OverkillRedirectFocusFireStrategy.java`
- `src/main/java/io/quarkmind/sc2/TerrainProvider.java`
- `src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java` *(Task 8, conditional)*
- `src/test/java/io/quarkmind/plugin/tactics/KiteStrategyTest.java`
- `src/test/java/io/quarkmind/plugin/tactics/FocusFireStrategyTest.java`

### Modified
- `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` — inject strategies, CDI wiring, remove extracted statics, update dispatch
- `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — inject `TerrainProvider`; blink state (Task 7)
- `src/main/java/io/quarkmind/domain/SC2Data.java` — STALKER cooldown 3→1; blink constants (Task 7)
- `src/main/java/io/quarkmind/domain/Unit.java` — add `blinkCooldownTicks` (Task 7)
- `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java` — `blinkReadyTags`, `shieldsLowTags` (Task 8)
- `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl` — BLINKING group + BLINK action (Task 8)
- `src/main/resources/application.properties` — strategy config defaults + emulated profile override
- `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java` — remove tests now covered by strategy tests
- `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` — new physics tests; Unit migration (Task 7)
- All other files constructing `Unit` with 8 args — migration for `blinkCooldownTicks` (Task 7, compiler-driven)

---

## Task 0: Create GitHub Issues

- [ ] **Step 1: Create E11 epic and sub-issues**

```bash
EPIC=$(gh issue create \
  --title "E11: Strategy Provider Pattern + Tactics Extensions" \
  --label "epic" \
  --body "CDI strategy provider pattern + terrain-aware kiting + multi-target focus-fire + timing fix + Blink (conditional). Spec: docs/superpowers/specs/2026-04-17-e11-strategy-pattern-tactics-extensions-design.md" \
  --json number -q .number)

E11_1=$(gh issue create --title "E11-1: KiteStrategy + FocusFireStrategy CDI strategy pattern" --label "enhancement" --body "Refs #$EPIC" --json number -q .number)
E11_2=$(gh issue create --title "E11-2: TerrainProvider CDI bean" --label "enhancement" --body "Refs #$EPIC" --json number -q .number)
E11_3=$(gh issue create --title "E11-3: Fix Stalker attack cooldown 3→1 tick" --label "bug" --body "Refs #$EPIC" --json number -q .number)
E11_4=$(gh issue create --title "E11-4: TerrainAwareKiteStrategy" --label "enhancement" --body "Refs #$EPIC" --json number -q .number)
E11_5=$(gh issue create --title "E11-5: OverkillRedirectFocusFireStrategy" --label "enhancement" --body "Refs #$EPIC" --json number -q .number)
E11_6=$(gh issue create --title "E11-6: Stalker Blink micro (conditional)" --label "enhancement" --body "Refs #$EPIC" --json number -q .number)

echo "Epic: $EPIC  Sub: $E11_1 $E11_2 $E11_3 $E11_4 $E11_5 $E11_6"
```

Note all issue numbers — used in commit messages below.

---

## Task 1: Strategy interfaces + extract defaults + CDI wiring

Refactoring only — no behaviour change. Existing tests verify correctness.

**Files:** KiteStrategy.java, FocusFireStrategy.java, DirectKiteStrategy.java, LowestHpFocusFireStrategy.java, KiteStrategyTest.java, FocusFireStrategyTest.java, DroolsTacticsTask.java, application.properties

- [ ] **Step 1: Write failing tests for DirectKiteStrategy**

Create `src/test/java/io/quarkmind/plugin/tactics/KiteStrategyTest.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.*;
import org.assertj.core.data.Offset;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class KiteStrategyTest {

    private static Unit unit(String tag, Point2d pos) {
        return new Unit(tag, UnitType.STALKER, pos, 80, 80, 80, 80, 0);
    }
    private static double dist(Point2d a, Point2d b) {
        double dx = a.x() - b.x(), dy = a.y() - b.y();
        return Math.sqrt(dx * dx + dy * dy);
    }

    @Test void direct_retreatsAwayFromEnemy() {
        Point2d result = new DirectKiteStrategy().retreatTarget(
            unit("u-0", new Point2d(5, 5)), List.of(unit("e-0", new Point2d(7, 5))), null);
        assertThat(result.x()).isLessThan(5f);
        assertThat(result.y()).isCloseTo(5f, Offset.offset(0.01f));
    }

    @Test void direct_stepSizeIsOnePoint0() {
        Point2d from = new Point2d(5, 5);
        Point2d result = new DirectKiteStrategy().retreatTarget(
            unit("u-0", from), List.of(unit("e-0", new Point2d(7, 5))), null);
        assertThat(dist(from, result)).isCloseTo(1.0, Offset.offset(0.01));
    }

    @Test void direct_picksNearestEnemy() {
        Point2d result = new DirectKiteStrategy().retreatTarget(
            unit("u-0", new Point2d(5, 5)),
            List.of(unit("e-near", new Point2d(6, 5)), unit("e-far", new Point2d(10, 5))), null);
        assertThat(result.x()).isLessThan(5f);
    }

    @Test void direct_ignoresNonNullTerrain() {
        TerrainGrid terrain = TerrainGrid.emulatedMap();
        DirectKiteStrategy s = new DirectKiteStrategy();
        List<Unit> enemies = List.of(unit("e-0", new Point2d(7, 5)));
        assertThat(s.retreatTarget(unit("u-0", new Point2d(5, 5)), enemies, terrain))
            .isEqualTo(s.retreatTarget(unit("u-0", new Point2d(5, 5)), enemies, null));
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn test -Dtest=KiteStrategyTest -q 2>&1 | grep "cannot find\|ERROR"
```

Expected: `cannot find symbol` for `DirectKiteStrategy`.

- [ ] **Step 3: Create KiteStrategy interface**

Create `src/main/java/io/quarkmind/plugin/tactics/KiteStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.Unit;
import java.util.List;

public interface KiteStrategy {
    /** @param terrain nullable — fall back to terrain-blind when null */
    Point2d retreatTarget(Unit unit, List<Unit> enemies, TerrainGrid terrain);
}
```

- [ ] **Step 4: Implement DirectKiteStrategy**

Create `src/main/java/io/quarkmind/plugin/tactics/DirectKiteStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.Unit;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;
import java.util.Comparator;
import java.util.List;

@ApplicationScoped
@Named("direct")
public class DirectKiteStrategy implements KiteStrategy {

    static final double KITE_STEP = 1.0;

    @Override
    public Point2d retreatTarget(Unit unit, List<Unit> enemies, TerrainGrid terrain) {
        Unit nearest = enemies.stream()
            .min(Comparator.comparingDouble(e -> distance(unit.position(), e.position())))
            .orElseThrow();
        double dx = unit.position().x() - nearest.position().x();
        double dy = unit.position().y() - nearest.position().y();
        double len = Math.sqrt(dx * dx + dy * dy);
        if (len < 0.001) return unit.position();
        return new Point2d(
            (float)(unit.position().x() + dx / len * KITE_STEP),
            (float)(unit.position().y() + dy / len * KITE_STEP));
    }

    static double distance(Point2d a, Point2d b) {
        double dx = a.x() - b.x(), dy = a.y() - b.y();
        return Math.sqrt(dx * dx + dy * dy);
    }
}
```

- [ ] **Step 5: Run KiteStrategyTest — expect PASS**

```bash
mvn test -Dtest=KiteStrategyTest -q
```

Expected: Tests run: 4, Failures: 0

- [ ] **Step 6: Write failing tests for LowestHpFocusFireStrategy**

Create `src/test/java/io/quarkmind/plugin/tactics/FocusFireStrategyTest.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class FocusFireStrategyTest {

    private static Unit stalker(String tag, Point2d pos) {
        return new Unit(tag, UnitType.STALKER, pos, 80, 80, 80, 80, 0);
    }
    private static Unit enemy(String tag, Point2d pos, int hp, int shields) {
        return new Unit(tag, UnitType.ZEALOT, pos, hp, 100, shields, 50, 0);
    }

    // LowestHpFocusFireStrategy

    @Test void lowestHp_allAttackLowestHpEnemy() {
        List<Unit> attackers = List.of(stalker("u-0", new Point2d(5,5)), stalker("u-1", new Point2d(5,6)));
        List<Unit> enemies   = List.of(enemy("e-strong", new Point2d(10,10), 100, 50),
                                       enemy("e-weak",   new Point2d(11,11), 10,  0));
        Map<String, Point2d> result = new LowestHpFocusFireStrategy().assignTargets(attackers, enemies);
        assertThat(result.get("u-0")).isEqualTo(new Point2d(11, 11));
        assertThat(result.get("u-1")).isEqualTo(new Point2d(11, 11));
    }

    @Test void lowestHp_singleEnemy() {
        Map<String, Point2d> result = new LowestHpFocusFireStrategy().assignTargets(
            List.of(stalker("u-0", new Point2d(5,5))),
            List.of(enemy("e-0", new Point2d(10,10), 80, 80)));
        assertThat(result.get("u-0")).isEqualTo(new Point2d(10, 10));
    }
}
```

- [ ] **Step 7: Run — expect compilation failure**

```bash
mvn test -Dtest=FocusFireStrategyTest -q 2>&1 | grep "cannot find\|ERROR"
```

- [ ] **Step 8: Create FocusFireStrategy interface**

Create `src/main/java/io/quarkmind/plugin/tactics/FocusFireStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.Unit;
import java.util.List;
import java.util.Map;

public interface FocusFireStrategy {
    /** @return map of attacker unit tag → target position */
    Map<String, Point2d> assignTargets(List<Unit> attackers, List<Unit> enemies);
}
```

- [ ] **Step 9: Implement LowestHpFocusFireStrategy**

Create `src/main/java/io/quarkmind/plugin/tactics/LowestHpFocusFireStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.Unit;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;
import java.util.*;

@ApplicationScoped
@Named("lowest-hp")
public class LowestHpFocusFireStrategy implements FocusFireStrategy {

    @Override
    public Map<String, Point2d> assignTargets(List<Unit> attackers, List<Unit> enemies) {
        Point2d target = enemies.stream()
            .min(Comparator.comparingInt(e -> e.health() + e.shields()))
            .map(Unit::position).orElseThrow();
        Map<String, Point2d> assignments = new HashMap<>();
        attackers.forEach(u -> assignments.put(u.tag(), target));
        return assignments;
    }
}
```

- [ ] **Step 10: Run FocusFireStrategyTest — expect PASS**

```bash
mvn test -Dtest=FocusFireStrategyTest -q
```

Expected: Tests run: 2, Failures: 0

- [ ] **Step 11: Wire strategies into DroolsTacticsTask**

In `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`:

Add imports:
```java
import io.quarkmind.plugin.tactics.FocusFireStrategy;
import io.quarkmind.plugin.tactics.KiteStrategy;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.inject.Instance;
import jakarta.enterprise.inject.literal.NamedLiteral;
import org.eclipse.microprofile.config.inject.ConfigProperty;
```

Add fields after `private final GoapPlanner planner`:
```java
    @Inject
    @ConfigProperty(name = "quarkmind.tactics.kite.strategy", defaultValue = "direct")
    String kiteStrategyName;

    @Inject
    @ConfigProperty(name = "quarkmind.tactics.focus-fire.strategy", defaultValue = "lowest-hp")
    String focusFireStrategyName;

    @Inject Instance<KiteStrategy>      kiteStrategies;
    @Inject Instance<FocusFireStrategy> focusFireStrategies;

    private KiteStrategy      kiteStrategy;
    private FocusFireStrategy focusFireStrategy;

    @PostConstruct
    void init() {
        kiteStrategy      = kiteStrategies.select(NamedLiteral.of(kiteStrategyName)).get();
        focusFireStrategy = focusFireStrategies.select(NamedLiteral.of(focusFireStrategyName)).get();
    }
```

In `execute()`, remove this line:
```java
        Optional<Unit> focusTarget = selectFocusTarget(enemies);
```

Update the dispatch call (remove `focusTarget`, add `null` for terrain — TerrainProvider wired in Task 2):
```java
                dispatch(plan.get(0), groupInfo.unitTags(), army, enemies, threat, bld, null);
```

Update `dispatch()` signature — replace `Optional<Unit> focusTarget` with `TerrainGrid terrain`:
```java
    private void dispatch(GoapAction action, List<String> unitTags,
                          List<Unit> army, List<Unit> enemies,
                          Point2d threat, List<Building> buildings,
                          TerrainGrid terrain) {
```

Add `import io.quarkmind.domain.TerrainGrid;`

Replace ATTACK case:
```java
            case "ATTACK" -> {
                List<Unit> attackers = army.stream()
                    .filter(u -> unitTags.contains(u.tag())).toList();
                Map<String, Point2d> targets = focusFireStrategy.assignTargets(attackers, enemies);
                unitTags.forEach(tag -> {
                    Point2d target = targets.getOrDefault(tag, threat != null ? threat : MAP_CENTER);
                    intentQueue.add(new AttackIntent(tag, target));
                });
            }
```

Replace KITE case:
```java
            case "KITE" -> {
                unitTags.forEach(tag ->
                    army.stream().filter(u -> u.tag().equals(tag)).findFirst()
                        .ifPresent(unit -> intentQueue.add(
                            new MoveIntent(tag, kiteStrategy.retreatTarget(unit, enemies, terrain)))));
            }
```

Delete static methods `kiteRetreatTarget`, `selectFocusTarget`, and constant `KITE_STEP` from `DroolsTacticsTask` (all now live in `DirectKiteStrategy`).

- [ ] **Step 12: Remove redundant tests from DroolsTacticsTaskTest**

In `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java`, delete any test methods that directly called `DroolsTacticsTask.kiteRetreatTarget(...)` or `DroolsTacticsTask.selectFocusTarget(...)` — those behaviours are now covered by `KiteStrategyTest` and `FocusFireStrategyTest`.

- [ ] **Step 13: Add config defaults**

Append to `src/main/resources/application.properties`:
```properties
quarkmind.tactics.kite.strategy=direct
quarkmind.tactics.focus-fire.strategy=lowest-hp
```

- [ ] **Step 14: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS — no behaviour changes.

- [ ] **Step 15: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/tactics/ \
        src/test/java/io/quarkmind/plugin/tactics/ \
        src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
        src/main/resources/application.properties \
        src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java
git commit -m "refactor(e11): extract KiteStrategy + FocusFireStrategy CDI strategy pattern — Refs #${EPIC} #${E11_1}"
```

---

## Task 2: TerrainProvider bean

**Files:** TerrainProvider.java (new), EmulatedGame.java, DroolsTacticsTask.java

No new tests — existing tests verify that null terrain leaves behaviour unchanged.

- [ ] **Step 1: Create TerrainProvider**

Create `src/main/java/io/quarkmind/sc2/TerrainProvider.java`:

```java
package io.quarkmind.sc2;

import io.quarkmind.domain.TerrainGrid;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class TerrainProvider {
    private TerrainGrid terrain;

    public void setTerrain(TerrainGrid terrain) { this.terrain = terrain; }

    /** Returns the active terrain, or null in mock/replay contexts. */
    public TerrainGrid get() { return terrain; }
}
```

- [ ] **Step 2: Inject TerrainProvider into EmulatedGame**

In `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`, add:

```java
import io.quarkmind.sc2.TerrainProvider;
```

Add injected field:
```java
    @Inject TerrainProvider terrainProvider;
```

In `setTerrainGrid(TerrainGrid terrainGrid)` (the test helper), add after `this.terrainGrid = terrainGrid;`:
```java
        terrainProvider.setTerrain(terrainGrid);
```

Search for any `@PostConstruct` or profile-init block where `TerrainGrid.emulatedMap()` is called and add the same call there.

- [ ] **Step 3: Inject TerrainProvider into DroolsTacticsTask**

In `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`:

```java
import io.quarkmind.sc2.TerrainProvider;
```

Add field:
```java
    @Inject TerrainProvider terrainProvider;
```

At the top of `execute()`, replace the `null` terrain placeholder:
```java
        TerrainGrid terrain = terrainProvider.get();
```

Update the dispatch call to pass `terrain` instead of `null`:
```java
                dispatch(plan.get(0), groupInfo.unitTags(), army, enemies, threat, bld, terrain);
```

- [ ] **Step 4: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS. Non-CDI unit tests don't wire TerrainProvider, so terrain stays null and DirectKiteStrategy ignores it.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/TerrainProvider.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java
git commit -m "feat(e11): TerrainProvider CDI bean — routes terrain from EmulatedGame to KiteStrategy — Refs #${EPIC} #${E11_2}"
```

---

## Task 3: Timing fix — STALKER cooldown 3 → 1

**Files:** SC2Data.java, EmulatedGameTest.java (threshold adjustments)

- [ ] **Step 1: Change STALKER cooldown**

In `src/main/java/io/quarkmind/domain/SC2Data.java`, update `attackCooldownInTicks()`:

```java
    public static int attackCooldownInTicks(UnitType type) {
        return switch (type) {
            case MARINE, HYDRALISK, STALKER                -> 1;
            case PROBE, ZEALOT, IMMORTAL, MARAUDER, ROACH -> 2;
            default                                        -> 2;
        };
    }
```

- [ ] **Step 2: Run tests — identify failures**

```bash
mvn test -q 2>&1 | grep -E "FAIL|expected|but was" | head -20
```

Expected: some E10 physics assertions in `EmulatedGameTest` fail because kiting triggers less often with cooldown=1 (unit is on cooldown for only 1 tick instead of 3 between shots).

- [ ] **Step 3: Adjust failing thresholds**

For each failing assertion, read the failure message (e.g., `expected: 45 but was: 32`) and adjust the numeric threshold to a value that the new timing produces reliably. Do **not** change test logic — only the numeric constants.

Pattern:
```java
// Before: assertThat(kitingHP).isGreaterThan(nonKitingHP + 30);
// After (example — use actual failure output):
assertThat(kitingHP).isGreaterThan(nonKitingHP + 8);
```

- [ ] **Step 4: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "fix(e11): Stalker attack cooldown 3→1 tick — matches real SC2 ~0.535s timing — Closes #${E11_3}"
```

---

## Task 4: TerrainAwareKiteStrategy

**Files:** TerrainAwareKiteStrategy.java, KiteStrategyTest.java, EmulatedGameTest.java, application.properties

- [ ] **Step 1: Write failing tests**

Append to `src/test/java/io/quarkmind/plugin/tactics/KiteStrategyTest.java`:

```java
    @Test void terrainAware_openTerrain_sameAsDirect() {
        TerrainGrid terrain = TerrainGrid.emulatedMap();
        List<Unit> enemies = List.of(unit("e-0", new Point2d(7, 5)));
        Unit u = unit("u-0", new Point2d(5, 5));
        assertThat(new TerrainAwareKiteStrategy().retreatTarget(u, enemies, terrain))
            .isEqualTo(new DirectKiteStrategy().retreatTarget(u, enemies, null));
    }

    @Test void terrainAware_wallAhead_findsWalkableAlternative() {
        // unit at (10,17), enemy at (10,15) → ideal retreat is up to (10,18) = WALL
        TerrainGrid terrain = TerrainGrid.emulatedMap();
        Point2d result = new TerrainAwareKiteStrategy().retreatTarget(
            unit("u-0", new Point2d(10, 17)),
            List.of(unit("e-0", new Point2d(10, 15))),
            terrain);
        assertThat(terrain.isWalkable((int) result.x(), (int) result.y())).isTrue();
        assertThat(result).isNotEqualTo(new Point2d(10, 18));
    }

    @Test void terrainAware_nullTerrain_behavesLikeDirect() {
        List<Unit> enemies = List.of(unit("e-0", new Point2d(7, 5)));
        Unit u = unit("u-0", new Point2d(5, 5));
        assertThat(new TerrainAwareKiteStrategy().retreatTarget(u, enemies, null))
            .isEqualTo(new DirectKiteStrategy().retreatTarget(u, enemies, null));
    }
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn test -Dtest=KiteStrategyTest -q 2>&1 | grep "cannot find"
```

- [ ] **Step 3: Implement TerrainAwareKiteStrategy**

Create `src/main/java/io/quarkmind/plugin/tactics/TerrainAwareKiteStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.domain.Unit;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;
import java.util.Comparator;
import java.util.List;

@ApplicationScoped
@Named("terrain-aware")
public class TerrainAwareKiteStrategy implements KiteStrategy {

    // Angular offsets tried in order: ideal, ±45°, ±90°, ±135°, 180°
    private static final double[] SWEEP = {
        0, Math.PI/4, -Math.PI/4, Math.PI/2, -Math.PI/2,
        3*Math.PI/4, -3*Math.PI/4, Math.PI
    };

    @Override
    public Point2d retreatTarget(Unit unit, List<Unit> enemies, TerrainGrid terrain) {
        Unit nearest = enemies.stream()
            .min(Comparator.comparingDouble(e -> DirectKiteStrategy.distance(unit.position(), e.position())))
            .orElseThrow();
        double dx = unit.position().x() - nearest.position().x();
        double dy = unit.position().y() - nearest.position().y();
        double len = Math.sqrt(dx * dx + dy * dy);
        if (len < 0.001) return unit.position();

        double idealAngle = Math.atan2(dy, dx);
        if (terrain == null) return candidateAt(unit.position(), idealAngle);

        for (double offset : SWEEP) {
            Point2d candidate = candidateAt(unit.position(), idealAngle + offset);
            if (terrain.isWalkable((int) candidate.x(), (int) candidate.y())) return candidate;
        }
        return unit.position(); // all 8 blocked — stay put
    }

    private static Point2d candidateAt(Point2d pos, double angle) {
        return new Point2d(
            (float)(pos.x() + Math.cos(angle) * DirectKiteStrategy.KITE_STEP),
            (float)(pos.y() + Math.sin(angle) * DirectKiteStrategy.KITE_STEP));
    }
}
```

- [ ] **Step 4: Run KiteStrategyTest — expect PASS**

```bash
mvn test -Dtest=KiteStrategyTest -q
```

Expected: Tests run: 7, Failures: 0

- [ ] **Step 5: Write physics E2E test**

In `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`, add a test that verifies that after several ticks with a unit near the wall row, no unit occupies a WALL tile. Follow the existing physics test setup pattern in that file (look at how existing combat tests configure enemies and run ticks).

```java
    @Test
    void terrainAwareKiting_doesNotStepIntoWallTile() {
        game.setTerrainGrid(TerrainGrid.emulatedMap());
        // Spawn an enemy just below the unit so ideal retreat direction is upward (toward wall)
        game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10, 10));
        // Run 5 ticks
        for (int i = 0; i < 5; i++) game.tick();
        // No friendly unit should be on a WALL tile
        TerrainGrid terrain = TerrainGrid.emulatedMap();
        for (Unit u : game.snapshot().myUnits()) {
            assertThat(terrain.heightAt((int) u.position().x(), (int) u.position().y()))
                .as("unit %s at %s", u.tag(), u.position())
                .isNotEqualTo(TerrainGrid.Height.WALL);
        }
    }
```

- [ ] **Step 6: Run EmulatedGameTest — expect PASS**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

- [ ] **Step 7: Add emulated profile override**

Append to `src/main/resources/application.properties`:
```properties
%emulated.quarkmind.tactics.kite.strategy=terrain-aware
```

- [ ] **Step 8: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/tactics/TerrainAwareKiteStrategy.java \
        src/test/java/io/quarkmind/plugin/tactics/KiteStrategyTest.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java \
        src/main/resources/application.properties
git commit -m "feat(e11): TerrainAwareKiteStrategy — angular sweep avoids wall tiles — Closes #${E11_4}"
```

---

## Task 5: OverkillRedirectFocusFireStrategy

**Files:** OverkillRedirectFocusFireStrategy.java, FocusFireStrategyTest.java

- [ ] **Step 1: Write failing tests**

Append to `src/test/java/io/quarkmind/plugin/tactics/FocusFireStrategyTest.java`:

```java
    // OverkillRedirectFocusFireStrategy
    // SC2Data.damagePerAttack(STALKER) = 13

    @Test void overkill_splitsMinimumAttackersOnPrimary() {
        // 5 Stalkers (13 dmg). Weak: 20 HP+shields. Strong: 150.
        // ceil(20/13) = 2 attackers on weak → 3 on strong.
        List<Unit> attackers = java.util.stream.IntStream.range(0, 5)
            .mapToObj(i -> stalker("u-" + i, new Point2d(5, 5 + i))).toList();
        List<Unit> enemies = List.of(
            enemy("e-weak",   new Point2d(10,10), 10, 10),   // 20 total
            enemy("e-strong", new Point2d(11,11), 100, 50));  // 150 total
        Map<String, Point2d> result =
            new OverkillRedirectFocusFireStrategy().assignTargets(attackers, enemies);
        long onWeak   = result.values().stream().filter(p -> p.equals(new Point2d(10,10))).count();
        long onStrong = result.values().stream().filter(p -> p.equals(new Point2d(11,11))).count();
        assertThat(onWeak).isEqualTo(2);
        assertThat(onStrong).isEqualTo(3);
    }

    @Test void overkill_allAttackPrimaryWhenFullVolleyNeeded() {
        // 3 Stalkers × 13 = 39 total. Primary has 50 HP. All must commit to primary.
        List<Unit> attackers = List.of(stalker("u-0", new Point2d(5,5)),
                                       stalker("u-1", new Point2d(5,6)),
                                       stalker("u-2", new Point2d(5,7)));
        List<Unit> enemies = List.of(enemy("e-0", new Point2d(10,10), 40, 10)); // 50 total
        Map<String, Point2d> result =
            new OverkillRedirectFocusFireStrategy().assignTargets(attackers, enemies);
        assertThat(result.values()).allMatch(p -> p.equals(new Point2d(10,10)));
    }

    @Test void overkill_singleEnemy_allAttackIt() {
        Map<String, Point2d> result = new OverkillRedirectFocusFireStrategy().assignTargets(
            List.of(stalker("u-0", new Point2d(5,5))),
            List.of(enemy("e-0", new Point2d(10,10), 80, 80)));
        assertThat(result.get("u-0")).isEqualTo(new Point2d(10,10));
    }
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn test -Dtest=FocusFireStrategyTest -q 2>&1 | grep "cannot find"
```

- [ ] **Step 3: Implement OverkillRedirectFocusFireStrategy**

Create `src/main/java/io/quarkmind/plugin/tactics/OverkillRedirectFocusFireStrategy.java`:

```java
package io.quarkmind.plugin.tactics;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.domain.Unit;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;
import java.util.*;

@ApplicationScoped
@Named("overkill-redirect")
public class OverkillRedirectFocusFireStrategy implements FocusFireStrategy {

    @Override
    public Map<String, Point2d> assignTargets(List<Unit> attackers, List<Unit> enemies) {
        List<Unit> sorted = enemies.stream()
            .sorted(Comparator.comparingInt(e -> e.health() + e.shields())).toList();
        Unit primary  = sorted.get(0);
        int  primaryHp = primary.health() + primary.shields();
        Map<String, Point2d> assignments = new HashMap<>();
        int accumulated = 0;
        boolean primaryKillable = false;
        for (Unit attacker : attackers) {
            if (!primaryKillable) {
                assignments.put(attacker.tag(), primary.position());
                accumulated += SC2Data.damagePerAttack(attacker.type());
                if (accumulated >= primaryHp) primaryKillable = true;
            } else {
                Point2d target = sorted.size() > 1 ? sorted.get(1).position() : primary.position();
                assignments.put(attacker.tag(), target);
            }
        }
        return assignments;
    }
}
```

- [ ] **Step 4: Run FocusFireStrategyTest — expect PASS**

```bash
mvn test -Dtest=FocusFireStrategyTest -q
```

Expected: Tests run: 5, Failures: 0

- [ ] **Step 5: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/tactics/OverkillRedirectFocusFireStrategy.java \
        src/test/java/io/quarkmind/plugin/tactics/FocusFireStrategyTest.java
git commit -m "feat(e11): OverkillRedirectFocusFireStrategy — min attackers on primary, rest to secondary — Closes #${E11_5}"
```

---

## Task 6: Blink assessment — pause before proceeding

All non-blink E11 work is now complete. Assess whether to implement Blink in E11.

**Punt to E12 if any of the following:**
- The two new WorldState keys (`shieldsLow`, `blinkReady`) produce an empty or wrong GOAP plan on first attempt and root cause is not immediately obvious
- Unit record migration (9th field) breaks more than 30 files and the compile loop is time-consuming
- Total estimated implementation time exceeds half a session

**If punting:**
```bash
gh issue create \
  --title "E12: Stalker Blink micro — GOAP BLINK action + EmulatedGame teleport" \
  --label "enhancement" \
  --body "Deferred from E11 (complexity assessment). Design: docs/superpowers/specs/2026-04-17-e11-strategy-pattern-tactics-extensions-design.md § Section 5. Refs #${E11_6}"
```

**If proceeding:** continue to Tasks 7–8.

---

## Task 7: Blink — Unit field + EmulatedGame mechanics

**Files:** Unit.java, SC2Data.java, EmulatedGame.java, EmulatedGameTest.java, all Unit construction sites (compiler-driven)

- [ ] **Step 1: Add blinkCooldownTicks to Unit record**

Modify `src/main/java/io/quarkmind/domain/Unit.java`:

```java
public record Unit(String tag, UnitType type, Point2d position,
                   int health, int maxHealth,
                   int shields, int maxShields,
                   int weaponCooldownTicks,
                   int blinkCooldownTicks) {}
```

- [ ] **Step 2: Fix all compilation errors**

```bash
mvn compile 2>&1 | grep "error:" | head -30
```

For each error site, append `, 0` to the Unit constructor call. Repeat until `mvn compile` succeeds. Sites include: `EmulatedGame.java`, `SimulatedGame.java`, `ReplaySimulatedGame.java`, `ObservationTranslator.java`, and all test files constructing `Unit` directly.

- [ ] **Step 3: Verify tests pass**

```bash
mvn test -q
```

Expected: BUILD SUCCESS — `blinkCooldownTicks=0` everywhere changes nothing.

- [ ] **Step 4: Add SC2Data blink constants**

In `src/main/java/io/quarkmind/domain/SC2Data.java`, append before the closing brace:

```java
    public static float blinkRange(UnitType type) {
        return type == UnitType.STALKER ? 8.0f : 0f;
    }

    public static int blinkCooldownInTicks(UnitType type) {
        return type == UnitType.STALKER ? 21 : 0;
    }

    public static int blinkShieldRestore(UnitType type) {
        return type == UnitType.STALKER ? 40 : 0;
    }
```

- [ ] **Step 5: Add blink state to EmulatedGame**

In `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`:

Add field alongside `unitCooldowns`:
```java
    private final Map<String, Integer> blinkCooldowns = new HashMap<>();
```

In the tick loop where `unitCooldowns` is decremented, also decrement `blinkCooldowns`:
```java
        blinkCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
```

In `snapshot()`, update the friendly unit mapping to stamp `blinkCooldownTicks`:
```java
        List<Unit> friendlyWithCooldown = myUnits.stream()
            .map(u -> new Unit(u.tag(), u.type(), u.position(),
                               u.health(), u.maxHealth(), u.shields(), u.maxShields(),
                               unitCooldowns.getOrDefault(u.tag(), 0),
                               blinkCooldowns.getOrDefault(u.tag(), 0)))
            .toList();
```

Add `executeBlink(String tag)` package-private method (same-package tests can call it directly):

```java
    void executeBlink(String tag) {
        myUnits.stream().filter(u -> u.tag().equals(tag)).findFirst().ifPresent(unit -> {
            Point2d dest = blinkDestination(unit);
            int restored = Math.min(unit.shields() + SC2Data.blinkShieldRestore(unit.type()),
                                    unit.maxShields());
            myUnits.remove(unit);
            myUnits.add(new Unit(unit.tag(), unit.type(), dest,
                                 unit.health(), unit.maxHealth(),
                                 restored, unit.maxShields(),
                                 unit.weaponCooldownTicks(),
                                 SC2Data.blinkCooldownInTicks(unit.type())));
        });
        blinkCooldowns.put(tag, SC2Data.blinkCooldownInTicks(UnitType.STALKER));
    }

    private Point2d blinkDestination(Unit unit) {
        if (enemyUnits.isEmpty()) return unit.position();
        Unit nearest = enemyUnits.stream()
            .min(Comparator.comparingDouble(e -> dist(unit.position(), e.position())))
            .orElseThrow();
        double dx = unit.position().x() - nearest.position().x();
        double dy = unit.position().y() - nearest.position().y();
        double len = Math.sqrt(dx * dx + dy * dy);
        if (len < 0.001) return unit.position();
        return new Point2d(
            (float)(unit.position().x() + dx / len * SC2Data.blinkRange(unit.type())),
            (float)(unit.position().y() + dy / len * SC2Data.blinkRange(unit.type())));
    }
```

(If `dist()` already exists in `EmulatedGame`, reuse it instead of adding a duplicate.)

- [ ] **Step 6: Write blink mechanics test**

In `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`:

```java
    @Test
    void blink_restoresShieldsAndMovesUnit() {
        game.reset();
        // Find a stalker (or skip if default spawn has none)
        String stalkerTag = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.STALKER)
            .map(Unit::tag).findFirst()
            .orElse(null);
        org.junit.jupiter.api.Assumptions.assumeTrue(stalkerTag != null, "No stalker in default spawn");

        game.setShieldsForTesting(stalkerTag, 10);
        game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(12, 10));

        Unit before = game.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(stalkerTag)).findFirst().orElseThrow();
        Point2d posBefore = before.position();

        game.executeBlink(stalkerTag);

        Unit after = game.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(stalkerTag)).findFirst().orElseThrow();
        assertThat(after.shields()).isEqualTo(50); // 10 + 40, maxShields=80
        assertThat(after.blinkCooldownTicks()).isEqualTo(SC2Data.blinkCooldownInTicks(UnitType.STALKER));
        assertThat(after.position()).isNotEqualTo(posBefore);
    }
```

- [ ] **Step 7: Run blink test**

```bash
mvn test -Dtest=EmulatedGameTest#blink_restoresShieldsAndMovesUnit -q
```

Expected: PASS (or skipped if no Stalker in default spawn).

- [ ] **Step 8: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/quarkmind/domain/Unit.java \
        src/main/java/io/quarkmind/domain/SC2Data.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git add -u   # pick up all other migrated Unit construction sites
git commit -m "feat(e11): Unit.blinkCooldownTicks + EmulatedGame blink mechanics — Refs #${EPIC} #${E11_6}"
```

---

## Task 8: Blink — Drools + GOAP

**⚠️ PUNT TRIGGER:** If after one debugging cycle the GOAP plan for "blinking" group is empty or wrong — stop, create the E12 issue (Task 6 above), and commit what's done.

**Files:** TacticsRuleUnit.java, StarCraftTactics.drl, DroolsTacticsTask.java, BlinkIntent.java (new), EmulatedGameTest.java

- [ ] **Step 1: Add DataStores to TacticsRuleUnit**

In `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java`, add after `onCooldownTags`:

```java
    private final DataStore<String> blinkReadyTags = DataSource.createStore();
    private final DataStore<String> shieldsLowTags = DataSource.createStore();
```

Add getters:
```java
    public DataStore<String> getBlinkReadyTags() { return blinkReadyTags; }
    public DataStore<String> getShieldsLowTags() { return shieldsLowTags; }
```

- [ ] **Step 2: Populate DataStores in DroolsTacticsTask.buildRuleUnit()**

In `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`, append to `buildRuleUnit()`:

```java
        army.stream()
            .filter(u -> u.type() == UnitType.STALKER && u.blinkCooldownTicks() == 0)
            .map(Unit::tag)
            .forEach(data.getBlinkReadyTags()::add);
        army.stream()
            .filter(u -> u.maxShields() > 0 && (double) u.shields() / u.maxShields() < 0.25)
            .map(Unit::tag)
            .forEach(data.getShieldsLowTags()::add);
```

Add `import io.quarkmind.domain.UnitType;` if not already present.

- [ ] **Step 3: Add BLINKING group to StarCraftTactics.drl**

In `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl`, add before "Group: kiting" (salience 205 > 200 so it wins over kiting):

```drl
rule "Group: blinking"
    salience 205
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() >= 0.3 ]
    /blinkReadyTags[ this == $u.tag() ]
    /shieldsLowTags[ this == $u.tag() ]
then
    groupDecisions.add("blinking:BLINK_ESCAPE:" + $u.tag());
    activeGroups.add("blinking");
end
```

Add Phase 2 emission after "Action: Retreat available":

```drl
rule "Action: Blink available"
    salience 110
when
    /activeGroups[ this == "blinking" ]
then
    actionDecisions.add("BLINK:1");
end
```

- [ ] **Step 4: Add BLINK GOAP action and BlinkIntent**

Create `src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java`:

```java
package io.quarkmind.sc2.intent;

import io.quarkmind.sc2.Intent;

public record BlinkIntent(String unitTag) implements Intent {}
```

In `DroolsTacticsTask.ACTION_TEMPLATES`, add:

```java
        "BLINK", new GoapAction("BLINK",
            Map.of("shieldsLow", true, "blinkReady", true, "enemyVisible", true),
            Map.of("shieldsLow", false, "inRange", false, "blinkReady", false), 1),
```

Add import:
```java
import io.quarkmind.sc2.intent.BlinkIntent;
```

- [ ] **Step 5: Add blinking cases to goalConditionKey and buildWorldState**

In `goalConditionKey()`:
```java
            case "BLINK_ESCAPE" -> "enemyEliminated";
```

In `buildWorldState()`:
```java
            case "blinking" -> new WorldState(Map.of(
                "lowHealth",       false,
                "enemyVisible",    true,
                "inRange",         true,
                "onCooldown",      false,
                "shieldsLow",      true,
                "blinkReady",      true,
                "unitSafe",        false,
                "enemyEliminated", false));
```

- [ ] **Step 6: Add BLINK dispatch case**

In `dispatch()`, add after the KITE case:

```java
            case "BLINK" -> unitTags.forEach(tag -> intentQueue.add(new BlinkIntent(tag)));
```

In `EmulatedGame`'s intent dispatch handler (wherever `AttackIntent`, `MoveIntent` etc. are processed), add handling for `BlinkIntent`:

```java
            case BlinkIntent bi -> executeBlink(bi.unitTag());
```

(Find the intent handler in EmulatedGame — search for `AttackIntent` to locate the switch.)

- [ ] **Step 7: Run tests — check for GOAP plan correctness**

```bash
mvn test -q 2>&1 | grep -E "FAIL|ERROR|Tests run"
```

If any test fails with a GOAP empty-plan symptom (units idle when they should blink), add a temporary debug line in `execute()`:
```java
log.debugf("[BLINK-DEBUG] group=blinking plan=%s", plan.stream().map(GoapAction::name).toList());
```

Run the failing test with `-Dquarkus.log.level=DEBUG` and inspect. **If the fix is not immediately obvious — punt to E12.**

- [ ] **Step 8: Write physics E2E test for blink**

In `EmulatedGameTest`:

```java
    @Test
    void blink_stalkerWithLowShields_shieldsIncrease() {
        game.reset();
        String stalkerTag = game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.STALKER)
            .map(Unit::tag).findFirst().orElse(null);
        org.junit.jupiter.api.Assumptions.assumeTrue(stalkerTag != null);

        game.setShieldsForTesting(stalkerTag, 10); // < 25% of 80
        game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(12, 10));

        for (int i = 0; i < 3; i++) game.tick();

        Unit after = game.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(stalkerTag)).findFirst().orElseThrow();
        assertThat(after.shields()).isGreaterThan(10); // blink restored shields
    }
```

- [ ] **Step 9: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 10: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java \
        src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl \
        src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java \
        src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java \
        src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e11): Stalker Blink — Drools BLINKING group + GOAP BLINK action + BlinkIntent — Closes #${E11_6} #${EPIC}"
```
