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

## Architecture

### New CaseFile Keys

```java
// In QuarkMindCaseFile
public static final String STRATEGY_INITIAL_ARCHETYPE  = "agent.strategy.initial.archetype";
public static final String SCOUTING_FINAL_ASSESSMENT   = "agent.scouting.final.assessment";
```

- `STRATEGY_INITIAL_ARCHETYPE` — written by `SC2StrategyRouterTask` on first successful routing only. Never overwritten on pivot. Captures the archetype that drove the initial strategy selection.
- `SCOUTING_FINAL_ASSESSMENT` — written by `DroolsScoutingTask` on each tick (overwritten). At game close, the CaseFile snapshot captures the most recent value.

### PostureCorrectnessEvaluator

**Package:** `io.quarkmind.agent`
**Type:** Plain Java class — no CDI, no framework dependencies.

```java
public class PostureCorrectnessEvaluator {

    public record Result(double convergence, boolean stable) {}

    public static Result evaluate(
            StrategyArchetype initialArchetype,
            List<PatternAssessment> finalAssessments) { ... }
}
```

**Scoring logic (phase-aware tiered):**

1. If `finalAssessments` is empty → `Result(0.0, false)` (no late data to compare)
2. Take `finalAssessments.getFirst()` (highest confidence — same selection as routing)
3. If exact archetype match → `1.0`
4. If same `ArchetypeCategory` AND same `GamePhase` → `0.5`
5. If initial is EARLY phase and final is MID or LATE phase → `0.5` (cross-phase transition, not penalised)
6. Otherwise → `0.0`
7. `stable = convergence >= 0.5`

### SC2CbrRetentionObserver Changes

In `onOutcome()`, after building the existing enrichment:

```java
String initialArchetypeStr = (String) snapshot.get(QuarkMindCaseFile.STRATEGY_INITIAL_ARCHETYPE);
List<PatternAssessment> finalAssessments = /* from SCOUTING_FINAL_ASSESSMENT in snapshot */;

if (initialArchetypeStr != null && finalAssessments != null) {
    var result = PostureCorrectnessEvaluator.evaluate(
            StrategyArchetype.valueOf(initialArchetypeStr), finalAssessments);
    // add to enrichment features
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
    │ writes STRATEGY_INITIAL_ARCHETYPE (once, never overwritten)
    │ writes STRATEGY_ROUTED_ARCHETYPE (may update on pivot)
    ▼
DroolsScoutingTask — each tick
    │ writes SCOUTING_FINAL_ASSESSMENT (overwritten each tick)
    ▼
Game close — CaseFile snapshot captured
    │
    ▼
SC2CbrRetentionObserver.onOutcome()
    │ reads STRATEGY_INITIAL_ARCHETYPE from snapshot
    │ reads SCOUTING_FINAL_ASSESSMENT from snapshot
    │ calls PostureCorrectnessEvaluator.evaluate()
    │ adds scouting_convergence + assessment_stable to CBR case
    ▼
CbrCaseMemoryStore.store()
```

## Changes Required

| File | Change |
|------|--------|
| `QuarkMindCaseFile.java` | Add `STRATEGY_INITIAL_ARCHETYPE`, `SCOUTING_FINAL_ASSESSMENT` keys |
| `SC2StrategyRouterTask.java` | Write `STRATEGY_INITIAL_ARCHETYPE` on first routing only |
| `DroolsScoutingTask.java` | Write `SCOUTING_FINAL_ASSESSMENT` on each tick |
| `PostureCorrectnessEvaluator.java` | **New** — phase-aware tiered scoring (in `agent/`) |
| `EnrichedGameData.java` | Add `scoutingConvergence`, `assessmentStable` fields |
| `SC2GameCbrCase.java` | Add feature mapping in `buildForGameEnriched()` |
| `SC2CbrRetentionObserver.java` | Call evaluator, pass results to enrichment |
| `PostureCorrectnessEvaluatorTest.java` | **New** — unit tests for scoring logic |
| `SC2CbrRetentionObserverTest.java` | Add test for convergence feature extraction |

## Testing Strategy

### PostureCorrectnessEvaluatorTest (unit, plain JUnit)
- Exact match → 1.0, stable=true
- Same category, same phase → 0.5, stable=true
- Cross-phase transition (EARLY → MID) → 0.5, stable=true
- Different category, same phase → 0.0, stable=false
- Empty final assessments → 0.0, stable=false
- Cross-race (initial TERRAN, final ZERG) → 0.0, stable=false

### SC2CbrRetentionObserverTest (unit, Mockito)
- Case with both keys present → features include `scouting_convergence` and `assessment_stable`
- Case with missing initial archetype → features omitted (graceful degradation)
- Case with missing final assessment → features omitted

## Known Limitations

1. **Convergence, not correctness** — measures internal consistency of the scouting system, not accuracy against reality. A consistently wrong classifier shows high convergence.
2. **Survivorship bias** — games where scouting confidence never reached the routing threshold have no `STRATEGY_INITIAL_ARCHETYPE` and are excluded from convergence tracking.
3. **Category granularity varies** — COMPOSITION has 22 archetypes, RUSH has 7. A category-level match of 0.5 means different things depending on the category.
4. **Cross-phase transitions all treated equally** — a RUSH → MACRO transition is scored the same as HARASS → COMPOSITION. Both get 0.5.
