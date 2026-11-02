# HANDOFF — quarkmind

## Last Session

Closed three children of epic #481 (EmulatedGame calibration). #493 added UnitDied tracker event processing — dead units now removed from PlayerState with supply deduction (97% tag-match rate, foodUsed error 52%→35%). #494 separated economy gate logic from resource deduction — tracker-driven intents now deduct mineral/vespene costs (mineralsCurrent error 654%→4%). #497 added per-field economy regression thresholds to DivergenceRegressionTest, replacing the single 420% MAPE ceiling with 5 individual field ceilings calibrated from post-fix baselines.

## Immediate Next Step

#496 (calibrate mining income model against oracle replays) — M/Med, the next calibration issue under #481. Collection rate errors are ~93% because EmulatedGame uses a flat mining rate vs SC2's saturation-based model.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export. Needs clean re-export before retraining.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — 928K replays, needs re-export when disk space permits.
- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).

## References

- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — updated accuracy numbers
- `blog/2026-11-02-mdp02-teaching-the-emulator-about-death.md` — diary entry covering #493/#494
