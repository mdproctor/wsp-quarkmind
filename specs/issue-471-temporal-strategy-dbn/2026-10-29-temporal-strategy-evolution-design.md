# Temporal Strategy Evolution — Design Spec

**Issue:** casehubio/quarkmind#471
**Branch:** issue-471-temporal-strategy-dbn
**Date:** 2026-10-29

## Overview

Replace the discrete phase-transition reinitialisation in QuarkMind's strategy classification with continuous temporal inference. Currently, when the game crosses a phase boundary (EARLY→MID→LATE), both `HypothesisTrackerFusion` and `DetectorOrchestrator` experience a discontinuity — the tracker is rebuilt from scratch with `seedPriors()` carrying only terminal weights. This design introduces continuous temporal smoothing using neocortex's `TemporalEvidenceFusionEngine` and a unified archetype set that eliminates phase-transition rebuilds entirely.

## Scope

Both classification systems are modified:

1. **DetectorOrchestrator** — replace `EvidenceFusionEngine` + `TemporalAssessmentTracker` with a single `TemporalEvidenceFusionEngine`
2. **HypothesisTrackerFusion** — eliminate phase-transition reinitialisation by using all 55 archetypes from all phases in a single tracker, with transition-aware temporal priors

A new offline pipeline learns transition probabilities from 149K replays at five time checkpoints.

## Component 1: DetectorOrchestrator temporal upgrade

### Current state

`DetectorOrchestrator` uses two objects for classification:
- `EvidenceFusionEngine engine` — non-temporal BJT inference
- `TemporalAssessmentTracker tracker` — sliding window of 5 posteriors, arithmetic mean

The flow in `classify()`:
1. Set all evidence on `engine`
2. `engine.update()` → raw posterior
3. `tracker.recordPosterior(posterior)` → sliding window
4. `tracker.getSmoothedPosterior()` → averaged posterior
5. Build assessments from smoothed posterior

### New state

Replace both with a single `TemporalEvidenceFusionEngine`:
- Constructor takes `temporalDecay` parameter (replaces `windowSize`)
- `classify()` flow becomes:
  1. Set all evidence on temporal engine
  2. `engine.update()` → temporally-smoothed posterior (decay blending handled internally)
  3. `engine.marginalize()` → smoothed posterior
  4. Build assessments

### Changes

- **Field changes:** Remove `EvidenceFusionEngine engine` and `TemporalAssessmentTracker tracker`. Add `TemporalEvidenceFusionEngine temporalEngine`.
- **Constructor:** Replace `int windowSize` with `double temporalDecay`. Default decay: `0.3` (30% carry-over from previous tick — equivalent smoothing to a ~3-tick window).
- **`initializeNetwork()`:** Call `temporalEngine.initialize(network, "Strategy", temporalDecay)` instead of separate engine + tracker init.
- **`classify()`:** Use `temporalEngine.setAllEvidence()`, `temporalEngine.update()`, `temporalEngine.marginalize()`. Remove `tracker.recordPosterior()` and `tracker.getSmoothedPosterior()`.
- **`detectTransition()`:** Implement directly — compare consecutive `marginalize()` results for dominant-hypothesis change. The `TemporalEvidenceFusionEngine` already stores `previousPosterior`.
- **`reset()`:** Call `temporalEngine.reset()`.
- **Sensitivity analysis:** `EvidenceFusionEngine.getVariableSensitivities()` is not on `TemporalEvidenceFusionEngine`. Add a delegation method or access the underlying instance. Alternatively, add `getVariableSensitivities()` to the temporal engine (neocortex change — small).

### Impact

- `TemporalAssessmentTracker` class becomes unused — delete it
- `TemporalAssessmentTrackerTest` — delete
- All `DetectorOrchestrator` tests update constructor calls (windowSize → temporalDecay)
- `DetectorOrchestratorTest`, `DetectorCatalogTest` — update assertions for decay-based smoothing

## Component 2: HypothesisTrackerFusion unified archetype set

### Current state

`HypothesisTrackerFusion.initialize()` rebuilds the entire `HypothesisTracker` when the `GamePhase` changes:
1. Snapshot terminal weights from old tracker
2. Load new phase-specific priors from `HypothesisPriorLoader`
3. `seedPriors()` overlays terminal weights onto new priors
4. Build new `HypothesisTracker` with phase-filtered archetypes (~15-20 per phase)

This causes a discontinuity — accumulated evidence is lost except for the terminal weight snapshot. The archetype set changes between phases, so some hypotheses vanish and new ones appear.

### New state

Initialize once with all 55 archetypes. No phase-transition rebuild. Temporal continuity is maintained by:

1. **Phase-window-weighted priors:** Each archetype has a `phaseWindow: [min, max]` in the taxonomy. The prior for each archetype is weighted by how much the current game time overlaps its phase window. An EARLY archetype at minute 8 gets a near-zero prior; a MID archetype at minute 8 is in its prime window.

2. **Transition-aware temporal decay:** After each classification tick, the posterior is blended with the previous tick's posterior weighted by the learned transition matrix. If the previous tick's leader was `MARINE_RUSH` and the transition matrix says P(BIO_TIMING | MARINE_RUSH) = 0.4, then `BIO_TIMING` gets a temporal boost.

3. **Phase-window decay function:** A soft sigmoid decay based on game time relative to each archetype's phase window. Inside the window: weight = 1.0. Outside: weight decays with a configurable ramp (e.g., 1 minute ramp on each side).

### Changes

- **`initialize()`:** Called once at game start. No phase parameter — loads all archetypes for the enemy race across all phases. Builds a single `HypothesisTracker` with 55 hypotheses (filtered to the ~18-23 for the enemy race).
- **Remove phase-transition logic:** No `currentPhase` tracking, no `seedPriors()`, no rebuild guard comparing phase/race/detectors.
- **New field: `transitionMatrix`** — `Map<String, Map<String, Double>>` loaded from classpath JSON. P(archetype_t | archetype_{t-1}) per matchup.
- **New field: `phaseWeights`** — `Map<String, Double>` recalculated each tick based on current game time and each archetype's phase window.
- **`classify()` changes:**
  1. Recalculate phase weights for current game time
  2. Apply transition-aware temporal prior (blend previous posterior with transition matrix)
  3. Run standard evidence fusion
  4. Weight results by phase window relevance
  5. Return assessments

- **New method: `computePhaseWeights(double gameTimeMinutes)`** — returns per-archetype weights based on phase windows. Uses sigmoid ramp at window boundaries.

- **New method: `applyTransitionPrior(Map<String, Double> previousPosterior, Map<String, Double> transitionMatrix)`** — returns predicted prior for current tick.

### Priors loading

`HypothesisPriorLoader` changes:
- New method: `loadTransitionCpts(Race enemyRace)` — loads `transitions/transition-cpts-vs_<race>.json`
- New method: `loadUnifiedPriors(Race enemyRace)` — loads priors for ALL archetypes of a race (not phase-filtered)
- Existing phase-specific loading methods kept for backward compatibility but deprecated

### Network size impact

Current: ~15-20 hypotheses per phase × 1 tracker rebuild per phase transition
New: ~18-23 hypotheses per race (all phases) × 1 tracker for the entire game

The HypothesisTracker maintains per-hypothesis `CompositionInstance` objects. Each instance runs its own junction tree. With 20→23 hypotheses, the per-tick cost increases ~15%. Acceptable given the game loop runs at 22.4 ticks/second (44.6ms budget per tick) and the detector pipeline currently uses <5ms.

## Component 3: Transition learning pipeline

### New test: `StrategyTransitionExportTest`

A `@Tag("report")` test that learns strategy transition probabilities from replay data.

**Input:** 149K replays (oracle + tournament datasets) processed through `ReplayValidationHarness` + `CascadingPatternClassifier`.

**Process:**
1. For each replay, classify the opponent's strategy at 5 checkpoints: 1, 3, 5, 7, 10 minutes
2. For each consecutive checkpoint pair (1→3, 3→5, 5→7, 7→10), record the (strategy_t, strategy_{t+1}) transition
3. Aggregate transitions per matchup (vs_terran, vs_zerg, vs_protoss)
4. Compute conditional probabilities: P(strategy_{t+1} | strategy_t)
5. Apply hybrid filtering:
   - All transitions in `strategy-taxonomy.yaml` are included regardless of frequency
   - Additional transitions are included only if frequency > 5% of games for that matchup
   - Below-threshold transitions get a uniform background probability

**Output:** Three JSON files in `src/main/resources/transitions/`:
- `transition-cpts-vs_terran.json`
- `transition-cpts-vs_zerg.json`
- `transition-cpts-vs_protoss.json`

Each file contains a map of `archetype → { archetype → probability }` representing P(next | current).

### Hierarchical smoothing

Same pattern as `ContextResponsePriorExportTest`:
- Raw counts → add-α smoothing (α=10.0) to avoid zero probabilities
- Taxonomy transitions get a prior count boost (equivalent to 50 additional observations) to ensure they dominate when data is sparse

### Validation

- `StrategyTransitionExportTest` also asserts:
  - Every taxonomy transition has a learned probability > 0.01
  - No transition has probability > 0.95 (no archetype is a deterministic successor)
  - Per-row probabilities sum to 1.0 (valid CPT)
  - At least 10 distinct transitions discovered per matchup above the 5% threshold

## Component 4: Phase-window decay function

### Specification

For each archetype with phase window `[wMin, wMax]` and current game time `t` (minutes):

```
ramp = 1.0  (minutes — configurable)

if t < wMin - ramp:
    weight = 0.01  (floor — never fully zero)
elif t < wMin:
    weight = 0.01 + 0.99 * sigmoid((t - wMin + ramp/2) / (ramp/4))
elif t <= wMax:
    weight = 1.0
elif t < wMax + ramp:
    weight = 0.01 + 0.99 * sigmoid((wMax + ramp/2 - t) / (ramp/4))
else:
    weight = 0.01
```

The floor of 0.01 prevents complete elimination — an EARLY strategy can still be detected in late game (rare but possible in real SC2, e.g., proxy rush at 12 minutes).

### Where it's applied

Phase weights are applied as multiplicative modifiers on the `HypothesisTracker` priors before evidence fusion. They do NOT modify the posteriors — evidence can still overcome a low phase weight if the detector signals are strong enough.

## Testing strategy

### Unit tests

- **`TemporalDetectorOrchestratorTest`** — new test replacing `DetectorOrchestratorTest` flow:
  - Temporal decay smooths jitter (equivalent to old window test)
  - Transition detection works with decay-based posteriors
  - Decay=0.0 behaves like non-temporal
  - Higher decay produces stronger smoothing

- **`UnifiedHypothesisTrackerFusionTest`** — new tests:
  - Single initialization with all archetypes (no phase parameter)
  - Phase-window decay reduces EARLY archetype probability at MID game times
  - Transition prior boosts expected successor archetypes
  - No rebuild on classify() calls across phase boundaries
  - Evidence overcomes low phase weight when signals are strong

- **`PhaseWindowDecayTest`** — unit test for the decay function:
  - Inside window → 1.0
  - Outside window → 0.01
  - Ramp region → smooth transition
  - Floor never zero

- **`StrategyTransitionCptTest`** — unit test for transition matrix loading and application:
  - Loaded CPT rows sum to 1.0
  - Taxonomy transitions present with boosted probability
  - Discovered transitions above threshold included
  - Below-threshold transitions at background level

### Integration tests

- **`StrategyTransitionExportTest`** (`@Tag("report")`) — the learning pipeline itself
- Update existing `PatternClassificationCalibrationTest` to use the unified tracker

### Regression

- Run `DivergenceRegressionTest` before and after to verify classification accuracy doesn't degrade
- Run `GameLoopBenchmarkTest` to verify per-tick latency stays within budget

## Dependency

QuarkMind's `inference-bayes` dependency must include the `TemporalEvidenceFusionEngine` class. This was landed in neocortex commit `7d140756`. Verify the SNAPSHOT version is installed: `mvn install -pl inference-bayes -DskipTests` in neocortex if needed.

`TemporalEvidenceFusionEngine` lacks `getVariableSensitivities()` and `getStats()` which `DetectorOrchestrator` uses. Two options:
1. Add these methods to `TemporalEvidenceFusionEngine` (small neocortex PR)
2. Expose the underlying `BayesInstance` for direct access

Option 1 is cleaner — small cross-repo change.

## References

- `HypothesisTrackerFusion.java:79-138` — current phase-transition reinitialisation
- `DetectorOrchestrator.java:179-217` — current classify() with TemporalAssessmentTracker
- `TemporalEvidenceFusionEngine.java` (neocortex inference-bayes) — temporal decay engine
- `TemporalAssessmentTracker.java` — sliding window smoother (to be replaced)
- `StrategyTaxonomy.java:33-110` — transition paths and phase windows
- `strategy-taxonomy.yaml` — 55 archetypes, 14 hand-authored transitions
- `ContextResponsePriorExportTest` — existing CPT export pattern (model for transition learning)
- `HypothesisPriorLoader.java` — classpath JSON prior loading
- `SC2HypothesisNetworkBuilder.java` — network construction
- neocortex#630 — temporal evidence fusion engine (landed as 7d140756)
- ADR-0010 — Bayesian evidence fusion replaces multi-class ONNX
