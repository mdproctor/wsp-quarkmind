# Drools-Based DominanceWeightStrategy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #240 — Drools-based DominanceWeightStrategy — rule-driven weight resolution
**Issue group:** #240

**Goal:** Add a Drools-based `DominanceWeightStrategy` that resolves dominance
weights via DRL rules matching enemy archetype assessments and game phase,
composing weight modifiers additively onto a temporal baseline.

**Architecture:** A new `DominanceWeightRuleUnit` with DRL rules emits
`WeightModifier` deltas that are applied to an `AnchorInterpolator` baseline.
`WeightContext` is enriched with `List<EnemyPatternAssessment>` so strategies
receive pattern signals via a pure function. `MultiFactorDominanceAssessor`
reads assessments from `ScoutingIntelBroker` synchronously.

**Tech Stack:** Drools Rule Units, Quarkus CDI, plain JUnit

## Global Constraints

- `DominanceWeights` must always sum to 1.0 (validated in record constructor)
- No dimension weight may go below `MINIMUM_WEIGHT` (0.05) after modifier application
- `resolve()` must remain a pure function — no mutable cached state
- DRL file lives alongside `StarCraftStrategy.drl` in `io.quarkmind.plugin.drools`
- `WeightModifier` belongs in `io.quarkmind.agent` (strategy layer, not domain)
- Default salience (0) for all rules — modifier pattern is additive
- Default weight strategy remains `temporal` — `drools` activates via config
- All tests plain JUnit — no `@QuarkusTest` needed

---

### Task 1: SPI Foundation and Signal Plumbing

Expand `WeightContext` with pattern assessments, extract `MINIMUM_WEIGHT`
to the SPI interface, inject `ScoutingIntelBroker` into the assessor,
and update all existing callers. This is foundation — everything else
builds on it.

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/WeightContext.java`
- Modify: `src/main/java/io/quarkmind/agent/DominanceWeightStrategy.java`
- Modify: `src/main/java/io/quarkmind/agent/SituationalDominanceWeightStrategy.java`
- Modify: `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java`
- Modify: `src/test/java/io/quarkmind/agent/TemporalDominanceWeightStrategyTest.java`
- Modify: `src/test/java/io/quarkmind/agent/SituationalDominanceWeightStrategyTest.java`
- Modify: `src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java`

**Interfaces:**
- Produces: `WeightContext(long gameFrame, String currentPhase, List<EnemyPatternAssessment> patternAssessments)`
- Produces: `DominanceWeightStrategy.MINIMUM_WEIGHT` constant (0.05)

- [ ] **Step 1: Expand WeightContext record**

Add `List<EnemyPatternAssessment> patternAssessments` as third field:

```java
public record WeightContext(
    long gameFrame,
    String currentPhase,
    List<EnemyPatternAssessment> patternAssessments
) {}
```

Use `ide_edit_member` on `WeightContext.java`, member=`WeightContext`, to
replace the record declaration. Add `import io.quarkmind.domain.EnemyPatternAssessment;`
and `import java.util.List;`.

- [ ] **Step 2: Add MINIMUM_WEIGHT to DominanceWeightStrategy interface**

```java
double MINIMUM_WEIGHT = 0.05;
```

Use `ide_insert_member` on `DominanceWeightStrategy.java`, position=`first`,
to add the constant before `id()`.

- [ ] **Step 3: Update SituationalDominanceWeightStrategy to use MINIMUM_WEIGHT**

Replace the `FLOOR` field with `DominanceWeightStrategy.MINIMUM_WEIGHT`.

Use `ide_refactor_safe_delete` on `FLOOR` field (line 17), then use
`ide_replace_text_in_file` to replace all `FLOOR` with
`DominanceWeightStrategy.MINIMUM_WEIGHT` in the file (2 occurrences in
the `resolve()` method: `Math.max(FLOOR, ...)` becomes
`Math.max(MINIMUM_WEIGHT, ...)`).

Note: since the class implements `DominanceWeightStrategy`, the constant
is inherited — use `MINIMUM_WEIGHT` directly (no qualifier needed).

- [ ] **Step 4: Update MultiFactorDominanceAssessor — inject ScoutingIntelBroker**

Add `ScoutingIntelBroker` field and update both constructors:

CDI constructor (line 37): add `ScoutingIntelBroker broker` parameter.
Test constructor (line 56): add `ScoutingIntelBroker broker` parameter.
Add field: `private final ScoutingIntelBroker broker;`

In `assess()` method, update the `WeightContext` construction (line 104)
to read pattern assessments from the broker:

```java
List<EnemyPatternAssessment> assessments = broker != null
    ? broker.current(ScoutingIntelType.PATTERN_ASSESSMENT,
                     ScoutingIntelPayload.PatternAssessment.class)
        .map(ScoutingIntelPayload.PatternAssessment::assessments)
        .orElse(List.of())
    : List.of();

WeightContext ctx = new WeightContext(state.gameFrame(),
    phase != null ? phase.phase() : null,
    assessments);
```

Add imports: `ScoutingIntelBroker`, `ScoutingIntelType`,
`ScoutingIntelPayload`, `EnemyPatternAssessment`, `java.util.List`.

- [ ] **Step 5: Update all existing test WeightContext constructor calls**

Every `new WeightContext(frame, phase)` becomes
`new WeightContext(frame, phase, List.of())`.

Files and call sites:
- `TemporalDominanceWeightStrategyTest.java`: lines 26, 36, 37
- `SituationalDominanceWeightStrategyTest.java`: lines 25, 34, 41, 48, 55, 61, 69, 79, 89, 99
- `MultiFactorDominanceAssessorTest.java`: line 175

Add `import java.util.List;` to each test file if not already present.

Update `MultiFactorDominanceAssessorTest` test constructor call (line 19-20)
to pass `null` as the broker parameter.

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `mvn test -q`
Expected: All ~1547 tests pass. Zero failures.

- [ ] **Step 7: Verify with IDE diagnostics**

Run `ide_diagnostics` on all modified files to confirm no compilation errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/WeightContext.java \
  src/main/java/io/quarkmind/agent/DominanceWeightStrategy.java \
  src/main/java/io/quarkmind/agent/SituationalDominanceWeightStrategy.java \
  src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java \
  src/test/java/io/quarkmind/agent/TemporalDominanceWeightStrategyTest.java \
  src/test/java/io/quarkmind/agent/SituationalDominanceWeightStrategyTest.java \
  src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#240): enrich WeightContext with pattern assessments, extract MINIMUM_WEIGHT to SPI

Refs #240"
```

---

### Task 2: WeightModifier Record and applyModifiers Logic

Create the `WeightModifier` record and the modifier application pipeline
(accumulate deltas, apply to baseline, floor, normalise). Test exhaustively
as a static method — no Drools needed yet.

**Files:**
- Create: `src/main/java/io/quarkmind/agent/WeightModifier.java`
- Create: `src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java` (partial — applyModifiers only)
- Create: `src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java`

**Interfaces:**
- Produces: `WeightModifier(double economyDelta, double armyDelta, double techDelta, double basesDelta, String reason)`
- Produces: `DroolsDominanceWeightStrategy.applyModifiers(DominanceWeights baseline, List<WeightModifier> modifiers) → DominanceWeights`

- [ ] **Step 1: Write failing tests for applyModifiers**

Create `DroolsDominanceWeightStrategyTest.java` with tests:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class DroolsDominanceWeightStrategyTest {

    private static final DominanceWeights BASELINE =
        new DominanceWeights(0.30, 0.30, 0.20, 0.20);

    @Test
    void applyModifiers_emptyList_returnsBaseline() {
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of());
        assertEquals(BASELINE, result);
    }

    @Test
    void applyModifiers_singleModifier_adjustsAndNormalises() {
        var mod = new WeightModifier(0.0, +0.10, 0.0, 0.0, "army boost");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(mod));
        assertTrue(result.army() > BASELINE.army());
        assertEquals(1.0, result.economy() + result.army()
            + result.tech() + result.bases(), 0.001);
    }

    @Test
    void applyModifiers_multipleModifiers_stackAdditively() {
        var mod1 = new WeightModifier(-0.05, +0.10, 0.0, 0.0, "rush");
        var mod2 = new WeightModifier(-0.05, +0.05, 0.0, 0.0, "phase");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(mod1, mod2));
        // economy: 0.30 - 0.05 - 0.05 = 0.20
        // army:    0.30 + 0.10 + 0.05 = 0.45
        // After normalise: sum = 0.20 + 0.45 + 0.20 + 0.20 = 1.05
        assertTrue(result.army() > result.economy());
        assertEquals(1.0, result.economy() + result.army()
            + result.tech() + result.bases(), 0.001);
    }

    @Test
    void applyModifiers_floorPreventsZero() {
        var mod = new WeightModifier(-0.50, 0.0, 0.0, 0.0, "extreme");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(mod));
        assertTrue(result.economy() >= DominanceWeightStrategy.MINIMUM_WEIGHT);
        assertEquals(1.0, result.economy() + result.army()
            + result.tech() + result.bases(), 0.001);
    }

    @Test
    void applyModifiers_negativeResult_flooredAtMinimum() {
        // Push economy far below zero
        var mod = new WeightModifier(-1.0, 0.0, 0.0, 0.0, "extreme negative");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(mod));
        assertTrue(result.economy() >= DominanceWeightStrategy.MINIMUM_WEIGHT);
    }

    @Test
    void applyModifiers_conflictingModifiers_netEffect() {
        var boost = new WeightModifier(+0.20, 0.0, 0.0, 0.0, "boost");
        var cut = new WeightModifier(-0.10, 0.0, 0.0, 0.0, "cut");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(boost, cut));
        // economy: 0.30 + 0.20 - 0.10 = 0.40 (before normalise)
        assertTrue(result.economy() > BASELINE.economy());
        assertEquals(1.0, result.economy() + result.army()
            + result.tech() + result.bases(), 0.001);
    }

    @Test
    void applyModifiers_allDimensionsClamped_stillSumToOne() {
        // Push all dimensions very negative — each floors at MINIMUM_WEIGHT
        var mod = new WeightModifier(-1.0, -1.0, -1.0, -1.0, "all negative");
        DominanceWeights result = DroolsDominanceWeightStrategy
            .applyModifiers(BASELINE, List.of(mod));
        assertEquals(0.25, result.economy(), 0.001);
        assertEquals(0.25, result.army(), 0.001);
        assertEquals(0.25, result.tech(), 0.001);
        assertEquals(0.25, result.bases(), 0.001);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=DroolsDominanceWeightStrategyTest -q`
Expected: FAIL — classes not found.

- [ ] **Step 3: Create WeightModifier record**

Create `src/main/java/io/quarkmind/agent/WeightModifier.java`:

```java
package io.quarkmind.agent;

public record WeightModifier(
    double economyDelta,
    double armyDelta,
    double techDelta,
    double basesDelta,
    String reason
) {}
```

- [ ] **Step 4: Implement applyModifiers in DroolsDominanceWeightStrategy**

Create `src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java`
with the static method only (CDI bean wiring comes in Task 3):

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;

import java.util.List;

public class DroolsDominanceWeightStrategy {

    static DominanceWeights applyModifiers(DominanceWeights baseline,
                                           List<WeightModifier> modifiers) {
        if (modifiers.isEmpty()) return baseline;

        double economy = baseline.economy();
        double army = baseline.army();
        double tech = baseline.tech();
        double bases = baseline.bases();

        for (WeightModifier mod : modifiers) {
            economy += mod.economyDelta();
            army += mod.armyDelta();
            tech += mod.techDelta();
            bases += mod.basesDelta();
        }

        economy = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, economy);
        army = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, army);
        tech = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, tech);
        bases = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, bases);

        double sum = economy + army + tech + bases;
        return new DominanceWeights(
            economy / sum, army / sum, tech / sum, bases / sum);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=DroolsDominanceWeightStrategyTest -q`
Expected: PASS — all 7 tests green.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/agent/WeightModifier.java \
  src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java \
  src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#240): WeightModifier record and applyModifiers logic with tests

Refs #240"
```

---

### Task 3: DominanceWeightRuleUnit, DRL Rules, and Strategy Bean

Build the Drools rule unit, write the DRL rules, complete the
`DroolsDominanceWeightStrategy` CDI bean, and write integration tests
for the full pipeline.

**Files:**
- Create: `src/main/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnit.java`
- Create: `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl`
- Modify: `src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java` (complete bean)
- Create: `src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java`
- Modify: `src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java` (add end-to-end tests)
- Modify: `CLAUDE.md` (add test classes to test list)

**Interfaces:**
- Consumes: `WeightContext(long, String, List<EnemyPatternAssessment>)` from Task 1
- Consumes: `WeightModifier` record from Task 2
- Consumes: `DroolsDominanceWeightStrategy.applyModifiers()` from Task 2
- Produces: `DroolsDominanceWeightStrategy` CDI bean with `id()="drools"` and `resolve(WeightContext)`

- [ ] **Step 1: Write failing rule unit tests**

Create `src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java`:

```java
package io.quarkmind.plugin.drools;

import io.quarkmind.agent.WeightModifier;
import io.quarkmind.domain.EnemyArchetype;
import io.quarkmind.domain.EnemyPatternAssessment;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.drools.ruleunits.api.RuleUnitProvider;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class DominanceWeightRuleUnitTest {

    private List<WeightModifier> fire(DominanceWeightRuleUnit data) {
        try (RuleUnitInstance<DominanceWeightRuleUnit> instance =
                 RuleUnitProvider.get().createRuleUnitInstance(data)) {
            instance.fire();
        }
        return data.getModifiers();
    }

    @Test
    void noSignals_noModifiers() {
        var data = new DominanceWeightRuleUnit();
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.isEmpty());
    }

    @Test
    void highConfidenceRush_emitsArmyBoost() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MARINE_RUSH, 0.7, 3000, "test"));
        List<WeightModifier> mods = fire(data);
        assertFalse(mods.isEmpty());
        double totalArmyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(totalArmyDelta > 0, "Army delta should be positive for rush");
        double totalEconomyDelta = mods.stream()
            .mapToDouble(WeightModifier::economyDelta).sum();
        assertTrue(totalEconomyDelta < 0, "Economy delta should be negative for rush");
    }

    @Test
    void moderateConfidenceRush_emitsSmallerBoost() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.ZERG_ZERGLING_RUSH, 0.4, 2000, "test"));
        List<WeightModifier> mods = fire(data);
        double totalArmyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(totalArmyDelta > 0);
        assertTrue(totalArmyDelta < 0.15,
            "Moderate rush should produce smaller boost than high confidence");
    }

    @Test
    void rushBelowThreshold_noModifier() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MARINE_RUSH, 0.29, 3000, "test"));
        List<WeightModifier> mods = fire(data);
        boolean hasRushMod = mods.stream()
            .anyMatch(m -> m.reason().toLowerCase().contains("rush"));
        assertFalse(hasRushMod);
    }

    @Test
    void pushArchetype_emitsTechAndArmyBoost() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_BIO_TIMING, 0.6, 5000, "test"));
        List<WeightModifier> mods = fire(data);
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        double techDelta = mods.stream()
            .mapToDouble(WeightModifier::techDelta).sum();
        assertTrue(armyDelta > 0);
        assertTrue(techDelta > 0);
    }

    @Test
    void harassArchetype_emitsTechBoost() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_BANSHEE_HARASS, 0.5, 6000, "test"));
        List<WeightModifier> mods = fire(data);
        double techDelta = mods.stream()
            .mapToDouble(WeightModifier::techDelta).sum();
        assertTrue(techDelta > 0);
    }

    @Test
    void macroArchetype_boostsEconomyAndTech() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.ZERG_MACRO, 0.6, 4000, "test"));
        List<WeightModifier> mods = fire(data);
        double economyDelta = mods.stream()
            .mapToDouble(WeightModifier::economyDelta).sum();
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(economyDelta > 0, "Macro should boost economy");
        assertTrue(armyDelta < 0, "Macro should reduce army weight");
    }

    @Test
    void phaseOnly_defensiveHold_boostsArmy() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("DEFENSIVE_HOLD");
        List<WeightModifier> mods = fire(data);
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(armyDelta > 0);
    }

    @Test
    void phaseOnly_earlyMacro_boostsEconomy() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("EARLY_MACRO");
        List<WeightModifier> mods = fire(data);
        double economyDelta = mods.stream()
            .mapToDouble(WeightModifier::economyDelta).sum();
        assertTrue(economyDelta > 0);
    }

    @Test
    void phaseOnly_midSkirmish_boostsArmy() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("MID_SKIRMISH");
        List<WeightModifier> mods = fire(data);
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(armyDelta > 0);
    }

    @Test
    void phaseOnly_earlyAggression_boostsArmy() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("EARLY_AGGRESSION");
        List<WeightModifier> mods = fire(data);
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(armyDelta > 0);
    }

    @Test
    void combinedSignal_rushAndDefensiveHold_stacksModifiers() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MARINE_RUSH, 0.7, 3000, "test"));
        data.getPhaseStore().add("DEFENSIVE_HOLD");
        List<WeightModifier> mods = fire(data);
        // Should have rush modifier + phase modifier + combined modifier
        assertTrue(mods.size() >= 3,
            "Expected rush + phase + combined modifiers, got " + mods.size());
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        assertTrue(armyDelta > 0.15,
            "Combined army delta should exceed individual rush modifier");
    }

    @Test
    void combinedSignal_macroAndEarlyMacro_stacksModifiers() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.PROTOSS_MACRO, 0.6, 4000, "test"));
        data.getPhaseStore().add("EARLY_MACRO");
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.size() >= 3);
        double economyDelta = mods.stream()
            .mapToDouble(WeightModifier::economyDelta).sum();
        assertTrue(economyDelta > 0.10,
            "Combined economy delta should exceed individual macro modifier");
    }

    @Test
    void combinedSignal_pushAndMidSkirmish_stacksModifiers() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MECH_PUSH, 0.6, 7000, "test"));
        data.getPhaseStore().add("MID_SKIRMISH");
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.size() >= 3);
        double armyDelta = mods.stream()
            .mapToDouble(WeightModifier::armyDelta).sum();
        double techDelta = mods.stream()
            .mapToDouble(WeightModifier::techDelta).sum();
        assertTrue(armyDelta > 0);
        assertTrue(techDelta > 0);
    }

    @Test
    void combinedSignal_rushAndEarlyAggression_stacksModifiers() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.ZERG_ZERGLING_RUSH, 0.6, 2000, "test"));
        data.getPhaseStore().add("EARLY_AGGRESSION");
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.size() >= 3);
    }

    @Test
    void multipleArchetypes_allFireAndStack() {
        var data = new DominanceWeightRuleUnit();
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MARINE_RUSH, 0.5, 3000, "test"));
        data.getPatternStore().add(new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_BIO_TIMING, 0.4, 3000, "test"));
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.size() >= 2, "Both archetypes should produce modifiers");
    }

    @Test
    void transitioning_noPhaseModifier() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("TRANSITIONING");
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.isEmpty(),
            "TRANSITIONING should produce no modifiers");
    }

    @Test
    void unknownPhase_noPhaseModifier() {
        var data = new DominanceWeightRuleUnit();
        data.getPhaseStore().add("SOME_UNKNOWN_PHASE");
        List<WeightModifier> mods = fire(data);
        assertTrue(mods.isEmpty());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=DominanceWeightRuleUnitTest -q`
Expected: FAIL — `DominanceWeightRuleUnit` not found.

- [ ] **Step 3: Create DominanceWeightRuleUnit**

Create `src/main/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnit.java`:

```java
package io.quarkmind.plugin.drools;

import io.quarkmind.agent.WeightModifier;
import io.quarkmind.domain.EnemyPatternAssessment;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;

import java.util.ArrayList;
import java.util.List;

public class DominanceWeightRuleUnit implements RuleUnitData {

    private final DataStore<EnemyPatternAssessment> patternStore = DataSource.createStore();
    private final DataStore<String>                 phaseStore   = DataSource.createStore();

    private final List<WeightModifier> modifiers = new ArrayList<>();

    public DataStore<EnemyPatternAssessment> getPatternStore() { return patternStore; }
    public DataStore<String>                 getPhaseStore()   { return phaseStore; }
    public List<WeightModifier>              getModifiers()    { return modifiers; }
}
```

- [ ] **Step 4: Write DominanceWeightAdjustment.drl**

Create `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl`:

```drl
package io.quarkmind.plugin.drools;
unit DominanceWeightRuleUnit;

import io.quarkmind.agent.WeightModifier;
import io.quarkmind.domain.EnemyArchetype;

// --- RUSH RESPONSE ---

rule "Rush response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().name().contains("RUSH")]
then
    modifiers.add(new WeightModifier(-0.10, +0.15, -0.05, 0.00,
        "High-confidence rush: " + $arch));
end

rule "Rush response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().name().contains("RUSH")]
then
    modifiers.add(new WeightModifier(-0.05, +0.08, -0.03, 0.00,
        "Moderate rush: " + $arch));
end

// --- PUSH RESPONSE ---

rule "Push response — bio timing or mech push"
when
    /patternStore[$arch: archetype(), confidence >= 0.5,
                  archetype() == EnemyArchetype.TERRAN_BIO_TIMING
                  || archetype() == EnemyArchetype.TERRAN_MECH_PUSH]
then
    modifiers.add(new WeightModifier(-0.10, +0.10, +0.05, -0.05,
        "Push detected: " + $arch));
end

// --- HARASS RESPONSE ---

rule "Harass response — banshee"
when
    /patternStore[$arch: archetype(), confidence >= 0.3,
                  archetype() == EnemyArchetype.TERRAN_BANSHEE_HARASS]
then
    modifiers.add(new WeightModifier(-0.10, +0.05, +0.10, -0.05,
        "Harass detected: " + $arch));
end

// --- MACRO RESPONSE ---

rule "Macro response"
when
    /patternStore[$arch: archetype(), confidence >= 0.5,
                  archetype().name().contains("MACRO")]
then
    modifiers.add(new WeightModifier(+0.08, -0.10, +0.05, -0.03,
        "Macro detected: " + $arch));
end

// --- PHASE MODIFIERS ---

rule "Phase: DEFENSIVE_HOLD"
when
    /phaseStore[this == "DEFENSIVE_HOLD"]
then
    modifiers.add(new WeightModifier(-0.10, +0.15, -0.05, 0.00,
        "Phase: DEFENSIVE_HOLD"));
end

rule "Phase: EARLY_AGGRESSION"
when
    /phaseStore[this == "EARLY_AGGRESSION"]
then
    modifiers.add(new WeightModifier(-0.05, +0.10, -0.05, 0.00,
        "Phase: EARLY_AGGRESSION"));
end

rule "Phase: EARLY_MACRO"
when
    /phaseStore[this == "EARLY_MACRO"]
then
    modifiers.add(new WeightModifier(+0.10, -0.10, +0.05, -0.05,
        "Phase: EARLY_MACRO"));
end

rule "Phase: MID_SKIRMISH"
when
    /phaseStore[this == "MID_SKIRMISH"]
then
    modifiers.add(new WeightModifier(-0.05, +0.10, -0.05, 0.00,
        "Phase: MID_SKIRMISH"));
end

// --- COMBINED SIGNALS ---

rule "Combined: Rush + DEFENSIVE_HOLD"
when
    /patternStore[confidence >= 0.5, archetype().name().contains("RUSH")]
    /phaseStore[this == "DEFENSIVE_HOLD"]
then
    modifiers.add(new WeightModifier(0.00, +0.05, 0.00, 0.00,
        "Combined: rush + defensive hold"));
end

rule "Combined: Macro + EARLY_MACRO"
when
    /patternStore[confidence >= 0.5, archetype().name().contains("MACRO")]
    /phaseStore[this == "EARLY_MACRO"]
then
    modifiers.add(new WeightModifier(+0.05, 0.00, 0.00, 0.00,
        "Combined: macro + early macro"));
end

rule "Combined: Push + MID_SKIRMISH"
when
    /patternStore[confidence >= 0.5,
                  archetype() == EnemyArchetype.TERRAN_BIO_TIMING
                  || archetype() == EnemyArchetype.TERRAN_MECH_PUSH]
    /phaseStore[this == "MID_SKIRMISH"]
then
    modifiers.add(new WeightModifier(0.00, +0.05, +0.03, 0.00,
        "Combined: push + mid skirmish"));
end

rule "Combined: Rush + EARLY_AGGRESSION"
when
    /patternStore[confidence >= 0.5, archetype().name().contains("RUSH")]
    /phaseStore[this == "EARLY_AGGRESSION"]
then
    modifiers.add(new WeightModifier(+0.03, +0.03, 0.00, 0.00,
        "Combined: rush + early aggression"));
end
```

- [ ] **Step 5: Run rule unit tests**

Run: `mvn test -Dtest=DominanceWeightRuleUnitTest -q`
Expected: PASS — all 18 tests green.

If any fail, debug the DRL pattern matching (common issues per GE-0056:
OOPath accessor syntax, record field naming, constraints on nested
method calls). Fix and re-run.

- [ ] **Step 6: Complete DroolsDominanceWeightStrategy CDI bean**

Update `DroolsDominanceWeightStrategy.java` to add the full bean:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import io.quarkmind.plugin.drools.DominanceWeightRuleUnit;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;

import java.util.List;

@ApplicationScoped
public class DroolsDominanceWeightStrategy implements DominanceWeightStrategy {

    private final RuleUnit<DominanceWeightRuleUnit> ruleUnit;
    private final AnchorInterpolator interpolator;

    @Inject
    DroolsDominanceWeightStrategy(
            RuleUnit<DominanceWeightRuleUnit> ruleUnit,
            MilestoneConfig config) {
        this.ruleUnit = ruleUnit;
        this.interpolator = new AnchorInterpolator(config.dominance().anchors());
    }

    DroolsDominanceWeightStrategy(
            RuleUnit<DominanceWeightRuleUnit> ruleUnit,
            List<MilestoneConfig.Dominance.WeightAnchor> anchors) {
        this.ruleUnit = ruleUnit;
        this.interpolator = new AnchorInterpolator(anchors);
    }

    @Override
    public String id() {
        return "drools";
    }

    @Override
    public DominanceWeights resolve(WeightContext context) {
        DominanceWeights baseline = interpolator.interpolate(context.gameFrame());
        DominanceWeightRuleUnit data = new DominanceWeightRuleUnit();

        if (context.currentPhase() != null) {
            data.getPhaseStore().add(context.currentPhase());
        }
        for (var a : context.patternAssessments()) {
            data.getPatternStore().add(a);
        }

        try (RuleUnitInstance<DominanceWeightRuleUnit> instance =
                 ruleUnit.createInstance(data)) {
            instance.fire();
        }

        return applyModifiers(baseline, data.getModifiers());
    }

    static DominanceWeights applyModifiers(DominanceWeights baseline,
                                           List<WeightModifier> modifiers) {
        if (modifiers.isEmpty()) return baseline;

        double economy = baseline.economy();
        double army = baseline.army();
        double tech = baseline.tech();
        double bases = baseline.bases();

        for (WeightModifier mod : modifiers) {
            economy += mod.economyDelta();
            army += mod.armyDelta();
            tech += mod.techDelta();
            bases += mod.basesDelta();
        }

        economy = Math.max(MINIMUM_WEIGHT, economy);
        army = Math.max(MINIMUM_WEIGHT, army);
        tech = Math.max(MINIMUM_WEIGHT, tech);
        bases = Math.max(MINIMUM_WEIGHT, bases);

        double sum = economy + army + tech + bases;
        return new DominanceWeights(
            economy / sum, army / sum, tech / sum, bases / sum);
    }
}
```

- [ ] **Step 7: Add end-to-end tests to DroolsDominanceWeightStrategyTest**

Add tests that exercise the full `resolve()` path with the real rule unit:

```java
// Add to DroolsDominanceWeightStrategyTest:

private static final List<MilestoneConfig.Dominance.WeightAnchor> ANCHORS = List.of(
    anchor(0, 0.30, 0.30, 0.20, 0.20),
    anchor(20160, 0.25, 0.35, 0.20, 0.20)
);

private DroolsDominanceWeightStrategy createStrategy() {
    RuleUnit<DominanceWeightRuleUnit> ruleUnit =
        RuleUnitProvider.get().getRuleUnit(DominanceWeightRuleUnit.class);
    return new DroolsDominanceWeightStrategy(ruleUnit, ANCHORS);
}

@Test
void id_returnsDrools() {
    assertEquals("drools", createStrategy().id());
}

@Test
void resolve_emptyContext_returnsBaseline() {
    var ctx = new WeightContext(5000, null, List.of());
    DominanceWeights result = createStrategy().resolve(ctx);
    assertNotNull(result);
    assertEquals(1.0, result.economy() + result.army()
        + result.tech() + result.bases(), 0.001);
}

@Test
void resolve_withRush_shiftsToArmy() {
    var ctx = new WeightContext(5000, null, List.of(
        new EnemyPatternAssessment(
            EnemyArchetype.TERRAN_MARINE_RUSH, 0.7, 3000, "test")));
    DominanceWeights baseline = new AnchorInterpolator(ANCHORS)
        .interpolate(5000);
    DominanceWeights result = createStrategy().resolve(ctx);
    assertTrue(result.army() > baseline.army());
    assertEquals(1.0, result.economy() + result.army()
        + result.tech() + result.bases(), 0.001);
}

@Test
void resolve_withPhaseAndPattern_composesAll() {
    var ctx = new WeightContext(5000, "DEFENSIVE_HOLD", List.of(
        new EnemyPatternAssessment(
            EnemyArchetype.ZERG_ZERGLING_RUSH, 0.6, 2000, "test")));
    DominanceWeights result = createStrategy().resolve(ctx);
    // Rush + defensive hold + combined signal → heavily army-weighted
    assertTrue(result.army() > 0.40, "Army should dominate: " + result.army());
    assertEquals(1.0, result.economy() + result.army()
        + result.tech() + result.bases(), 0.001);
}

// Helper — create a WeightAnchor for testing
private static MilestoneConfig.Dominance.WeightAnchor anchor(
        long frame, double economy, double army, double tech, double bases) {
    return new MilestoneConfig.Dominance.WeightAnchor() {
        public long frame() { return frame; }
        public double economyWeight() { return economy; }
        public double armyWeight() { return army; }
        public double techWeight() { return tech; }
        public double basesWeight() { return bases; }
    };
}
```

Add imports: `RuleUnitProvider`, `DominanceWeightRuleUnit`, `WeightContext`,
`EnemyPatternAssessment`, `EnemyArchetype`, `AnchorInterpolator`,
`MilestoneConfig`.

- [ ] **Step 8: Run all tests**

Run: `mvn test -Dtest=DominanceWeightRuleUnitTest,DroolsDominanceWeightStrategyTest -q`
Expected: PASS — all tests green.

Then run: `mvn test -q`
Expected: All ~1547+ tests pass. Zero failures.

- [ ] **Step 9: Verify with IDE diagnostics**

Run `ide_diagnostics` on all new and modified files.

- [ ] **Step 10: Update CLAUDE.md test lists**

Add `DominanceWeightRuleUnitTest` and `DroolsDominanceWeightStrategyTest`
to the unit test list in the `## Testing Patterns` section of CLAUDE.md.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add \
  src/main/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnit.java \
  src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl \
  src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java \
  src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java \
  src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java \
  CLAUDE.md
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#240): Drools-based DominanceWeightStrategy — rule unit, DRL rules, CDI bean

Refs #240"
```
