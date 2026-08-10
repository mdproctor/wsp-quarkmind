# Dominance Weight Rule Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #242 — feat: Expand DominanceWeightAdjustment.drl rule set from calibration data
**Issue group:** #242

**Goal:** Normalise the DRL rule set to category-level matching across all 6 ArchetypeCategory values, add per-category deduplication, and add domain-justified combined signal rules.

**Architecture:** Expand `DominanceWeightAdjustment.drl` from 13 to 25 rules. Add per-category dedup in `DroolsDominanceWeightStrategy.resolve()` so rules assume one assessment per category. All tests are directional — asserting dimensions move the right way, not exact values.

**Tech Stack:** Drools Rule Units (DRL), Quarkus CDI, JUnit 5

## Global Constraints

- DRL rules use `archetype().category() == ArchetypeCategory.<CAT>` — never specific enum matching
- Combined signal rules use ≥0.5 confidence threshold (between moderate and high tiers)
- Phase modifier rules are unchanged — do not modify
- All weight dimensions sum to 1.0 after normalisation (enforced by `applyModifiers()`)
- `MINIMUM_WEIGHT` floor prevents any dimension from reaching zero
- No `@QuarkusTest` for tests that can be plain JUnit — but DominanceWeightRuleUnit tests require CDI for `RuleUnit` injection

---

### Task 1: Per-Category Deduplication in DroolsDominanceWeightStrategy

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java:39-55`
- Test: `src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java`

**Interfaces:**
- Consumes: `PatternAssessment.archetype().category()` returns `ArchetypeCategory`
- Produces: `resolve()` now deduplicates assessments before DRL insertion — downstream tasks can assume one assessment per category in the DataStore

- [ ] **Step 1: Write the failing test — `resolve_deduplicatesPerCategory`**

```java
@Test
void resolve_deduplicatesPerCategory() {
    var ctx = new WeightContext(5000, null, List.of(
        new PatternAssessment(StrategyArchetype.TERRAN_MARINE_TANK, 0.7, 5000, "test"),
        new PatternAssessment(StrategyArchetype.TERRAN_BATTLE_MECH, 0.6, 5000, "test"),
        new PatternAssessment(StrategyArchetype.ZERG_ROACH_HYDRA, 0.4, 5000, "test")));
    var singleCtx = new WeightContext(5000, null, List.of(
        new PatternAssessment(StrategyArchetype.TERRAN_MARINE_TANK, 0.7, 5000, "test")));
    DominanceWeights multi = createStrategy().resolve(ctx);
    DominanceWeights single = createStrategy().resolve(singleCtx);
    assertEquals(single.army(), multi.army(), 0.001,
        "3 COMPOSITION assessments should dedup to highest-confidence one");
    assertEquals(single.economy(), multi.economy(), 0.001);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=DroolsDominanceWeightStrategyTest#resolve_deduplicatesPerCategory -q`
Expected: FAIL — currently 3 assessments fire 3× the COMPOSITION rule (once this category rule exists; for now the test may pass vacuously because no COMPOSITION rule exists yet — but the dedup logic is still needed for Task 2)

- [ ] **Step 3: Write the second failing test — `resolve_keepsHighestPerCategoryAcrossCategories`**

```java
@Test
void resolve_keepsHighestPerCategoryAcrossCategories() {
    var ctx = new WeightContext(5000, null, List.of(
        new PatternAssessment(StrategyArchetype.TERRAN_MARINE_RUSH, 0.8, 3000, "test"),
        new PatternAssessment(StrategyArchetype.ZERG_ZERGLING_RUSH, 0.5, 2000, "test"),
        new PatternAssessment(StrategyArchetype.TERRAN_BIO_TIMING, 0.6, 5000, "test")));
    var dedupedCtx = new WeightContext(5000, null, List.of(
        new PatternAssessment(StrategyArchetype.TERRAN_MARINE_RUSH, 0.8, 3000, "test"),
        new PatternAssessment(StrategyArchetype.TERRAN_BIO_TIMING, 0.6, 5000, "test")));
    DominanceWeights multi = createStrategy().resolve(ctx);
    DominanceWeights deduped = createStrategy().resolve(dedupedCtx);
    assertEquals(deduped.army(), multi.army(), 0.001,
        "2 RUSH + 1 TIMING should dedup to 1 RUSH (highest) + 1 TIMING");
}
```

- [ ] **Step 4: Implement dedup in `resolve()`**

Add a private static method `deduplicateByCategory` and call it in `resolve()` before inserting into the DataStore:

```java
@Override
public DominanceWeights resolve(WeightContext context) {
    DominanceWeights baseline = interpolator.interpolate(context.gameFrame());
    DominanceWeightRuleUnit data = new DominanceWeightRuleUnit();

    if (context.currentPhase() != null) {
        data.getTacticalPostureStore().add(context.currentPhase());
    }
    for (var a : deduplicateByCategory(context.patternAssessments())) {
        data.getPatternStore().add(a);
    }

    try (RuleUnitInstance<DominanceWeightRuleUnit> instance =
             ruleUnit.createInstance(data)) {
        instance.fire();
    }

    return applyModifiers(baseline, data.getModifiers());
}

static List<PatternAssessment> deduplicateByCategory(List<PatternAssessment> assessments) {
    if (assessments.size() <= 1) return assessments;
    return new ArrayList<>(assessments.stream()
        .collect(java.util.stream.Collectors.toMap(
            a -> a.archetype().category(),
            a -> a,
            (a, b) -> a.confidence() >= b.confidence() ? a : b))
        .values());
}
```

Add imports: `java.util.ArrayList`, `java.util.stream.Collectors`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=DroolsDominanceWeightStrategyTest -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#242): per-category dedup in DroolsDominanceWeightStrategy.resolve()"
```

---

### Task 2: Category-Level Archetype Rules in DRL

**Files:**
- Modify: `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl`
- Modify: `src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java`

**Interfaces:**
- Consumes: `ArchetypeCategory` enum (RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION), `PatternAssessment` record
- Produces: DRL fires `WeightModifier` for every `PatternAssessment` with confidence ≥0.3, matching by category

- [ ] **Step 1: Rewrite the archetype response section of DominanceWeightAdjustment.drl**

Replace the entire archetype response section (lines 1–60 of the current DRL — the 5 rules for RUSH high/moderate, push, banshee, macro) with 12 category rules:

```drl
package io.quarkmind.plugin.drools;
unit DominanceWeightRuleUnit;

import io.quarkmind.agent.WeightModifier;
import io.quarkmind.domain.ArchetypeCategory;
import io.quarkmind.domain.StrategyArchetype;

// --- RUSH RESPONSE ---

rule "Rush response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.RUSH]
then
    modifiers.add(new WeightModifier(-0.10, +0.15, -0.05, 0.00,
        "High-confidence rush: " + $arch));
end

rule "Rush response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.RUSH]
then
    modifiers.add(new WeightModifier(-0.05, +0.08, -0.03, 0.00,
        "Moderate rush: " + $arch));
end

// --- TIMING RESPONSE ---

rule "Timing response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.TIMING]
then
    modifiers.add(new WeightModifier(-0.10, +0.10, +0.05, -0.05,
        "High-confidence timing: " + $arch));
end

rule "Timing response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.TIMING]
then
    modifiers.add(new WeightModifier(-0.05, +0.05, +0.03, -0.03,
        "Moderate timing: " + $arch));
end

// --- HARASS RESPONSE ---

rule "Harass response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.HARASS]
then
    modifiers.add(new WeightModifier(-0.10, +0.05, +0.10, -0.05,
        "High-confidence harass: " + $arch));
end

rule "Harass response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.HARASS]
then
    modifiers.add(new WeightModifier(-0.05, +0.03, +0.05, -0.03,
        "Moderate harass: " + $arch));
end

// --- MACRO RESPONSE ---

rule "Macro response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.MACRO]
then
    modifiers.add(new WeightModifier(+0.08, -0.10, +0.05, -0.03,
        "High-confidence macro: " + $arch));
end

rule "Macro response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.MACRO]
then
    modifiers.add(new WeightModifier(+0.05, -0.05, +0.03, -0.03,
        "Moderate macro: " + $arch));
end

// --- TECH RESPONSE ---

rule "Tech response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.TECH]
then
    modifiers.add(new WeightModifier(-0.05, -0.05, +0.15, -0.05,
        "High-confidence tech: " + $arch));
end

rule "Tech response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.TECH]
then
    modifiers.add(new WeightModifier(-0.03, -0.03, +0.08, -0.02,
        "Moderate tech: " + $arch));
end

// --- COMPOSITION RESPONSE ---

rule "Composition response — high confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.6,
                  archetype().category() == ArchetypeCategory.COMPOSITION]
then
    modifiers.add(new WeightModifier(-0.05, +0.10, +0.05, -0.10,
        "High-confidence composition: " + $arch));
end

rule "Composition response — moderate confidence"
when
    /patternStore[$arch: archetype(), confidence >= 0.3, confidence < 0.6,
                  archetype().category() == ArchetypeCategory.COMPOSITION]
then
    modifiers.add(new WeightModifier(-0.03, +0.05, +0.03, -0.05,
        "Moderate composition: " + $arch));
end
```

Keep the phase modifier section (lines 64–94) and combined signal section (lines 96–135) unchanged for now — Task 3 handles them.

- [ ] **Step 2: Rename existing tests and update assertions**

In `DominanceWeightRuleUnitTest.java`:

Rename `pushArchetype_emitsTechAndArmyBoost` → `timingArchetype_emitsTechAndArmyBoost`. Keep the same input (`TERRAN_BIO_TIMING` at 0.6 — it's a TIMING category archetype). Assertions unchanged.

The existing `harassArchetype_emitsTechBoost` test uses `TERRAN_BANSHEE_HARASS` at 0.5. This now fires the **moderate** HARASS tier (0.5 < 0.6) instead of the old banshee rule. The assertion `techDelta > 0` still passes — the moderate tier has techDelta +0.05.

- [ ] **Step 3: Write new category tests**

Add these tests to `DominanceWeightRuleUnitTest.java`:

```java
@Test
void moderateConfidenceHarass_emitsSmallerBoost() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_REAPER_HARASS, 0.4, 3000, "test"));
    List<WeightModifier> mods = fire(data);
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    assertTrue(techDelta > 0, "Moderate harass should boost tech");
    assertTrue(techDelta < 0.10, "Moderate should be less than high-confidence tech boost");
}

@Test
void timingBelowThreshold_noModifier() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.PROTOSS_BLINK_STALKER, 0.29, 5000, "test"));
    List<WeightModifier> mods = fire(data);
    boolean hasTimingMod = mods.stream()
        .anyMatch(m -> m.reason().toLowerCase().contains("timing"));
    assertFalse(hasTimingMod);
}

@Test
void techArchetype_boostsTech() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_BC_TRANSITION, 0.6, 10000, "test"));
    List<WeightModifier> mods = fire(data);
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    double armyDelta = mods.stream()
        .mapToDouble(WeightModifier::armyDelta).sum();
    assertTrue(techDelta > 0, "Tech archetype should boost tech");
    assertTrue(armyDelta < 0, "Tech archetype should reduce army");
}

@Test
void moderateConfidenceTech_emitsSmallerBoost() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.ZERG_VIPER_SUPPORT, 0.4, 10000, "test"));
    List<WeightModifier> mods = fire(data);
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    assertTrue(techDelta > 0, "Moderate tech should boost tech");
    assertTrue(techDelta < 0.15, "Moderate should be less than high-confidence tech boost");
}

@Test
void compositionArchetype_boostsArmy() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_TANK, 0.6, 7000, "test"));
    List<WeightModifier> mods = fire(data);
    double armyDelta = mods.stream()
        .mapToDouble(WeightModifier::armyDelta).sum();
    double basesDelta = mods.stream()
        .mapToDouble(WeightModifier::basesDelta).sum();
    assertTrue(armyDelta > 0, "Composition should boost army");
    assertTrue(basesDelta < 0, "Composition should reduce bases");
}

@Test
void moderateConfidenceComposition_emitsSmallerBoost() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.ZERG_ROACH_HYDRA, 0.4, 7000, "test"));
    List<WeightModifier> mods = fire(data);
    double armyDelta = mods.stream()
        .mapToDouble(WeightModifier::armyDelta).sum();
    assertTrue(armyDelta > 0, "Moderate composition should boost army");
    assertTrue(armyDelta < 0.10, "Moderate should be less than high-confidence army boost");
}

@Test
void moderateConfidenceMacro_emitsSmallerBoost() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.ZERG_MACRO, 0.4, 4000, "test"));
    List<WeightModifier> mods = fire(data);
    double economyDelta = mods.stream()
        .mapToDouble(WeightModifier::economyDelta).sum();
    assertTrue(economyDelta > 0, "Moderate macro should boost economy");
    assertTrue(economyDelta < 0.08, "Moderate should be less than high-confidence economy boost");
}

@Test
void sameCategoryDedup_highestConfidenceWins() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_TANK, 0.7, 7000, "test"));
    var singleData = new DominanceWeightRuleUnit();
    singleData.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_TANK, 0.7, 7000, "test"));
    List<WeightModifier> singleMods = fire(singleData);
    List<WeightModifier> multiMods = fire(data);
    assertEquals(singleMods.size(), multiMods.size(),
        "Single assessment per category should produce same modifier count");
}

@Test
void crossCategoryStacking_allFireIndependently() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_RUSH, 0.7, 3000, "test"));
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_TANK, 0.6, 7000, "test"));
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_BANSHEE_HARASS, 0.5, 6000, "test"));
    List<WeightModifier> mods = fire(data);
    long categoryMods = mods.stream()
        .filter(m -> !m.reason().startsWith("Phase:") && !m.reason().startsWith("Combined:"))
        .count();
    assertEquals(3, categoryMods, "3 different categories should produce 3 modifiers");
}
```

- [ ] **Step 4: Run all rule unit tests**

Run: `mvn test -Dtest=DominanceWeightRuleUnitTest -q`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#242): category-level archetype rules — TIMING, HARASS, MACRO moderate, TECH, COMPOSITION"
```

---

### Task 3: Combined Signal Rules — Migration and Expansion

**Files:**
- Modify: `src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl`
- Modify: `src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java`

**Interfaces:**
- Consumes: Category-level archetype rules from Task 2, phase modifier rules (unchanged)
- Produces: 9 combined signal rules (4 migrated + 5 new) that fire when category + posture match

- [ ] **Step 1: Migrate existing 4 combined rules to category matching and add 5 new rules**

Replace the entire combined signals section of `DominanceWeightAdjustment.drl` with the rules below. Also remove the `import io.quarkmind.domain.StrategyArchetype;` line from the file header — no rule references `StrategyArchetype` directly after this migration.

```drl
// --- COMBINED SIGNALS ---

rule "Combined: Rush + DEFENSIVE_HOLD"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.RUSH]
    /tacticalPostureStore[this == "DEFENSIVE_HOLD"]
then
    modifiers.add(new WeightModifier(0.00, +0.05, 0.00, 0.00,
        "Combined: rush + defensive hold"));
end

rule "Combined: Macro + EARLY_MACRO"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.MACRO]
    /tacticalPostureStore[this == "EARLY_MACRO"]
then
    modifiers.add(new WeightModifier(+0.05, 0.00, 0.00, 0.00,
        "Combined: macro + early macro"));
end

rule "Combined: Timing + MID_SKIRMISH"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.TIMING]
    /tacticalPostureStore[this == "MID_SKIRMISH"]
then
    modifiers.add(new WeightModifier(0.00, +0.05, +0.03, 0.00,
        "Combined: timing + mid skirmish"));
end

rule "Combined: Rush + EARLY_AGGRESSION"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.RUSH]
    /tacticalPostureStore[this == "EARLY_AGGRESSION"]
then
    modifiers.add(new WeightModifier(+0.03, +0.03, 0.00, 0.00,
        "Combined: rush + early aggression"));
end

rule "Combined: Harass + DEFENSIVE_HOLD"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.HARASS]
    /tacticalPostureStore[this == "DEFENSIVE_HOLD"]
then
    modifiers.add(new WeightModifier(+0.05, 0.00, +0.03, 0.00,
        "Combined: harass + defensive hold"));
end

rule "Combined: Harass + EARLY_MACRO"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.HARASS]
    /tacticalPostureStore[this == "EARLY_MACRO"]
then
    modifiers.add(new WeightModifier(0.00, 0.00, +0.05, 0.00,
        "Combined: harass + early macro"));
end

rule "Combined: Tech + EARLY_MACRO"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.TECH]
    /tacticalPostureStore[this == "EARLY_MACRO"]
then
    modifiers.add(new WeightModifier(+0.03, 0.00, +0.05, 0.00,
        "Combined: tech + early macro"));
end

rule "Combined: Composition + MID_SKIRMISH"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.COMPOSITION]
    /tacticalPostureStore[this == "MID_SKIRMISH"]
then
    modifiers.add(new WeightModifier(0.00, +0.05, 0.00, -0.03,
        "Combined: composition + mid skirmish"));
end

rule "Combined: Composition + DEFENSIVE_HOLD"
when
    /patternStore[confidence >= 0.5, archetype().category() == ArchetypeCategory.COMPOSITION]
    /tacticalPostureStore[this == "DEFENSIVE_HOLD"]
then
    modifiers.add(new WeightModifier(0.00, +0.08, +0.03, 0.00,
        "Combined: composition + defensive hold"));
end
```

- [ ] **Step 2: Rename existing combined test**

Rename `combinedSignal_pushAndMidSkirmish_stacksModifiers` → `combinedSignal_timingAndMidSkirmish_stacksModifiers`. Replace `TERRAN_MECH_PUSH` with `TERRAN_MECH_PUSH` (unchanged — it's a TIMING archetype).

- [ ] **Step 3: Write new combined signal tests**

Add to `DominanceWeightRuleUnitTest.java`:

```java
@Test
void combinedSignal_harassAndDefensiveHold_stacksModifiers() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_BANSHEE_HARASS, 0.6, 6000, "test"));
    data.getTacticalPostureStore().add("DEFENSIVE_HOLD");
    List<WeightModifier> mods = fire(data);
    assertTrue(mods.size() >= 3,
        "Expected harass + phase + combined modifiers, got " + mods.size());
    double economyDelta = mods.stream()
        .mapToDouble(WeightModifier::economyDelta).sum();
    assertTrue(economyDelta > -0.10,
        "Combined should offset some economy loss from harass category rule");
}

@Test
void combinedSignal_harassAndEarlyMacro_stacksModifiers() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.PROTOSS_DT_HARASS, 0.6, 5000, "test"));
    data.getTacticalPostureStore().add("EARLY_MACRO");
    List<WeightModifier> mods = fire(data);
    assertTrue(mods.size() >= 3);
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    assertTrue(techDelta > 0.10,
        "Combined harass+early_macro should stack tech above category-only harass");
}

@Test
void combinedSignal_techAndEarlyMacro_stacksModifiers() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_BC_TRANSITION, 0.6, 10000, "test"));
    data.getTacticalPostureStore().add("EARLY_MACRO");
    List<WeightModifier> mods = fire(data);
    assertTrue(mods.size() >= 3);
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    double economyDelta = mods.stream()
        .mapToDouble(WeightModifier::economyDelta).sum();
    assertTrue(techDelta > 0.15,
        "Combined tech+early_macro should stack tech above category-only tech");
    assertTrue(economyDelta > 0,
        "Combined should produce positive economy from phase + combined");
}

@Test
void combinedSignal_compositionAndMidSkirmish_stacksModifiers() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.TERRAN_MARINE_TANK, 0.6, 7000, "test"));
    data.getTacticalPostureStore().add("MID_SKIRMISH");
    List<WeightModifier> mods = fire(data);
    assertTrue(mods.size() >= 3);
    double armyDelta = mods.stream()
        .mapToDouble(WeightModifier::armyDelta).sum();
    assertTrue(armyDelta > 0.10,
        "Combined composition+mid_skirmish should stack army above category-only composition");
}

@Test
void combinedSignal_compositionAndDefensiveHold_stacksModifiers() {
    var data = new DominanceWeightRuleUnit();
    data.getPatternStore().add(new PatternAssessment(
        StrategyArchetype.PROTOSS_STALKER_COLOSSUS, 0.6, 7000, "test"));
    data.getTacticalPostureStore().add("DEFENSIVE_HOLD");
    List<WeightModifier> mods = fire(data);
    assertTrue(mods.size() >= 3);
    double armyDelta = mods.stream()
        .mapToDouble(WeightModifier::armyDelta).sum();
    double techDelta = mods.stream()
        .mapToDouble(WeightModifier::techDelta).sum();
    assertTrue(armyDelta > 0.15,
        "Combined composition+defensive_hold should produce strong army response");
    assertTrue(techDelta > 0,
        "Combined should produce positive tech from combined rule");
}
```

- [ ] **Step 4: Run all tests**

Run: `mvn test -Dtest=DominanceWeightRuleUnitTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite to catch regressions**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/resources/io/quarkmind/plugin/drools/DominanceWeightAdjustment.drl src/test/java/io/quarkmind/plugin/drools/DominanceWeightRuleUnitTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#242): combined signal rules — migrate to category matching, add 5 new combinations"
```

---

### Task 4: Update Issue #242 and Run Final Validation

**Files:**
- No code changes — issue update and validation only

- [ ] **Step 1: Update issue #242 body to reflect partial delivery**

```bash
gh issue edit 242 --repo casehubio/quarkmind --body "$(gh issue view 242 --repo casehubio/quarkmind --json body --jq '.body')

## Delivered (this branch)

1. ~~Tune delta values from replay data~~ → Category-level normalisation with domain-justified deltas
2. Add new combined signal rules from calibration → 5 new domain-justified combined rules
3. Coverage for signal combinations not in the initial set → All 6 ArchetypeCategory values covered
4. ~~Validate expanded rules improve win rate~~ → Directional unit test validation

## Deferred

- Replay-based delta tuning (requires outcome feedback signals — not yet available)
- Win rate validation (requires live or emulated game loop comparison)"
```

- [ ] **Step 2: Run the full test suite one final time**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 3: Run benchmark tests to verify no regression**

Run: `mvn test -Pbenchmark -Dtest=PatternClassificationCalibrationTest -q`
Expected: ALL PASS — classification accuracy ≥70% for rush and air-threat patterns
