# Detector CPT Learning — Design Spec

**Issue:** casehubio/quarkmind#450
**Branch:** issue-450-detector-cpts-learned-from-data
**Date:** 2026-10-25

## Problem

`CascadingPatternClassifier.buildDefaultOrchestrator()` constructs Bayesian network CPTs with a uniform `0.8 / 0.2` split: P(HIGH | archetype) = 0.8 if the detector is a positive indicator for that archetype, 0.2 otherwise. Every detector gets the same strength regardless of how discriminative it actually is. This produces suboptimal posteriors — a weak detector and a strong detector contribute equally to strategy classification.

We have 1.57M labelled replays with both strategy labels (from the heuristic labeller) and the ONNX detector models trained on this data. The neocortex `ParameterLearner.fit()` API is ready (issue #644, closed). The ingredients exist to replace guesses with data.

## Solution Overview

An offline batch learning pipeline that:
1. Runs reconstituted replays through the **production Java inference path** (EmulatedGame → StrategyFeatures → BinaryDetectorRunner)
2. Extracts detector outputs at game-phase checkpoints (3min, 5min)
3. Pairs with heuristic strategy labels from `.label.json` sidecars
4. Calls `ParameterLearner.fit()` to learn CPTs from data
5. Serializes the learned network to JSON resource files (one per matchup)
6. At startup, loads learned CPTs instead of the 0.8/0.2 defaults

## Architecture

### Data Flow

```
Reconstituted replays (.json)     .label.json sidecars
         │                              │
         ▼                              │
    EmulatedGame (tick to checkpoint)    │
         │                              │
         ▼                              │
    StrategyFeatureExtractor            │
         │                              │
         ▼                              │
    BinaryDetectorRunner (ONNX)         │
         │                              │
         ▼                              ▼
    Map<String, String> observations
    { "Strategy"="RUSH",
      "early_rush_vs_terran"="HIGH",
      "early_greedy_vs_terran"="LOW", ... }
         │
         ▼
    ParameterLearner.fit(structure, observations)
         │
         ▼
    Learned BayesNetwork
         │
         ▼
    JSON resource file (learned-cpts-vs-terran.json, etc.)
```

### Training Data Extraction

A new `@Tag("report")` test — `CptLearningExportTest` — following the established pattern of `ReconstitutionExportTest`:

1. **Iterate reconstituted replays** from `../quarkmind-classifier/data/reconstituted/` with their `.label.json` sidecars
2. **Filter** to replays with non-UNKNOWN labels and sufficient game length (≥3min for EARLY, ≥5min for MID)
3. **Group by matchup** (vs_terran, vs_zerg, vs_protoss) — each matchup has different active detectors
4. **For each replay at each phase checkpoint:**
   - Parse the reconstituted JSON to extract tracker events
   - Feed tracker events through EmulatedGame, ticking to the checkpoint frame (3min = loop 4032, 5min = loop 6720)
   - Extract `StrategyFeatures` from the EmulatedGame state via `StrategyFeatureExtractor`
   - Run `BinaryDetectorRunner.runActiveDetectors()` for the phase and enemy race
   - Threshold each detector output: ≥0.5 → "HIGH", <0.5 → "LOW"
   - Record the observation: `Map.of("Strategy", label, det1, "HIGH", det2, "LOW", ...)`

### Parameter Learning

For each matchup:

1. Build the network **structure** via existing `SC2NetworkBuilder.build()` (topology only — CPT values don't matter, they'll be overwritten)
2. Call `ParameterLearner.fit(structure, observations)` — returns a new `BayesNetwork` with MLE-learned CPTs (Laplace smoothing α=1.0)
3. Extract learned CPTs and serialize to JSON

### JSON Resource Format

One file per matchup, stored at `quarkmind-sc2/src/main/resources/classifier/learned-cpts-vs-{race}.json`:

```json
{
  "matchup": "vs_terran",
  "learnedFrom": {
    "replayCount": 12345,
    "observationCount": 24690,
    "extractorVersion": "abc123",
    "generatedAt": "2026-10-25T12:00:00Z"
  },
  "priors": {
    "RUSH": 0.15,
    "MACRO_ECONOMY": 0.35,
    "BIO_TIMING": 0.20,
    ...
  },
  "detectors": {
    "early_rush_vs_terran": {
      "outcomes": ["HIGH", "LOW"],
      "cpt": {
        "RUSH": [0.92, 0.08],
        "MACRO_ECONOMY": [0.05, 0.95],
        "BIO_TIMING": [0.12, 0.88],
        ...
      }
    },
    ...
  }
}
```

The format stores CPTs in a human-readable archetype-keyed map rather than raw `double[][]` arrays — reviewable in PRs, and the archetype ordering is explicit rather than implicit.

### Network Loading

`SC2NetworkBuilder` gains a `load(InputStream)` method:

1. Parse the JSON resource
2. Build a `BayesNetwork` via `BayesNetworkBuilder`:
   - Root `"Strategy"` variable with learned priors
   - Each detector variable with learned CPTs
3. Return the assembled network

The existing `build(priors, detectorNames, cpts)` method remains unchanged — it's the fallback path for profiles without learned CPTs.

### Integration into DetectorOrchestrator and CascadingPatternClassifier

The integration point is `DetectorOrchestrator.initializeNetwork(detectorNames)`, which currently calls `SC2NetworkBuilder.build()` every time the active detector set changes. The learned CPTs are per-matchup, and the active detectors are filtered by enemy race at classify time.

**DetectorOrchestrator changes:**

1. Constructor gains an optional `Map<String, BayesNetwork> learnedNetworks` — keyed by matchup (e.g., `"vs_terran"`)
2. `initializeNetwork()` checks if a learned network exists for the current detector set — if yes, uses it directly; if no, falls back to `SC2NetworkBuilder.build()`
3. The learned networks are pre-loaded at construction time, not per-tick

**CascadingPatternClassifier.buildDefaultOrchestrator() changes:**

1. For each matchup in the detector catalog, attempt to load `classifier/learned-cpts-vs-{race}.json` from classpath
2. **If found:** parse via `SC2NetworkBuilder.load()`, store in the learned networks map
3. **If not found:**
   - In `%sc2`, `%emulated`, `%coach` profiles: throw `IllegalStateException` — classification quality matters
   - In `%mock`, `%test`, `%replay` profiles: fall back to the 0.8/0.2 defaults with a WARN log
4. Pass the learned networks map to `DetectorOrchestrator` constructor

The profile check uses a Quarkus config property (`quarkmind.detector.require-learned-cpts`, defaulting to `false`, set to `true` in sc2/emulated/coach profiles).

### Validation

A companion test — `CptLearningValidationTest` — measures accuracy improvement:

1. Hold out 20% of replays as a test set during `CptLearningExportTest`
2. Run classification on the test set with **learned CPTs** vs **hand-tuned 0.8/0.2**
3. Compare: top-1 accuracy, mean posterior for correct archetype, calibration (do 80% predictions match 80% actual?)
4. Assert learned CPTs produce strictly better top-1 accuracy than hand-tuned baseline

## Scope

### In scope
- `CptLearningExportTest` — batch extraction + learning + JSON export
- `SC2NetworkBuilder.load()` — JSON resource loading
- `CascadingPatternClassifier` — integration with profile-gated fallback
- `CptLearningValidationTest` — accuracy comparison
- JSON resource files for 3 matchups

### Out of scope
- Online CPT refinement during live games (future: `OnlineParameterUpdater`)
- Structure learning (the DAG topology is fixed — naive Bayes star)
- Neocortex serialization API (we serialize in quarkmind, not in the library)
- Changes to ONNX detector models or the heuristic labeller

## Acceptance Criteria

- [ ] CPTs learned from ≥10K labelled replays per matchup
- [ ] Classification accuracy improves over hand-tuned 0.8/0.2 baseline (measured on held-out set)
- [ ] Learned CPTs serialized as JSON resources, loadable at startup
- [ ] Profile-gated fallback: fail fast in sc2/emulated/coach, warn+fallback in mock/test/replay
- [ ] `CptLearningExportTest` is reproducible (same inputs → same outputs)

## References

- `SC2NetworkBuilder.java` — current network construction (star topology, 0.8/0.2 CPTs)
- `CascadingPatternClassifier.java:385-415` — hand-coded CPT assignment
- `DetectorCatalog.java` — 9 detector definitions with phase/race/archetype mappings
- `DetectorOrchestrator.java` — Bayesian fusion consumer
- `ParameterLearner.java` (neocortex) — MLE batch learning API
- `OnlineParameterUpdater.java` (neocortex) — incremental learning (future)
- `ReconstitutionExportTest.java` — pattern for batch replay processing tests
- `BinaryDetectorRunner.java` — production ONNX inference path
- `StrategyFeatureExtractor.java` — production feature extraction
- neocortex #644 — parameter learning implementation (closed)
- neocortex #652 — online parameter updater (closed)
