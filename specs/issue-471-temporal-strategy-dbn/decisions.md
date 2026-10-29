# Decisions — #471 Temporal Strategy Evolution

## D1: Scope — both classification systems

**Choice:** Both DetectorOrchestrator and HypothesisTrackerFusion
**Alternatives:**
- HypothesisTrackerFusion only — simpler scope but leaves DetectorOrchestrator with the crude sliding-window smoother
- DetectorOrchestrator only — doesn't address the phase-transition reinitialisation problem
**Rationale:** Both systems have temporal concerns that the same solution addresses. DetectorOrchestrator's TemporalAssessmentTracker is a crude version of what TemporalEvidenceFusionEngine already does; HypothesisTrackerFusion's phase-transition reinit is the primary pain point.
**Trade-offs:** Larger scope increases implementation effort and risk surface
**Sources:** HypothesisTrackerFusion.java:79-138 (phase transition logic), DetectorOrchestrator.java:179-217 (classify with TemporalAssessmentTracker)
**Exploration:** quick
**Status:** captured

## D2: Learning mode — offline from replays

**Choice:** Offline pipeline from replay data
**Alternatives:**
- Online during gameplay — less data, harder to validate, more complex
- Both (offline seed, online update) — maximum adaptability but highest complexity
**Rationale:** Consistent with existing prior-learning pattern (ContextResponsePriorExportTest, HypothesisPriorLoader). 149K replays provide rich data. Classpath JSON loading is proven.
**Trade-offs:** No real-time adaptation — transition probabilities are static until retrained
**Sources:** ContextResponsePriorExportTest (existing CPT export pattern), HypothesisPriorLoader.java (classpath JSON loading)
**Exploration:** quick
**Status:** captured

## D3: Temporal model — TemporalEvidenceFusionEngine as-is

**Choice:** Use neocortex's existing TemporalEvidenceFusionEngine (linear blending with configurable decay)
**Alternatives:**
- Transition-CPT-based model — more principled but requires matrix ops and new learning pipeline
- Full two-slice DBN — classical formulation but requires engine changes in neocortex
**Rationale:** TemporalEvidenceFusionEngine already delivered by neocortex#630. Linear blending is simple, proven, and the decay parameter is adjustable at runtime. No cross-repo engine work needed.
**Trade-offs:** Linear blending is less principled than a proper transition model — it doesn't encode which strategies transition to which. But transition-awareness can be added via the transition priors without changing the engine.
**Sources:** TemporalEvidenceFusionEngine.java (neocortex inference-bayes), neocortex#630 (closed, landed as 7d140756)
**Exploration:** quick
**Status:** captured

## D4: Transition handling — hybrid taxonomy + discovered

**Choice:** Taxonomy transitions plus high-confidence discovered transitions (>5% frequency threshold)
**Alternatives:**
- Taxonomy-only — interpretable but misses real transitions not yet hand-authored
- Data-driven discovery — learns all pairs but risks noisy/nonsensical transitions
**Rationale:** The 14 hand-authored transitions are a strong prior. Replay data will reveal common transitions not yet in the taxonomy (e.g., PROTOSS_DT_RUSH → PROTOSS_BLINK_STALKER). The 5% threshold filters noise while allowing genuine discoveries.
**Trade-offs:** Adds a calibration parameter (frequency threshold). Discovered transitions lack coaching advice until hand-authored.
**Sources:** strategy-taxonomy.yaml transitions section (14 paths), 149K replays with timestamped strategy snapshots
**Exploration:** quick
**Status:** captured

## D5: Phase model — single unified archetype set

**Choice:** All 55 archetypes from all phases in one tracker from the start
**Alternatives:**
- Phase-gated with smooth handoff — keeps separate sets but smooths transitions over N ticks
- Expanding archetype set — starts small, adds archetypes as game progresses via BayesInstance.addVariable()
**Rationale:** Eliminates the phase-transition reinitialisation entirely. Phase windows (already in taxonomy) provide natural probability decay — an EARLY archetype's probability falls as the game passes its phase window. No network rebuild, no seedPriors() hack.
**Trade-offs:** Larger Bayesian network (55 hypotheses vs ~15-20 per phase). More computation per tick. Some archetype labels only make sense in their phase — MARINE_RUSH at minute 15 should have near-zero probability, which the phase window handles.
**Sources:** StrategyTaxonomy.java:33-110 (init with phase windows), ArchetypeEntry.phaseWindow()
**Exploration:** quick
**Status:** captured

## D6: Replay checkpoints — 1, 3, 5, 7, 10 minutes

**Choice:** Five checkpoints at 1, 3, 5, 7, 10 minutes
**Alternatives:**
- Every minute 1-15 — maximum resolution but 15× more data, mostly no-transition noise
- 3 and 5 minutes only — misses MID→LATE transitions entirely
**Rationale:** Covers EARLY (1-3), EARLY→MID boundary (3-5), MID (5-7), MID→LATE boundary (7-10). Aligns with phase windows. 5 checkpoints × 149K replays = rich transition data without excessive noise.
**Trade-offs:** Misses very late transitions (>10 min). Can extend later if needed.
**Sources:** ContextResponsePriorExportTest (existing 3/5 min checkpoint pattern), strategy-taxonomy.yaml phase windows
**Exploration:** quick
**Status:** captured

## D7: Integration architecture — layered temporal replacement

**Choice:** Approach A — layered temporal replacement
**Alternatives:**
- Approach B (temporal wrapper) — less invasive but adds indirection, transition application outside Bayesian framework
- Approach C (network-level temporal edges) — most principled but couples temporal logic into network structure, hard MAP evidence discards uncertainty
**Rationale:** Uses existing TemporalEvidenceFusionEngine directly for DetectorOrchestrator (clean 1:1 replacement). For HypothesisTrackerFusion, eliminates phase-transition reinit by using unified archetype set. Each piece (DetectorOrchestrator swap, unified tracker, transition learning) can be implemented and tested independently.
**Trade-offs:** Unified archetype set makes the network larger. TemporalEvidenceFusionEngine's linear blending doesn't encode transition structure — but transition-aware priors compensate.
**Sources:** EvidenceFusionEngine.java, TemporalEvidenceFusionEngine.java, HypothesisTrackerFusion.java, DetectorOrchestrator.java
**Exploration:** quick
**Status:** captured
