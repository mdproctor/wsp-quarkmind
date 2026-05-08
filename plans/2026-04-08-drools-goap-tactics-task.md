# Drools-backed GOAP TacticsTask Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `BasicTacticsTask` with `DroolsTacticsTask` — a GOAP planner where Drools classifies unit groups and emits an action library each tick, and Java A* finds the cheapest action sequence per group.

**Architecture:** Drools fires once per tick to produce group decisions (Phase 1) and action strings (Phase 2). Java parses the output, builds `GoapAction` objects with hardcoded preconditions/effects, and runs A* per unit group over `WorldState` clones. The first action in each plan is dispatched as an `AttackIntent` or `MoveIntent`. DEFEND bypasses GOAP and emits `MoveIntent` to Nexus directly.

**Tech Stack:** Drools Rule Units (`drools-ruleunits-api`), plain Java A*, JUnit 5 + AssertJ, `@QuarkusTest` for CDI integration tests.

---

## File Map

**Create:**
- `src/main/java/org/acme/starcraft/plugin/tactics/WorldState.java` — immutable boolean-condition map; cheap to clone for A*
- `src/main/java/org/acme/starcraft/plugin/tactics/GoapAction.java` — name + preconditions + effects + cost; `isApplicable()` + `applyTo()`
- `src/main/java/org/acme/starcraft/plugin/tactics/GoapPlanner.java` — A* over `WorldState` nodes; no CDI
- `src/main/java/org/acme/starcraft/plugin/drools/TacticsRuleUnit.java` — `RuleUnitData` with `DataStore<Unit> army/enemies/inRangeTags/activeGroups`; plain `List<String>` outputs
- `src/main/java/org/acme/starcraft/plugin/DroolsTacticsTask.java` — active CDI bean; orchestrates Drools → parse → A* → dispatch
- `src/main/resources/org/acme/starcraft/plugin/drools/StarCraftTactics.drl` — Phase 1 (group classification) + Phase 2 (action emission) rules
- `src/test/java/org/acme/starcraft/plugin/GoapPlannerTest.java` — plain JUnit; tests WorldState, GoapAction, and GoapPlanner
- `src/test/java/org/acme/starcraft/plugin/DroolsTacticsRuleUnitTest.java` — `@QuarkusTest`; tests DRL rules directly via `RuleUnit<TacticsRuleUnit>`
- `src/test/java/org/acme/starcraft/plugin/DroolsTacticsTaskIT.java` — `@QuarkusTest`; end-to-end via `TacticsTask.execute(CaseFile)`

**Modify:**
- `src/main/java/org/acme/starcraft/plugin/BasicTacticsTask.java` — remove `@ApplicationScoped` and `@CaseType` (demote; keep class for reference)
- `docs/library-research.md` — mark GOAP chosen; note gdx-ai deferred
- `NATIVE.md` — confirm no new non-native dependency introduced

---

## Task 1: WorldState + GoapAction (plain JUnit TDD)

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/tactics/WorldState.java`
- Create: `src/main/java/org/acme/starcraft/plugin/tactics/GoapAction.java`
- Create: `src/test/java/org/acme/starcraft/plugin/GoapPlannerTest.java`

- [ ] **Step 1: Write failing tests for WorldState and GoapAction**

Create `src/test/java/org/acme/starcraft/plugin/GoapPlannerTest.java`:

```java
package org.acme.starcraft.plugin;

import org.acme.starcraft.plugin.tactics.GoapAction;
import org.acme.starcraft.plugin.tactics.GoapPlanner;
import org.acme.starcraft.plugin.tactics.WorldState;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GoapPlannerTest {

    // ---- WorldState ----

    @Test
    void worldStateWithReturnsCopyWithNewValue() {
        WorldState original = new WorldState(Map.of("inRange", false));
        WorldState updated  = original.with("inRange", true);
        assertThat(updated.get("inRange")).isTrue();
        assertThat(original.get("inRange")).isFalse(); // immutable
    }

    @Test
    void worldStateSatisfiesReturnsTrueWhenConditionIsTrue() {
        WorldState state = new WorldState(Map.of("unitSafe", true));
        assertThat(state.satisfies("unitSafe")).isTrue();
    }

    @Test
    void worldStateSatisfiesReturnsFalseWhenConditionIsFalse() {
        WorldState state = new WorldState(Map.of("unitSafe", false));
        assertThat(state.satisfies("unitSafe")).isFalse();
    }

    @Test
    void worldStateSatisfiesReturnsFalseForAbsentCondition() {
        WorldState state = new WorldState(new HashMap<>());
        assertThat(state.satisfies("unknown")).isFalse();
    }

    // ---- GoapAction ----

    @Test
    void goapActionIsApplicableWhenPreconditionsMet() {
        GoapAction attack = attack();
        WorldState state  = new WorldState(Map.of("inRange", true, "enemyVisible", true));
        assertThat(attack.isApplicable(state)).isTrue();
    }

    @Test
    void goapActionNotApplicableWhenPreconditionUnmet() {
        GoapAction attack = attack();
        WorldState state  = new WorldState(Map.of("inRange", false, "enemyVisible", true));
        assertThat(attack.isApplicable(state)).isFalse();
    }

    @Test
    void goapActionApplyToProducesNewStateWithEffects() {
        GoapAction move = moveToEngage();
        WorldState state = new WorldState(Map.of("inRange", false, "enemyVisible", true));
        WorldState result = move.applyTo(state);
        assertThat(result.get("inRange")).isTrue();
        assertThat(state.get("inRange")).isFalse(); // original unchanged
    }

    // ---- GoapPlanner ----

    @Test
    void plannerReturnsEmptyWhenGoalAlreadySatisfied() {
        GoapPlanner planner = new GoapPlanner();
        WorldState state = new WorldState(Map.of("enemyEliminated", true));
        List<GoapAction> plan = planner.plan(state, "enemyEliminated", List.of(attack()));
        assertThat(plan).isEmpty();
    }

    @Test
    void plannerReturnsEmptyWhenGoalUnreachable() {
        GoapPlanner planner = new GoapPlanner();
        WorldState state = new WorldState(Map.of("inRange", false, "enemyVisible", false));
        List<GoapAction> plan = planner.plan(state, "enemyEliminated", List.of(attack()));
        assertThat(plan).isEmpty();
    }

    @Test
    void plannerFindsDirectPlanForInRangeUnit() {
        GoapPlanner planner = new GoapPlanner();
        WorldState state = new WorldState(Map.of("inRange", true, "enemyVisible", true,
                                                  "lowHealth", false, "enemyEliminated", false));
        List<GoapAction> plan = planner.plan(state, "enemyEliminated", List.of(moveToEngage(), attack()));
        assertThat(plan).extracting(GoapAction::name).containsExactly("ATTACK");
    }

    @Test
    void plannerFindsChainedPlanForOutOfRangeUnit() {
        GoapPlanner planner = new GoapPlanner();
        WorldState state = new WorldState(Map.of("inRange", false, "enemyVisible", true,
                                                  "lowHealth", false, "enemyEliminated", false));
        List<GoapAction> plan = planner.plan(state, "enemyEliminated", List.of(moveToEngage(), attack()));
        assertThat(plan).extracting(GoapAction::name).containsExactly("MOVE_TO_ENGAGE", "ATTACK");
    }

    @Test
    void plannerFindsRetreatForLowHealthUnit() {
        GoapPlanner planner = new GoapPlanner();
        WorldState state = new WorldState(Map.of("lowHealth", true, "unitSafe", false));
        List<GoapAction> plan = planner.plan(state, "unitSafe", List.of(retreat(), attack(), moveToEngage()));
        assertThat(plan).extracting(GoapAction::name).containsExactly("RETREAT");
    }

    @Test
    void plannerPicksCheaperOfTwoPaths() {
        // Two ways to eliminate enemy: cheap chain or expensive direct
        GoapAction cheapMove   = new GoapAction("CHEAP_MOVE",
            Map.of("enemyVisible", true, "inRange", false), Map.of("inRange", true), 1);
        GoapAction cheapAttack = new GoapAction("CHEAP_ATTACK",
            Map.of("inRange", true, "enemyVisible", true), Map.of("enemyEliminated", true), 1);
        GoapAction expensive   = new GoapAction("EXPENSIVE_ATTACK",
            Map.of("inRange", true, "enemyVisible", true), Map.of("enemyEliminated", true), 10);
        WorldState state = new WorldState(Map.of("inRange", true, "enemyVisible", true,
                                                  "enemyEliminated", false));
        GoapPlanner planner = new GoapPlanner();
        List<GoapAction> plan = planner.plan(state, "enemyEliminated",
            List.of(cheapMove, expensive, cheapAttack));
        assertThat(plan).extracting(GoapAction::name).containsExactly("CHEAP_ATTACK");
    }

    // ---- Helpers ----

    private GoapAction attack() {
        return new GoapAction("ATTACK",
            Map.of("inRange", true, "enemyVisible", true),
            Map.of("enemyEliminated", true), 2);
    }

    private GoapAction moveToEngage() {
        return new GoapAction("MOVE_TO_ENGAGE",
            Map.of("enemyVisible", true, "inRange", false),
            Map.of("inRange", true), 1);
    }

    private GoapAction retreat() {
        return new GoapAction("RETREAT",
            Map.of("lowHealth", true),
            Map.of("unitSafe", true), 1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
mvn test -Dtest=GoapPlannerTest -q 2>&1 | tail -5
```

Expected: compilation errors — `WorldState`, `GoapAction`, `GoapPlanner` do not exist yet.

- [ ] **Step 3: Implement WorldState**

Create `src/main/java/org/acme/starcraft/plugin/tactics/WorldState.java`:

```java
package org.acme.starcraft.plugin.tactics;

import java.util.HashMap;
import java.util.Map;

public record WorldState(Map<String, Boolean> conditions) {

    public WorldState with(String key, boolean value) {
        Map<String, Boolean> copy = new HashMap<>(conditions);
        copy.put(key, value);
        return new WorldState(copy);
    }

    public boolean get(String key) {
        return Boolean.TRUE.equals(conditions.get(key));
    }

    public boolean satisfies(String goalCondition) {
        return get(goalCondition);
    }
}
```

- [ ] **Step 4: Implement GoapAction**

Create `src/main/java/org/acme/starcraft/plugin/tactics/GoapAction.java`:

```java
package org.acme.starcraft.plugin.tactics;

import java.util.Map;

public record GoapAction(
    String name,
    Map<String, Boolean> preconditions,
    Map<String, Boolean> effects,
    int cost
) {
    public boolean isApplicable(WorldState state) {
        return preconditions.entrySet().stream()
            .allMatch(e -> state.get(e.getKey()) == e.getValue());
    }

    public WorldState applyTo(WorldState state) {
        WorldState result = state;
        for (var entry : effects.entrySet()) {
            result = result.with(entry.getKey(), entry.getValue());
        }
        return result;
    }
}
```

- [ ] **Step 5: Run WorldState + GoapAction tests only (planner tests will still fail)**

```bash
mvn test -Dtest=GoapPlannerTest#worldState*+GoapPlannerTest#goapAction* -q 2>&1 | tail -5
```

Expected: WorldState and GoapAction tests PASS. Planner tests still fail (GoapPlanner not yet implemented).

- [ ] **Step 6: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/tactics/WorldState.java \
        src/main/java/org/acme/starcraft/plugin/tactics/GoapAction.java \
        src/test/java/org/acme/starcraft/plugin/GoapPlannerTest.java
git commit -m "feat(goap): WorldState + GoapAction records — immutable, A*-friendly

Refs #2"
```

---

## Task 2: GoapPlanner — A* Search (plain JUnit TDD)

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/tactics/GoapPlanner.java`

- [ ] **Step 1: Implement GoapPlanner**

Create `src/main/java/org/acme/starcraft/plugin/tactics/GoapPlanner.java`:

```java
package org.acme.starcraft.plugin.tactics;

import java.util.*;

public class GoapPlanner {

    /**
     * A* search over WorldState nodes.
     *
     * @param initial        starting world state
     * @param goalCondition  boolean key that must be true in the goal state
     * @param actions        all available actions (planner picks applicable ones)
     * @return cheapest action sequence, or empty list if goal already satisfied or unreachable
     */
    public List<GoapAction> plan(WorldState initial, String goalCondition, List<GoapAction> actions) {
        if (initial.satisfies(goalCondition)) return List.of();

        record Node(WorldState state, List<GoapAction> plan, int cost) {}

        PriorityQueue<Node> open = new PriorityQueue<>(
            Comparator.comparingInt(n -> n.cost() + heuristic(n.state(), goalCondition)));
        open.add(new Node(initial, List.of(), 0));

        Set<Map<String, Boolean>> visited = new HashSet<>();

        while (!open.isEmpty()) {
            Node current = open.poll();
            if (current.state().satisfies(goalCondition)) return current.plan();
            if (!visited.add(current.state().conditions())) continue;

            for (GoapAction action : actions) {
                if (action.isApplicable(current.state())) {
                    WorldState next = action.applyTo(current.state());
                    List<GoapAction> newPlan = new ArrayList<>(current.plan());
                    newPlan.add(action);
                    open.add(new Node(next, newPlan, current.cost() + action.cost()));
                }
            }
        }
        return List.of();
    }

    private int heuristic(WorldState state, String goalCondition) {
        return state.satisfies(goalCondition) ? 0 : 1;
    }
}
```

- [ ] **Step 2: Run all GoapPlannerTest tests**

```bash
mvn test -Dtest=GoapPlannerTest -q 2>&1 | tail -10
```

Expected: all 12 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/tactics/GoapPlanner.java
git commit -m "feat(goap): GoapPlanner — A* search over WorldState nodes

Refs #2"
```

---

## Task 3: TacticsRuleUnit + StarCraftTactics.drl + Rule Tests

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/drools/TacticsRuleUnit.java`
- Create: `src/main/resources/org/acme/starcraft/plugin/drools/StarCraftTactics.drl`
- Create: `src/test/java/org/acme/starcraft/plugin/DroolsTacticsRuleUnitTest.java`

**Note:** `DataSource.createStore()` requires Quarkus boot (GE-0053). All rule unit tests must be `@QuarkusTest`.

- [ ] **Step 1: Implement TacticsRuleUnit**

Create `src/main/java/org/acme/starcraft/plugin/drools/TacticsRuleUnit.java`:

```java
package org.acme.starcraft.plugin.drools;

import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;
import org.acme.starcraft.domain.Unit;

import java.util.ArrayList;
import java.util.List;

/**
 * Rule Unit data context for Drools tactics evaluation.
 *
 * <p>Per GE-0053: only JDK types and {@link DataStore}{@code <T>} (generic type erased)
 * may appear as plain field types. Application types must use DataStore wrapping.
 *
 * <p>Phase 1 rules (salience 200+) classify units into groups, writing to
 * {@link #groupDecisions} and inserting group IDs into {@link #activeGroups}.
 *
 * <p>Phase 2 rules (salience 100+) match on {@link #activeGroups} and emit
 * applicable action names to {@link #actionDecisions}.
 */
public class TacticsRuleUnit implements RuleUnitData {

    /** All player army units. */
    private final DataStore<Unit>   army           = DataSource.createStore();

    /** Visible enemy units. */
    private final DataStore<Unit>   enemies        = DataSource.createStore();

    /**
     * Tags of player units within Stalker effective range of any enemy.
     * Pre-computed in Java (Euclidean distance) before rule firing to avoid
     * floating-point arithmetic in DRL.
     */
    private final DataStore<String> inRangeTags    = DataSource.createStore();

    /**
     * Active group IDs inserted by Phase 1 rules (e.g. "low-health", "in-range").
     * Phase 2 rules pattern-match on this store — DataStore insertions trigger
     * Drools re-evaluation, ensuring Phase 2 fires after Phase 1.
     */
    private final DataStore<String> activeGroups   = DataSource.createStore();

    /** Phase 1 output: "groupId:GOAL:unitTag" strings. */
    private final List<String>      groupDecisions  = new ArrayList<>();

    /** Phase 2 output: "ACTION_NAME:cost" strings. */
    private final List<String>      actionDecisions = new ArrayList<>();

    /** ATTACK / DEFEND / MACRO — set before rule firing. */
    private String strategyGoal = "MACRO";

    public DataStore<Unit>   getArmy()             { return army; }
    public DataStore<Unit>   getEnemies()           { return enemies; }
    public DataStore<String> getInRangeTags()       { return inRangeTags; }
    public DataStore<String> getActiveGroups()      { return activeGroups; }
    public List<String>      getGroupDecisions()    { return groupDecisions; }
    public List<String>      getActionDecisions()   { return actionDecisions; }
    public String            getStrategyGoal()      { return strategyGoal; }
    public void              setStrategyGoal(String g) { this.strategyGoal = g; }
}
```

- [ ] **Step 2: Write the DRL rules**

Create `src/main/resources/org/acme/starcraft/plugin/drools/StarCraftTactics.drl`:

```drl
package org.acme.starcraft.plugin.drools;
unit TacticsRuleUnit;

// ---------------------------------------------------------------------------
// Phase 1 — Group Classification (salience 200+)
//
// Classifies each army unit into exactly one group based on health and enemy
// proximity. Only fires for ATTACK strategy.
//   - DEFEND: handled in Java (bypass GOAP), no rules fire
//   - MACRO:  no rules fire; army holds position
//
// Output format for groupDecisions: "groupId:GOAL:unitTag"
// activeGroups DataStore is also updated so Phase 2 rules fire correctly.
//
// Syntax notes:
//   - No-constraint pattern: /army  (NOT /army[])
//   - Record accessors: this.method() to avoid DRL keyword clashes
//   - health/maxHealth cast to double avoids integer division
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

rule "Group: in range"
    salience 200
when
    eval(strategyGoal.equals("ATTACK"))
    $u: /army[ (double) this.health() / this.maxHealth() >= 0.3 ]
    /inRangeTags[ this == $u.tag() ]
then
    groupDecisions.add("in-range:ENEMY_ELIMINATED:" + $u.tag());
    activeGroups.add("in-range");
end

rule "Group: out of range"
    salience 190
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
//
// Fires once per active group. Matches on activeGroups DataStore (not
// groupDecisions List) to guarantee Drools re-evaluates after Phase 1.
//
// Output format for actionDecisions: "ACTION_NAME:cost"
// All emitted actions are available to all group planners — A* determines
// applicability via WorldState precondition checks.
// ---------------------------------------------------------------------------

rule "Action: Retreat available"
    salience 110
when
    /activeGroups[ this == "low-health" ]
then
    actionDecisions.add("RETREAT:1");
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

- [ ] **Step 3: Write the rule unit tests**

Create `src/test/java/org/acme/starcraft/plugin/DroolsTacticsRuleUnitTest.java`:

```java
package org.acme.starcraft.plugin;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.domain.Point2d;
import org.acme.starcraft.domain.Unit;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.plugin.drools.TacticsRuleUnit;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for StarCraftTactics.drl rules.
 *
 * <p>Requires {@code @QuarkusTest} — {@code DataSource.createStore()} is initialised
 * at Quarkus build time and unavailable in plain JUnit (GE-0053).
 *
 * <p>Each test documents one specific rule behaviour.
 */
@QuarkusTest
class DroolsTacticsRuleUnitTest {

    @Inject RuleUnit<TacticsRuleUnit> ruleUnit;

    // ---- Phase 1: Group Classification ----

    @Test
    void unitBelowThirtyPercentHealthClassifiedAsLowHealth() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 20, 100)), List.of(enemy()), List.of());
        fire(data);
        assertThat(data.getGroupDecisions())
            .anyMatch(g -> g.startsWith("low-health:UNIT_SAFE:s-0"));
    }

    @Test
    void unitWithinRangeClassifiedAsInRange() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 80, 80)), List.of(enemy()), List.of("s-0"));
        fire(data);
        assertThat(data.getGroupDecisions())
            .anyMatch(g -> g.startsWith("in-range:ENEMY_ELIMINATED:s-0"));
    }

    @Test
    void healthyUnitWithEnemyNotInRangeClassifiedAsOutOfRange() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 80, 80)), List.of(enemy()), List.of());
        fire(data);
        assertThat(data.getGroupDecisions())
            .anyMatch(g -> g.startsWith("out-of-range:ENEMY_ELIMINATED:s-0"));
    }

    @Test
    void mixedArmyProducesThreeGroups() {
        List<Unit> army = List.of(
            stalker("low",  20, 100),  // low health
            stalker("near", 80, 80),   // in range
            stalker("far",  80, 80)    // out of range
        );
        TacticsRuleUnit data = attack(army, List.of(enemy()), List.of("near"));
        fire(data);
        assertThat(data.getGroupDecisions())
            .anyMatch(g -> g.startsWith("low-health:"))
            .anyMatch(g -> g.startsWith("in-range:"))
            .anyMatch(g -> g.startsWith("out-of-range:"));
    }

    @Test
    void noEnemiesProducesNoGroups() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 80, 80)), List.of(), List.of());
        fire(data);
        assertThat(data.getGroupDecisions()).isEmpty();
    }

    @Test
    void emptyArmyProducesNoGroups() {
        TacticsRuleUnit data = attack(List.of(), List.of(enemy()), List.of());
        fire(data);
        assertThat(data.getGroupDecisions()).isEmpty();
    }

    @Test
    void macroStrategyProducesNoGroups() {
        TacticsRuleUnit data = new TacticsRuleUnit();
        data.setStrategyGoal("MACRO");
        stalker("s-0", 80, 80);
        data.getArmy().add(stalker("s-0", 80, 80));
        data.getEnemies().add(enemy());
        fire(data);
        assertThat(data.getGroupDecisions()).isEmpty();
    }

    // ---- Phase 2: Action Emission ----

    @Test
    void lowHealthGroupEmitsRetreatAction() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 20, 100)), List.of(enemy()), List.of());
        fire(data);
        assertThat(data.getActionDecisions()).contains("RETREAT:1");
    }

    @Test
    void inRangeGroupEmitsAttackAction() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 80, 80)), List.of(enemy()), List.of("s-0"));
        fire(data);
        assertThat(data.getActionDecisions()).contains("ATTACK:2");
    }

    @Test
    void outOfRangeGroupEmitsMoveToEngageAction() {
        TacticsRuleUnit data = attack(List.of(stalker("s-0", 80, 80)), List.of(enemy()), List.of());
        fire(data);
        assertThat(data.getActionDecisions()).contains("MOVE_TO_ENGAGE:1");
    }

    // ---- Helpers ----

    private void fire(TacticsRuleUnit data) {
        try (RuleUnitInstance<TacticsRuleUnit> instance = ruleUnit.createInstance(data)) {
            instance.fire();
        }
    }

    private TacticsRuleUnit attack(List<Unit> army, List<Unit> enemies, List<String> inRangeTags) {
        TacticsRuleUnit data = new TacticsRuleUnit();
        data.setStrategyGoal("ATTACK");
        army.forEach(data.getArmy()::add);
        enemies.forEach(data.getEnemies()::add);
        inRangeTags.forEach(data.getInRangeTags()::add);
        return data;
    }

    private Unit stalker(String tag, int health, int maxHealth) {
        return new Unit(tag, UnitType.STALKER, new Point2d(10, 10), health, maxHealth);
    }

    private Unit enemy() {
        return new Unit("e-0", UnitType.ZEALOT, new Point2d(15, 15), 100, 100);
    }
}
```

- [ ] **Step 4: Run the rule unit tests**

```bash
mvn test -Dtest=DroolsTacticsRuleUnitTest -q 2>&1 | tail -10
```

Expected: all 10 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/drools/TacticsRuleUnit.java \
        src/main/resources/org/acme/starcraft/plugin/drools/StarCraftTactics.drl \
        src/test/java/org/acme/starcraft/plugin/DroolsTacticsRuleUnitTest.java
git commit -m "feat(goap): TacticsRuleUnit + StarCraftTactics.drl — Drools group classification and action emission

Phase 1 rules classify units (low-health/in-range/out-of-range) using activeGroups
DataStore so Phase 2 rules fire correctly after Phase 1.

Refs #3"
```

---

## Task 4: DroolsTacticsTask + Demote BasicTacticsTask

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/DroolsTacticsTask.java`
- Create: `src/test/java/org/acme/starcraft/plugin/DroolsTacticsTaskIT.java`
- Modify: `src/main/java/org/acme/starcraft/plugin/BasicTacticsTask.java`

- [ ] **Step 1: Write the integration test first**

Create `src/test/java/org/acme/starcraft/plugin/DroolsTacticsTaskIT.java`:

```java
package org.acme.starcraft.plugin;

import io.casehub.annotation.CaseType;
import io.casehub.core.DefaultCaseFile;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.TacticsTask;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.AttackIntent;
import org.acme.starcraft.sc2.intent.MoveIntent;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * End-to-end integration tests for DroolsTacticsTask.
 *
 * <p>Each test documents one complete pipeline scenario: CaseFile state → Drools
 * group classification → A* planning → Intent emission.
 */
@QuarkusTest
class DroolsTacticsTaskIT {

    @Inject @CaseType("starcraft-game") TacticsTask tacticsTask;
    @Inject IntentQueue intentQueue;

    @BeforeEach @AfterEach
    void drainQueue() { intentQueue.drainAll(); }

    // ---- ATTACK: unit group routing ----

    @Test
    void attackAllUnitsInRangeEmitsAttackIntents() {
        // Both stalkers within Stalker range (6 tiles) of enemy
        var cf = caseFile("ATTACK",
            List.of(stalker("s-0", new Point2d(10, 10), 80, 80),
                    stalker("s-1", new Point2d(11, 10), 80, 80)),
            List.of(enemy(new Point2d(14, 10))),  // distance ~4 — in range
            new Point2d(14, 10));
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending())
            .hasSize(2)
            .allMatch(i -> i instanceof AttackIntent);
    }

    @Test
    void attackAllUnitsOutOfRangeEmitsMoveIntents() {
        // Stalkers far from enemy — out of range
        var cf = caseFile("ATTACK",
            List.of(stalker("s-0", new Point2d(10, 10), 80, 80)),
            List.of(enemy(new Point2d(30, 30))),  // distance ~28 — out of range
            new Point2d(30, 30));
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .allMatch(i -> i instanceof MoveIntent);
        assertThat(((MoveIntent) intentQueue.pending().get(0)).targetLocation())
            .isEqualTo(new Point2d(30, 30));
    }

    @Test
    void attackLowHealthUnitsRetreatsToNexus() {
        Point2d nexusPos = new Point2d(8, 8);
        var cf = caseFile("ATTACK",
            List.of(stalker("s-0", new Point2d(10, 10), 20, 100)),  // 20% health — low
            List.of(enemy(new Point2d(30, 30))),
            new Point2d(30, 30));
        cf.put(StarCraftCaseFile.MY_BUILDINGS, List.of(nexus(nexusPos)));
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .allMatch(i -> i instanceof MoveIntent);
        assertThat(((MoveIntent) intentQueue.pending().get(0)).targetLocation())
            .isEqualTo(nexusPos);
    }

    @Test
    void attackMixedArmyEmitsCorrectIntentPerGroup() {
        // s-0: low health → retreat; s-1: in range → attack; s-2: out of range → move
        var cf = caseFile("ATTACK",
            List.of(stalker("s-0", new Point2d(10, 10), 20, 100),   // low health
                    stalker("s-1", new Point2d(10, 10), 80, 80),    // in range (same pos)
                    stalker("s-2", new Point2d(10, 10), 80, 80)),   // out of range (no enemy nearby)
            List.of(enemy(new Point2d(14, 10))),  // s-1 in range; s-0/s-2 technically also, but s-0 low health
            new Point2d(14, 10));
        tacticsTask.execute(cf);
        // At minimum: 3 intents (one per unit)
        assertThat(intentQueue.pending()).hasSize(3);
        // s-0 gets MoveIntent (retreat)
        assertThat(intentQueue.pending().stream()
            .filter(i -> i instanceof MoveIntent mi && mi.unitTag().equals("s-0")))
            .hasSize(1);
        // s-1 gets AttackIntent
        assertThat(intentQueue.pending().stream()
            .filter(i -> i instanceof AttackIntent ai && ai.unitTag().equals("s-1")))
            .hasSize(1);
    }

    // ---- DEFEND ----

    @Test
    void defendAllUnitsMovesToNexus() {
        Point2d nexusPos = new Point2d(8, 8);
        var cf = caseFile("DEFEND",
            List.of(stalker("s-0", new Point2d(10, 10), 80, 80),
                    stalker("s-1", new Point2d(20, 20), 80, 80)),
            List.of(enemy(new Point2d(12, 12))),
            new Point2d(12, 12));
        cf.put(StarCraftCaseFile.MY_BUILDINGS, List.of(nexus(nexusPos)));
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending())
            .hasSize(2)
            .allMatch(i -> i instanceof MoveIntent mi && mi.targetLocation().equals(nexusPos));
    }

    // ---- MACRO ----

    @Test
    void macroProducesNoIntents() {
        var cf = caseFile("MACRO",
            List.of(stalker("s-0", new Point2d(10, 10), 80, 80)),
            List.of(enemy(new Point2d(12, 12))),
            null);
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ---- Edge cases ----

    @Test
    void noArmyProducesNoIntents() {
        var cf = caseFile("ATTACK", List.of(), List.of(enemy(new Point2d(12, 12))), new Point2d(12, 12));
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending()).isEmpty();
    }

    @Test
    void noEnemiesProducesNoIntents() {
        var cf = caseFile("ATTACK",
            List.of(stalker("s-0", new Point2d(10, 10), 80, 80)),
            List.of(), null);
        tacticsTask.execute(cf);
        assertThat(intentQueue.pending()).isEmpty();
    }

    // ---- Helpers ----

    private DefaultCaseFile caseFile(String strategy, List<Unit> army,
                                     List<Unit> enemies, Point2d nearestThreat) {
        var cf = new DefaultCaseFile("test-" + System.nanoTime(), "starcraft-game", null, null);
        cf.put(StarCraftCaseFile.STRATEGY,      strategy);
        cf.put(StarCraftCaseFile.ARMY,          army);
        cf.put(StarCraftCaseFile.ENEMY_UNITS,   enemies);
        cf.put(StarCraftCaseFile.MY_BUILDINGS,  List.of());
        cf.put(StarCraftCaseFile.READY,         Boolean.TRUE);
        if (nearestThreat != null) cf.put(StarCraftCaseFile.NEAREST_THREAT, nearestThreat);
        return cf;
    }

    private Unit stalker(String tag, Point2d pos, int health, int maxHealth) {
        return new Unit(tag, UnitType.STALKER, pos, health, maxHealth);
    }

    private Unit enemy(Point2d pos) {
        return new Unit("e-0", UnitType.ZEALOT, pos, 100, 100);
    }

    private Building nexus(Point2d pos) {
        return new Building("n-0", BuildingType.NEXUS, pos, 1500, 1500, true);
    }
}
```

- [ ] **Step 2: Demote BasicTacticsTask**

Edit `src/main/java/org/acme/starcraft/plugin/BasicTacticsTask.java` — remove the two CDI annotations so it is no longer the active bean:

Remove lines:
```java
@ApplicationScoped
@CaseType("starcraft-game")
```

The class stays intact (tests for it remain green — they instantiate it directly with `new`).

- [ ] **Step 3: Implement DroolsTacticsTask**

Create `src/main/java/org/acme/starcraft/plugin/DroolsTacticsTask.java`:

```java
package org.acme.starcraft.plugin;

import io.casehub.annotation.CaseType;
import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.TacticsTask;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.drools.TacticsRuleUnit;
import org.acme.starcraft.plugin.tactics.GoapAction;
import org.acme.starcraft.plugin.tactics.GoapPlanner;
import org.acme.starcraft.plugin.tactics.WorldState;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.AttackIntent;
import org.acme.starcraft.sc2.intent.MoveIntent;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.jboss.logging.Logger;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Drools-backed GOAP {@link TacticsTask} — third R&D integration.
 *
 * <p>Each tick:
 * <ol>
 *   <li>DEFEND: bypasses GOAP — emits {@link MoveIntent} to Nexus for all units.</li>
 *   <li>ATTACK: fires {@link TacticsRuleUnit} to classify units into groups (Phase 1)
 *       and emit applicable action names (Phase 2).</li>
 *   <li>Java A* ({@link GoapPlanner}) finds the cheapest action sequence per group.</li>
 *   <li>First action in each plan is dispatched as an Intent.</li>
 * </ol>
 *
 * <p>Replaces {@link BasicTacticsTask} as the active CDI bean.
 */
@ApplicationScoped
@CaseType("starcraft-game")
public class DroolsTacticsTask implements TacticsTask {

    static final Point2d MAP_CENTER = new Point2d(64, 64);
    static final double STALKER_RANGE = 6.0;

    /** Hardcoded action templates — preconditions, effects, cost. */
    private static final Map<String, GoapAction> ACTION_TEMPLATES = Map.of(
        "RETREAT",        new GoapAction("RETREAT",
            Map.of("lowHealth", true),
            Map.of("unitSafe", true), 1),
        "ATTACK",         new GoapAction("ATTACK",
            Map.of("inRange", true, "enemyVisible", true),
            Map.of("enemyEliminated", true), 2),
        "MOVE_TO_ENGAGE", new GoapAction("MOVE_TO_ENGAGE",
            Map.of("enemyVisible", true, "inRange", false),
            Map.of("inRange", true), 1)
    );

    private static final Logger log = Logger.getLogger(DroolsTacticsTask.class);

    private final RuleUnit<TacticsRuleUnit> ruleUnit;
    private final IntentQueue intentQueue;
    private final GoapPlanner planner = new GoapPlanner();

    @Inject
    public DroolsTacticsTask(RuleUnit<TacticsRuleUnit> ruleUnit, IntentQueue intentQueue) {
        this.ruleUnit    = ruleUnit;
        this.intentQueue = intentQueue;
    }

    @Override public String getId()   { return "tactics.drools-goap"; }
    @Override public String getName() { return "Drools GOAP Tactics"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys()  { return Set.of(); }

    @Override
    @SuppressWarnings("unchecked")
    public void execute(CaseFile caseFile) {
        String strategy    = caseFile.get(StarCraftCaseFile.STRATEGY,      String.class).orElse("MACRO");
        List<Unit> army    = (List<Unit>)     caseFile.get(StarCraftCaseFile.ARMY,         List.class).orElse(List.of());
        List<Unit> enemies = (List<Unit>)     caseFile.get(StarCraftCaseFile.ENEMY_UNITS,  List.class).orElse(List.of());
        List<Building> bld = (List<Building>) caseFile.get(StarCraftCaseFile.MY_BUILDINGS, List.class).orElse(List.of());
        Point2d threat     = caseFile.get(StarCraftCaseFile.NEAREST_THREAT, Point2d.class).orElse(null);

        if (army.isEmpty()) return;

        // DEFEND bypasses GOAP
        if ("DEFEND".equals(strategy)) {
            dispatchDefend(army, bld);
            return;
        }

        // MACRO: hold position
        if (!"ATTACK".equals(strategy)) return;

        // No enemies — nothing to do
        if (enemies.isEmpty()) return;

        // Step 1: compute in-range tags in Java (avoid float arithmetic in DRL)
        Set<String> inRangeSet = computeInRangeTags(army, enemies);

        // Step 2: fire Drools — group classification + action emission
        TacticsRuleUnit data = buildRuleUnit(army, enemies, inRangeSet, strategy);
        try (RuleUnitInstance<TacticsRuleUnit> instance = ruleUnit.createInstance(data)) {
            instance.fire();
        }

        // Step 3: parse groups and build available actions list
        Map<String, GroupInfo> groups = parseGroups(data.getGroupDecisions());
        List<GoapAction> availableActions = parseActions(data.getActionDecisions());

        if (groups.isEmpty() || availableActions.isEmpty()) return;

        // Step 4: plan and dispatch per group
        for (Map.Entry<String, GroupInfo> entry : groups.entrySet()) {
            String    groupId   = entry.getKey();
            GroupInfo groupInfo = entry.getValue();
            WorldState ws       = buildWorldState(groupId, !enemies.isEmpty());
            List<GoapAction> plan = planner.plan(ws, groupInfo.goalCondition(), availableActions);
            if (!plan.isEmpty()) {
                dispatch(plan.get(0), groupInfo.unitTags(), threat, bld);
            }
            log.debugf("[DROOLS-GOAP] group=%s goal=%s plan=%s units=%d",
                groupId, groupInfo.goalCondition(),
                plan.stream().map(GoapAction::name).toList(),
                groupInfo.unitTags().size());
        }
    }

    // ---- Internal helpers ----

    private Set<String> computeInRangeTags(List<Unit> army, List<Unit> enemies) {
        Set<String> result = new HashSet<>();
        for (Unit unit : army) {
            for (Unit enemy : enemies) {
                if (distance(unit.position(), enemy.position()) <= STALKER_RANGE) {
                    result.add(unit.tag());
                    break;
                }
            }
        }
        return result;
    }

    private double distance(Point2d a, Point2d b) {
        double dx = a.x() - b.x();
        double dy = a.y() - b.y();
        return Math.sqrt(dx * dx + dy * dy);
    }

    private TacticsRuleUnit buildRuleUnit(List<Unit> army, List<Unit> enemies,
                                          Set<String> inRangeTags, String strategy) {
        TacticsRuleUnit data = new TacticsRuleUnit();
        data.setStrategyGoal(strategy);
        army.forEach(data.getArmy()::add);
        enemies.forEach(data.getEnemies()::add);
        inRangeTags.forEach(data.getInRangeTags()::add);
        return data;
    }

    /** Parses "groupId:GOAL:unitTag" strings into a map of groupId → GroupInfo. */
    private Map<String, GroupInfo> parseGroups(List<String> groupDecisions) {
        Map<String, GroupInfo> groups = new LinkedHashMap<>();
        for (String decision : groupDecisions) {
            String[] parts = decision.split(":", 3);
            if (parts.length < 3) continue;
            String groupId      = parts[0];
            String goalKey      = goalConditionKey(parts[1]);
            String unitTag      = parts[2];
            groups.computeIfAbsent(groupId, k -> new GroupInfo(goalKey, new ArrayList<>()))
                  .unitTags().add(unitTag);
        }
        return groups;
    }

    /** Maps GOAP goal name to WorldState condition key. */
    private String goalConditionKey(String goalName) {
        return switch (goalName) {
            case "UNIT_SAFE"         -> "unitSafe";
            case "ENEMY_ELIMINATED"  -> "enemyEliminated";
            default                  -> goalName.toLowerCase();
        };
    }

    /** Parses "ACTION_NAME:cost" strings into GoapAction list using hardcoded templates. */
    private List<GoapAction> parseActions(List<String> actionDecisions) {
        return actionDecisions.stream()
            .map(d -> d.split(":")[0])
            .map(ACTION_TEMPLATES::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }

    /** Builds initial WorldState for a group based on its classification. */
    private WorldState buildWorldState(String groupId, boolean enemyVisible) {
        return switch (groupId) {
            case "low-health"    -> new WorldState(Map.of(
                "lowHealth",        true,
                "enemyVisible",     enemyVisible,
                "inRange",          false,
                "unitSafe",         false,
                "enemyEliminated",  false));
            case "in-range"      -> new WorldState(Map.of(
                "lowHealth",        false,
                "enemyVisible",     true,
                "inRange",          true,
                "unitSafe",         false,
                "enemyEliminated",  false));
            case "out-of-range"  -> new WorldState(Map.of(
                "lowHealth",        false,
                "enemyVisible",     true,
                "inRange",          false,
                "unitSafe",         false,
                "enemyEliminated",  false));
            default              -> new WorldState(Map.of("enemyEliminated", false));
        };
    }

    private void dispatch(GoapAction action, List<String> unitTags,
                          Point2d threat, List<Building> buildings) {
        switch (action.name()) {
            case "ATTACK" -> {
                Point2d target = threat != null ? threat : MAP_CENTER;
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
        }
    }

    private void dispatchDefend(List<Unit> army, List<Building> buildings) {
        Point2d rally = buildings.stream()
            .filter(b -> b.type() == BuildingType.NEXUS)
            .findFirst()
            .map(Building::position)
            .orElse(MAP_CENTER);
        army.forEach(unit -> intentQueue.add(new MoveIntent(unit.tag(), rally)));
    }

    /** Transient grouping result — not persisted between ticks. */
    private record GroupInfo(String goalCondition, List<String> unitTags) {}
}
```

- [ ] **Step 4: Run the integration test**

```bash
mvn test -Dtest=DroolsTacticsTaskIT -q 2>&1 | tail -10
```

Expected: all 8 tests PASS.

- [ ] **Step 5: Run the full test suite**

```bash
mvn test -q 2>&1 | tail -10
```

Expected: all tests PASS (count increases from 128). Verify `BasicTacticsTaskTest` still passes — it instantiates `BasicTacticsTask` directly with `new`, so removing the CDI annotations doesn't affect it.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/DroolsTacticsTask.java \
        src/main/java/org/acme/starcraft/plugin/BasicTacticsTask.java \
        src/test/java/org/acme/starcraft/plugin/DroolsTacticsTaskIT.java
git commit -m "feat(tactics): DroolsTacticsTask — Drools GOAP unit group planning

Drools classifies units into groups (low-health/in-range/out-of-range) and
emits applicable actions; Java A* finds cheapest plan per group; first action
dispatched as AttackIntent or MoveIntent. BasicTacticsTask demoted.

Closes #3, Refs #4"
```

---

## Task 5: Documentation + NATIVE.md

**Files:**
- Modify: `docs/library-research.md`
- Modify: `NATIVE.md`

- [ ] **Step 1: Update library-research.md**

In `docs/library-research.md` §2.3 (GOAP), add at the top of the section:

```markdown
**Status: CHOSEN** — implemented in Phase 3 as `DroolsTacticsTask` (Drools action compiler + Java A*).
See `docs/superpowers/specs/2026-04-08-drools-goap-tactics-design.md`.

gdx-ai (§2.2) deferred — JVM-only dep, no GraalVM metadata. Revisit for Phase 4 pathfinding.
```

- [ ] **Step 2: Update NATIVE.md**

In the `## Dependencies` table, add a row confirming no new non-native dependency:

```markdown
| drools-goap tactics | — | ✅ Native-safe | Pure Java A* planner; no new dependency; Drools already tracked via casehub-core |
```

- [ ] **Step 3: Commit**

```bash
git add docs/library-research.md NATIVE.md
git commit -m "docs: mark GOAP chosen in library-research; confirm native-safe in NATIVE.md

Closes #2, Closes #5"
```

---

## Self-Review

**Spec coverage check:**

| Spec section | Covered by |
|---|---|
| Architecture flow | Task 4 (DroolsTacticsTask.execute) |
| WorldState | Task 1 |
| GoapAction | Task 1 |
| GoapPlanner A* | Task 2 |
| TacticsRuleUnit fields | Task 3 |
| StarCraftTactics.drl Phase 1 | Task 3 |
| StarCraftTactics.drl Phase 2 | Task 3 |
| DroolsTacticsTask orchestration | Task 4 |
| DEFEND bypass GOAP | Task 4 (`dispatchDefend`) |
| Demote BasicTacticsTask | Task 4 Step 2 |
| GoapPlannerTest (7 scenarios) | Task 1 + Task 2 |
| TacticsRuleUnitTest (10 scenarios) | Task 3 |
| DroolsTacticsTaskIT (8 scenarios) | Task 4 |
| NATIVE.md update | Task 5 |
| library-research.md update | Task 5 |

**All spec requirements covered. No placeholders. Type names consistent across all tasks.**
