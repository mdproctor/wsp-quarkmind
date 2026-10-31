# Decisions — issue-405-calibrated-multiplication

## D1: Scope — assess + fix if warranted

**Choice:** Measure classifier impact first; implement calibration in same branch if distortions hurt accuracy
**Alternatives:**
- Assessment only — measure and document, fix in follow-up issue. Slower turnaround.
- Skip to fix — jump straight to implementation. Risks fixing something that doesn't matter for classification.
**Rationale:** vs_zerg classifier accuracy (47-61% fine) correlates with the biggest per-type distortions (Overlord +350, Zergling ±698, Baneling -229). Assessment confirms causation before committing to a fix, but keeping both in one branch avoids round-trip overhead.
**Trade-offs:** If assessment shows distortions don't matter, the branch closes with just measurement data — no wasted implementation work.
**Sources:** oracle-accuracy-baseline.md, cross-version-generalisation.md, issue #405 body
**Exploration:** quick
**Status:** captured

## D2: Fix layer — in-extractor

**Choice:** Change StrippedReplayFeatureExtractor's multiplication logic directly
**Alternatives:**
- Post-processing layer — adjust feature vectors after extraction. More flexible but indirect; oracle comparison tests wouldn't validate it.
- Both layers — in-extractor structural fix + post-processing fine-tuning. Maximum accuracy potential but maximum complexity.
**Rationale:** The oracle baseline tests (OracleAccuracyBaselineTest, DivergenceRegressionTest) compare extractor output directly against ground truth. Fixing inside the extractor means existing tests validate the improvement without new infrastructure.
**Trade-offs:** Coupling the correction to the extractor means re-extraction is required for any factor change (mitigated by configurable factors).
**Sources:** StrippedReplayFeatureExtractor.java:210-226, oracle-accuracy-baseline.md
**Exploration:** quick
**Status:** captured

## D3: Success metric — classifier retrain + compare

**Choice:** Full pipeline: implement fix → re-extract features → retrain classifier → compare per-matchup accuracy
**Alternatives:**
- Feature vector delta analysis — compare vectors against oracle without retraining. Faster but doesn't answer "does classification improve?"
- Oracle divergence only — cheapest, most indirect. Lower divergence doesn't guarantee better classification.
**Rationale:** The question is "do per-type distortions cause classifier misclassification?" Only retraining and comparing answers that directly. The pipeline already exists (reconstitution → training → cross-version validation).
**Trade-offs:** Full pipeline run is slower (~30 min extraction + training). Worth it for a definitive answer.
**Sources:** cross-version-generalisation.md, quarkmind-classifier pipeline
**Exploration:** quick
**Status:** captured

## D4: Calibration approach — per-type calibrated multipliers

**Choice:** Per-type correction factors derived from oracle data, replacing uniform building-count multiplication
**Alternatives:**
- Rate-limiting — infer building usage from temporal command spacing. High risk: 43% CmdEvent detection rate makes temporal reasoning unreliable.
- Hybrid pool (revisited) — pool-based busy-until tracking with modified gating. High risk: #404 showed pool approach dropped accuracy from 88% to 56%.
**Rationale:** Simplest, most directly testable. One lookup table replaces the building-count logic. Each unit type gets its own correction factor. Avoids the temporal reasoning pitfalls that sank pool tracking.
**Trade-offs:** Conflates multiple error sources into one factor (detection rate + building count + timing). May not generalise to very different building counts. Loses the causal model.
**Sources:** StrippedReplayFeatureExtractor.java:143 (MAX_MULTIPLICATION), issue #404 (pool failure), oracle-accuracy-baseline.md T1 divergences
**Exploration:** quick
**Status:** captured

## D5: Factor storage — configurable per dataset

**Choice:** Store calibration factors in a resource file (e.g., calibration-factors.json), loadable per dataset
**Alternatives:**
- Hardcoded constants from oracle — simpler but locked to one dataset. Oracle is representative of pro play; ladder may differ.
- Auto-derived at extraction time — compute on-the-fly. Adds runtime complexity and requires defining "expected" ratios.
**Rationale:** Different datasets (oracle, tournament, ladder) may have different CmdEvent detection characteristics. A configurable file lets factors be re-derived without code changes. The derive-from-oracle workflow produces the file; the extractor consumes it.
**Trade-offs:** Adds a configuration surface (file must exist and be valid). Mitigated by defaulting to current behaviour (uniform multiplication) when no file is present.
**Sources:** issue #405 approaches section, calibration-milestones-are-waypoints protocol
**Exploration:** quick
**Status:** captured
