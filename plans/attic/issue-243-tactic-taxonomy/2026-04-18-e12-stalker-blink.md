# E12: Stalker Blink Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Stalker Blink micro to the GOAP tactics pipeline — when a Stalker has critically low shields and blink is off cooldown, the BLINKING Drools group fires and the GOAP planner selects BLINK as the first action, causing a teleport 8 tiles away with shield restoration.

**Architecture:** Nine-step migration: Unit record gains a 9th field (blinkCooldownTicks), EmulatedGame gains a blinkCooldowns map and executeBlink() method, TacticsRuleUnit gains two new DataStores, the DRL gains a new salience-205 group, and DroolsTacticsTask wires it all together. BlinkIntent is a new sealed Intent subtype dispatched by EmulatedEngine.

**Tech Stack:** Java records (Unit), Drools RuleUnit/DataStore, GOAP (GoapAction/GoapPlanner), EmulatedGame physics, Quarkus CDI (integration test only)

**Issue:** Closes #82

---

## File Map

| File | Change |
|------|--------|
| `src/main/java/io/quarkmind/domain/Unit.java` | Add 9th field `blinkCooldownTicks` |
| `src/main/java/io/quarkmind/domain/SC2Data.java` | Add 3 blink constants |
| `src/main/java/io/quarkmind/sc2/intent/Intent.java` | Add BlinkIntent to permits |
| `src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java` | Create new record |
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | blinkCooldowns map, executeBlink(), snapshot(), applyIntent(), reset(), resolveCombat(), spawnFriendlyUnitForTesting() |
| `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java` | Add blinkReadyTags + shieldsLowTags DataStores |
| `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl` | BLINKING group rule + Blink action rule |
| `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java` | ACTION_TEMPLATES, goalConditionKey, buildWorldState, buildRuleUnit, dispatch, computeBlinkReadyTags, computeShieldsLowTags |
| All 24 files with `new Unit(` | Add `, 0` as 9th argument |
| `src/test/java/io/quarkmind/sc2/emulated/BlinkMechanicsTest.java` | Create — mechanics unit tests |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add blink survival E2E test |
| `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java` | Add computeBlinkReadyTags + computeShieldsLowTags tests |

---

## Task 1: Add `blinkCooldownTicks` to Unit record and migrate all call sites

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/Unit.java`
- Modify: all 24 files containing `new Unit(`

- [ ] **Step 1: Add the 9th field to Unit**

Edit `src/main/java/io/quarkmind/domain/Unit.java`:

```java
package io.quarkmind.domain;

public record Unit(String tag, UnitType type, Point2d position,
                   int health, int maxHealth,
                   int shields, int maxShields,
                   int weaponCooldownTicks,
                   int blinkCooldownTicks) {}
```

- [ ] **Step 2: Run compile to see all broken call sites**

```bash
mvn compile -q 2>&1 | grep "error:" | head -40
```

Expected: ~70 errors, all "wrong number of arguments to constructor".

- [ ] **Step 3: Fix all 70 call sites — append `, 0` as the 9th argument**

Every `new Unit(tag, type, pos, health, maxHealth, shields, maxShields, cooldown)` becomes `new Unit(tag, type, pos, health, maxHealth, shields, maxShields, cooldown, 0)`.

The 24 files to update (check each for `new Unit(`):

**Main sources:**
- `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (~16 call sites)
- `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`
- `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`
- `src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java`

**Test sources:**
- `src/test/java/io/quarkmind/agent/GameStateTranslatorTest.java`
- `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`
- `src/test/java/io/quarkmind/domain/GameStateTest.java`
- `src/test/java/io/quarkmind/plugin/BasicEconomicsTaskTest.java`
- `src/test/java/io/quarkmind/plugin/BasicScoutingTaskTest.java`
- `src/test/java/io/quarkmind/plugin/BasicStrategyTaskTest.java`
- `src/test/java/io/quarkmind/plugin/BasicTacticsTaskTest.java`
- `src/test/java/io/quarkmind/plugin/DroolsScoutingTaskIT.java`
- `src/test/java/io/quarkmind/plugin/DroolsStrategyTaskTest.java`
- `src/test/java/io/quarkmind/plugin/DroolsTacticsRuleUnitTest.java`
- `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskIT.java`
- `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java`
- `src/test/java/io/quarkmind/plugin/EconomicsDecisionServiceTest.java`
- `src/test/java/io/quarkmind/plugin/EconomicsFlowTest.java`
- `src/test/java/io/quarkmind/plugin/ScoutingSessionManagerTest.java`
- `src/test/java/io/quarkmind/plugin/tactics/FocusFireStrategyTest.java`
- `src/test/java/io/quarkmind/plugin/tactics/KiteStrategyTest.java`
- `src/test/java/io/quarkmind/qa/GameStateBroadcasterTest.java`
- `src/test/java/io/quarkmind/sc2/emulated/DamageCalculatorTest.java`
- `src/test/java/io/quarkmind/sc2/emulated/VisibilityGridTest.java`

**Pattern:** In each file, for every `new Unit(` call, count the existing arguments and append `, 0` before the closing `)`.

Example before:
```java
new Unit("probe-0", UnitType.PROBE, new Point2d(9, 9), 45, 45, 20, 20, 0)
```
Example after:
```java
new Unit("probe-0", UnitType.PROBE, new Point2d(9, 9), 45, 45, 20, 20, 0, 0)
```

Note: `EmulatedGame.snapshot()` has a special `new Unit(...)` that stamps `weaponCooldownTicks` — leave it as-is for now (it gets updated in Task 4 to also stamp blinkCooldownTicks). For now add `, 0` to keep it compiling.

- [ ] **Step 4: Verify compile succeeds**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS, no errors.

- [ ] **Step 5: Run full test suite to confirm no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS, 437+ tests passing (count may increase as we add tests).

- [ ] **Step 6: Commit**

```bash
git add -u
git commit -m "feat(e12): add blinkCooldownTicks as 9th field to Unit record — migrate 70 call sites

Refs #82"
```

---

## Task 2: Add blink constants to SC2Data

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

- [ ] **Step 1: Add three blink methods at the end of SC2Data (before the final `}`)**

```java
    /** Blink range in tiles (STALKER only). */
    public static float blinkRange(UnitType type) {
        return type == UnitType.STALKER ? 8.0f : 0.0f;
    }

    /** Ticks before blink can be used again. 21 ticks ≈ 10.5s at 500ms/tick. */
    public static int blinkCooldownInTicks(UnitType type) {
        return type == UnitType.STALKER ? 21 : 0;
    }

    /** Shields restored on blink (capped at maxShields). */
    public static int blinkShieldRestore(UnitType type) {
        return type == UnitType.STALKER ? 40 : 0;
    }
```

- [ ] **Step 2: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java
git commit -m "feat(e12): SC2Data blink constants — blinkRange, blinkCooldownInTicks, blinkShieldRestore

Refs #82"
```

---

## Task 3: Add BlinkIntent sealed type

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java`
- Modify: `src/main/java/io/quarkmind/sc2/intent/Intent.java`

- [ ] **Step 1: Create BlinkIntent**

```java
package io.quarkmind.sc2.intent;

public record BlinkIntent(String unitTag) implements Intent {}
```

- [ ] **Step 2: Add BlinkIntent to Intent's permits clause**

Edit `src/main/java/io/quarkmind/sc2/intent/Intent.java`:

```java
package io.quarkmind.sc2.intent;

public sealed interface Intent permits BuildIntent, TrainIntent, AttackIntent, MoveIntent, BlinkIntent {
    String unitTag();
}
```

- [ ] **Step 3: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/intent/BlinkIntent.java \
        src/main/java/io/quarkmind/sc2/intent/Intent.java
git commit -m "feat(e12): BlinkIntent sealed type

Refs #82"
```

---

## Task 4: EmulatedGame blink mechanics

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

This task adds: `blinkCooldowns` map, `executeBlink()`, `blinkRetreatTarget()`, `spawnFriendlyUnitForTesting()`, and wires blink cooldowns into `reset()`, `resolveCombat()`, `snapshot()`, `applyIntent()`, and unit-death cleanup.

- [ ] **Step 1: Add blinkCooldowns field alongside unitCooldowns**

In the field declarations section (around line 39), add after `enemyCooldowns`:

```java
    private final Map<String, Integer> blinkCooldowns = new HashMap<>();
```

- [ ] **Step 2: Clear blinkCooldowns in reset()**

In `reset()`, after `unitCooldowns.clear()`:

```java
        blinkCooldowns.clear();
```

- [ ] **Step 3: Decrement blinkCooldowns in resolveCombat()**

In `resolveCombat()`, after the two existing `replaceAll` decrements (lines 375–376):

```java
        blinkCooldowns.replaceAll((tag, cd) -> Math.max(0, cd - 1));
```

- [ ] **Step 4: Clean up blinkCooldowns on unit death**

In `resolveCombat()`, in the `myUnits.removeIf` block after `unitCooldowns.remove(u.tag())`:

```java
                        blinkCooldowns.remove(u.tag());
```

- [ ] **Step 5: Update snapshot() to stamp blinkCooldownTicks**

Replace the existing `friendlyWithCooldown` mapping in `snapshot()`:

```java
        List<Unit> friendlyWithCooldown = myUnits.stream()
            .map(u -> new Unit(u.tag(), u.type(), u.position(),
                               u.health(), u.maxHealth(), u.shields(), u.maxShields(),
                               unitCooldowns.getOrDefault(u.tag(), 0),
                               blinkCooldowns.getOrDefault(u.tag(), 0)))
            .toList();
```

- [ ] **Step 6: Add blinkRetreatTarget() private method**

Add after the existing `nearestInRange()` method:

```java
    private Point2d blinkRetreatTarget(Unit unit) {
        Unit nearest = enemyUnits.stream()
            .min(Comparator.comparingDouble(e -> distance(unit.position(), e.position())))
            .orElse(null);
        if (nearest == null) return unit.position();
        double dx = unit.position().x() - nearest.position().x();
        double dy = unit.position().y() - nearest.position().y();
        double len = Math.sqrt(dx * dx + dy * dy);
        if (len < 0.001) return unit.position();
        float step = SC2Data.blinkRange(unit.type());
        double baseAngle = Math.atan2(dy, dx);
        // Try direct direction first, then sweep if wall
        for (int i = 0; i <= 4; i++) {
            for (int sign : new int[]{1, -1}) {
                double angle = baseAngle + sign * i * Math.PI / 4;
                Point2d candidate = new Point2d(
                    (float)(unit.position().x() + Math.cos(angle) * step),
                    (float)(unit.position().y() + Math.sin(angle) * step));
                if (terrainGrid == null || !terrainGrid.isWall(candidate)) return candidate;
                if (i == 0) break; // only one attempt for the direct direction
            }
        }
        return unit.position(); // all directions blocked — stay put
    }
```

Note: `distance()` is already a static method in EmulatedGame.

- [ ] **Step 7: Add executeBlink() package-private method**

Add after `blinkRetreatTarget()`:

```java
    void executeBlink(String tag) {
        myUnits.replaceAll(u -> {
            if (!u.tag().equals(tag)) return u;
            Point2d dest = blinkRetreatTarget(u);
            int restored = Math.min(u.shields() + SC2Data.blinkShieldRestore(u.type()), u.maxShields());
            unitTargets.put(tag, dest);
            blinkCooldowns.put(tag, SC2Data.blinkCooldownInTicks(u.type()));
            attackingUnits.remove(tag); // blink cancels attack mode
            return new Unit(u.tag(), u.type(), dest,
                            u.health(), u.maxHealth(), restored, u.maxShields(), 0, 0);
        });
    }
```

- [ ] **Step 8: Handle BlinkIntent in applyIntent()**

In the `applyIntent()` switch statement, add a case for BlinkIntent:

```java
    public void applyIntent(Intent intent) {
        switch (intent) {
            case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation(), false);
            case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation(), true);
            case TrainIntent  t -> handleTrain(t);
            case BuildIntent  b -> handleBuild(b);
            case BlinkIntent  b -> executeBlink(b.unitTag());
        }
    }
```

- [ ] **Step 9: Add spawnFriendlyUnitForTesting() test helper**

At the end of EmulatedGame, before the final closing `}` of the class, add:

```java
    /** Test helper — adds a friendly unit directly to myUnits. Package-private. */
    void spawnFriendlyUnitForTesting(UnitType type, Point2d position) {
        int hp  = SC2Data.maxHealth(type);
        int sh  = SC2Data.maxShields(type);
        String tag = "test-friendly-" + nextTag++;
        myUnits.add(new Unit(tag, type, position, hp, hp, sh, sh, 0, 0));
    }
```

- [ ] **Step 10: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 11: Run tests**

```bash
mvn test -q
```

Expected: all existing tests pass (blinkCooldownTicks=0 for all existing call sites so behaviour unchanged).

- [ ] **Step 12: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git commit -m "feat(e12): EmulatedGame blink mechanics — blinkCooldowns, executeBlink, snapshot stamping

Refs #82"
```

---

## Task 5: TacticsRuleUnit — add blinkReadyTags and shieldsLowTags DataStores

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java`

- [ ] **Step 1: Add two new DataStores and getters**

Add after `onCooldownTags` field and its getter:

```java
    private final DataStore<String> blinkReadyTags = DataSource.createStore();
    private final DataStore<String> shieldsLowTags = DataSource.createStore();
```

And add getters:

```java
    public DataStore<String> getBlinkReadyTags() { return blinkReadyTags; }
    public DataStore<String> getShieldsLowTags() { return shieldsLowTags; }
```

Full updated class:

```java
package io.quarkmind.plugin.drools;

import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;
import io.quarkmind.domain.Unit;

import java.util.ArrayList;
import java.util.List;

public class TacticsRuleUnit implements RuleUnitData {

    private final DataStore<Unit>   army           = DataSource.createStore();
    private final DataStore<Unit>   enemies        = DataSource.createStore();
    private final DataStore<String> inRangeTags    = DataSource.createStore();
    private final DataStore<String> onCooldownTags = DataSource.createStore();
    private final DataStore<String> blinkReadyTags = DataSource.createStore();
    private final DataStore<String> shieldsLowTags = DataSource.createStore();
    private final DataStore<String> activeGroups   = DataSource.createStore();
    private final List<String>      groupDecisions  = new ArrayList<>();
    private final List<String>      actionDecisions = new ArrayList<>();
    private String strategyGoal = "MACRO";

    public DataStore<Unit>   getArmy()             { return army; }
    public DataStore<Unit>   getEnemies()           { return enemies; }
    public DataStore<String> getInRangeTags()       { return inRangeTags; }
    public DataStore<String> getOnCooldownTags()    { return onCooldownTags; }
    public DataStore<String> getBlinkReadyTags()    { return blinkReadyTags; }
    public DataStore<String> getShieldsLowTags()    { return shieldsLowTags; }
    public DataStore<String> getActiveGroups()      { return activeGroups; }
    public List<String>      getGroupDecisions()    { return groupDecisions; }
    public List<String>      getActionDecisions()   { return actionDecisions; }
    public String            getStrategyGoal()      { return strategyGoal; }
    public void              setStrategyGoal(String g) { this.strategyGoal = g; }
}
```

- [ ] **Step 2: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/drools/TacticsRuleUnit.java
git commit -m "feat(e12): TacticsRuleUnit — blinkReadyTags and shieldsLowTags DataStores

Refs #82"
```

---

## Task 6: StarCraftTactics.drl — BLINKING group rule and Blink action rule

**Files:**
- Modify: `src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl`

- [ ] **Step 1: Add UnitType import at the top of the DRL**

After the `unit TacticsRuleUnit;` line, add:

```drl
import io.quarkmind.domain.UnitType;
```

- [ ] **Step 2: Add "Group: blinking" rule at salience 205**

Insert between the "Group: low health" (salience 210) rule and the "Group: kiting" (salience 200) rule, in the Phase 1 comment block. Also update the group comment to include the new group:

```drl
rule "Group: blinking"
    salience 205
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ type == UnitType.STALKER, (double) this.health() / this.maxHealth() >= 0.3 ]
    /blinkReadyTags[ this == $u.tag() ]
    /shieldsLowTags[ this == $u.tag() ]
then
    groupDecisions.add("blinking:BLINKING:" + $u.tag());
    activeGroups.add("blinking");
end
```

- [ ] **Step 3: Add "Action: Blink available" rule at salience 105**

Insert in the Phase 2 section, between "Action: Retreat available" (salience 110) and "Action: Kite available" (salience 100):

```drl
rule "Action: Blink available"
    salience 105
when
    /activeGroups[ this == "blinking" ]
then
    actionDecisions.add("BLINK:1");
end
```

- [ ] **Step 4: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Run tests**

```bash
mvn test -q
```

Expected: all existing tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/io/quarkmind/plugin/drools/StarCraftTactics.drl
git commit -m "feat(e12): StarCraftTactics.drl — BLINKING group (salience 205) and Blink action rules

Refs #82"
```

---

## Task 7: DroolsTacticsTask — wire blink into GOAP pipeline

**Files:**
- Modify: `src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java`

This task touches five areas: ACTION_TEMPLATES, `goalConditionKey()`, `buildWorldState()`, `buildRuleUnit()`, `dispatch()`, and two new static helper methods.

- [ ] **Step 1: Add BlinkIntent import at the top of the file**

Add alongside the existing intent imports:

```java
import io.quarkmind.sc2.intent.BlinkIntent;
```

- [ ] **Step 2: Add BLINK to ACTION_TEMPLATES**

The `ACTION_TEMPLATES` map currently uses `Map.of(...)` with 4 entries. Java's `Map.of` supports up to 10 entries — add BLINK as the 5th:

```java
    private static final Map<String, GoapAction> ACTION_TEMPLATES = Map.of(
        "RETREAT",        new GoapAction("RETREAT",
            Map.of("lowHealth", true),
            Map.of("unitSafe", true), 1),
        "ATTACK",         new GoapAction("ATTACK",
            Map.of("inRange", true, "enemyVisible", true, "onCooldown", false),
            Map.of("enemyEliminated", true), 2),
        "MOVE_TO_ENGAGE", new GoapAction("MOVE_TO_ENGAGE",
            Map.of("enemyVisible", true, "inRange", false),
            Map.of("inRange", true), 1),
        "KITE",           new GoapAction("KITE",
            Map.of("inRange", true, "onCooldown", true, "enemyVisible", true),
            Map.of("onCooldown", false), 1),
        "BLINK",          new GoapAction("BLINK",
            Map.of("shieldsLow", true, "blinkReady", true, "enemyVisible", true),
            Map.of("shieldsLow", false, "inRange", false, "blinkReady", false), 1)
    );
```

- [ ] **Step 3: Add "BLINKING" case to goalConditionKey()**

```java
    private String goalConditionKey(String goalName) {
        return switch (goalName) {
            case "UNIT_SAFE"        -> "unitSafe";
            case "ENEMY_ELIMINATED" -> "enemyEliminated";
            case "KITING"           -> "enemyEliminated"; // plan: KITE → ATTACK
            case "BLINKING"         -> "enemyEliminated"; // plan: BLINK → MOVE_TO_ENGAGE → ATTACK
            default                 -> goalName.toLowerCase();
        };
    }
```

- [ ] **Step 4: Add "blinking" case to buildWorldState()**

```java
    private WorldState buildWorldState(String groupId, boolean enemyVisible) {
        return switch (groupId) {
            case "low-health"   -> new WorldState(Map.of(
                "lowHealth",       true,
                "enemyVisible",    enemyVisible,
                "inRange",         false,
                "unitSafe",        false,
                "enemyEliminated", false));
            case "in-range"     -> new WorldState(Map.of(
                "lowHealth",       false,
                "enemyVisible",    true,
                "inRange",         true,
                "onCooldown",      false,
                "unitSafe",        false,
                "enemyEliminated", false));
            case "out-of-range" -> new WorldState(Map.of(
                "lowHealth",       false,
                "enemyVisible",    true,
                "inRange",         false,
                "unitSafe",        false,
                "enemyEliminated", false));
            case "kiting" -> new WorldState(Map.of(
                "lowHealth",       false,
                "enemyVisible",    true,
                "inRange",         true,
                "onCooldown",      true,
                "unitSafe",        false,
                "enemyEliminated", false));
            case "blinking" -> new WorldState(Map.of(
                "shieldsLow",      true,
                "blinkReady",      true,
                "enemyVisible",    enemyVisible,
                "inRange",         true,
                "onCooldown",      false,
                "lowHealth",       false,
                "unitSafe",        false,
                "enemyEliminated", false));
            default             -> new WorldState(Map.of("enemyEliminated", false));
        };
    }
```

- [ ] **Step 5: Add two new static helper methods**

Add after `computeOnCooldownTags()`:

```java
    static Set<String> computeBlinkReadyTags(List<Unit> army) {
        return army.stream()
            .filter(u -> u.type() == UnitType.STALKER && u.blinkCooldownTicks() == 0)
            .map(Unit::tag)
            .collect(Collectors.toSet());
    }

    static Set<String> computeShieldsLowTags(List<Unit> army) {
        return army.stream()
            .filter(u -> u.shields() < u.maxShields() * 0.25)
            .map(Unit::tag)
            .collect(Collectors.toSet());
    }
```

- [ ] **Step 6: Update buildRuleUnit() signature and body**

Replace the existing `buildRuleUnit()` method:

```java
    private TacticsRuleUnit buildRuleUnit(List<Unit> army, List<Unit> enemies,
                                           Set<String> inRangeTags, Set<String> onCooldownTags,
                                           Set<String> blinkReadyTags, Set<String> shieldsLowTags,
                                           String strategy) {
        TacticsRuleUnit data = new TacticsRuleUnit();
        data.setStrategyGoal(strategy);
        army.forEach(data.getArmy()::add);
        enemies.forEach(data.getEnemies()::add);
        inRangeTags.forEach(data.getInRangeTags()::add);
        onCooldownTags.forEach(data.getOnCooldownTags()::add);
        blinkReadyTags.forEach(data.getBlinkReadyTags()::add);
        shieldsLowTags.forEach(data.getShieldsLowTags()::add);
        return data;
    }
```

- [ ] **Step 7: Update execute() to compute blink sets and pass them to buildRuleUnit()**

In `execute()`, after the existing `onCooldownSet` computation:

```java
        Set<String> blinkReadySet  = computeBlinkReadyTags(army);
        Set<String> shieldsLowSet  = computeShieldsLowTags(army);
```

Update the `buildRuleUnit` call:

```java
        TacticsRuleUnit data = buildRuleUnit(army, enemies, inRangeSet, onCooldownSet, blinkReadySet, shieldsLowSet, strategy);
```

- [ ] **Step 8: Add BLINK case to dispatch()**

In the `dispatch()` switch, add after the KITE case:

```java
            case "BLINK" -> {
                unitTags.forEach(tag -> intentQueue.add(new BlinkIntent(tag)));
            }
```

- [ ] **Step 9: Add UnitType import if not already present**

Check the imports — `UnitType` is in `io.quarkmind.domain.*` which is already imported. No change needed.

- [ ] **Step 10: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 11: Run tests**

```bash
mvn test -q
```

Expected: all existing tests pass.

- [ ] **Step 12: Commit**

```bash
git add src/main/java/io/quarkmind/plugin/DroolsTacticsTask.java
git commit -m "feat(e12): DroolsTacticsTask — BLINK action wired into GOAP (templates, worldstate, dispatch)

Refs #82"
```

---

## Task 8: BlinkMechanicsTest — unit tests for EmulatedGame blink physics

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/emulated/BlinkMechanicsTest.java`

- [ ] **Step 1: Write the test class**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.BlinkIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BlinkMechanicsTest {

    EmulatedGame game;

    @BeforeEach
    void setUp() {
        game = new EmulatedGame();
        game.configureWave(9999, 1, UnitType.ZEALOT);
        game.reset();
        // Clear the default probes; spawn a single Stalker for blink tests
        game.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(10.0f, 10.0f));
    }

    private Unit stalker() {
        return game.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.STALKER)
            .findFirst()
            .orElseThrow(() -> new AssertionError("No Stalker found"));
    }

    @Test
    void blinkSetsBlinkCooldownInSnapshot() {
        String tag = stalker().tag();
        game.applyIntent(new BlinkIntent(tag));
        assertThat(stalker().blinkCooldownTicks())
            .isEqualTo(SC2Data.blinkCooldownInTicks(UnitType.STALKER)); // 21
    }

    @Test
    void blinkRestoresShieldsCappedAtMaxShields() {
        // Set shields to 50 so 50 + 40 restore > 80 max → should cap at 80
        String tag = stalker().tag();
        game.setShieldsForTesting(tag, 50);
        game.applyIntent(new BlinkIntent(tag));
        assertThat(stalker().shields()).isEqualTo(SC2Data.maxShields(UnitType.STALKER)); // 80
    }

    @Test
    void blinkRestoresPartialShieldsWhenRoomExists() {
        // Set shields to 10 so 10 + 40 = 50 < 80 max → gets full 40 restore
        String tag = stalker().tag();
        game.setShieldsForTesting(tag, 10);
        game.applyIntent(new BlinkIntent(tag));
        assertThat(stalker().shields()).isEqualTo(50);
    }

    @Test
    void blinkCooldownDecrementsEachTick() {
        String tag = stalker().tag();
        game.applyIntent(new BlinkIntent(tag));
        assertThat(stalker().blinkCooldownTicks()).isEqualTo(21);
        game.tick();
        assertThat(stalker().blinkCooldownTicks()).isEqualTo(20);
        game.tick();
        assertThat(stalker().blinkCooldownTicks()).isEqualTo(19);
    }

    @Test
    void blinkMovesUnitAtLeastBlinkRangeFromStart() {
        // Spawn an enemy to give blinkRetreatTarget a direction
        game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10.0f, 10.3f));
        String tag = stalker().tag();
        Point2d before = stalker().position();
        game.applyIntent(new BlinkIntent(tag));
        Point2d after = stalker().position();
        double dx = after.x() - before.x();
        double dy = after.y() - before.y();
        double dist = Math.sqrt(dx * dx + dy * dy);
        assertThat(dist).isGreaterThanOrEqualTo(SC2Data.blinkRange(UnitType.STALKER) - 0.01);
    }

    @Test
    void blinkCancelsAttackMode() {
        // Ensure blink clears the unit from attacking state (no follow-up auto-attack)
        String tag = stalker().tag();
        game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10.0f, 10.3f));
        // Manually blink — after blink, unit should not be in attack mode
        game.applyIntent(new BlinkIntent(tag));
        // Advance a tick — if attacking, the stalker would deal damage to the adjacent zealot
        // After blink the stalker is 8 tiles away — zealot at 10,10.3 is out of 5-tile range
        game.tick();
        // Enemy still alive = stalker is not attacking after blink
        assertThat(game.snapshot().enemyUnits()).isNotEmpty();
    }
}
```

- [ ] **Step 2: Run the new tests**

```bash
mvn test -Dtest=BlinkMechanicsTest -q
```

Expected: all 6 tests PASS.

- [ ] **Step 3: Run full suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add src/test/java/io/quarkmind/sc2/emulated/BlinkMechanicsTest.java
git commit -m "test(e12): BlinkMechanicsTest — cooldown stamping, shield restore cap, position jump, attack cancel

Refs #82"
```

---

## Task 9: E2E physics test + DroolsTacticsTask static helper tests

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`
- Modify: `src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java`

### Part A — EmulatedGameTest E2E survival test

- [ ] **Step 1: Add blink survival test to EmulatedGameTest**

Add at the end of the existing `EmulatedGameTest` class, in a new "E12 blink" section:

```java
    // ---- E12 blink tests ----

    @Test
    void stalkerThatBlinksRetainsMoreHpThanNonBlinkingControl() {
        // Scenario: one Stalker vs one ZEALOT at melee range.
        // Blink condition: shields < 25% of 80 = < 20 shields.
        // Run A: Stalker blinks when shields are critically low.
        // Run B: Stalker never blinks (control).
        // Assert: Run A Stalker has higher HP+shields after 30 ticks.

        // ----- Run A: with blink -----
        EmulatedGame runA = new EmulatedGame();
        runA.configureWave(9999, 1, UnitType.ZEALOT);
        runA.reset();
        runA.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(10.0f, 10.0f));
        runA.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10.0f, 10.5f));

        String stalkerTagA = runA.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.STALKER).findFirst().orElseThrow().tag();

        boolean blinkUsedA = false;
        for (int t = 0; t < 30; t++) {
            GameState state = runA.snapshot();
            Unit stalker = state.myUnits().stream()
                .filter(u -> u.tag().equals(stalkerTagA)).findFirst().orElse(null);
            if (stalker == null) break; // died
            // Trigger blink when shields critically low and blink ready
            if (!blinkUsedA && stalker.shields() < stalker.maxShields() * 0.25
                    && stalker.blinkCooldownTicks() == 0) {
                runA.applyIntent(new BlinkIntent(stalkerTagA));
                blinkUsedA = true;
            } else {
                runA.applyIntent(new AttackIntent(stalkerTagA, runA.snapshot().enemyUnits().get(0).position()));
            }
            runA.tick();
        }

        // ----- Run B: no blink (control) -----
        EmulatedGame runB = new EmulatedGame();
        runB.configureWave(9999, 1, UnitType.ZEALOT);
        runB.reset();
        runB.spawnFriendlyUnitForTesting(UnitType.STALKER, new Point2d(10.0f, 10.0f));
        runB.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(10.0f, 10.5f));

        String stalkerTagB = runB.snapshot().myUnits().stream()
            .filter(u -> u.type() == UnitType.STALKER).findFirst().orElseThrow().tag();

        for (int t = 0; t < 30; t++) {
            List<Unit> enemies = runB.snapshot().enemyUnits();
            if (enemies.isEmpty()) break;
            Unit stalker = runB.snapshot().myUnits().stream()
                .filter(u -> u.tag().equals(stalkerTagB)).findFirst().orElse(null);
            if (stalker == null) break;
            runB.applyIntent(new AttackIntent(stalkerTagB, enemies.get(0).position()));
            runB.tick();
        }

        int hpA = runA.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(stalkerTagA))
            .mapToInt(u -> u.health() + u.shields()).findFirst().orElse(0);
        int hpB = runB.snapshot().myUnits().stream()
            .filter(u -> u.tag().equals(stalkerTagB))
            .mapToInt(u -> u.health() + u.shields()).findFirst().orElse(0);

        assertThat(blinkUsedA).as("blink should have triggered in run A").isTrue();
        assertThat(hpA).as("blinking Stalker should survive with more total HP+shields")
            .isGreaterThan(hpB);
    }
```

- [ ] **Step 2: Add required import to EmulatedGameTest**

Add at the top of the imports (alongside existing `sc2.intent.*` imports):

```java
import io.quarkmind.sc2.intent.BlinkIntent;
import io.quarkmind.sc2.intent.AttackIntent;
```

(Check if AttackIntent is already imported — if so, skip that line.)

- [ ] **Step 3: Run the new E2E test**

```bash
mvn test -Dtest=EmulatedGameTest#stalkerThatBlinksRetainsMoreHpThanNonBlinkingControl -q
```

Expected: PASS.

### Part B — DroolsTacticsTaskTest static helper tests

- [ ] **Step 4: Add tests for computeBlinkReadyTags and computeShieldsLowTags to DroolsTacticsTaskTest**

Add a helper method and two test methods to the existing `DroolsTacticsTaskTest` class. First, check how existing unit helpers look in that file, then add:

```java
    // Helper already exists in test file — reuse the existing unit() factory.
    // If it doesn't exist, add:
    private static Unit unit(String tag, UnitType type, Point2d pos, int shields, int maxShields, int blinkCd) {
        int hp = SC2Data.maxHealth(type);
        return new Unit(tag, type, pos, hp, hp, shields, maxShields, 0, blinkCd);
    }

    @Test
    void computeBlinkReadyTagsReturnsStalkerWithCooldownZero() {
        List<Unit> army = List.of(
            new Unit("s-0", UnitType.STALKER, new Point2d(0,0), 80, 80, 80, 80, 0, 0),  // ready
            new Unit("s-1", UnitType.STALKER, new Point2d(0,0), 80, 80, 80, 80, 0, 5),  // on cooldown
            new Unit("z-0", UnitType.ZEALOT,  new Point2d(0,0), 100, 100, 50, 50, 0, 0) // not a Stalker
        );
        Set<String> result = DroolsTacticsTask.computeBlinkReadyTags(army);
        assertThat(result).containsExactly("s-0");
    }

    @Test
    void computeShieldsLowTagsReturnsBelowTwentyFivePercent() {
        List<Unit> army = List.of(
            new Unit("s-0", UnitType.STALKER, new Point2d(0,0), 80, 80, 19, 80, 0, 0), // 19 < 20 (25% of 80)
            new Unit("s-1", UnitType.STALKER, new Point2d(0,0), 80, 80, 20, 80, 0, 0), // 20 == 25%, not low
            new Unit("s-2", UnitType.STALKER, new Point2d(0,0), 80, 80,  0, 80, 0, 0)  // 0 shields — low
        );
        Set<String> result = DroolsTacticsTask.computeShieldsLowTags(army);
        assertThat(result).containsExactlyInAnyOrder("s-0", "s-2");
    }
```

Note: The existing test class will already have `import io.quarkmind.domain.*` and `import java.util.*` — check before adding duplicate imports.

- [ ] **Step 5: Run DroolsTacticsTaskTest**

```bash
mvn test -Dtest=DroolsTacticsTaskTest -q
```

Expected: all tests PASS.

- [ ] **Step 6: Run full suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 7: Commit**

```bash
git add src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java \
        src/test/java/io/quarkmind/plugin/DroolsTacticsTaskTest.java
git commit -m "test(e12): blink survival E2E + computeBlinkReadyTags/ShieldsLowTags unit tests — Closes #82

Refs #82"
```

---

## Self-Review

**Spec coverage check:**
- ✅ Unit record 9th field — Task 1
- ✅ SC2Data blink constants — Task 2
- ✅ BlinkIntent + Intent permits — Task 3
- ✅ EmulatedGame blinkCooldowns map, decrement, death cleanup — Task 4
- ✅ EmulatedGame executeBlink() — teleport + shield restore + cooldown reset — Task 4
- ✅ EmulatedGame snapshot() stamps blinkCooldownTicks — Task 4
- ✅ TacticsRuleUnit blinkReadyTags + shieldsLowTags — Task 5
- ✅ DRL BLINKING group salience 205 (above kiting 200, below low-health 210) — Task 6
- ✅ DRL Action: Blink available — Task 6
- ✅ BLINK GoapAction preconditions/effects/cost — Task 7
- ✅ buildWorldState("blinking") — Task 7
- ✅ goalConditionKey("BLINKING") — Task 7
- ✅ dispatch() BLINK case emits BlinkIntent — Task 7
- ✅ BlinkMechanicsTest — Task 8
- ✅ E2E survival comparison test — Task 9
- ✅ computeBlinkReadyTags/ShieldsLowTags unit tests — Task 9

**Type consistency:** `blinkCooldownTicks` used consistently as int throughout. `BlinkIntent(String unitTag)` matches `Intent.unitTag()`. `executeBlink(String tag)` matches `BlinkIntent.unitTag()` usage in `applyIntent()`.

**Placeholder scan:** None found.
