# HANDOFF — quarkmind

## Last Session

**#484 closed — targeted emulator calibration.** Built diagnostic infrastructure (IntentExecutionListener, ReplayDivergenceScanner, ReplayDivergenceTracer) and applied a rerouting fix for wrong_building_type rejections. Accuracy improved from 98.8% to 99.1% at 5min (+22 units). Discovered baseline was stale (98.8%, not 99.8% as documented) — three EmulatedGame changes (#401, #479, #480) since Oct 13 regressed accuracy undetected because report tests are excluded from the default suite.

**Key finding:** 354 `wrong_building_type` rejections across 236 player runs from TrackerIntentExtractor building tag resolution issues. The rerouting fix rescued cases where a correct building type exists. Remaining gap requires fixing `lookupBuildingTag()` in TrackerIntentExtractor — filed as #485.

**#485 created — TrackerIntentExtractor building tag resolution.** Fix `lookupBuildingTag()` to reduce wrong_building_type rejections. Target ≥99.5% at 5min. L/High.

**SC2EGSet confirmed complete:** all 71 tournament ZIPs (2016-2024, ~17,930 replays) already downloaded and processed (234 NPZ files). Updated replay index with Dataset 8 summary.

## Immediate Next Step

**#485** — TrackerIntentExtractor building tag resolution. Use the diagnostic tools from #484 (scanner, tracer, listener) to identify and fix the remaining building tag misresolutions.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export. Needs clean re-export before retraining.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — 928K replays, needs re-export when disk space permits.
- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).
