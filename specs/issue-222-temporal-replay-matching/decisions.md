# Decisions — Issue #222: Temporal Replay Matching

## D1: Design Focus — End-to-End First

**Choice:** End-to-end pipeline with minimal timeline (economy + phase sequence), then iterate
**Alternatives:**
- Representation depth first — design full temporal feature set before wiring retrieval. More upfront design but delays validation.
**Rationale:** The engine already has DTW, EditDistance, TimeSeries/DiscreteSequence field types fully built. The work is wiring, not algorithm design. End-to-end validates the concept fast.
**Trade-offs:** Minimal feature set may miss important temporal signals; iterate after validation.
**Sources:** neocortex memory-api (`DtwSimilarity`, `EditDistanceSimilarity`, `FeatureField.TimeSeries`), Tier 2 decisions D5
**Exploration:** quick
**Status:** captured

## D2: Timeline Data — Economy + Army Curves

**Choice:** ~6 numeric fields per time step: our_workers, our_minerals, our_army_supply, enemy_workers, enemy_minerals, enemy_army_supply. Plus phase_sequence upgraded to DiscreteSequence.
**Alternatives:**
- Dominance trajectory (3 fields: dominance_army/economy/overall) — more abstract, fewer dimensions, but loses raw economic signal
- Both economy + dominance (~9 fields) — richer but more DTW dimensions for marginal gain in the first pass
**Rationale:** Economy and army curves are the most directly meaningful temporal signals for SC2. Workers, minerals, and army supply capture the macro story of a game. Dominance scores are derived from these and lose granularity.
**Trade-offs:** Does not include dominance scores or tech trajectory. Can add in a later iteration.
**Sources:** `PlayerEconomyStats` fields, `GameState` record, issue #222 acceptance criteria (DTW on economy/army trajectories)
**Exploration:** quick
**Status:** captured

## D3: Query Timing — Periodic (Every ~2 Minutes)

**Choice:** Query CBR every ~2 minutes with the timeline so far
**Alternatives:**
- Phase transitions only — fewer queries but misses slow-evolving patterns between transitions
- Piggyback on strategy router — minimal wiring but couples temporal prediction to scouting events
**Rationale:** DTW handles variable-length sequences natively. Periodic queries give progressive refinement as the game unfolds. 2-minute interval keeps retrieval load predictable (~5-10 queries per game).
**Trade-offs:** Some redundant queries when the game state hasn't meaningfully changed. Acceptable given DTW is cheap at 20-40 steps.
**Sources:** `AgentOrchestrator.gameTick()` runs every 500ms, `GameTickExecutor.execute()` has GameState access
**Exploration:** quick
**Status:** captured

## D4: Prediction Surface — CaseFile Keys with Structured Prediction

**Choice:** Write structured `TemporalPrediction` to CaseFile keys. Other plugins read it as advisory context.
**Alternatives:**
- Advisory channel message — integrates with coaching pipeline but heavier integration for a first pass
- Return with ScoredCbrCase — simpler but pushes interpretation to consumers
**Rationale:** CaseFile keys are the established inter-plugin communication mechanism. Strategy, tactics, and coaching plugins already read CaseFile keys. Clean separation: temporal CBR produces predictions, consumers decide what to do with them.
**Trade-offs:** No advisory pipeline integration in the first pass. Coaching plugin would need to read CaseFile keys rather than receiving advisory messages.
**Sources:** `QuarkMindCaseFile` key namespace (`agent.*` for plugin-produced state), `SC2StrategyRouterTask` reads/writes pattern
**Exploration:** quick
**Status:** captured

## D5: Phase Sequence Upgrade — In-Place to DiscreteSequence

**Choice:** Upgrade `phase_sequence` from `CategoricalList` to `DiscreteSequence` with `EditDistanceSpec`
**Alternatives:**
- Add separate `phase_timeline` field alongside existing `phase_sequence` — backwards compatible but redundant data
**Rationale:** One field, better similarity. Edit distance is strictly superior to Jaccard for ordered sequences. Existing Tier 2 queries also improve. Neocortex handles the schema migration transparently.
**Trade-offs:** Existing stored cases will be re-scored with edit distance instead of Jaccard. Similarity scores for phase_sequence will change, potentially affecting strategy router results. Acceptable because the router weights phase_sequence lightly.
**Sources:** `SC2CbrSchemaRegistrar` current registration, `FeatureField.discreteSequence()` API
**Exploration:** quick
**Status:** captured

## D6: Timeline Sampling Architecture — New TimelineSampler Bean

**Choice:** Standalone `@ApplicationScoped` CDI bean injected into `GameTickExecutor`. Samples GameState every ~30s, accumulates in a list. `SC2CbrRetentionObserver` injects it and calls `getTimeline()` at game close.
**Alternatives:**
- Extend `SC2CbrRetentionObserver` directly — mixes sampling concern with retention. Observer doesn't currently receive GameState per-tick.
**Rationale:** Clean separation: sampler owns periodic accumulation, observer owns game-close retention. Follows the established pattern of `GameTickExecutor` calling into specialised components (e.g., `SummarisationLifecycle.tick()`). Sampler is independently testable.
**Trade-offs:** One more CDI bean. Trivial overhead.
**Sources:** `GameTickExecutor.execute()` line 59 (`engine.observe()`), `SummarisationLifecycle.tick()` pattern
**Exploration:** quick
**Status:** captured

## D7: Live Temporal Retrieval — New TemporalCbrTask

**Choice:** New `@ApplicationScoped @CaseType("starcraft-game")` TaskDefinition. Separate from `SC2StrategyRouterTask`. Activates when timeline has ≥4 samples (~2 min). Produces temporal predictions to CaseFile.
**Alternatives:**
- Extend `SC2StrategyRouterTask` — conflates initial routing with ongoing prediction. Router's activation gates and pivot logic would need reworking.
**Rationale:** Temporal prediction is advisory (informing strategy/tactics), not routing (selecting a strategy). Different activation timing (periodic vs. scouting events), different query weights, different outputs. Clean TaskDefinition separation.
**Trade-offs:** New plugin in the orchestrator chain. Ordering relative to other tasks needs consideration.
**Sources:** `SC2StrategyRouterTask` structure, `TaskDefinition` contract, plugin seam pattern
**Exploration:** quick
**Status:** captured

## D8: DTW Warping Constraint — Sakoe-Chiba Band, Window=3

**Choice:** `SakoeChibaBand(3)` — allows ±90 seconds of temporal shift (3 time steps at 30s intervals)
**Alternatives:**
- Unconstrained — maximum flexibility but can produce pathological alignments. O(n²) instead of O(n*w).
- Itakura parallelogram (slope=2) — better theoretical properties for globally similar pace but harder to reason about.
**Rationale:** SC2 games have similar macro structure but events shift by 1-3 minutes. ±90 seconds covers typical timing variation (early push at minute 3 vs. minute 4.5). Also faster: O(n*w) with w=3 vs. O(n²).
**Trade-offs:** Cannot match games where equivalent events happen >90 seconds apart. Acceptable for 10-20 minute games. Can widen later if matching quality is poor.
**Sources:** `DtwSimilarity` API, `WarpingConstraint.SakoeChibaBand`, `LbKeogh` lower-bound pruning
**Exploration:** quick
**Status:** captured

## D9: Edit Distance Substitution Costs — Semantic Similarity Matrix

**Choice:** Define pairwise substitution similarities between phases (e.g., EARLY_MACRO↔LATE_MACRO=0.7, EARLY_MACRO↔AGGRESSIVE_PUSH=0.2)
**Alternatives:**
- Uniform costs — simpler, no hand-tuning. Phase ordering still matters.
**Rationale:** Phase labels encode semantic meaning. Substituting within a posture family (early→late macro) is cheaper than substituting across families (macro→aggression). Makes edit distance phase-aware.
**Trade-offs:** Requires manual tuning of the substitution matrix. Need to cover all TacticalPosture values. Can start with a reasonable matrix and calibrate.
**Sources:** `EditDistanceSpec(substitutionSimilarities, insertCost, deleteCost)` API, `TacticalPosture` enum values
**Exploration:** quick
**Status:** captured
**Depends on:** D5 (phase sequence as DiscreteSequence)

## D10: Query Weight Distribution — Temporal-Heavy for Temporal Queries

**Choice:** TemporalCbrTask query: timeline=0.50 (DTW), phase_sequence=0.30 (EditDistance), enemy_archetype=0.10, matchup=0.10. SC2StrategyRouterTask query unchanged.
**Alternatives:**
- Balanced weighting (timeline=0.30, phase_sequence=0.20, archetype=0.20, rest distributed) — finds games similar across all dimensions but less temporal focus.
**Rationale:** Temporal queries should find temporally similar games. The strategy router already handles archetype-focused retrieval. Two queries with different weight profiles serve different purposes.
**Trade-offs:** Temporal query may match games with different archetypes if their economy curves are similar. Archetype at 0.10 is still a weak filter. Acceptable for advisory/predictive use.
**Sources:** Current strategy router weights (archetype=0.5, race=0.15, matchup=0.15, confidence=0.2), neocortex weight normalisation
**Exploration:** quick
**Status:** captured
**Depends on:** D7 (separate TemporalCbrTask)

## D11: Independent Sampler (Not TemporalWindowAccumulator Reuse)

**Choice:** New lightweight `TimelineSampler` that captures economy/army values from GameState every 30 seconds. Decoupled from ONNX pipeline.
**Alternatives:**
- Reuse `TemporalWindowAccumulator` — extract 6 CBR fields from existing 269-feature windows. Avoids duplicate sampling but couples CBR to ONNX pipeline.
**Rationale:** The accumulator's 134-feature tensor is overbuilt for CBR's 6-field timeline. Direct GameState sampling is simpler, more maintainable, and won't break if ONNX features change. Different consumers, different concerns.
**Trade-offs:** Two components sampling GameState at 30s intervals. Negligible overhead — sampling is a record copy, not computation.
**Sources:** `TemporalWindowAccumulator` (134 features/player, 10 windows), `StrategyFeatureExtractor`
**Exploration:** quick
**Status:** captured
