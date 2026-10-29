# Decisions — #470 Learned Scouting Network Structure

## D1: Time scope for structure learning

**Choice:** Per-phase (EARLY, MID, LATE) per-matchup — learn separate structures for each game phase, matching the existing observation CPT infrastructure
**Alternatives:**
- Single 5min checkpoint — simplest but ignores genuine distributional differences across game stages; misaligns with existing per-phase observation CPT files
- Multiple explicit checkpoints (3min, 5min) — splits the dataset without aligning to the platform's phase abstraction
- Merged across phases — more data but mixes signals from different game stages, confuses independence tests
**Rationale:** The existing observation CPTs are organized per-phase per-matchup (`observation-cpts-{PHASE}-vs_{race}.json`, 9 files total). The PC algorithm's conditional independence tests are distribution-sensitive — in EARLY, `first_building` and `gas_timing` carry the most signal; by MID, `tech_path` and `army_composition` dominate. Learning structure per-phase respects these distributional differences and aligns with the existing `GamePhase` enum (EARLY, MID, LATE) used throughout `HypothesisPriorLoader`, `DetectorOrchestrator`, and `CascadingPatternClassifier`.
**Trade-offs:** 3 phases × 3 matchups = 9 structure learning runs per algorithm variant. Each partition has fewer observations than the full dataset, but distributional purity matters more than volume for structure learning.
**Sources:** HypothesisPriorLoader.loadObservationCpts(GamePhase, Race), PriorExportPipeline.exportToJson(), ContextResponsePriorExportTest (CHECKPOINT_3MIN + CHECKPOINT_5MIN), observation-cpts-{PHASE}-vs_{race}.json resources
**Exploration:** quick
**Status:** revised (R1-01: corrected factual error — existing pipeline uses both 3min and 5min checkpoints and EARLY/MID/LATE phases, not 5min only)

## D2: Structure learning algorithms

**Choice:** Run four variants: unconstrained PC, unconstrained hill-climbing BIC, constrained hill-climbing with domain invariants, and LLM-seeded (D3). Compare all against hand-specified star baseline.
**Alternatives:**
- PC only — follows issue spec, principled but may miss denser structures
- Hill-climbing only — often outperforms PC on finite discrete data but deviates from issue
- Unconstrained only — misses the middle ground between no priors and full LLM priors
**Rationale:** API surface is identical for all variants. Constrained hill-climbing uses `StructuralConstraints` with domain invariants (`enemy_strategy` as root, no reverse edges from observations to `enemy_strategy`) — principled structural priors that don't require LLM infrastructure. This gives a four-point comparison: no priors → domain invariants → LLM priors → hand-specified, measuring the value of each level of domain knowledge injection.
**Trade-offs:** Four variants × 9 phase-matchup combinations = 36 structure learning runs. Comparison matrix is larger but each run is fast relative to replay parsing.
**Sources:** neocortex StructureLearner.pc(data, test, constraints), StructureLearner.hillClimbing(data, constraints, restarts, seed), StructuralConstraints
**Exploration:** quick
**Status:** revised (R1-10: added constrained hill-climbing variant)

## D3: LLM structural priors integration

**Choice:** Data-first with LLM-seeded comparison — run unconstrained PC/HC as baseline, then run again with LLM-proposed StructuralConstraints
**Alternatives:**
- LLM-seeded only — faster but no unconstrained baseline to measure LLM prior value
- Skip LLM priors — simpler, avoids LLM dependency, but misses domain knowledge injection
**Rationale:** Three-way comparison (hand-specified vs unconstrained-learned vs LLM-seeded-learned) cleanly measures the value of each approach. `NetworkElicitor` exists in `io.quarkmind.inference.bayes.llm.NetworkElicitor` (neocortex `inference-bayes-llm` module, landed via neocortex#677). Its `propose(domainDescription)` returns an `ElicitationResult` containing a `NetworkSpec`. `StructuralConstraints.fromSpec(NetworkSpec)` extracts LOCKED → required and REJECTED → forbidden edges. Requires adding `inference-bayes-llm` as a test-scoped dependency to quarkmind-sc2.
**Trade-offs:** Adds `inference-bayes-llm` test dependency and requires LLM call during the report test. Adds latency to the export pipeline.
**Sources:** neocortex inference-bayes-llm NetworkElicitor, StructuralConstraints.fromSpec()
**Exploration:** quick
**Status:** revised (R1-02: corrected package reference — NetworkElicitor is in inference-bayes-llm module, not inference-bayes; clarified dependency requirement)

## D4: Pipeline placement

**Choice:** @Tag("report") test in quarkmind-sc2
**Alternatives:**
- Standalone CLI module — cleaner separation but over-engineered for a one-shot export
- Runtime CDI bean — unnecessary unless we want dynamic re-learning
**Rationale:** Consistent with existing export pipeline (ContextResponsePriorExportTest, CptLearningExportTest). Runs on demand, writes to src/main/resources/. StructureLearner remains test-scoped. Note: runtime integration infrastructure already exists — `DetectorOrchestrator.learnedNetworks` loads `BayesNetwork` objects keyed by `{phase}_{vs_race}` and uses them in `initializeNetwork(detectorNames, phase, enemyRace)`. The runtime path requires no new infrastructure — only a decision to promote learned structures from test-export to runtime-loaded resources.
**Trade-offs:** StructureLearner becomes a test-scoped dependency only — runtime learning would use the existing `LearnedCptLoader` → `DetectorOrchestrator.learnedNetworks` path with no StructureLearner dependency
**Sources:** ContextResponsePriorExportTest, CptLearningExportTest, DetectorOrchestrator.initializeNetwork(), LearnedCptLoader
**Exploration:** quick
**Status:** revised (R1-11: acknowledged existing runtime learned-network infrastructure)

## D5: Builder API — NetworkSpec-based construction

**Choice:** Express both hand-specified star and learned structure as NetworkSpec. Both paths use `NetworkSpec.toBuilder().buildJunctionTree()` for construction. The hand-specified star is no longer special — just one NetworkSpec among several. Swapping topology is a resource file choice, not a code change.
**Alternatives:**
- Toggle flag between two hard-coded paths — duplicate construction code, any accuracy difference could be an artifact of different code paths
- Keep buildIntelTree as primary path, add separate learned path — two divergent code paths for the same purpose
**Rationale:** There are two independent network builders today: `SC2HypothesisNetworkBuilder.buildIntelTree()` (builds JunctionTree for HypothesisTracker with observations + detectors + drools_evidence) and `SC2NetworkBuilder.build()` (builds BayesNetwork for DetectorOrchestrator with detectors only). Both should be unified through NetworkSpec. For learned structures: `StructureResult.networkSpec()` → serialize as JSON. For the hand-specified star: refactor `buildIntelTree` to PRODUCE a NetworkSpec (encoding its clamping, normalization, and drools_evidence as spec-level concerns), then use `NetworkSpec.toBuilder().buildJunctionTree()`. The current per-variable clamping (`Math.max(0.01, Math.min(0.99, val))`) moves to spec creation time rather than consumption time. `DetectorOrchestrator.learnedNetworks` already consumes `BayesNetwork` objects via `NetworkSpec.toBayesNetwork()` through `LearnedCptLoader`.
**Trade-offs:** Refactoring buildIntelTree to produce a NetworkSpec requires expressing drools_evidence and all clamping in NetworkSpec format. Both SC2HypothesisNetworkBuilder and SC2NetworkBuilder must be updated.
**Sources:** SC2HypothesisNetworkBuilder.buildIntelTree(), SC2NetworkBuilder.build(), DetectorOrchestrator.learnedNetworks, LearnedCptLoader, neocortex NetworkSpec.toBuilder(), StructureResult.networkSpec()
**Exploration:** quick
**Status:** revised (R1-03: corrected API description — buildIntelTree does not accept NetworkSpec; R1-04: expanded scope to cover both SC2HypothesisNetworkBuilder and SC2NetworkBuilder/DetectorOrchestrator)

## D6: Matchup scope

**Choice:** Per-matchup — learn separate topologies for vs_protoss, vs_terran, vs_zerg
**Alternatives:**
- Unified — one topology for all matchups, simpler but misses matchup-specific independence structure
- Both — compare unified vs per-matchup, most thorough but adds another comparison dimension
**Rationale:** Different matchups have different informative observations (first_building matters more vs Protoss than vs Zerg). Consistent with per-matchup CPT exports. With ~50K replays per matchup and 7 observation variables (3-6 states each) + 1 strategy variable (5-9 archetypes), effective sample size per conditioning configuration is: for a 2-parent conditioning set (e.g., 9 archetypes × 4 gas_timing states = 36 cells), ~1400 observations per cell — adequate for chi-squared and Fisher exact tests. For deeper conditioning sets (3+ parents), some cells may be sparse, triggering Fisher exact fallbacks in `IndependenceTest.adaptive()`. This is acceptable — the PC algorithm handles sparse cells gracefully by failing to reject independence, which produces sparser (more conservative) structures.
**Trade-offs:** Three separate learning runs per algorithm variant per phase. If one matchup has sparse data, learned structure may be less reliable — but this is detectable from the StructureDiagnostics returned by StructureResult.
**Sources:** ContextResponsePriorExportTest (per-matchup CPT export), PriorExportPipeline, IndependenceTest.adaptive()
**Exploration:** quick
**Status:** revised (R1-07: added effective sample size analysis for conditioning configurations)

## D7: Variable scope — staged approach

**Choice:** Phase 1: Observations only (7 observation variables + enemy_strategy). Phase 2 (follow-up): Include detectors, requiring data join infrastructure.
**Alternatives:**
- Observations + detectors from the start — more complete but requires solving the data join problem upfront
- Observations only, permanently — simpler but misses potential cross-edges between observations and detectors
**Rationale:** The observation tabular data already exists in the prior export pipeline (`PriorExportPipeline.learnObservationCpts()`). No new data infrastructure is needed for Phase 1. Detector data comes from a separate pipeline (`CptLearningExportTest`, sourced from ONNX classifier output at `quarkmind-classifier/output/cpt_observations/`). For Phase 2, tabular rows must contain both observation states AND detector outputs for the same replay at the same checkpoint — this join doesn't exist today and requires either running ONNX classifiers during replay parsing or joining offline by replay hash and checkpoint. Phase 2 must also resolve the detector state cardinality discrepancy: `CptLearningExportTest` uses 3 states (HIGH/MEDIUM/LOW) while `SC2HypothesisNetworkBuilder.buildIntelTree()` uses 2 states (HIGH/LOW). When detectors are eventually included, structure should be learned from the full variable set but detector CPTs should be replaced with ONNX-learned ones after structure learning, since those use raw neural network confidence scores and are higher quality than re-fitted tabular CPTs.
**Trade-offs:** Phase 1 cannot discover cross-edges between observations and detectors. Phase 2 requires non-trivial data pipeline work. Re-fitting detector CPTs from tabular data loses information from the ONNX classifiers — the hybrid approach (learn structure, keep ONNX CPTs) requires selective parameter re-fitting that `ParameterLearner.fit()` doesn't currently support (it re-fits everything).
**Depends on:** D1 (phase determines data partitioning), D2 (algorithms), D6 (per-matchup scope)
**Sources:** PriorExportPipeline.learnObservationCpts(), CptLearningExportTest, SC2HypothesisNetworkBuilder.buildIntelTree(), DetectorOrchestrator
**Exploration:** quick
**Status:** revised (R1-05: staged into Phase 1/Phase 2; R1-06: detector CPT quality concern addressed; R1-12: dependencies expanded)

## D8: Comparison methodology

**Choice:** Cross-validated log-likelihood as primary metric. Brier score and per-archetype classification accuracy as secondary metrics. K-fold cross-validation at the replay level.
**Alternatives:**
- BIC/AIC only — model selection criteria that don't validate on held-out data; useful for algorithm-internal optimization but not for cross-topology comparison
- Accuracy only — doesn't measure calibration quality (a network can have high accuracy but poorly calibrated posteriors)
- Calibration accuracy only — ambiguous term; log-likelihood is the standard information-theoretic measure of probabilistic model quality
**Rationale:** Log-likelihood on held-out data directly measures how well the network predicts unseen observations. Brier score measures calibration — whether predicted probabilities match empirical frequencies. Per-archetype accuracy catches cases where a topology is good on average but poor for specific strategies. K-fold at the replay level (all observations from one replay in the same fold) prevents data leakage from correlated observations within the same game. Statistical significance via paired bootstrap test on per-replay log-likelihood scores. Practical significance threshold: a learned topology must improve log-likelihood by at least 5% over the hand-specified star to justify switching (buffer against overfitting to the training distribution).
**Trade-offs:** Log-likelihood rewards density (more edges = higher likelihood on training data). The 5% improvement threshold and cross-validation guard against this, but BIC's explicit complexity penalty provides an additional safeguard — BIC scores from D2's hill-climbing variant serve as a cross-check.
**Sources:** neocortex ParameterLearner.fit(), StructureDiagnostics
**Exploration:** quick
**Status:** captured (surfaced by R1-08)

## D9: Pipeline lifecycle — coexistence with CptLearningExportTest

**Choice:** Coexist. The new structure learning pipeline runs alongside CptLearningExportTest. Both export to `LearnedCptResource` format wrapping `NetworkSpec`.
**Alternatives:**
- Replace CptLearningExportTest — premature until the learned structure is validated
- Merge into one pipeline — couples structure learning and CPT export, making each harder to iterate independently
**Rationale:** `CptLearningExportTest` learns detector CPTs from ONNX classifier outputs using a fixed star topology and exports `NetworkSpec` to `src/main/resources/classifier/`. The new pipeline learns network STRUCTURE from observation data and exports to the same format. If the learned structure outperforms the star: the detector CPTs from `CptLearningExportTest` are preserved and plugged into the learned structure's `NetworkSpec`. The old pipeline becomes the source of detector CPTs; the new pipeline becomes the source of topology. If the hand-specified star wins: the structure learning pipeline becomes a validation tool confirming the hand-specified topology is near-optimal.
**Trade-offs:** Two export pipelines produce overlapping artifacts temporarily. Output format consistency (`LearnedCptResource` wrapping `NetworkSpec`) ensures the `LearnedCptLoader` can consume either pipeline's output without code changes.
**Sources:** CptLearningExportTest, LearnedCptResource, LearnedCptLoader, DetectorOrchestrator.learnedNetworks
**Exploration:** quick
**Status:** captured (surfaced by R1-09)
