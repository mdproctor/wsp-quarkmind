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
2. Extracts detector outputs at game-phase checkpoints (3min, 5min, 7min)
3. Discretizes detector outputs into three buckets: LOW (<0.3), MEDIUM (0.3–0.7), HIGH (>0.7)
4. Pairs with heuristic strategy labels from `.label.json` sidecars
5. Calls `ParameterLearner.fit()` to learn CPTs from data
6. Serializes the learned network as `NetworkSpec` JSON (one per matchup × phase = 9 files)
7. At startup, loads learned CPTs via `NetworkSpec.toBayesNetwork()` instead of the 0.8/0.2 defaults

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
    Learned BayesNetwork → NetworkSpec
         │
         ▼
    JSON resource files (9: 3 matchups × 3 phases)
```

### Training Data Extraction

A new `@Tag("report")` test — `CptLearningExportTest` — following the established pattern of `ReconstitutionExportTest`:

1. **Iterate reconstituted replays** from `../quarkmind-classifier/data/reconstituted/` with their `.label.json` sidecars
2. **Filter** to replays with non-UNKNOWN labels and sufficient game length (≥3min for EARLY, ≥5min for MID, ≥7min for LATE)
3. **Group by matchup** (vs_terran, vs_zerg, vs_protoss) — each matchup has different active detectors
4. **For each replay at each phase checkpoint** (3min = loop 4032, 5min = loop 6720, 7min = loop 9408):
   - Parse the reconstituted JSON to extract tracker events
   - Feed tracker events through EmulatedGame, ticking to the checkpoint frame
   - Extract `StrategyFeatures` from the EmulatedGame state via `StrategyFeatureExtractor`
   - Run `BinaryDetectorRunner.runActiveDetectors()` for the phase and enemy race
   - Discretize each detector output into three buckets: <0.3 → "LOW", 0.3–0.7 → "MEDIUM", >0.7 → "HIGH"
   - Record the observation: `Map.of("Strategy", label, det1, "HIGH", det2, "LOW", det3, "MEDIUM", ...)`
5. **Persist observations** as intermediate JSON (one file per matchup × phase) alongside the learned CPTs — enables re-learning without re-extracting, and seeds `OnlineParameterUpdater` for future migration

### Three-Bucket Discretization

Production inference in `DetectorOrchestrator.classify()` uses continuous soft evidence — a detector scoring 0.95 is much stronger evidence than one scoring 0.51. Binary HIGH/LOW discretization loses this magnitude information during training.

Three buckets (LOW/MEDIUM/HIGH) capture the distinction between confident detection, borderline detection, and confident absence. The learned CPTs then model P(HIGH|RUSH), P(MEDIUM|RUSH), P(LOW|RUSH) separately. At inference time, detector outputs are discretized to the same three buckets before setting evidence via the `EvidenceFusionEngine`.

The thresholds (0.3 and 0.7) are chosen to roughly partition the continuous score range into equal-information bands. They can be calibrated from the actual detector output distributions in the training data.

### Parameter Learning

For each matchup × phase combination (9 total):

1. Build the network **structure** — `Strategy` root with phase-specific detector children (3 outcomes each: HIGH/MEDIUM/LOW)
2. Call `ParameterLearner.fit(structure, observations)` — returns a new `BayesNetwork` with MLE-learned CPTs (Laplace smoothing α=1.0)
3. Convert the learned `BayesNetwork` to a `NetworkSpec` via a helper that walks the variables and CPTs

### Persistence via NetworkSpec

Neocortex's `NetworkSpec` record hierarchy (`NetworkSpec`, `VariableSpec`, `EdgeSpec`, `CptSpec.DirectCpt`) is the purpose-built model for Bayesian network structure + CPTs. All are Java records — Jackson handles serialization natively.

Each learned network is wrapped in a metadata envelope:

```java
record LearnedCptResource(
    String matchup,           // "vs_terran"
    String phase,             // "EARLY"
    LearnedFrom learnedFrom,
    NetworkSpec network
) {}

record LearnedFrom(
    int replayCount,
    int observationCount,
    String extractorVersion,
    String generatedAt
) {}
```

Stored at `quarkmind-sc2/src/main/resources/classifier/learned-cpts-{phase}-vs-{race}.json` (9 files).

At load time: `ObjectMapper.readValue(is, LearnedCptResource.class)` → `resource.network().toBayesNetwork()`.

**Jackson sealed interface handling:** `CptSpec` is a sealed interface with record implementations (`DirectCpt`, `NoisyOr`, `Uniform`). The existing `ResponseParser` in `inference-bayes-llm` already deserializes `CptSpec` via Jackson using a type discriminator. If the core module doesn't have `@JsonTypeInfo` annotations, add them — a one-time annotation addition, not a new serialization framework.

### Integration into DetectorOrchestrator and CascadingPatternClassifier

The integration point is `DetectorOrchestrator.initializeNetwork(detectorNames)`, which currently calls `SC2NetworkBuilder.build()` every time the active detector set changes. Learned networks are per matchup × phase, matching the active detector set exactly.

**DetectorOrchestrator changes:**

1. Constructor gains an optional `Map<String, BayesNetwork> learnedNetworks` — keyed by `"{phase}_vs_{race}"` (e.g., `"EARLY_vs_terran"`)
2. `initializeNetwork()` checks if a learned network exists for the current phase + race key — if yes, uses it directly; if no, falls back to `SC2NetworkBuilder.build()`
3. The learned networks are pre-loaded at construction time, not per-tick

**CascadingPatternClassifier.buildDefaultOrchestrator() changes:**

1. For each matchup × phase in the detector catalog, attempt to load `classifier/learned-cpts-{phase}-vs-{race}.json` from classpath
2. **If found:** deserialize `LearnedCptResource`, call `network().toBayesNetwork()`, store in the learned networks map
3. **If not found:**
   - In `%sc2`, `%emulated`, `%coach` profiles: throw `IllegalStateException` — classification quality matters
   - In `%mock`, `%test`, `%replay` profiles: fall back to the 0.8/0.2 defaults with a WARN log
4. Pass the learned networks map to `DetectorOrchestrator` constructor

The profile check uses a Quarkus config property (`quarkmind.detector.require-learned-cpts`, defaulting to `false`, set to `true` in sc2/emulated/coach profiles).

**Semantic validation at load time:** After deserializing, validate:
- CPT rows sum to 1.0 (within ε)
- Archetype names match `StrategyArchetype` enum values
- Detector names match `DetectorCatalog` entries for the stated phase and race
- Priors are non-zero

### Validation

A companion test — `CptLearningValidationTest` — measures accuracy improvement:

1. Hold out 20% of replays as a test set during `CptLearningExportTest`
2. Run classification on the test set with **learned CPTs** vs **hand-tuned 0.8/0.2**
3. Compare: top-1 accuracy, mean posterior for correct archetype, calibration (do 80% predictions match 80% actual?)
4. Assert learned CPTs produce strictly better top-1 accuracy than hand-tuned baseline

## Assumptions and Limitations

- **Label quality:** Strategy labels from the heuristic labeller are treated as ground truth. If the labeller has systematic confusion (e.g., mislabelling MACRO_ECONOMY as BIO_TIMING), the learned CPTs encode those errors. Out of scope for this issue — label quality improvements belong to the classifier pipeline.
- **Star topology:** The naive Bayes assumption (detectors conditionally independent given strategy) is retained. Detectors like `early_rush` and `early_greedy` are definitionally anti-correlated, which naive Bayes can't model. Structure learning would address this but is out of scope.
- **Laplace smoothing α=1.0:** With 10K+ observations per matchup, α=1.0 is negligible for common archetypes. For rare archetypes (e.g., CANNON_RUSH at <2% of games), the smoothing constant matters more. If rare-archetype CPTs look degenerate, α can be tuned — but start with the default.

## Scope

### In scope
- `CptLearningExportTest` — batch extraction + learning + NetworkSpec JSON export
- `LearnedCptResource` — wrapper record for metadata + NetworkSpec
- `CascadingPatternClassifier` — integration with profile-gated fallback
- `DetectorOrchestrator` — learned network injection
- `CptLearningValidationTest` — accuracy comparison
- NetworkSpec JSON resource files (9: 3 matchups × 3 phases)
- Observation persistence for future online learning migration

### Out of scope
- Online CPT refinement during live games (future: `OnlineParameterUpdater` — the persisted observations provide the seeding interface)
- Structure learning (the DAG topology is fixed — naive Bayes star)
- Changes to ONNX detector models or the heuristic labeller
- Jackson annotations on neocortex `CptSpec` (if needed — tracked as a dependency)

## Acceptance Criteria

- [ ] CPTs learned from ≥10K labelled replays per matchup
- [ ] Classification accuracy improves over hand-tuned 0.8/0.2 baseline (measured on held-out set)
- [ ] Learned CPTs serialized as NetworkSpec JSON, loadable at startup via `toBayesNetwork()`
- [ ] Profile-gated fallback: fail fast in sc2/emulated/coach, warn+fallback in mock/test/replay
- [ ] Semantic validation at load time (CPT sums, archetype names, detector names)
- [ ] `CptLearningExportTest` is reproducible (same inputs → same outputs)
- [ ] Observations persisted alongside learned CPTs for future online learning

## References

- `SC2NetworkBuilder.java` — current network construction (star topology, 0.8/0.2 CPTs)
- `CascadingPatternClassifier.java:385-415` — hand-coded CPT assignment
- `DetectorCatalog.java` — 9 detector definitions with phase/race/archetype mappings
- `DetectorOrchestrator.java` — Bayesian fusion consumer
- `ParameterLearner.java` (neocortex) — MLE batch learning API
- `OnlineParameterUpdater.java` (neocortex) — incremental learning (future seeding interface)
- `NetworkSpec.java` (neocortex) — persistence format with `toBayesNetwork()` conversion
- `CptSpec.java` (neocortex) — sealed interface for CPT representations
- `ReconstitutionExportTest.java` — pattern for batch replay processing tests
- `BinaryDetectorRunner.java` — production ONNX inference path
- `StrategyFeatureExtractor.java` — production feature extraction
- neocortex #644 — parameter learning implementation (closed)
- neocortex #652 — online parameter updater (closed)
