# Decisions — #213 IEM10 Replay Validation & Accuracy Benchmarking

## D1: ONNX model availability

**Choice:** Design for both — framework works Drools-only when ONNX unavailable, activates full cascade when model present. ONNX models have now landed in neocortex (three per-race models).
**Alternatives:**
- Wait for model — park #213 until neocortex delivers. Unnecessary, models are available.
- Mock ONNX tier — synthetic classifier as stand-in. Doesn't test real inference.
**Rationale:** The cascade already skips ONNX when the model isn't resolvable. With models now available, we wire the full path.
**Trade-offs:** None significant — graceful degradation is already built in.
**Sources:** CascadingPatternClassifier.java constructor (CDI Instance resolution), neocortex StrategyClassifierOnnxTest.java
**Exploration:** quick
**Status:** captured

## D2: Test placement for cascade validation

**Choice:** New dedicated class — CascadeValidationCalibrationTest
**Alternatives:**
- Extend existing PatternClassificationCalibrationTest — grows an already large test.
- Separate + refactor shared logic — more work, unclear benefit given different test patterns.
**Rationale:** Cascade validation has different concerns (multi-tier comparison, tier hit rates) from the existing calibration test (single-tier accuracy at 3-min). Dedicated class keeps each focused.
**Trade-offs:** Some duplication of replay loading and ground-truth derivation logic. Acceptable — the classification loop is fundamentally different (full classify() vs static methods).
**Sources:** PatternClassificationCalibrationTest.java
**Exploration:** quick
**Status:** captured

## D3: Latency benchmark placement

**Choice:** Separate class — CascadeLatencyBenchmarkTest (@Tag("benchmark"))
**Alternatives:**
- Combined with accuracy validation — fewer files but mixes concerns.
**Rationale:** Consistent with existing EmulatedGameBenchmarkTest pattern. Latency and accuracy have different runtime profiles and assertion thresholds.
**Trade-offs:** Additional test class.
**Sources:** EmulatedGameBenchmarkTest.java, GameLoopBenchmarkTest.java
**Exploration:** quick
**Status:** captured

## D4: Output format

**Choice:** Stdout tables + one-time committed snapshot to docs/benchmarks/
**Alternatives:**
- Stdout only — no persistent record.
- Time-series tracking — unnecessary; numbers change on discrete events (rule changes, model retraining), not gradually.
**Rationale:** Accuracy and latency numbers are baseline snapshots, not trends. Stdout for development, committed snapshot for the record.
**Trade-offs:** Snapshot may go stale — but that's acceptable; rerun overwrites.
**Sources:** Existing benchmark tests all write to target/ (ephemeral).
**Exploration:** quick
**Status:** captured

## D5: Assertion strategy

**Choice:** Assert selectively — hard-assert ≥70% rush/air accuracy on Drools-only config (already proven), report ONNX-only and cascade metrics informationally. ONNX-specific thresholds added when model integration is validated.
**Alternatives:**
- Hard-assert all thresholds now — premature without ONNX model validation history.
- Informational only — loses regression protection on the Drools tier.
**Rationale:** The Drools tier already meets the 70% rush/air bar. Protecting that baseline catches regressions. ONNX-only and cascade configs are new — no baseline exists to assert against. Other metrics are exploratory until the cascade is validated across multiple runs.
**Trade-offs:** Weaker assertions initially — tightened incrementally.
**Sources:** PatternClassificationCalibrationTest assertions
**Exploration:** quick
**Status:** revised (review clarified assertion scope per config)

## D6: Cascade validation loop approach

**Choice:** Full cascade with mode switching — run each game 3x with different threshold configs (Drools-only, ONNX-only, cascade).
**Alternatives:**
- Instrumented single pass — what-if tracking per tier. Couples test to internal cascade logic.
- Static method reuse — plain JUnit but doesn't exercise real cascade routing.
**Rationale:** Tests the real production code path. 177 total runs (59 games × 3 modes) is fast — pure computation, no I/O. Mode switching via threshold reconfiguration is clean.
**Trade-offs:** 3x execution time. Negligible at ~183 ticks per game.
**Sources:** CascadingPatternClassifier threshold configuration, DroolsScoutingTask wiring
**Exploration:** quick
**Status:** captured

## D7: Scope — include feature extractor rewrite

**Choice:** Include StrategyFeatureExtractor rewrite + race-aware model routing in #213.
**Alternatives:**
- Split into separate issue — cleaner separation but adds coordination overhead.
**Rationale:** Tight feedback loop — wire and validate together. The validation is meaningless without correct feature extraction.
**Trade-offs:** Larger issue scope. Acceptable — the wiring is prerequisite, not optional.
**Sources:** neocortex StrategyClassifierOnnxTest.java (model input shapes), sc2egset_extractor.py (feature definitions)
**Exploration:** quick
**Status:** captured

## D8: Race detection from replays

**Choice:** Plumb enemy race from replay metadata into the cascade. IEM10 games carry matchup strings (PvT, PvZ, PvP) — extract opponent race and pass to CascadingPatternClassifier for per-race model selection. AI Arena replays are all PvP (Protoss opponents).
**Alternatives:**
- Infer race from scouted units — fragile early-game, unnecessary when metadata is available.
**Rationale:** Race is known metadata in replays. In live games, race is always known from the lobby. No inference needed.
**Trade-offs:** None — race is always available in both replay and live contexts.
**Sources:** IEM10JsonSimulatedGame.enumerate() (provides matchup), ScoutingCalibrationTest grouping by matchup
**Exploration:** quick
**Depends on:** D7 (race-aware routing is part of the feature extractor rewrite)
**Status:** captured

## D9: Shared test infrastructure

**Choice:** Extract deriveGroundTruth(), replay loading, and ClassificationResult into a shared ReplayClassificationTestSupport utility class. Both PatternClassificationCalibrationTest and the new CascadeValidationCalibrationTest use it.
**Alternatives:**
- Accept duplication — simpler short-term, diverges over time.
**Rationale:** Ground truth derivation and replay loading are non-trivial logic that must stay consistent across test classes. Extraction prevents drift.
**Trade-offs:** Refactoring existing test class (minor).
**Sources:** PatternClassificationCalibrationTest.java (deriveGroundTruth, classifyGame, ClassificationResult)
**Exploration:** quick
**Depends on:** D2 (new test class consumes the extracted utility)
**Status:** captured
