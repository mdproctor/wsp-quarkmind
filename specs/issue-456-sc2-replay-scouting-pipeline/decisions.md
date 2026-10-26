## D1: HypothesisTracker role

**Choice:** General-purpose hypothesis tracking tool — not just scouting, but also strategy selection and plan driving. Opponent detection feeds response selection.
**Alternatives:**
- Scouting-only — simpler but misses the real value of hypothesis-driven decision-making
- Plan/script switching only — needs outcome data that doesn't exist yet
**Rationale:** The full loop (detect opponent → select response) is where the value lies. Both sides use the same composition mechanism.
**Trade-offs:** Broader scope means more integration work, but the underlying mechanism (composed Bayesian networks + hypothesis tracking) is the same regardless.
**Sources:** casehubio/neocortex#692 (HypothesisTracker), casehubio/neocortex#705 (HypothesisPriorLearner), CascadingPatternClassifier.java, DetectorOrchestrator.java
**Exploration:** quick
**Status:** captured

## D2: Training data sources

**Choice:** Replay win/loss outcomes (bulk prior learning) + expert-defined matchup knowledge (prior overrides/constraints)
**Alternatives:**
- Replay outcomes only — misses expert knowledge where data is sparse
- Expert-defined only — doesn't learn from data
- Own game outcomes only — too small initially
**Rationale:** Data gives base rates, expert knowledge corrects where data is sparse or misleading. Both feed HypothesisPriorLearner (batch mode for replays, manual overrides for expert knowledge).
**Trade-offs:** Need to define how expert overrides interact with learned priors.
**Sources:** quarkmind-classifier .label.json sidecars, HypothesisPriorLearner.fromOutcomes()
**Exploration:** quick
**Status:** captured

## D3: Integration with existing cascade

**Choice:** Replace DetectorOrchestrator's BayesNetwork with HypothesisTracker as the fusion layer. Drools evidence markers and ONNX detector outputs feed it as observations.
**Alternatives:**
- Sit above the cascade — less invasive but adds a redundant layer
- Run in parallel — good for validation but redundant long-term
**Rationale:** DetectorOrchestrator already builds a per-tick Bayesian network structurally similar to HypothesisTracker, but without learned priors or path elimination. One probabilistic model is better than two competing ones.
**Trade-offs:** Reworking the existing fusion means touching a well-tested path. Need to ensure the HypothesisTracker produces equivalent or better assessments.
**Sources:** DetectorOrchestrator.java, CascadingPatternClassifier.java, SC2NetworkBuilder.java
**Exploration:** quick
**Status:** captured

## D4: Network composition approach

**Choice:** Composed networks — opponent detection and response selection modelled in one composition graph, with trackers at whatever granularity makes sense. It's all just composition.
**Alternatives:**
- Two separate independent trackers — artificial separation of what's fundamentally one composed system
- Single flat tracker — loses the compositional structure
**Rationale:** NetworkComposition is a DAG of composed junction trees. Opponent and response variables compose naturally. Multiple trackers can share composition slots.
**Trade-offs:** Composed network is more complex to define and train CPTs for, but the mechanism already exists in neocortex.
**Sources:** HypothesisTracker.builder(NetworkComposition), NetworkComposition API
**Exploration:** quick
**Status:** captured

## D5: Observation model

**Choice:** Discrete named events — variables with discrete states (first_building=gateway|forge|nexus, gas_timing=early|normal|late, etc.). Maps to observeHard().
**Alternatives:**
- ONNX outputs as soft evidence — reuses existing detectors but loses interpretability
- Both discrete + ONNX soft evidence — richer but more variables to define and learn CPTs for
**Rationale:** Discrete events map directly to the HypothesisTracker API (observeHard). They're interpretable, testable, and the discretization logic is well-defined.
**Trade-offs:** Need to define the right discretization boundaries. Loses continuous probability information from ONNX detectors.
**Sources:** HypothesisTracker.observeHard(), StrategyFeatureExtractor.java, FeatureIndexMaps.java
**Exploration:** quick
**Status:** captured

## D6: Module placement

**Choice:** quarkmind-sc2 — SC2-specific network structure, variable definitions, and discretization logic.
**Alternatives:**
- quarkmind-core — violates layering rule since variables are SC2-specific
- New module — not enough code to justify separation
**Rationale:** Layering rule: SC2 game knowledge belongs in quarkmind-sc2, not in the foundation.
**Trade-offs:** None significant.
**Sources:** CLAUDE.md layering rule
**Exploration:** quick
**Status:** captured

## D7: Discrete observation extraction

**Choice:** New ScoutingObservationExtractor that reads existing game_json and produces discrete event records.
**Alternatives:**
- Extend ReplayFeatureExtractor — mixes continuous and discrete concerns
- Python-side in classifier — adds Python dependency for Java-consumed output
**Rationale:** Keeps continuous (CNN training) and discrete (Bayesian hypothesis) pipelines separate. Clean single-responsibility.
**Trade-offs:** Another extractor to maintain, but the output format is fundamentally different.
**Sources:** ReplayFeatureExtractor.java, TrackerEventFeatureExtractor.java
**Exploration:** quick
**Depends on:** D5 (observation model)
**Status:** captured
