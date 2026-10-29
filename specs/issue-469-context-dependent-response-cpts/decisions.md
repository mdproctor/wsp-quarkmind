## D1: Game state extraction approach

**Choice:** Direct PlayerStats parsing from reconstituted replay JSON
**Alternatives:**
- EmulatedGame replay — accurate but prohibitively slow for 149K replays
- Python extraction via extract_state.py — mature but cross-language boundary with Java consumers
**Rationale:** Fast (JSON parsing only), reuses existing replay loading infrastructure, works on all 149K replays. PlayerStats fires every 160 loops with collection rates; UnitBorn/UnitDied give army/building counts. Checkpoint may be ±80 loops off — negligible for discretization at 0.8/1.2 thresholds.
**Trade-offs:** Slightly less precise than EmulatedGame (nearest PlayerStats vs exact loop), but the discretization thresholds are wide enough that this doesn't matter.
**Sources:** PriorExportPipeline.java, TrackerEventFeatureExtractor.java:97-119, DroolsScoutingTask.java:680-707
**Exploration:** quick
**Status:** captured

## D2: CPT construction and integration approach

**Choice:** Extend PriorExportPipeline with context-aware methods
**Alternatives:**
- New ContextCptPipeline class — cleaner separation but more scaffolding, duplicates replay loading
- Python pipeline in quarkmind-classifier — better data tooling but cross-language boundary
**Rationale:** Minimal new classes, follows existing pipeline pattern. Unconditional learnResponsePriors() remains untouched and serves as the marginal prior for hierarchical smoothing. PriorExportPipeline is only 157 lines — room to grow.
**Trade-offs:** Pipeline class grows, but the context extension is a natural progression.
**Sources:** PriorExportPipeline.java:37-60, SC2HypothesisNetworkBuilder.java:194-235
**Exploration:** quick
**Status:** captured

## D3: Data sparsity handling

**Choice:** Hierarchical smoothing — pool toward marginal (context-free) prior with strength proportional to cell count
**Alternatives:**
- Trust Laplace smoothing — simpler but sparse cells are nearly uniform, no regression toward known-good marginal
- Explicit fallback threshold — sharper control but adds a tuning parameter
**Rationale:** P_smoothed = (n / (n + α)) * P_cell + (α / (n + α)) * P_marginal. Data-rich cells shift proportionally; sparse cells regress toward Phase 4 unconditional prior. Best statistical properties — empirical Bayes shrinkage.
**Trade-offs:** Adds a concentration parameter α to tune. More complex than plain Laplace but the implementation is a weighted average — straightforward.
**Sources:** Phase 4 response priors serve as P_marginal (response-priors-vs_*.json)
**Exploration:** quick
**Status:** captured

## D4: Extraction checkpoints

**Choice:** Both 3-min and 5-min checkpoints — each replay contributes two snapshots
**Alternatives:**
- Single checkpoint at 5-min — cleaner signal but half the data
- Phase-matched checkpoint — 3-min for rushes, 5-min for macro archetypes — most principled but adds per-archetype logic
**Rationale:** Doubles the training data. Captures game state evolution. A player "behind" at 3-min but "ahead" at 5-min contributes to both context cells. Matches existing EARLY/MID phase checkpoint pattern.
**Trade-offs:** A single replay can contribute conflicting context states at different checkpoints, but this accurately reflects how game state evolves.
**Sources:** ScoutingObservationExtractor.java threshold constants, DroolsCoarseLabelExportTest.java:166-189
**Exploration:** quick
**Status:** captured
