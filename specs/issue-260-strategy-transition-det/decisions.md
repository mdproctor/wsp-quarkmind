## D1: Consumer scope

**Choice:** All four consumers — coaching trigger + advisory dispatch + workbench WebSocket + CBR retention
**Alternatives:**
- Coaching only — smaller branch but creates mechanical follow-up work for advisory/workbench/CBR hookup
- Coaching + advisory + workbench — defers CBR which needs schema thinking, but CBR retention is a thin observer
**Rationale:** The transition event record carries full metadata for all consumers. Each consumer is a thin observer subscribing to a CDI event — wiring all four now avoids follow-up issues for mechanical hookup.
**Trade-offs:** Slightly larger branch surface; CBR retention schema may evolve as game-outcome correlation analysis matures.
**Sources:** `PatternAssessmentPublished.java` (existing CDI event pattern), `WorkbenchEnricher.java` (existing observer pattern), `SC2CbrRetentionObserver.java` (existing CBR observer)
**Exploration:** quick
**Status:** captured

## D2: Detection location

**Choice:** `CascadingPatternClassifier` — the classifier already owns cumulative confidence history per archetype
**Alternatives:**
- `DroolsScoutingTask` — already tracks prevAssessments but would duplicate confidence-tracking state
- New Drools rule unit — maximum expressiveness but adds complexity to the scouting pipeline
- `MomentDetectionTask` — clean separation but adds CDI event coupling between scouting and moment detection
**Rationale:** The classifier naturally knows when archetype A was dominant and archetype B overtook it. Detection is co-located with the confidence data it depends on. Add a 'prev dominant' field, detect when dominant changes, return a `TransitionEvent` alongside `CascadeResult`.
**Trade-offs:** The classifier gains transition-detection responsibility beyond pure classification. Acceptable because transition detection is fundamentally a classification-over-time operation.
**Sources:** `CascadingPatternClassifier.java:52-53` (cumulativeConfidence EnumMap), `CascadingPatternClassifier.java:294-305` (mergeCumulative — decay + update), `DroolsScoutingTask.java:438-446` (assessmentsChanged — existing change detection)
**Exploration:** quick
**Status:** captured

## D3: Transition path model

**Choice:** YAML-defined transition paths in `strategy-taxonomy.yaml` with curated coaching text
**Alternatives:**
- Runtime-only detection — simpler but coaching quality depends entirely on LLM reasoning over raw archetypes
- Hybrid (YAML for coaching, runtime for detection) — best of both but two code paths
**Rationale:** Grounds coaching advice in curated domain knowledge. Known transitions get specific advice text ("they're switching to mech — get Immortals"). Unknown transitions are still detected but flagged as novel. The YAML is the single source of truth for transition domain knowledge, consistent with how the taxonomy already works for archetype signatures and counters.
**Trade-offs:** Requires YAML authoring effort for each transition path. Transitions not in YAML are detected but lack curated coaching text — they fall back to counter data from the `to` archetype.
**Sources:** `strategy-taxonomy.yaml` (existing YAML structure for archetypes), `StrategyTaxonomy.java` (existing YAML loader), #243 spec Section 7 (transitions listed as follow-up)
**Exploration:** quick
**Status:** captured

## D4: Transition trigger mechanics

**Choice:** Dominant-swap with hysteresis — track dominant archetype, fire transition when a different archetype overtakes with confidence gap
**Alternatives:**
- Absolute threshold crossing — simpler but susceptible to rapid oscillation in noisy classifications
- Sustained dominance window — most robust against noise but introduces detection latency, could miss fast transitions
**Rationale:** Hysteresis prevents flip-flopping between closely-matched archetypes. A transition fires when: (1) a different archetype overtakes the current dominant, (2) the new dominant's confidence exceeds a threshold (e.g., 0.4), and (3) the new dominant exceeds the old by a gap margin (e.g., 0.15). The gap margin is the key noise filter.
**Trade-offs:** Requires tuning two parameters (min confidence threshold, gap margin). A gap margin too high delays legitimate transitions; too low allows noise transitions. Calibration test against replay data will validate.
**Sources:** `CascadingPatternClassifier.java:34` (DISPATCH_THRESHOLD = existing threshold), `CascadingPatternClassifier.java:35` (DECAY_PER_FRAME = existing decay rate), `CascadingPatternClassifier.java:294-305` (mergeCumulative — confidence mechanics)
**Exploration:** quick
**Status:** captured

## D5: Initial YAML coverage

**Choice:** All three matchups (PvT, PvZ, PvP) — ~5-8 common transition paths per matchup (~15-24 total)
**Alternatives:**
- PvT only — faster to ship but leaves PvZ and PvP coaching without transition awareness
**Rationale:** YAML authoring is mechanical once the schema is defined. Most transitions follow obvious EARLY→MID→LATE phase progressions within a race. Validates the model across all matchups immediately.
**Trade-offs:** More YAML authoring upfront, but all paths are from Protoss perspective (consistent with existing counter scope) and follow known SC2 metagame patterns.
**Sources:** `strategy-taxonomy.yaml` (existing archetype definitions per race), #243 spec Section 7 (transitions listed as follow-up scope)
**Exploration:** quick
**Status:** captured
