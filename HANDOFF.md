# HANDOFF — quarkmind

## Last Session

**#405 closed — calibrated multiplication assessment.** Assessed per-type calibration factors for StrippedReplayFeatureExtractor. Found and fixed MULE calldown bug (MuleCalldownIntent silently dropped — gap reduced from -623 to -148). Calibration factors degrade aggregate T1 accuracy (99.3% → 96.2%) because compensating errors are structural: SCV over-count (+862) balances MULE (-623) + Drone (-244) under-counts. Calibration infrastructure retained but dormant. Concluded that the right path is fixing emulator divergences, not extraction heuristics.

**#484 created — targeted emulator calibration.** Use real SC2 as oracle on strategic replay subsets to find and fix specific emulator physics gaps (Marine 98.1%, WidowMine 87.9%, Hellion 94.9% at 5min). Also expand complete dataset via SC2EGSet (17,930 replays on Zenodo) to reduce dependence on stripped replay reconstitution.

**Disk cleanup:** Deleted slots attic (57 GB), venvs, node_modules. Classifier venv needs recreating. Partially-written blizzard_ladder_4.9.2 reconstituted data deleted (94 GB corrupt partial export). blizzard_ladder_4.9.3 reconstituted data has mixed timestamps (partial calibrated overwrite from Oct 31) — needs clean re-export.

## Immediate Next Step

**#484** — targeted emulator calibration via SC2 oracle comparison. Or expand complete replay dataset via SC2EGSet download first.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export. Needs clean re-export before retraining.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — 928K replays, needs re-export when disk space permits.
- **Classifier venv deleted** for disk space — recreate with `cd quarkmind-classifier && python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'`
- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).
