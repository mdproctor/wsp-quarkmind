# Scouting Convergence Evaluation — Retrospective Assessment Stability at Game Close

**Issue:** #269
**Branch:** issue-269-posture-correctness-eval
**Date:** 2026-08-12

## Problem

At game close, the CBR case captures what strategy was selected and whether the game was won or lost — but not whether the scouting system's early assessment was reliable. Two games might both be wins with the same strategy, but in one the early scouting read was stable throughout the game, while in the other the bot pivoted after discovering the initial read was wrong. CBR retrieval can't distinguish these cases.

## Solution

Add two features to the CBR case at game close:

- **`scouting_convergence`** (double, 0.0–1.0) — did the initial archetype assessment match the final assessment?
- **`assessment_stable`** (boolean) — was convergence >= 0.5 (at least category-correct)?

These enable CBR retrieval queries like: "in past games where our early scouting was reliable, which strategy worked best against this archetype?"

## What This Measures

**Convergence, not correctness.** The feature compares the scouting system's first routing assessment against its own final assessment — not against an external ground truth. A scouting system that consistently misidentifies MACRO as HARASS would show high convergence. The feature names reflect this distinction deliberately.

**Scope note:** Issue #269 originally proposed `posture_correct` and `scouting_accuracy` as features measuring correctness against the opponent's actual composition. During design review, convergence was identified as the achievable first step — true correctness requires a composition-derived reverse-classifier (D1 alternative). The issue body should be updated to reflect this scope, and a follow-up issue filed for composition-based correctness evaluation if needed.

## Architecture

### New CaseFile Keys

```java
// In QuarkMindCaseFile
public static final String STRATEGY_INITIAL_ARCHETYPE  = "agent.strategy.initial.archetype";
public static final String SCOUTING_FINAL_ASSESSMENT   = "agent.scouting.final.assessment";
```

- `STRATEGY_INITIAL_ARCHETYPE` — written by `SC2StrategyRouterTask` on first successful routing only. Uses a write-once guard:
  ```java
  if (ctx.get(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE) == null) {
      ctx.set(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE, archetype.name());
  }
  ```
  Without this guard, a pivot would overwrite the initial archetype, collapsing convergence to 1.0 by construction.

- `SCOUTING_FINAL_ASSESSMENT` — written by `DroolsScoutingTask` **unconditionally** after computing assessments, outside the `assessmentsChanged()` / `patternAssessmentDispatchEnabled` guards. The CaseFile write must not be gated by dispatch preferences — disabling pattern assessment dispatch should not silently disable convergence tracking. The write is a single `ctx.set()` call after `PatternClassifier.allAssessments()` returns, regardless of whether assessments changed or will be published.

### ScoutingConvergenceEvaluator

**Package:** `io.quarkmind.agent`
**Type:** Plain Java class — no CDI, no framework dependencies.

```java
public class ScoutingConvergenceEvaluator {

    public record Result(double convergence, boolean stable) {}

    public static Result evaluate(
            StrategyArchetype initialArchetype,
            List<PatternAssessment> finalAssessments) { ... }
}
```

**Scoring logic (category-aware tiered):**

1. If `finalAssessments` is empty → `Result(0.0, false)` (no late data to compare)
2. Take `finalAssessments.getFirst()` (highest confidence — same selection as routing)
3. If exact archetype match → `1.0`
4. If same `ArchetypeCategory` (any phase) → `0.5`
5. Otherwise → `0.0`
6. `stable = convergence >= 0.5`

Phase transitions are handled naturally by rule 4: same category across phases scores 0.5 regardless of which phases are involved (EARLY→MID, MID→LATE, EARLY→LATE). Cross-race mismatches always score 0.0 because no two archetypes of different races share an enum value — the exact match (rule 3) won't fire, and category match (rule 4) fires only within the `getFirst()` result which carries its own race.

### SC2CbrRetentionObserver Changes

In `onOutcome()`, after building the existing enrichment:

```java
String initialArchetypeStr = (String) snapshot.get(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE);
@SuppressWarnings("unchecked")
List<PatternAssessment> finalAssessments =
        (List<PatternAssessment>) snapshot.get(QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT);

double convergence = 0.0;
boolean stable = false;
if (initialArchetypeStr != null && finalAssessments != null && !finalAssessments.isEmpty()) {
    var result = ScoutingConvergenceEvaluator.evaluate(
            StrategyArchetype.valueOf(initialArchetypeStr), finalAssessments);
    convergence = result.convergence();
    stable = result.stable();
}
```

### EnrichedGameData Extension

Add two fields:

```java
double scoutingConvergence,    // 0.0–1.0
boolean assessmentStable       // convergence >= 0.5
```

### SC2GameCbrCase Feature Mapping

```java
features.put("scouting_convergence", FeatureValue.number(e.scoutingConvergence()));
features.put("assessment_stable", FeatureValue.string(String.valueOf(e.assessmentStable())));
```

## Data Flow

```
Game start
    │
    ▼
SC2StrategyRouterTask — first routing
    │ writes STRATEGY_INITIAL_ARCHETYPE (once, write-once guard)
    │ writes STRATEGY_ROUTED_ARCHETYPE (may update on pivot)
    ▼
DroolsScoutingTask — each tick
    │ writes SCOUTING_FINAL_ASSESSMENT (unconditional, outside dispatch guards)
    ▼
Game close — CaseFile snapshot captured
    │
    ▼
SC2CbrRetentionObserver.onOutcome()
    │ reads STRATEGY_INITIAL_ARCHETYPE from snapshot
    │ reads SCOUTING_FINAL_ASSESSMENT from snapshot
    │ calls ScoutingConvergenceEvaluator.evaluate()
    │ adds scouting_convergence + assessment_stable to CBR case
    ▼
CbrCaseMemoryStore.store()
```

## Changes Required

| File | Change |
|------|--------|
| `QuarkMindCaseFile.java` | Add `STRATEGY_INITIAL_ARCHETYPE`, `SCOUTING_FINAL_ASSESSMENT` keys |
| `SC2StrategyRouterTask.java` | Write `STRATEGY_INITIAL_ARCHETYPE` with write-once guard on first routing |
| `DroolsScoutingTask.java` | Write `SCOUTING_FINAL_ASSESSMENT` unconditionally after computing assessments |
| `ScoutingConvergenceEvaluator.java` | **New** — category-aware tiered scoring (in `agent/`) |
| `EnrichedGameData.java` | Add `scoutingConvergence`, `assessmentStable` fields |
| `SC2GameCbrCase.java` | Add feature mapping in `buildForGameEnriched()` |
| `SC2CbrRetentionObserver.java` | Call evaluator, pass results to enrichment |
| `ScoutingConvergenceEvaluatorTest.java` | **New** — unit tests for scoring logic |
| `SC2CbrRetentionObserverTest.java` | Add test for convergence feature extraction |

## Testing Strategy

### ScoutingConvergenceEvaluatorTest (unit, plain JUnit)
- Exact match → 1.0, stable=true
- Same category, same phase → 0.5, stable=true
- Same category, cross-phase (EARLY→MID) → 0.5, stable=true
- Same category, cross-phase (MID→LATE) → 0.5, stable=true
- Different category, same phase → 0.0, stable=false
- Different category, cross-phase → 0.0, stable=false
- Empty final assessments → 0.0, stable=false

### SC2CbrRetentionObserverTest (unit, Mockito)
- Case with both keys present → features include `scouting_convergence` and `assessment_stable`
- Case with missing initial archetype → convergence defaults to 0.0
- Case with missing final assessment → convergence defaults to 0.0

## Known Limitations

1. **Convergence, not correctness** — measures internal consistency of the scouting system, not accuracy against reality. A consistently wrong classifier shows high convergence. True correctness evaluation (composition-derived reverse-classifier) is a potential follow-up.
2. **Survivorship bias** — games where scouting confidence never reached the routing threshold have no `STRATEGY_INITIAL_ARCHETYPE` and are excluded from convergence tracking.
3. **Category granularity varies** — COMPOSITION has 22 archetypes, RUSH has 7. A category-level match of 0.5 means different things depending on the category.
