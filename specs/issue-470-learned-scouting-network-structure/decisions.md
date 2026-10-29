# Decisions — #470 Learned Scouting Network Structure

## D1: Time checkpoint for structure learning

**Choice:** Single checkpoint at 5min
**Alternatives:**
- Multiple checkpoints (3min, 5min) — reveals temporal topology differences but splits the dataset
- Merged checkpoints — more data but mixes early/late signals, confuses independence tests
**Rationale:** 5min has the richest scouting data, matches existing CPT export pipeline, simpler. Temporal variation can be explored in a follow-up.
**Trade-offs:** Misses potential early-game topology differences (e.g. gas_timing may be more informative at 3min than 5min)
**Sources:** PriorExportPipeline.learnObservationCpts(), ContextResponsePriorExportTest
**Exploration:** quick
**Status:** captured

## D2: Structure learning algorithms

**Choice:** Run both PC and hill-climbing BIC, compare results
**Alternatives:**
- PC only — follows issue spec, principled but may miss denser structures
- Hill-climbing only — often outperforms PC on finite discrete data but deviates from issue
**Rationale:** API surface is identical for both. Modest extra work to run both and let data decide which structure produces better calibration accuracy.
**Trade-offs:** Slightly more complex comparison matrix (hand-specified × PC × HC × LLM-seeded variants)
**Sources:** neocortex StructureLearner.pc(), StructureLearner.hillClimbing()
**Exploration:** quick
**Status:** captured

## D3: LLM structural priors integration

**Choice:** Data-first with LLM-seeded comparison — run unconstrained PC/HC as baseline, then run again with LLM-proposed StructuralConstraints
**Alternatives:**
- LLM-seeded only — faster but no unconstrained baseline to measure LLM prior value
- Skip LLM priors — simpler, avoids LLM dependency, but misses domain knowledge injection
**Rationale:** Three-way comparison (hand-specified vs unconstrained-learned vs LLM-seeded-learned) cleanly measures the value of each approach.
**Trade-offs:** Requires LLM call during report test (NetworkElicitor.propose()), adds latency to the export pipeline
**Sources:** neocortex NetworkElicitor, StructuralConstraints.fromSpec()
**Exploration:** quick
**Status:** captured

## D4: Pipeline placement

**Choice:** @Tag("report") test in quarkmind-sc2
**Alternatives:**
- Standalone CLI module — cleaner separation but over-engineered for a one-shot export
- Runtime CDI bean — unnecessary unless we want dynamic re-learning
**Rationale:** Consistent with existing export pipeline (ContextResponsePriorExportTest, StrategyTransitionExportTest). Runs on demand, writes to src/main/resources/. No runtime StructureLearner dependency.
**Trade-offs:** StructureLearner becomes a test-scoped dependency only — if we ever want runtime learning, must revisit
**Sources:** ContextResponsePriorExportTest, StrategyTransitionExportTest
**Exploration:** quick
**Status:** captured

## D5: Builder API — NetworkSpec-based construction

**Choice:** Express both hand-specified star and learned structure as NetworkSpec. buildIntelTree(NetworkSpec) constructs from any spec. Hand-specified star becomes the default template; learned structure is an alternative loaded from exported JSON.
**Alternatives:**
- Toggle flag between two hard-coded paths — duplicate construction code, any accuracy difference could be an artifact of different code paths
- buildLearnedIntelTree(NetworkSpec) as a separate method — over-engineered, same builder logic applies to both topologies
**Rationale:** Single code path for both topologies. Swapping is a resource file choice, not a code change. The hand-specified star is no longer special — just one NetworkSpec among several.
**Trade-offs:** Must express the current hand-coded star topology as a NetworkSpec (minor refactor of buildIntelTree)
**Sources:** SC2HypothesisNetworkBuilder.buildIntelTree(), neocortex NetworkSpec
**Exploration:** quick
**Status:** captured
