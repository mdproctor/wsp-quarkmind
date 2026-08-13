# Scouting Convergence Evaluation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #269 — feat: posture correctness evaluation — retrospective scouting accuracy at game close
**Issue group:** #269

**Goal:** Add `scouting_convergence` and `assessment_stable` features to CBR cases, comparing the initial archetype assessment against the final assessment at game close.

**Architecture:** A pure-function `ScoutingConvergenceEvaluator` in `agent/` computes a tiered score (exact=1.0, same race+category=0.5, else=0.0). Two new CaseFile keys capture the initial archetype (write-once) and the latest assessment (unconditional per-tick). `SC2CbrRetentionObserver` reads both from the snapshot and enriches the CBR case. Features are stored but not yet wired into retrieval queries — query integration is a follow-up.

**Tech Stack:** Java 21, Quarkus, JUnit 5, Mockito, AssertJ

## Global Constraints

- Domain model classes (`domain/`) must remain plain Java — no CDI, no Quarkus imports
- `ScoutingConvergenceEvaluator` is a plain Java class in `agent/` — no CDI
- CaseFile key names follow existing convention: `agent.strategy.*` / `agent.scouting.*`
- Never use `@QuarkusTest` for tests that can be plain JUnit
- All commits reference `Refs #269`

---

### Task 1: ScoutingConvergenceEvaluator — pure scoring logic

**Files:**
- Create: `src/main/java/io/quarkmind/agent/ScoutingConvergenceEvaluator.java`
- Test: `src/test/java/io/quarkmind/agent/ScoutingConvergenceEvaluatorTest.java`

**Interfaces:**
- Consumes: `StrategyArchetype` (from `domain/`), `PatternAssessment` (from `domain/`)
- Produces: `ScoutingConvergenceEvaluator.Result(double convergence, boolean stable)` — used by Task 3

- [ ] **Step 1: Write the failing tests**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ScoutingConvergenceEvaluatorTest {

    @Test
    void exactMatch_returnsOnePointZero() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.ZERG_ROACH_RUSH,
                List.of(new PatternAssessment(StrategyArchetype.ZERG_ROACH_RUSH, 0.8, 5000, "test")));
        assertThat(result.convergence()).isEqualTo(1.0);
        assertThat(result.stable()).isTrue();
    }

    @Test
    void sameCategorySamePhase_returnsHalf() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.ZERG_ZERGLING_RUSH,
                List.of(new PatternAssessment(StrategyArchetype.ZERG_ROACH_RUSH, 0.7, 5000, "test")));
        assertThat(result.convergence()).isEqualTo(0.5);
        assertThat(result.stable()).isTrue();
    }

    @Test
    void sameCategoryCrossPhase_returnsHalf() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.TERRAN_MARINE_TANK,
                List.of(new PatternAssessment(StrategyArchetype.TERRAN_MECH_LATE, 0.6, 10000, "test")));
        assertThat(result.convergence()).isEqualTo(0.5);
        assertThat(result.stable()).isTrue();
    }

    @Test
    void differentCategory_returnsZero() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.ZERG_ZERGLING_RUSH,
                List.of(new PatternAssessment(StrategyArchetype.ZERG_MACRO, 0.8, 5000, "test")));
        assertThat(result.convergence()).isEqualTo(0.0);
        assertThat(result.stable()).isFalse();
    }

    @Test
    void emptyAssessments_returnsZero() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.ZERG_ROACH_RUSH, List.of());
        assertThat(result.convergence()).isEqualTo(0.0);
        assertThat(result.stable()).isFalse();
    }

    @Test
    void crossRaceDifferentCategory_returnsZero() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.TERRAN_MARINE_RUSH,
                List.of(new PatternAssessment(StrategyArchetype.ZERG_BROOD_LORD, 0.7, 10000, "test")));
        assertThat(result.convergence()).isEqualTo(0.0);
        assertThat(result.stable()).isFalse();
    }

    @Test
    void crossRaceSameCategory_returnsZero() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.TERRAN_MARINE_RUSH,
                List.of(new PatternAssessment(StrategyArchetype.ZERG_ZERGLING_RUSH, 0.7, 5000, "test")));
        assertThat(result.convergence()).isEqualTo(0.0);
        assertThat(result.stable()).isFalse();
    }

    @Test
    void usesFirstAssessment_notSecond() {
        var result = ScoutingConvergenceEvaluator.evaluate(
                StrategyArchetype.ZERG_ROACH_RUSH,
                List.of(
                    new PatternAssessment(StrategyArchetype.ZERG_MACRO, 0.9, 5000, "first"),
                    new PatternAssessment(StrategyArchetype.ZERG_ROACH_RUSH, 0.4, 5000, "second")));
        assertThat(result.convergence()).isEqualTo(0.0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=ScoutingConvergenceEvaluatorTest -q`
Expected: Compilation failure — `ScoutingConvergenceEvaluator` does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyArchetype;

import java.util.List;

public class ScoutingConvergenceEvaluator {

    public record Result(double convergence, boolean stable) {}

    public static Result evaluate(StrategyArchetype initialArchetype,
                                  List<PatternAssessment> finalAssessments) {
        if (finalAssessments.isEmpty()) {
            return new Result(0.0, false);
        }
        StrategyArchetype finalArchetype = finalAssessments.getFirst().archetype();

        double convergence;
        if (initialArchetype == finalArchetype) {
            convergence = 1.0;
        } else if (initialArchetype.race() == finalArchetype.race()
                   && initialArchetype.category() == finalArchetype.category()) {
            convergence = 0.5;
        } else {
            convergence = 0.0;
        }
        return new Result(convergence, convergence >= 0.5);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=ScoutingConvergenceEvaluatorTest -q`
Expected: All 8 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/ScoutingConvergenceEvaluator.java src/test/java/io/quarkmind/agent/ScoutingConvergenceEvaluatorTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#269): ScoutingConvergenceEvaluator — category-aware tiered scoring

Refs #269"
```

---

### Task 2: CaseFile key writes — initial archetype + final assessment

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java:53` — add two key constants
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java:198-202` — add write-once guard
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:321` — add unconditional CaseFile write
- Modify: `src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:390-397` — add key to `produces()`
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java` — add write-once tests
- Test: `src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java` — add unconditional write test

**Interfaces:**
- Consumes: `QuarkMindCaseFile` keys, `PatternAssessment` (from `domain/`)
- Produces: `STRATEGY_INITIAL_ARCHETYPE` written to CaseFile, `SCOUTING_FINAL_ASSESSMENT` written to CaseFile — both read by Task 3

- [ ] **Step 1: Add CaseFile key constants**

In `QuarkMindCaseFile.java`, after the `SCOUTING_DISPATCH_FRAME` line (line 53), add:

```java
public static final String STRATEGY_INITIAL_ARCHETYPE  = "agent.strategy.initial.archetype";
public static final String SCOUTING_FINAL_ASSESSMENT   = "agent.scouting.final.assessment";
```

- [ ] **Step 2: Write failing test for write-once guard in SC2StrategyRouterTask**

In `SC2StrategyRouterTaskTest.java`, add:

```java
@Test
void firstRouting_writesInitialArchetype() {
    // Set up broker to return a confident assessment
    when(broker.current(ScoutingIntelType.PATTERN_ASSESSMENT))
            .thenReturn(Optional.of(new PatternAssessmentPayload(
                    List.of(new PatternAssessment(
                            StrategyArchetype.ZERG_ROACH_RUSH, 0.8, 1000, "test")))));

    task.execute(ctx);

    assertThat(ctx.getString(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE))
            .isEqualTo("ZERG_ROACH_RUSH");
}

@Test
void secondRouting_doesNotOverwriteInitialArchetype() {
    // First routing — ZERG_ROACH_RUSH
    when(broker.current(ScoutingIntelType.PATTERN_ASSESSMENT))
            .thenReturn(Optional.of(new PatternAssessmentPayload(
                    List.of(new PatternAssessment(
                            StrategyArchetype.ZERG_ROACH_RUSH, 0.8, 1000, "test")))));
    task.execute(ctx);

    // Second routing — pivot to ZERG_MACRO (different context triggers re-routing)
    when(broker.current(ScoutingIntelType.PATTERN_ASSESSMENT))
            .thenReturn(Optional.of(new PatternAssessmentPayload(
                    List.of(new PatternAssessment(
                            StrategyArchetype.ZERG_MACRO, 0.9, 5000, "test")))));
    task.execute(ctx);

    assertThat(ctx.getString(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE))
            .isEqualTo("ZERG_ROACH_RUSH");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest#firstRouting_writesInitialArchetype+secondRouting_doesNotOverwriteInitialArchetype -q`
Expected: FAIL — key not written

- [ ] **Step 4: Implement write-once guard in SC2StrategyRouterTask**

In `SC2StrategyRouterTask.java`, after line 200 (`ctx.set(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, archetype.name())`), add:

```java
if (ctx.getString(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE) == null) {
    ctx.set(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE, archetype.name());
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2StrategyRouterTaskTest -q`
Expected: All tests PASS

- [ ] **Step 6: Write failing test for unconditional assessment write in DroolsScoutingTask**

In `DroolsScoutingTaskTest.java`, add a test verifying `SCOUTING_FINAL_ASSESSMENT` is written to the CaseContext even when `patternAssessmentDispatchEnabled` is false. The exact test structure depends on the existing test setup — use the existing mock patterns in the file. The assertion:

```java
@SuppressWarnings("unchecked")
List<PatternAssessment> stored =
        (List<PatternAssessment>) ctx.getAs(QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT, List.class);
assertThat(stored).isNotNull().isNotEmpty();
```

- [ ] **Step 7: Run test to verify it fails**

Run: `mvn test -Dtest=DroolsScoutingTaskTest#<testName> -q`
Expected: FAIL — key not written

- [ ] **Step 8: Implement unconditional write in DroolsScoutingTask**

In `DroolsScoutingTask.java`, at line 321 after `var assessments = PatternClassifier.allAssessments(...)`, add the unconditional write BEFORE the `assessmentsChanged` guard:

```java
var assessments = PatternClassifier.allAssessments(cumulativeConfidence, frame);
ctx.set(QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT, assessments);
```

Also add `QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT` to DroolsScoutingTask's `produces()` set (line 390-397), and `QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE` to SC2StrategyRouterTask's `produces()` set (line 104-106).

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn test -Dtest=DroolsScoutingTaskTest -q`
Expected: All tests PASS

- [ ] **Step 10: Run full test suite**

Run: `mvn test -q`
Expected: All tests PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java src/main/java/io/quarkmind/agent/cbr/SC2StrategyRouterTask.java src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java src/test/java/io/quarkmind/agent/cbr/SC2StrategyRouterTaskTest.java src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#269): CaseFile keys for initial archetype (write-once) and final assessment

Refs #269"
```

---

### Task 3: CBR enrichment wiring — convergence features in the case

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java` — add two fields
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java:54-94` — add feature mapping
- Modify: `src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java:104-214` — call evaluator, pass to enrichment
- Test: `src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java` — add convergence tests

**Interfaces:**
- Consumes: `ScoutingConvergenceEvaluator.Result` (from Task 1), `STRATEGY_INITIAL_ARCHETYPE` + `SCOUTING_FINAL_ASSESSMENT` keys (from Task 2)
- Produces: `scouting_convergence` and `assessment_stable` features in the stored CBR case

- [ ] **Step 1: Extend EnrichedGameData with two new fields**

In `EnrichedGameData.java`, add after `unitTradeRatio`:

```java
double scoutingConvergence,
boolean assessmentStable
```

- [ ] **Step 2: Add feature mapping in SC2GameCbrCase.buildForGameEnriched()**

After the `unit_trade_ratio` line in `buildForGameEnriched()`:

```java
features.put("scouting_convergence", FeatureValue.number(e.scoutingConvergence()));
features.put("assessment_stable", FeatureValue.number(e.assessmentStable() ? 1.0 : 0.0));
```

- [ ] **Step 3: Write failing test for convergence feature extraction**

In `SC2CbrRetentionObserverTest.java`, add:

```java
@Test
void onOutcome_includesConvergenceFeatures_whenBothKeysPresent() {
    var assessments = List.of(
            new PatternAssessment(StrategyArchetype.ZERG_ROACH_RUSH, 0.85, 8000, "final"));

    CaseOutcomeEvent event = buildEvent("WIN", Map.of(
            QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.early-pressure",
            QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
            QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.85,
            QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE, "ZERG_ROACH_RUSH",
            QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT, assessments));

    observer.onOutcome(event);

    verify(store).store(
            argThat(c -> {
                var f = c.features();
                return f.containsKey("scouting_convergence")
                       && ((Number) f.get("scouting_convergence").toRawValue()).doubleValue() == 1.0
                       && ((Number) f.get("assessment_stable").toRawValue()).doubleValue() == 1.0;
            }),
            any(), any(), any(), any(), any(), any());
}

@Test
void onOutcome_defaultsConvergenceToZero_whenInitialArchetypeMissing() {
    CaseOutcomeEvent event = buildEvent("WIN", Map.of(
            QuarkMindCaseFile.STRATEGY_SELECTED_ID, "strategy.drools",
            QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE, "ZERG_ROACH_RUSH",
            QuarkMindCaseFile.STRATEGY_ROUTED_CONFIDENCE, 0.7));

    observer.onOutcome(event);

    verify(store).store(
            argThat(c -> {
                var f = c.features();
                return f.containsKey("scouting_convergence")
                       && ((Number) f.get("scouting_convergence").toRawValue()).doubleValue() == 0.0
                       && ((Number) f.get("assessment_stable").toRawValue()).doubleValue() == 0.0;
            }),
            any(), any(), any(), any(), any(), any());
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest#onOutcome_includesConvergenceFeatures_whenBothKeysPresent+onOutcome_defaultsConvergenceToZero_whenInitialArchetypeMissing -q`
Expected: FAIL — constructor mismatch or missing features

- [ ] **Step 5: Update SC2CbrRetentionObserver.onOutcome() to call evaluator**

In `onOutcome()`, after the `opponentId` extraction (around line 153) and before constructing `EnrichedGameData`, add:

```java
String initialArchetypeStr = (String) snapshot.get(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE);
@SuppressWarnings("unchecked")
List<PatternAssessment> finalAssessments =
        (List<PatternAssessment>) snapshot.get(QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT);

double scoutingConvergence = 0.0;
boolean assessmentStable = false;
if (initialArchetypeStr != null && finalAssessments != null && !finalAssessments.isEmpty()) {
    var convergenceResult = ScoutingConvergenceEvaluator.evaluate(
            StrategyArchetype.valueOf(initialArchetypeStr), finalAssessments);
    scoutingConvergence = convergenceResult.convergence();
    assessmentStable = convergenceResult.stable();
}
```

Then pass `scoutingConvergence` and `assessmentStable` to the `EnrichedGameData` constructor.

- [ ] **Step 6: Update all EnrichedGameData constructor call sites**

The `EnrichedGameData` constructor gains two new trailing parameters (`scoutingConvergence`, `assessmentStable`). All call sites must be updated:

- `SC2CbrRetentionObserver.onOutcome()` — production call site (pass computed values)
- `SC2GameCbrCaseTest.java:82` — `buildForGameEnriched_allFeaturesPopulated` (append `0.0, false`)
- `SC2GameCbrCaseTest.java:110` — `buildForGameEnriched_optionalTimingFeaturesOmittedWhenEmpty` (append `0.0, false`)
- `SC2GameCbrCaseTest.java:128` — `buildForGameEnriched_withOutcomePreservesEnrichment` (append `0.0, false`)
- `SC2GameCbrCaseTest.java:146` — `buildForGameEnriched_includesEngagementFeatures` (append `0.0, false`)
- Any other test constructing `EnrichedGameData` directly in `SC2CbrRetentionObserverTest`

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -Dtest=SC2CbrRetentionObserverTest -q`
Expected: All tests PASS

- [ ] **Step 8: Run full test suite**

Run: `mvn test -q`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/main/java/io/quarkmind/agent/cbr/EnrichedGameData.java src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#269): scouting convergence features in CBR case enrichment

Refs #269"
```
