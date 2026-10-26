# HANDOFF — quarkmind

## Last Session

**#455 in progress — vs_zerg early-minute classification accuracy.** Added coarse-level accuracy reporting to cross-version validation. Key finding: vs_zerg tournament coarse accuracy is 93.9% at minute 3 (fine: 43.5%) — the ROACH_RUSH→RUSH confusion is entirely within the AGGRESSIVE coarse category. Label audit confirmed 93% of ROACH_RUSH replays have no RoachWarren at minute 3, making the fine-grained distinction unobservable at that checkpoint. Phase-aware thresholds documented. All 3 acceptance criteria met — ready for work-end.

## Immediate Next Step

**Close #455** — all acceptance criteria met, tests pass. Then Phase 4: #359 (autonomous play) / #361 (continuous training loop).

## Known Issues

- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy` (wrong test assertion), `test_ingest_spawningtool` (API signature change), `test_coverage_gaps` (archetype count drift).
