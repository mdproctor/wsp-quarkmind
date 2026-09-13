## D1: Consumer scope

**Choice:** All four consumers — coaching trigger + advisory dispatch + workbench WebSocket + CBR retention
**Alternatives:**
- Coaching only — smaller branch but creates mechanical follow-up work for advisory/workbench/CBR hookup
- Coaching + advisory + workbench — defers CBR which needs schema thinking, but CBR retention is a thin observer
**Rationale:** The transition event carries a `StrategyTransition` record (from/to archetypes, confidence values, detected frame, optional `TransitionPath`). The four consumers are: (1) coaching trigger and (2) advisory dispatch via `GameMomentType.STRATEGY_TRANSITION` through the moment pipeline — zero new wiring; (3) workbench via `@Observes StrategyTransitionPublished` — parallel to existing `PatternAssessmentPublished` observer pattern; (4) CBR retention via `@Observes StrategyTransitionPublished` — collects transition objects mid-game, enriches features at game-end in `onOutcome()`. Note: `SC2AdvisoryCbrRetentionObserver` is NOT a transition consumer — it observes `CaseOutcomeEvent` for advisory invocation success tracking, a separate concern. The CBR retention observer's event subscription is thin (one line: add to list); its complex enrichment logic (phases, moments, arcs, dominance) runs at game-end in `onOutcome()`, not in response to the transition event.
**Trade-offs:** Slightly larger branch surface; CBR retention schema may evolve as game-outcome correlation analysis matures.
**Sources:** `PatternAssessmentPublished.java` (existing CDI event pattern), `WorkbenchEnricher.java` (existing observer pattern), `SC2CbrRetentionObserver.java` (existing CBR observer — `CaseOutcomeObserver`, not a transition observer), `SC2AdvisoryCbrRetentionObserver.java` (advisory-level CBR — `CaseOutcomeObserver`, separate concern)
**Exploration:** quick
**Status:** revised — clarified "full metadata" to name the record, specified thin subscription vs. game-end enrichment, confirmed consumer count

## D2: Detection location

**Choice:** `CascadingPatternClassifier` — the classifier already owns cumulative confidence history per archetype
**Alternatives:**
- `DroolsScoutingTask` — already tracks prevAssessments but would duplicate confidence-tracking state
- New Drools rule unit — maximum expressiveness but adds complexity to the scouting pipeline
- `MomentDetectionTask` — clean separation but adds CDI event coupling between scouting and moment detection
**Rationale:** The classifier naturally knows when archetype A was dominant and archetype B overtook it. Detection is co-located with the confidence data it depends on. Add a 'prev dominant' field, detect when dominant changes, return a `TransitionEvent` alongside `CascadeResult`. The classifier is already not stateless — it maintains `cumulativeConfidence` (EnumMap), `llmFallbackEnabled`, `lastLlmFallbackFrame`, `lastProcessedLlmArchetype`. Adding `prevDominant` is one field and ~15 lines of detection logic — a natural extension, not responsibility creep. Downstream delivery still uses the moment pipeline: `DroolsScoutingTask` publishes `ScoutingIntelPayload.TransitionDetected`, `MomentDetectionTask.drl` picks it up and emits `GameMomentType.STRATEGY_TRANSITION`, which flows through `CoachingTriggerBuilder` and `AdvisoryTriggerBuilder` with zero new consumer wiring. The transition detection is not a "fourth classification tier" — it's orthogonal to the Drools→ONNX→LLM cascade, comparing consecutive results rather than adding a new classification source.
**Trade-offs:** The classifier gains transition-detection responsibility beyond pure classification. Acceptable because transition detection is fundamentally a classification-over-time operation.
**Sources:** `CascadingPatternClassifier.java:52-53` (cumulativeConfidence EnumMap), `CascadingPatternClassifier.java:55-60` (existing temporal state: LLM fallback tracking), `CascadingPatternClassifier.java:294-305` (mergeCumulative — decay + update), `DroolsScoutingTask.java:438-446` (assessmentsChanged — existing change detection, fundamentally different from transition detection)
**Exploration:** quick
**Status:** captured

## D3: Transition path model

**Choice:** YAML-defined transition paths in `strategy-taxonomy.yaml` with curated coaching text
**Alternatives:**
- Runtime-only detection — simpler but coaching quality depends entirely on LLM reasoning over raw archetypes
- Hybrid (YAML for coaching, runtime for detection) — best of both but two code paths
**Rationale:** Grounds coaching advice in curated domain knowledge. Known transitions get specific advice text ("they're switching to mech — get Immortals"). Unknown transitions are still detected but flagged as novel. The YAML is the single source of truth for transition domain knowledge, consistent with how the taxonomy already works for archetype signatures and counters. Note: YAML defines coaching text and display names, NOT detection parameters. Detection is handled by D2 (classifier). `TemporalCbrTask` serves a different purpose — it predicts what MIGHT happen based on similar historical games; it doesn't detect that a transition HAS occurred. The YAML and CBR temporal prediction are complementary: one provides curated expert coaching advice, the other provides statistical forecasting. Transition-as-archetype patterns (e.g., `TERRAN_BC_TRANSITION`) are handled naturally — they appear as the `to` field in YAML transition paths.
**Trade-offs:** Requires YAML authoring effort for each transition path. Transitions not in YAML are detected but lack curated coaching text — they fall back to counter data from the `to` archetype.
**Sources:** `strategy-taxonomy.yaml` (existing YAML structure for archetypes), `StrategyTaxonomy.java` (existing YAML loader), `TemporalCbrTask.java` (CBR temporal prediction — complementary, not overlapping), `StrategyArchetype.java:62` (TERRAN_BC_TRANSITION — transition-as-archetype pattern), #243 spec Section 7 (transitions listed as follow-up)
**Exploration:** quick
**Status:** revised — clarified YAML scope (coaching text only, not detection), documented relationship to TemporalCbrTask and transition-as-archetype pattern

## D4: Transition trigger mechanics

**Choice:** Dominant-swap with hysteresis — track dominant archetype, fire transition when a different archetype overtakes with confidence gap
**Alternatives:**
- Absolute threshold crossing — simpler but susceptible to rapid oscillation in noisy classifications
- Sustained dominance window — most robust against noise but introduces detection latency, could miss fast transitions
**Rationale:** Hysteresis prevents flip-flopping between closely-matched archetypes. A transition fires when: (1) a different archetype overtakes the current dominant, (2) the new dominant's confidence exceeds a threshold (e.g., 0.4), and (3) the new dominant exceeds the old by a gap margin (e.g., 0.15). The gap margin is the key noise filter.
**Intermediate state behavior:** The dominant archetype is tracked volatilely — `prevDominant` updates to whichever archetype has the highest cumulative confidence, regardless of whether the gap threshold is met. Only the transition EVENT requires the gap. During near-tie confidence, no transition fires; coaching continues based on the latest sorted pattern assessment list.
**Decay/gap interaction:** The exponential decay (`DECAY_PER_FRAME = 0.99948`) interacts with the gap margin. Over 100 frames (~4.5s), confidence decays ~5%. The gap between archetypes is partly an artifact of differential evidence arrival, not just strategic shift. This is a parameter coupling — calibration against replay data must validate that the gap margin correctly separates genuine transitions from decay-induced crossovers.
**Evidence disappearance mitigation:** When enemy visibility is lost (scout dies), two mechanisms prevent false transitions: (1) `mergeCumulative` applies uniform decay to ALL archetypes — if no new evidence arrives for any archetype, all decay at the same rate and relative ordering is preserved; (2) `TRANSITION_MIN_CONFIDENCE = 0.4` ensures the "to" archetype needs substantial cumulative evidence, which noise alone cannot produce. The failure mode "evidence gap → noise dominant" requires differential evidence arrival, which doesn't happen when no enemies are visible.
**Trade-offs:** Requires tuning two parameters (min confidence threshold, gap margin). A gap margin too high delays legitimate transitions; too low allows noise transitions. Calibration test against replay data will validate. The decay/gap interaction is the critical coupling to calibrate.
**Sources:** `CascadingPatternClassifier.java:34` (DISPATCH_THRESHOLD = existing threshold), `CascadingPatternClassifier.java:35` (DECAY_PER_FRAME = existing decay rate), `CascadingPatternClassifier.java:294-305` (mergeCumulative — uniform decay behavior), `DroolsScoutingTask.java:436` (THRESHOLDS — existing threshold-crossing detection, different from gap margin)
**Exploration:** quick
**Status:** revised — documented intermediate state behavior, decay/gap interaction, and evidence disappearance mitigation

## D5: Initial YAML coverage

**Choice:** All three matchups (PvT, PvZ, PvP) — ~5-8 common transition paths per matchup (~15-24 total)
**Alternatives:**
- PvT only — faster to ship but leaves PvZ and PvP coaching without transition awareness
**Rationale:** YAML schema structure is straightforward once defined. Coaching text authoring requires SC2 domain expertise to map transition patterns to effective responses — this is content creation, not mechanical work. Most transitions follow known EARLY→MID→LATE phase progressions within a race. PvP is structurally harder than PvT/PvZ (same unit pool on both sides means archetype discrimination depends more on production facility targeting and tech building scouting) but the spec accounts for this with fewer PvP paths (3 in current draft vs. 4 each for PvT/PvZ). Validates the model across all matchups immediately.
**Trade-offs:** More YAML authoring upfront, but all paths are from Protoss perspective (consistent with existing counter scope) and follow known SC2 metagame patterns. PvP paths may need refinement as classifier accuracy data for mirror matchups becomes available.
**Sources:** `strategy-taxonomy.yaml` (existing archetype definitions per race), `StrategyArchetype.java` (73 archetypes across all races and phases), #243 spec Section 7 (transitions listed as follow-up scope)
**Exploration:** quick
**Status:** revised — corrected "mechanical" characterisation, acknowledged PvP structural differences

## D6: CDI event delivery mechanism

**Choice:** Dual-path — `StrategyTransitionPublished` CDI event for rich-data consumers + `GameMomentType.STRATEGY_TRANSITION` through the moment pipeline for coaching/advisory
**Alternatives:**
- CDI event only (new `StrategyTransitionPublished`) — requires new `@Observes` methods in every consumer including coaching/advisory, duplicating the moment pipeline
- GameMomentType only (new `STRATEGY_TRANSITION_DETECTED` value) — reuses coaching/advisory pipeline with zero new wiring, but `GameMoment.context()` (`Map<String, Object>`) cannot carry the full `StrategyTransition` record (confidences, `TransitionPath` with coaching text) that workbench and CBR consumers need
- CascadeResult extension only — couples transition semantics to the classification return path; callers that don't need transitions are forced to handle the expanded contract
**Rationale:** Different consumers need different data granularity. Coaching and advisory only need the moment type to determine urgency tier — `GameMomentType.STRATEGY_TRANSITION` through the existing pipeline handles this with zero new wiring in `CoachingTriggerBuilder.mapMomentToTier()` and `AdvisoryTriggerBuilder.mapMomentTypeToTrigger()`. Workbench needs from/to archetypes, confidence values, and transition path display data. CBR retention needs the full `StrategyTransition` object to store as features. These consumers use `@Observes StrategyTransitionPublished`. The flow is: classifier detects → `DroolsScoutingTask` publishes `ScoutingIntelPayload.TransitionDetected` AND fires `StrategyTransitionPublished` CDI event → `MomentDetectionTask.drl` rule picks up the intel payload and emits `GameMomentType.STRATEGY_TRANSITION` → coaching/advisory consume the moment.
**Trade-offs:** Two event paths for one detection. But each path matches its consumers' existing patterns — moments for coaching/advisory, CDI events for workbench/CBR.
**Sources:** `CoachingTriggerBuilder.java:63-70` (mapMomentToTier — switch on GameMomentType), `AdvisoryTriggerBuilder.java:61-68` (mapMomentTypeToTrigger — switch on GameMomentType), `WorkbenchEnricher.java:30` (@Observes PatternAssessmentPublished — existing CDI observer pattern), `SC2CbrRetentionObserver.java:84-94` (existing event collection pattern), `MomentDetectionTask.drl:49-57` (existing rule pattern for scouting intel → moment)
**Exploration:** quick — surfaced by reviewer
**Status:** captured

## D7: Relationship to SC2StrategyRouterTask pivot detection

**Choice:** Parallel signals from shared data — transition detection and pivot detection are independent consumers of pattern assessment data, serving different downstream purposes
**Alternatives:**
- Transition event triggers router re-evaluation — couples coaching signal to strategy selection
- Router consumes transition events instead of pattern assessments — breaks existing routing logic
**Rationale:** `SC2StrategyRouterTask` reads the current best pattern assessment from `ScoutingIntelBroker`, checks if the archetype changed, increments `pivotCount`, and fires `StrategySelectionPublished`. This is strategy selection — "which counter-strategy should WE play?" The transition event drives coaching — "the opponent is switching to mech, here's advice." These serve different downstream effects (strategy selection vs. coaching advice) and should remain independent. A single archetype change in the classifier produces one router pivot (via pattern assessment) and one transition event (via transition detection). They don't double-count because they feed different pipelines. The router's `maxPivots` cap operates independently of transition event count.
**Trade-offs:** Two components independently track archetype changes. This is the correct separation of concerns — merging them would couple coaching frequency to routing decisions.
**Sources:** `SC2StrategyRouterTask.java:117-216` (execute — archetype change detection, pivotCount, routing), `StrategySelectionPublished.java:5` (routing event record with pivotCount)
**Exploration:** quick — surfaced by reviewer
**Status:** captured

## D8: Relationship to existing TECH_TRANSITION_DETECTED

**Choice:** Complementary signals — `TECH_TRANSITION_DETECTED` is a build-order detection (early warning from what they're building), `STRATEGY_TRANSITION` is a composition classification shift (confirmed transition from what they have)
**Alternatives:**
- Subsume TECH_TRANSITION_DETECTED into STRATEGY_TRANSITION — loses the early warning signal
- Merge into a single signal with source tag — adds complexity to moment handling for no consumer benefit
- Deduplicate at the coaching level — suppress coaching for STRATEGY_TRANSITION if a recent TECH_TRANSITION_DETECTED fired for the same archetype
**Rationale:** `TECH_TRANSITION_DETECTED` fires from `MomentDetectionTask.drl` when a `BuildOrder` scouting intel event has a non-UNKNOWN detected value. This is a point-in-time observation: "we scouted their build order." `STRATEGY_TRANSITION` fires when the dominant archetype shifts in cumulative classification. These are different evidence sources and fire at different times — build order detection fires earlier (when we see the Factory being built), composition classification fires later (when enough Tanks are observed to shift the dominant archetype). Both are coaching-relevant: the first is "watch out, they might be switching," the second is "they've switched — here's what to do." The existing `CoachingTriggerBuilder.canFire()` cooldown mechanism prevents rapid-fire coaching for closely-spaced moments of the same urgency tier (both map to `STRATEGIC`).
**Trade-offs:** Two transition-related moments can fire for the same strategic shift. The cooldown mechanism handles this, but the coaching text should complement rather than repeat.
**Sources:** `MomentDetectionTask.drl:49-57` (Tech Transition Detected rule — fires on BuildOrder intel), `CoachingTriggerBuilder.java:56-60` (canFire — cooldown mechanism prevents rapid-fire), `CoachingTriggerBuilder.java:67` (both map to CoachingUrgencyTier.STRATEGIC)
**Exploration:** quick — surfaced by reviewer
**Status:** captured

## D9: Transition event persistence

**Choice:** Three persistence paths — CaseContext (per-tick), ScoutingIntelBroker (scouting subscription), CBR features (game-end enrichment)
**Alternatives:**
- Add casehub-ledger audit trail — out of scope for this issue, can be added independently
- Publish on qhorus channel — no external consumers identified for transition data yet
**Rationale:** The transition event flows through three persistence paths that already exist in the scouting pipeline: (1) `CaseContext.set(STRATEGY_TRANSITION, enriched)` — available to any task in the current tick; (2) `publishIntel(TransitionDetected)` → `ScoutingIntelBroker` — available to any subscriber of scouting intel; (3) `SC2CbrRetentionObserver` collects transitions mid-game, adds `transition_count` and `transitions` feature values to `SC2GameCbrCase` at game-end. These mirror how `PatternAssessmentPublished` flows through the same three paths. Ledger and qhorus persistence are independent concerns that can be added without changing this design.
**Sources:** `DroolsScoutingTask.java:389-398` (publishIntel — CaseContext + broker + CDI event pattern), `SC2CbrRetentionObserver.java:109-235` (game-end enrichment pattern)
**Exploration:** quick — surfaced by reviewer
**Status:** captured
