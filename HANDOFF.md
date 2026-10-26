# HANDOFF — quarkmind

## Last Session

Two issues closed:

**#455 closed — vs_zerg early-minute classification accuracy.** Added coarse-level accuracy to cross-version validation. vs_zerg tournament coarse accuracy is 93.9% at minute 3 (fine: 43.5%). Label audit confirmed 93% of ROACH_RUSH replays have no RoachWarren at minute 3. Phase-aware thresholds documented. Blog: "Your Classifier Isn't Broken — Your Labels Are" with SVG charts.

**#454 closed — detector output distribution diagnostic.** Ran all 23 binary detectors over training data. 13/23 bimodal, 8/23 have MEDIUM < 5%. Recommendation: per-detector discretization (binary for bimodal, 3-bucket for spread). Findings posted to #450. Blog: "Not All Detectors Are Binary" with SVG chart.

## Immediate Next Step

**#456 (SC2 replay pipeline: scouting observations for hypothesis prior learning)** — cross-repo with neocortex#705. Or **#403 (smoke test playbook suite)** for independent M/Low work.

## Known Issues

- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).
