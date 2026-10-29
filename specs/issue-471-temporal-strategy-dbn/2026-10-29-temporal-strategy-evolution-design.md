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

### Changes — composition approach

Rather than a monolithic replacement, compose temporal blending with the existing `EvidenceFusionEngine` to preserve its full API (`mapInference()`, `explain()`, `getVariableSensitivities()`, `getStats()`). The `TemporalEvidenceFusionEngine` pattern (linear blending of posteriors) is simple enough to implement in the quarkmind layer without requiring the neocortex class directly.

- **Field changes:** Remove `TemporalAssessmentTracker tracker`. Keep `EvidenceFusionEngine engine`. Add `double temporalDecay` and `double[] previousPosterior` fields.
- **Constructor:** Replace `int windowSize` with `double temporalDecay`. Default decay: `0.3` (30% carry-over from previous tick — equivalent smoothing to a ~3-tick window).
- **`classify()`:** After `engine.update()` and `engine.marginalize()`, apply temporal blending inline: `smoothed[i] = (1 - decay) * current[i] + decay * previous[i]`. Store `smoothed` as `previousPosterior` for next tick. Build assessments from smoothed posterior.
- **`detectTransition()`:** Compare consecutive smoothed posteriors for dominant-hypothesis change.
- **`reset()`:** Call `engine.reset()` and clear `previousPosterior`.
- **Preserved API:** `getMapStrategy()`, `getExplanationTrace()`, `getSensitivities()`, `getStats()` all continue to work via the underlying `EvidenceFusionEngine`.

This supersedes ADR-0010's choice of windowing over decay. The windowing approach (TemporalAssessmentTracker) was correct for its time — it provided stability without introducing temporal coupling. But with transition-aware priors and phase-window decay now providing structural temporal awareness, the simpler decay blending is sufficient for tick-to-tick smoothing, and windowing's equal-weight-across-window assumption is actually less appropriate than exponential decay for strategy evolution (recent evidence should matter more than evidence 5 ticks ago).

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

Initialize once with all archetypes for the enemy race (~18-23 per race, ~69 total across all races but only one race's archetypes are active per game). No phase-transition rebuild. Temporal continuity is maintained by:

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
  2. Apply phase weights as multiplicative modifiers on tracker priors (before evidence)
  3. Set per-tick detector evidence and drools markers (same as today)
  4. Run standard evidence fusion via `HypothesisTracker` observe/setPerTickEvidence
  5. Get raw posteriors from tracker weights
  6. Apply transition-aware temporal blending: `smoothed[j] = (1 - decay) * raw[j] + decay * sum_i(prev[i] * P(j|i))` where P(j|i) is the learned transition CPT and `prev` is the previous tick's smoothed posterior
  7. Return assessments built from smoothed posterior

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

The HypothesisTracker maintains per-hypothesis `CompositionInstance` objects. Each instance runs its own junction tree. Per-race archetype counts: Terran ~23, Zerg ~20, Protoss ~21. This is larger than the current per-phase networks (~15-20) but not dramatically so because the tracker is already race-filtered.

**Prior dilution mitigation:** With ~23 hypotheses, uniform priors give each ~4.3% — close to the current ~5% per phase-filtered network. Phase-window decay further concentrates probability mass: at minute 2, only EARLY archetypes (~6-9 per race) have full weight, giving each an effective prior of ~11-17%. The phase-window decay function is critical to avoiding dilution — it must be applied before evidence fusion, not after.

**Latency:** Inference is O(N × D) for star topology. At ~23 archetypes × ~15 detectors, the per-tick cost is manageable within the 44.6ms budget (detector pipeline currently uses <5ms). Run `GameLoopBenchmarkTest` before and after to verify.

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

**Note on LATE-game coverage:** The 10-minute cap means MID→LATE transitions (typically 10-15 min) have limited data. LATE archetypes will have transition priors dominated by the taxonomy's hand-authored entries and the hierarchical smoothing α. This is acceptable for v1 — LATE-game strategy is more stable (fewer transitions) and can be extended to 15 minutes in a follow-up if data shows gaps.

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

Option 1 is cleaner — small cross-repo change. This is NOT a blocker: DetectorOrchestrator can initially skip sensitivity analysis, or access the underlying instance directly. The neocortex PR can follow.

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
