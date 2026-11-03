# HANDOFF — quarkmind

## Last Session

Closed #499: gas worker assignment was missing from replay building injection/sync paths, causing 98% vespene collection rate error in the CmdEvent path. Fix added `onGasBuildingCompleted` calls to `injectReplayBuilding`, `injectReplayBuildingWithCost`, and `syncReplayBuilding`, with position-based dedup in `WorkerAssignment` to prevent double-assignment. Error dropped to 34%. Filed #500 for individual worker simulation — the remaining 34% is architectural (aggregate formula vs per-worker tracking).

## Immediate Next Step

#500 — design and implement individual worker simulation in EmulatedGame. Each worker tracks its task (MINING_MINERALS, HARVESTING_GAS, IDLE, BUILDING). Income emerges from simulation; existing formulas become validators.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export. Needs clean re-export before retraining.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — 928K replays, needs re-export when disk space permits.
- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).
- **Pre-existing:** Economy MAPE ceiling (150%) exceeded at 211% — predates #499, not a regression.

## References

- `blog/2026-11-03-mdp01-when-the-formula-is-the-bug.md` — diary entry covering the diagnosis and design insight
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — updated accuracy numbers
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerAssignment.java` — position dedup added
