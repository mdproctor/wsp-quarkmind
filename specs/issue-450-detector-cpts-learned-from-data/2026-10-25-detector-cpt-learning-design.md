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
4. **For each replay at each phase checkpoint** (EARLY at frame 4031, MID at frame 6719, LATE at frame 9407 — one frame before the `resolvePhase()` boundary, so EARLY features are extracted while `resolvePhase()` still returns EARLY):
   - Parse the reconstituted JSON to extract tracker events
   - Feed tracker events through EmulatedGame, ticking to the checkpoint frame
   - Extract `StrategyFeatures` from the EmulatedGame state via `StrategyFeatureExtractor`
   - Run `BinaryDetectorRunner.runActiveDetectors()` for the **hardcoded phase** (EARLY/MID/LATE — do not call `resolvePhase()` on the checkpoint frame) and enemy race
   - Map the `.label.json` fine-grained label to a catalog label via `CatalogLabelMapper.map(archetype, race)` — skip replays with unmapped labels
   - Discretize each detector output into three buckets: <0.3 → "LOW", 0.3–0.7 → "MEDIUM", >0.7 → "HIGH"
   - Record the observation: `Map.of("Strategy", catalogLabel, det1, "HIGH", det2, "LOW", det3, "MEDIUM", ...)`
5. **Persist observations** as intermediate JSON (one file per matchup × phase) alongside the learned CPTs — enables re-learning without re-extracting, and seeds `OnlineParameterUpdater` for future migration

### Strategy Label Mapping

The `.label.json` sidecars use fine-grained labels from the `StrategyArchetype` enum (e.g., `TERRAN_MARINE_RUSH`, `PROTOSS_CANNON_RUSH`), but the network's Strategy variable uses the detector catalog labels (e.g., `RUSH`, `CANNON_RUSH`, `MACRO_ECONOMY`). A mapping function is required:

The mapping is derived from `DetectorCatalog`'s `positiveArchetypes` lists. For each matchup, every `StrategyArchetype` that appears in any detector's positive list maps to the corresponding catalog label. Examples:
- `TERRAN_MARINE_RUSH` → `RUSH` (via `early_rush` positive for TERRAN)
- `PROTOSS_CANNON_RUSH` → `CANNON_RUSH` (via `early_cheese` positive for PROTOSS)
- `ZERG_ROACH_RUSH` → `ROACH_RUSH` (via `early_rush` positive for ZERG)

Unmapped `StrategyArchetype` values (those not appearing in any detector's positive list for the given matchup) are **excluded from training** — they represent strategies the detector network has no variable for, so including them would add noise.

This mapping is implemented as a static utility method `CatalogLabelMapper.map(StrategyArchetype, Race) → Optional<String>`, testable in isolation.

### Three-Bucket Discretization

Production inference in `DetectorOrchestrator.classify()` uses continuous soft evidence — a detector scoring 0.95 is much stronger evidence than one scoring 0.51. Binary HIGH/LOW discretization loses this magnitude information during training.

Three buckets (LOW/MEDIUM/HIGH) capture the distinction between confident detection, borderline detection, and confident absence. The learned CPTs then model P(HIGH|RUSH), P(MEDIUM|RUSH), P(LOW|RUSH) separately.

**Training-time discretization:** Detector outputs → discrete labels for `ParameterLearner.fit()`:
- Score < 0.3 → "LOW"
- Score 0.3–0.7 → "MEDIUM"
- Score > 0.7 → "HIGH"

**Inference-time evidence mapping:** `DetectorOrchestrator.classify()` maps continuous detector scores to 3-element soft likelihood vectors using a piecewise linear function, preserving magnitude information:

```
score → [P(HIGH), P(MEDIUM), P(LOW)]
  0.0 → [0.0,  0.0,  1.0]
  0.3 → [0.0,  0.5,  0.5]
  0.5 → [0.0,  1.0,  0.0]
  0.7 → [0.5,  0.5,  0.0]
  1.0 → [1.0,  0.0,  0.0]
```

Hard evidence (≥ `hardEvidenceThreshold` = 0.95): `[1.0, 0.0, 0.0]`. Hard negative (≤ 0.05): `[0.0, 0.0, 1.0]`.

**Fallback compatibility:** When the fallback path fires (2-outcome `SC2NetworkBuilder.build()` network), the evidence-setting code must detect the outcome count and use the existing 2-element mapping. This is determined from `engine.getOutcomeNames().length` — if 2, use the current binary mapping; if 3, use the piecewise linear mapping above.

**Threshold calibration:** The initial thresholds (0.3/0.7) must be validated against actual detector output distributions during `CptLearningExportTest`. If detector outputs are bimodal (clustering near 0.0 and 1.0), the MEDIUM bucket may be nearly empty — in that case, revert to binary discretization. Log the per-detector bucket distribution as part of the export report.

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

**Jackson sealed interface handling:** `CptSpec` is a sealed interface with record implementations (`DirectCpt`, `NoisyOr`, `Uniform`). Jackson cannot polymorphically deserialize sealed interfaces without type annotations. **Prerequisite:** add `@JsonTypeInfo`/`@JsonSubTypes` annotations to `CptSpec` in neocortex's `inference-bayes` module. This is a small, one-time change — a few annotations on the sealed interface and its implementations. File a neocortex issue for this before implementation begins.

**BayesNetwork → NetworkSpec conversion:** No `fromBayesNetwork()` method exists. Add a static factory `NetworkSpec.fromBayesNetwork(BayesNetwork)` to neocortex that walks the graph nodes, extracts variables/edges/CPTs, and builds the spec. This belongs in neocortex alongside `toBayesNetwork()` — the conversion should live close to the type. File a neocortex issue for this alongside the Jackson annotations.

### Integration into DetectorOrchestrator and CascadingPatternClassifier

The integration point is `DetectorOrchestrator.initializeNetwork(detectorNames)`, which currently calls `SC2NetworkBuilder.build()` every time the active detector set changes. Learned networks are per matchup × phase, matching the active detector set exactly.

**DetectorOrchestrator changes:**

1. Constructor gains an optional `Map<String, BayesNetwork> learnedNetworks` — keyed by `"{phase}_vs_{race}"` (e.g., `"EARLY_vs_terran"`)
2. `initializeNetwork()` signature changes: `initializeNetwork(List<String> detectorNames, GamePhase phase, Race enemyRace)` — the phase and race are needed to look up the learned network. The call site in `CascadingPatternClassifier.classify()` already has both values.
3. If a learned network exists for the phase+race key, use it directly; otherwise fall back to `SC2NetworkBuilder.build()`
4. The evidence-mapping logic in `classify()` checks `outcomeNames.length` to select between 3-element (learned) and 2-element (fallback) likelihood vectors
5. The learned networks are pre-loaded at construction time, not per-tick

**CascadingPatternClassifier changes:**

`buildDefaultOrchestrator()` is currently `private static` and cannot access Quarkus config. The learned-CPT loading moves to a new CDI bean — `LearnedCptLoader`:

```java
@ApplicationScoped
class LearnedCptLoader {
    @ConfigProperty(name = "quarkmind.detector.require-learned-cpts",
                    defaultValue = "false")
    boolean requireLearnedCpts;

    Map<String, BayesNetwork> load() { ... }
}
```

1. For each matchup × phase in the detector catalog, attempt to load `classifier/learned-cpts-{phase}-vs-{race}.json` from classpath
2. **If found:** deserialize `LearnedCptResource`, call `network().toBayesNetwork()`, store in map
3. **If not found and `requireLearnedCpts` is true:** throw `IllegalStateException`
4. **If not found and `requireLearnedCpts` is false:** log WARN, omit from map (orchestrator falls back to `build()`)

Config property set per profile:
- `%sc2`, `%emulated`, `%coach`: `quarkmind.detector.require-learned-cpts=true`
- All others: default `false`

The `CascadingPatternClassifier` constructor (already CDI-managed) receives the loaded map from `LearnedCptLoader` and passes it to `DetectorOrchestrator`.

**Note on issue #450 divergence:** Issue #450's acceptance criteria state "hand-tuned CPTs used if learned data unavailable." This spec intentionally changes that to fail-fast in production profiles because silently using garbage CPTs is worse than failing — the whole point of this work is that the hand-tuned values are unreliable. The fallback path remains for dev/test profiles where classification quality is incidental.

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
- `CatalogLabelMapper` — fine-grained StrategyArchetype → catalog label mapping
- `LearnedCptResource` — wrapper record for metadata + NetworkSpec
- `LearnedCptLoader` — CDI bean for classpath loading with profile-gated fallback
- `CascadingPatternClassifier` — integration to receive learned networks
- `DetectorOrchestrator` — learned network injection, 3-element evidence mapping, `initializeNetwork(names, phase, race)` signature change
- `CptLearningValidationTest` — accuracy comparison
- NetworkSpec JSON resource files (9: 3 matchups × 3 phases)
- Observation persistence for future online learning migration

### Prerequisites (neocortex cross-repo)
- `@JsonTypeInfo`/`@JsonSubTypes` annotations on `CptSpec` sealed interface — required for Jackson deserialization
- `NetworkSpec.fromBayesNetwork(BayesNetwork)` — required for converting learned networks to serializable spec

### Out of scope
- Online CPT refinement during live games (future: `OnlineParameterUpdater` — the persisted observations provide the seeding interface)
- Structure learning (the DAG topology is fixed — naive Bayes star)
- Changes to ONNX detector models or the heuristic labeller

## Acceptance Criteria

- [ ] CPTs learned from ≥10K labelled replays per matchup
- [ ] Classification accuracy improves over hand-tuned 0.8/0.2 baseline by ≥2% top-1 accuracy (measured on held-out 20% set)
- [ ] Learned CPTs serialized as NetworkSpec JSON, loadable at startup via `toBayesNetwork()`
- [ ] Profile-gated fallback: fail fast in sc2/emulated/coach, warn+fallback in mock/test/replay
- [ ] Semantic validation at load time (CPT row sums, catalog label names, detector names per phase/race)
- [ ] `CptLearningExportTest` is reproducible (same inputs → same outputs)
- [ ] Observations persisted alongside learned CPTs for future online learning
- [ ] Discretization threshold validation: log per-detector bucket distributions, revert to binary if MEDIUM bucket is <5% of observations

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
