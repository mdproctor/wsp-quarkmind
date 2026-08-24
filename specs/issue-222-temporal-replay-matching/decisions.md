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

## D2: Timeline Data — Own-Side Economy + Army Curves

**Choice:** 3 numeric fields per time step: our_workers, our_minerals, our_army_supply. Plus phase_sequence upgraded to DiscreteSequence. Enemy fields excluded.
**Alternatives:**
- 6 fields including enemy data (enemy_workers, enemy_minerals, enemy_army_supply) — richer but enemy economy data is not available through the SC2 API during live games. `ObservationTranslator.translate()` sets both `playerEconomy` and `enemyEconomy` to `PlayerEconomyStats.EMPTY`. Even scouted enemy unit counts are partial. DTW between live partial-enemy data and stored full-replay data would systematically bias toward games where the opponent appeared weakest.
- Dominance trajectory (3 fields: dominance_army/economy/overall) — more abstract, fewer dimensions, but loses raw economic signal
- Both economy + dominance (~9 fields) — richer but more DTW dimensions for marginal gain in the first pass
**Rationale:** Own-side economy and army curves are directly observable from `GameState` and capture the macro story of a game. Workers counted from `myUnits`, minerals from `PlayerCommon.getMinerals()`, army supply from unit supply sums. These fields are fully reliable in both live and stored contexts — no fog-of-war asymmetry.
**Trade-offs:** Loses cross-player temporal comparison. Can add enemy fields in a future iteration that introduces a scouting-based estimation model or replay-only retrieval mode.
**Sources:** `ObservationTranslator.translate()` line 160 — `PlayerEconomyStats.EMPTY` for both sides, `GameState` record, `PlayerCommon` fields
**Exploration:** quick
**Status:** revised (R1-02: removed enemy fields — SC2 API fog-of-war makes them unreliable for live queries)

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
**Trade-offs:** Existing stored cases will be re-scored with edit distance instead of Jaccard. The strategy router does NOT query `phase_sequence` — its weights are `enemy_archetype=0.5, enemy_race=0.15, matchup=0.15, assessment_confidence=0.2`. The upgrade only affects retrieval queries that explicitly weight `phase_sequence`, which today means none. The change is purely forward-looking for D10's temporal query.
**Sources:** `SC2CbrSchemaRegistrar` current registration, `FeatureField.discreteSequence()` API, `SC2StrategyRouterTask.execute()` query weights
**Exploration:** quick
**Status:** revised (R1-09: corrected factual error — strategy router does not query phase_sequence)

## D6: Timeline Sampling Architecture — New TimelineSampler Bean

**Choice:** Standalone `@ApplicationScoped` CDI bean injected into `GameTickExecutor`. Samples GameState every ~30s, accumulates in a list. `SC2CbrRetentionObserver` injects it and calls `getTimeline()` at game close. Observes `GameStarted` directly (`@Observes GameStarted`) to clear accumulated samples between games. Receives full `GameState` via `tick(GameState)`, not just the frame number.
**Alternatives:**
- Extend `SC2CbrRetentionObserver` directly — mixes sampling concern with retention. Observer doesn't currently receive GameState per-tick.
- Reset via `MomentBroker.onGameStarted` (like SummarisationLifecycle) — couples timeline sampling to the summarisation pipeline. The sampler is a CBR concern, not a summarisation concern.
- Follow `SummarisationLifecycle.tick(long gameFrame)` pattern — insufficient, because the sampler needs full `GameState` to extract economy/army values. `GameTickExecutor.execute()` has `gameState` as a local variable (from `engine.observe()`), so passing it directly is straightforward.
**Rationale:** Clean separation: sampler owns periodic accumulation, observer owns game-close retention. Direct `@Observes GameStarted` for reset is simpler than routing through MomentBroker — the sampler has no relationship to the moment/phase/arc summarisation pipeline. The `tick(GameState)` signature is the natural integration point since GameTickExecutor already holds the GameState.
**Trade-offs:** One more CDI bean. Trivial overhead. Different tick signature from SummarisationLifecycle (GameState vs long), but this reflects a genuine difference in data requirements.
**Sources:** `GameTickExecutor.execute()` line 58 (`engine.observe()`), `SummarisationLifecycle.tick()` pattern, `MomentBroker.onGameStarted()` reset pattern, `SC2CbrRetentionObserver.onGameStarted()` existing reset
**Exploration:** quick
**Status:** revised (R1-11, R1-12: added lifecycle reset via @Observes GameStarted and tick(GameState) parameter)

## D7: Live Temporal Retrieval — New TemporalCbrTask

**Choice:** New `@ApplicationScoped @CaseType("starcraft-game")` TaskDefinition. Separate from `SC2StrategyRouterTask`. Activates when timeline has ≥4 samples (~2 min). Produces temporal predictions to CaseFile.
**Alternatives:**
- Extend `SC2StrategyRouterTask` — conflates initial routing with ongoing prediction. Router's activation gates and pivot logic would need reworking.
**Rationale:** Temporal prediction is advisory (informing strategy/tactics), not routing (selecting a strategy). Different activation timing (periodic vs. scouting events), different query weights, different outputs. Clean TaskDefinition separation.
**Trade-offs:** New plugin in the orchestrator chain. Ordering relative to other tasks needs consideration (see D13).
**Sources:** `SC2StrategyRouterTask` structure, `TaskDefinition` contract, plugin seam pattern
**Exploration:** quick
**Status:** captured
**Depends on:** D12 (scope decision), D13 (task ordering)

## D8: DTW Warping Constraint — Sakoe-Chiba Band, Window=3

**Choice:** `SakoeChibaBand(3)` — allows ±90 seconds of temporal shift (3 time steps at 30s intervals)
**Alternatives:**
- Unconstrained — maximum flexibility but can produce pathological alignments. O(n²) instead of O(n*w).
- Itakura parallelogram (slope=2) — better theoretical properties for globally similar pace but harder to reason about.
**Rationale:** SC2 games have similar macro structure but events shift by 1-3 minutes. ±90 seconds covers typical timing variation (early push at minute 3 vs. minute 4.5). For same-length comparisons (post-game retrieval), provides O(n*w) with w=3 vs O(n²). For unequal-length sequences (live partial-timeline queries), `DtwSimilarity.computeJStart()` widens the effective window to `max(3, |n-m|)` — so with a 4-step query against a 30-step stored timeline, the window becomes 26 (effectively unconstrained). The warping constraint's performance benefit is specific to same-length comparisons; live partial-timeline performance relies on `LbKeogh` lower-bound pruning (see D14).
**Trade-offs:** Cannot match games where equivalent events happen >90 seconds apart in same-length comparisons. Acceptable for 10-20 minute games. Can widen later if matching quality is poor.
**Sources:** `DtwSimilarity` API, `WarpingConstraint.SakoeChibaBand`, `DtwSimilarity.computeJStart()` window widening logic, `LbKeogh` lower-bound pruning
**Exploration:** quick
**Status:** revised (R1-06: noted window widening for unequal-length sequences)

## D9: Edit Distance Substitution Costs — Semantic Similarity Matrix

**Choice:** Define pairwise substitution similarities between phases (e.g., EARLY_MACRO↔MID_SKIRMISH=0.3, DEFENSIVE_HOLD↔EARLY_AGGRESSION=0.2). Extract posture values from the free-form `String posture` in `TacticalPosture` into a proper enum (`TacticalPostureType`) to enable compile-time validation of the substitution matrix.
**Alternatives:**
- Uniform costs — simpler, no hand-tuning. Phase ordering still matters but loses semantic distance.
- Keep posture as free-form String with documented protocol — fragile because `GamePhaseSummariser` can add new values without updating the matrix, causing them to silently default to 0.0 similarity (maximally dissimilar) in `EditDistanceSimilarity.lookupSimilarity()`.
**Rationale:** Phase labels encode semantic meaning. Substituting within a posture family (EARLY_MACRO→TRANSITIONING) is cheaper than across families (EARLY_MACRO→DEFENSIVE_HOLD). `TacticalPosture` is a record with `String posture` — the values (`DEFENSIVE_HOLD`, `EARLY_AGGRESSION`, `MID_SKIRMISH`, `TRANSITIONING`, `EARLY_MACRO`) are produced by `GamePhaseSummariser`. An enum constrains the value set at compile time and makes the substitution matrix exhaustively checkable. Without the enum, a renamed or added phase silently falls back to 0.0 similarity — maximally dissimilar to everything.
**Trade-offs:** Enum extraction requires updating `TacticalPosture(String posture, ...)` to `TacticalPosture(TacticalPostureType posture, ...)` and all call sites in `GamePhaseSummariser`. Small, mechanical migration. Worth doing for correctness.
**Sources:** `TacticalPosture` record, `GamePhaseSummariser.doSummarise()` phase assignment, `EditDistanceSimilarity.lookupSimilarity()` default-to-0.0 behaviour, `EditDistanceSpec(substitutionSimilarities, insertCost, deleteCost)` API
**Exploration:** quick
**Status:** revised (R1-07: TacticalPosture is a record not an enum; propose TacticalPostureType enum extraction)
**Depends on:** D5 (phase sequence as DiscreteSequence)

## D10: Query Weight Distribution — Temporal-Heavy with Matchup Hard Filter

**Choice:** TemporalCbrTask query: `timeline=0.55` (DTW), `phase_sequence=0.30` (EditDistance), `enemy_archetype=0.15`. Matchup applied as a hard `CbrFilter` (exact match) via `CbrQuery.withFilter("matchup", CbrFilter.contains(matchup))`, not as a weighted similarity feature. SC2StrategyRouterTask query unchanged.
**Alternatives:**
- Weighted matchup at 0.10 — allows cross-matchup matches when economy curves are similar. But SC2 temporal patterns are fundamentally matchup-dependent: PvZ timing windows, economic curves, and threat profiles bear no resemblance to PvT patterns. Cross-matchup temporal matching produces misleading similarity.
- Balanced weighting (timeline=0.30, phase_sequence=0.20, archetype=0.20, rest distributed) — less temporal focus.
**Rationale:** Matchup fundamentally scopes the search space in SC2. A PvZ economy curve has no meaningful temporal similarity to a PvT curve — the timing windows, tech transitions, and army compositions are entirely different games. Hard filtering eliminates cross-matchup contamination before similarity scoring begins. The freed weight goes to temporal features (timeline +0.05) and enemy_archetype (+0.05) for better discrimination within the correct matchup. `enemy_race` is redundant when matchup is a hard filter since matchup encodes the opponent race (PvZ → Zerg).
**Trade-offs:** Hard filter reduces the candidate pool, which may affect recall if matchup is incorrectly identified. Acceptable because matchup is always known (our race is fixed, opponent race is observable from the first scouted unit).
**Sources:** `CbrQuery.withFilter(String, CbrFilter)` API, `CbrFilter.contains(String)` for exact match, current strategy router weights
**Exploration:** quick
**Status:** revised (R1-05: matchup as hard CbrFilter instead of weighted feature)
**Depends on:** D7 (separate TemporalCbrTask)

## D11: Independent Sampler (Not TemporalWindowAccumulator Reuse)

**Choice:** New lightweight `TimelineSampler` that captures economy/army values from GameState every 30 seconds. Decoupled from ONNX pipeline.
**Alternatives:**
- Reuse `TemporalWindowAccumulator` — extract 6 CBR fields from existing 269-feature windows. Avoids duplicate sampling but couples CBR to ONNX pipeline.
**Rationale:** The accumulator's 134-feature tensor is overbuilt for CBR's 3-field timeline. Direct GameState sampling is simpler, more maintainable, and won't break if ONNX features change. Different consumers, different concerns.
**Trade-offs:** Two components sampling GameState at 30s intervals. Negligible overhead — sampling is a record copy, not computation.
**Sources:** `TemporalWindowAccumulator` (134 features/player, 10 windows), `StrategyFeatureExtractor`
**Exploration:** quick
**Status:** revised (R1-02: field count updated from 6 to 3 after removing enemy fields)

## D12: Scope — Live Periodic Prediction in Issue #222

**Choice:** Include live periodic prediction (D3, D7) in the issue #222 deliverable, not just store-and-retrieve.
**Alternatives:**
- Store-and-retrieve only — sample at game close, query later. Matches the minimal reading of acceptance criteria. But "Retrieved temporal matches usable as predictive context" (AC4) is unvalidatable without actually running predictions during a game.
- Live prediction as a follow-up issue — cleaner scope boundary but delays the core value proposition. The issue body explicitly describes predictive CBR ("in a similar game, the opponent's army spike at minute 8 was followed by an all-in at minute 9 — prepare defences now") and calls it the "most research-grade feature" — research-grade means it needs experimentation, not deferral.
**Rationale:** The store-and-retrieve pipeline is infrastructure; the live prediction is the feature. Shipping infrastructure without the feature validates plumbing, not value. D3 and D7 ARE explicit decisions about live querying — the scope expansion was intentional, not implicit. However, it deserves to be surfaced as a first-class scope decision rather than flowing as a consequence of other choices.
**Trade-offs:** Larger scope for issue #222. The live prediction path introduces partial-timeline DTW (4-10 steps vs 20-40 stored), a new TaskDefinition in the tick chain, and CaseFile key consumers. These are tractable given the existing infrastructure.
**Sources:** Issue #222 acceptance criteria (AC4: "usable as predictive context"), issue body (predictive CBR use case)
**Exploration:** quick (surfaced by R1-03)
**Status:** captured

## D13: Task Chain Ordering — Temporal Task After Summarisation

**Choice:** `TemporalCbrTask` runs at the default position (after summarisation) in the tick chain. Its ID prefix (e.g., `temporal.cbr`) is NOT added to `QuarkMindCaseHub.PHASE_ORDER`. Temporal predictions are available to strategy/tactics on the next tick (500ms later).
**Alternatives:**
- Add `"temporal."` prefix before `"strategy-routing."` in PHASE_ORDER — predictions available in the same tick. But temporal prediction is advisory context, not routing input. It should not delay or gate strategy selection.
- Add `"temporal."` prefix after `"scouting."` — temporal prediction runs after scouting but before strategy. Plausible if temporal predictions should directly influence the strategy router, but this couples two independent retrieval mechanisms.
**Rationale:** Temporal predictions run every ~2 minutes. A 500ms delay (one tick) before strategy/tactics can consume them is <0.5% of the update interval — negligible. Running after summarisation means the temporal task sees the most recent phase data. The advisory nature of temporal predictions (informing, not routing) means same-tick availability is not required. The default append-after-summarisation position avoids modifying PHASE_ORDER, which is a deliberate architectural boundary.
**Trade-offs:** Strategy decisions in the tick immediately after a temporal query won't see the new predictions until one tick later. Acceptable for an advisory feature.
**Sources:** `QuarkMindCaseHub.PHASE_ORDER`, `QuarkMindCaseHub.phaseIndex()` — unknown prefix returns `PHASE_ORDER.size()` (after summarisation)
**Exploration:** quick (surfaced by R1-04)
**Status:** captured

## D14: Performance Strategy — Validate First, LbKeogh as Optimisation Path

**Choice:** Ship without explicit performance optimisation. If profiling shows the 500-case <1s target is not met, apply `LbKeogh` lower-bound pruning first, then `TrendSpec` pre-filtering.
**Alternatives:**
- Implement LbKeogh from the start — pre-compute envelopes per stored case, reject non-matching cases in O(n) before O(n*m) DTW. Guarantees sub-second at scale but adds complexity before validating the concept.
- Implement TrendSpec pre-filtering — use pre-computed scalar features (SLOPE, DELTA, VOLATILITY) to filter before DTW. Second-order optimisation, useful if LbKeogh pruning is insufficient.
**Rationale:** Back-of-envelope: DTW over 3 fields × 30 steps × 500 cases. Each DTW computation is O(n*m) = O(30*30) = 900 cells, each computing ~3 Euclidean distances = ~2700 operations. 500 cases: ~1.35M operations. On modern hardware, this completes in well under 100ms. The 500-case <1s target is met without any pruning. LbKeogh (`LbKeogh.computeEnvelope()`, `LbKeogh.lowerBound()`) and TrendSpec (`TrendType.SLOPE`, `DELTA`, `VOLATILITY`, `ACCELERATION`, `CHANGE_POINTS`) are available in the neocortex API and should be the first and second optimisation levers if empirical profiling contradicts this estimate, or if the case base grows beyond 500.
**Trade-offs:** Risk of discovering performance issues post-ship. Mitigated by the back-of-envelope analysis showing comfortable margin. D1's "end-to-end first" philosophy applies — premature optimisation before validating the concept is bad engineering.
**Sources:** `LbKeogh.computeEnvelope()`, `LbKeogh.lowerBound()`, `TrendSpec`, `TrendType` enum, `DtwSimilarity.compute()` cost model
**Exploration:** quick (surfaced by R1-08)
**Status:** captured
**Depends on:** D1 (end-to-end first), D8 (warping constraint)

## D15: Schema Registration — TimeSeries Field Addition

**Choice:** Add `FeatureField.timeSeries(...)` to `SC2CbrSchemaRegistrar.register()` alongside existing flat fields. Use `SimilaritySpec.DtwSpec(SakoeChibaBand(3))` for DTW similarity. No TrendSpec in the first pass.
**Alternatives:**
- Separate schema for temporal cases — isolates temporal features but fragments the case base. Existing cases would not be retrievable by temporal queries.
**Rationale:** `CbrFeatureSchema.of()` accepts any `FeatureField` including `TimeSeries` — the registration call is syntactically identical to flat fields. The `TimeSeries` record constructor validates inner fields (requires timestamp + at least one non-timestamp Numeric), so schema correctness is enforced at registration time. Existing cases stored without the timeline field will have null/empty values, which DTW handles naturally (empty sequence → 0.0 similarity for that field). Schema evolution is transparent in neocortex.
**Trade-offs:** Schema grows more complex with nested inner fields. Acceptable — the `FeatureField.timeSeries()` factory encapsulates the complexity.
**Sources:** `SC2CbrSchemaRegistrar.register()`, `FeatureField.timeSeries(name, timestampField, spec, innerFields...)`, `CbrFeatureSchema.of()`, `TimeSeries` constructor validation
**Exploration:** quick (surfaced by R1-13)
**Status:** captured
**Depends on:** D8 (DtwSpec constraint)
