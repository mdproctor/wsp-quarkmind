# Learned Scouting Network Structure — Design Spec

**Issue:** casehubio/quarkmind#470
**Branch:** issue-470-learned-scouting-network-structure
**Date:** 2026-10-29

## Problem

QuarkMind's scouting Bayesian network uses a hand-specified naïve Bayes star topology: `enemy_strategy` as root, all 7 observation variables and N detector variables as direct children, all conditionally independent given the strategy. This topology was designed by intuition — which observations matter for which archetypes was a human judgment call.

Neocortex now has structure learning (PC algorithm, hill-climbing BIC, LLM+data fusion) that can discover the optimal network topology from data, potentially finding cross-observation dependencies the star topology misses (e.g. `gas_timing` and `tech_path` may be conditionally dependent given certain strategies).

## Solution

Replace the hand-specified topology with a data-learned structure, using neocortex's `StructureLearner` to discover conditional independence relationships from reconstituted replay data. The hand-specified star becomes one `NetworkSpec` among several — a baseline to compare against, not a privileged construction path.

## Architecture

### Three components, one data flow

1. **Tabular Data Extractor** — A `@Tag("report")` test (`StructureLearningExportTest`) reads reconstituted replay JSON, extracts per-phase per-matchup tabular data as `List<Map<String,String>>`. Each row contains `enemy_strategy` + 7 observation variable states. 9 partitions total: 3 phases (EARLY, MID, LATE) × 3 matchups (vs_protoss, vs_terran, vs_zerg).

2. **Structure Learner** — Calls neocortex `StructureLearner.pc()` and `.hillClimbing()` on each partition. Four variants per partition:
   - Unconstrained PC
   - Unconstrained hill-climbing BIC
   - Constrained hill-climbing with domain invariants (`enemy_strategy` as root, no reverse edges from observations to `enemy_strategy`)
   - LLM-seeded hill-climbing (via `NetworkElicitor.propose()` → `StructuralConstraints.fromSpec()`)

   Each variant produces a `StructureResult` with `networkSpec()` (the learned DAG) and `toBayesNetwork(data)` (full BN with CPTs).

3. **Comparator** — K-fold cross-validated evaluation. Primary metric: log-likelihood on held-out data. Secondary: Brier score and per-archetype classification accuracy. All topologies — including the hand-specified star — evaluated with data-fitted CPTs from `ParameterLearner.fit()` to isolate the structure contribution. Star with hand-tuned CPTs as additional secondary baseline. 5% improvement threshold for switching. Paired bootstrap test for statistical significance.

### Data flow

```
reconstituted replays (JSON)
    │
    ▼
StructureLearningExportTest
    │  extracts per-phase per-matchup tabular data
    │  (EARLY/MID/LATE × vs_protoss/vs_terran/vs_zerg)
    │
    ▼
List<Map<String,String>> per partition (9 partitions)
    │
    ├──► StructureLearner.pc(data, alpha)                    [unconstrained]
    ├──► StructureLearner.hillClimbing(data)                  [unconstrained]
    ├──► StructureLearner.hillClimbing(data, constraints)     [domain invariants]
    └──► NetworkElicitor.propose(desc) → constraints
         └──► StructureLearner.hillClimbing(data, constraints) [LLM-seeded]
    │
    ▼
4 × StructureResult per partition (36 total)
    │
    ▼
K-fold cross-validation comparator
    │  log-likelihood, Brier score, per-archetype accuracy
    │  hand-specified star (data-fitted CPTs) as baseline
    │
    ▼
Winner per partition → NetworkSpec
    │  augment with drools_evidence
    │
    ▼
src/main/resources/priors/learned-topology-{PHASE}-vs_{race}.json
    │
    ▼
SC2HypothesisNetworkBuilder.buildIntelTree(NetworkSpec)
    │
    ▼
JunctionTree → CompositionBuilder → HypothesisTracker
```

### Post-structure augmentation

Every learned `NetworkSpec` destined for the "intel" slot gets `drools_evidence` appended as a `VariableSpec` with `CptSpec.DirectCpt` using the existing soft-identity CPT (0.8 diagonal, 0.2/(N-1) off-diagonal, conditioned on `enemy_strategy`). This is a post-learning augmentation step — `drools_evidence` is a platform integration artifact absent from training data, but required at runtime by `HypothesisTrackerFusion.classify()` (5 usage sites: `setPerTickEvidence`, `clearPerTickEvidence`, and `marginalize` calls on the "intel" slot).

### Builder refactor

`SC2HypothesisNetworkBuilder.buildIntelTree()` is refactored to:
1. Produce a `NetworkSpec` encoding its current topology, clamping (`Math.max(0.01, Math.min(0.99, val))`), normalization, and drools_evidence
2. Both hand-specified and learned topologies flow through the same `NetworkSpec → JunctionTree` construction path via `NetworkSpec.toBuilder().buildJunctionTree()`
3. The hand-specified star is the default `NetworkSpec`; learned topologies are alternatives loaded from JSON resources

This unifies construction across `SC2HypothesisNetworkBuilder` (observations + detectors + drools for HypothesisTracker) and aligns with `DetectorOrchestrator.learnedNetworks` which already consumes `BayesNetwork` objects via `NetworkSpec.toBayesNetwork()` through `LearnedCptLoader`.

## Per-phase structure learning

The existing observation CPTs are organized per-phase per-matchup (`observation-cpts-{PHASE}-vs_{race}.json`, 9 files total). The PC algorithm's conditional independence tests are distribution-sensitive — in EARLY, `first_building` and `gas_timing` carry the most signal; by MID, `tech_path` and `army_composition` dominate. Learning structure per-phase respects these distributional differences and aligns with the `GamePhase` enum (EARLY, MID, LATE) used throughout `HypothesisPriorLoader`, `DetectorOrchestrator`, and `CascadingPatternClassifier`.

3 phases × 3 matchups = 9 partitions. With ~50K replays per matchup, each partition has adequate sample size for chi-squared and Fisher exact tests. For deeper conditioning sets (3+ parents), sparse cells trigger Fisher exact fallbacks in `IndependenceTest.adaptive()` — this produces sparser, more conservative structures, which is acceptable.

## Comparison methodology

All topologies are evaluated with data-fitted CPTs to isolate the structure contribution:

| Topology | CPT source | Purpose |
|----------|-----------|---------|
| Hand-specified star | Data-fitted | Primary baseline — same parameter source as learned |
| Hand-specified star | Hand-tuned (current production) | Secondary baseline — practical "what's deployed" |
| Unconstrained PC | Data-fitted | Pure data-driven |
| Unconstrained HC | Data-fitted | Pure data-driven |
| Constrained HC | Data-fitted | Domain invariants only |
| LLM-seeded HC | Data-fitted | Full domain knowledge |

**Metrics:**
- **Primary:** Cross-validated log-likelihood (information-theoretic model quality)
- **Secondary:** Brier score (calibration quality), per-archetype classification accuracy
- **K-fold:** At replay level (all observations from one replay in same fold) to prevent data leakage
- **Significance:** Paired bootstrap test on per-replay log-likelihood scores
- **Threshold:** 5% log-likelihood improvement required to justify switching from star

## Pipeline coexistence

The new structure learning pipeline coexists with `CptLearningExportTest`. Both export to `LearnedCptResource` format wrapping `NetworkSpec`:

- `CptLearningExportTest` → learns detector CPTs from ONNX classifier outputs (fixed star topology)
- `StructureLearningExportTest` → learns network structure from observation data

If the learned structure wins: detector CPTs from `CptLearningExportTest` are preserved and plugged into the learned structure's `NetworkSpec`. The old pipeline becomes the detector CPT source; the new pipeline becomes the topology source.

## Testing strategy

- **Unit test:** `SC2HypothesisNetworkBuilder` refactored to produce `NetworkSpec` — existing test assertions still pass (same junction tree, same posteriors)
- **Unit test:** Hand-specified star expressed as `NetworkSpec` produces identical posteriors to the current hard-coded builder
- **Report test:** `StructureLearningExportTest` — the main pipeline. Extracts data, learns structures, compares, exports winners. Prints comparison table to stdout
- **Calibration gate:** `PatternClassificationCalibrationTest` must still pass with learned topology — no regression below current thresholds (≥70% rush/air-threat at 3min)

## Dependencies

| Dependency | Scope | What it provides |
|-----------|-------|-----------------|
| `casehub-neocortex-inference-bayes` | test | StructureLearner, ParameterLearner, NetworkSpec, StructuralConstraints |
| `casehub-neocortex-inference-bayes-llm` | test | NetworkElicitor (LLM structural priors) |
| Reconstituted replay data | data | `../quarkmind-classifier/data/reconstituted/` |

## Scope boundary

**In scope (this issue):**
- Structure learning across 7 observation variables + `enemy_strategy`, per-phase per-matchup
- Builder refactor: `buildIntelTree()` → `NetworkSpec`-based construction
- Four-variant comparison pipeline with cross-validation
- Export winning topologies as JSON resources
- Post-structure drools_evidence augmentation

**Out of scope (Phase 2 follow-up):**
- Including detector variables in structure learning (requires observation-detector data join)
- Selective parameter re-fitting (keep ONNX detector CPTs, only re-fit observation CPTs from learned structure)

## References

- `SC2HypothesisNetworkBuilder.buildIntelTree()` — current hand-specified star topology
- `PriorExportPipeline.learnObservationCpts()` — observation CPT extraction from replays
- `ContextResponsePriorExportTest` — existing per-phase per-matchup CPT export pipeline
- `CptLearningExportTest` — existing detector CPT export pipeline
- `DetectorOrchestrator.learnedNetworks` — runtime learned network consumption
- `HypothesisTrackerFusion.classify()` — drools_evidence dependency (5 usage sites)
- `LearnedCptLoader`, `LearnedCptResource` — runtime network loading infrastructure
- neocortex `StructureLearner` — PC algorithm and hill-climbing BIC (casehubio/neocortex#675)
- neocortex `NetworkElicitor` — LLM structural prior proposal (casehubio/neocortex#677)
- neocortex `StructuralConstraints.fromSpec()` — converts NetworkSpec edge status to constraints
- neocortex `ParameterLearner.fit()` — Bayesian CPT fitting with Dirichlet prior
