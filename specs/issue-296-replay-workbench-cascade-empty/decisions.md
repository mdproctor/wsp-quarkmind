## D1: Cascade empty-result fix strategy

**Choice:** Visible-enemy fallback + generic rule weight tuning
**Alternatives:**
- Lower DISPATCH_THRESHOLD per profile — lets noise through, misleading workbench
- Improve generic rule weights only — no safety net for unusual compositions
**Rationale:** COMPOSITION_UNKNOWN fallback guarantees the pipeline never starves when enemies are visible. Generic rule weight tuning makes it a genuine last resort. Combined approach: proper classifications when the composition is recognisable, honest "unknown" when it isn't.
**Trade-offs:** Adds a new StrategyArchetype enum value; downstream consumers (strategy, commentary) need to handle COMPOSITION_UNKNOWN sensibly rather than treating it like a real classification.
**Sources:** CascadingPatternClassifier.java:192 (existing fallback path), PatternClassification.drl (generic rules), StrategyTaxonomy.java:151 (hardcoded 0.5 weight), strategy-taxonomy.yaml (signature specs)
**Exploration:** quick
**Status:** captured

**Tuning approach:** Heuristic, replay-driven. Weights and thresholds are calibrated by watching replays and verifying classifications against known game states — not by formula. Tests assert calibration ranges, not exact values.

## D2: Testing architecture — recorded event replay with incremental composition

**Choice:** Committed event fixtures at each layer boundary, with isolation tests and incremental composition tests
**Alternatives:**
- End-to-end only — catches total failure, zero diagnostic signal on which layer broke
- Synthetic-input unit tests only — disconnected from real data, can pass while actual replays produce nothing (the current situation)
**Rationale:** Each layer has a well-defined input/output contract at its boundary. By capturing real events from replay runs and committing them as fixtures, we get: (1) isolation — test each layer against real data without running preceding layers, (2) composition — incrementally stack layers (L1, L1+L2, L1+L2+L3, ...) proving they compose correctly, (3) calibration — the fixtures ARE the tuning artifacts, run and re-run as weights change.
**Trade-offs:** Fixture files need updating when layer contracts change (new fields on records). Mitigated by keeping fixtures minimal — only the fields each layer actually reads.
**Sources:** PatternClassificationCalibrationTest (existing calibration pattern), CascadeValidationCalibrationTest (existing cascade calibration), ReplayValidationHarness (existing replay infrastructure)
**Exploration:** quick
**Status:** captured

**Layer boundaries and fixture types:**

| Boundary | Input fixture | Output fixture |
|----------|--------------|----------------|
| L1: Drools evidence | UnitEvent[], ExpansionEvent[], gameTime, armyNearBase | EvidenceMarker[], ConfidenceRevision[] |
| L2: Cascade routing | EvidenceMarker[], ConfidenceRevision[], StrategyFeatures, Race, frame | CascadeResult (assessments[], llmTriggered) |
| L3: Scouting publish | CascadeResult, broker subscription state | PatternAssessmentPublished event (or absent) |
| L4: Workbench enricher | PatternAssessmentPublished | WorkbenchEvent (type, payload) |
| L5: WebSocket delivery | WorkbenchEvent | JSON text message |

**Composition test progression:**
- L1 isolation: replay unit events → assert evidence markers
- L2 isolation: replay evidence markers → assert cascade result
- L1+L2: replay unit events → run Drools → feed cascade → assert assessments
- L1+L2+L3: same + scouting task publish → assert CDI event fires
- L1+L2+L3+L4: same + workbench enricher → assert WorkbenchEvent broadcast
- Full stack: same + WebSocket → assert client receives JSON

**Fixture source:** Captured from actual replay runs against the AI Arena and IEM10 datasets. Committed to `quarkmind-sc2/src/test/resources/fixtures/pipeline/`. One directory per replay per layer.
**Depends on:** D1 (cascade fix determines what L2 output looks like)

## D3: Downstream handling of COMPOSITION_UNKNOWN

**Choice:** Transparent passthrough — COMPOSITION_UNKNOWN is a real StrategyArchetype enum value (phase=MID, category=COMPOSITION) that flows through the normal pipeline
**Alternatives:**
- Marker event with separate handling — splits pipeline into "real" and "fallback" paths, every consumer needs two code paths
- Suppress and show "no data" — strategy/commentary still get nothing, defeats the purpose
**Rationale:** Low-confidence (0.35) assessment through the standard pipeline is honest and simple. Strategy selects a safe default. Commentary describes what's visible. Workbench shows "Unknown composition (0.35)". No consumer special-casing needed — the low confidence naturally limits its influence.
**Trade-offs:** COMPOSITION_UNKNOWN appears in strategy selection and commentary as a real (if low-confidence) archetype. Consumers that do `switch` on StrategyArchetype need a case for it. Sealed-type exhaustiveness catches these at compile time.
**Sources:** StrategyArchetype.java (sealed enum, exhaustive switches), DroolsStrategyTask (strategy consumer), CommentaryWorkerFactory (commentary consumer)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (fallback produces COMPOSITION_UNKNOWN)
