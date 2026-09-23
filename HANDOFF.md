# HANDOFF — quarkmind

## Last Session

Completed Task 9 (FeatureAlignmentTest) for #306 — the alignment test found and fixed three bugs in the Python extractor (sc2egset_extractor.py):

1. **UnitBorn buildings uncounted**: Starting bases (Nexus/CC/Hatchery) arrive via UnitBorn, but the extractor only counted buildings from UnitInit events. Starting buildings were invisible in the feature vector.
2. **UnitInit+UnitDone double-counting**: Building counts incremented on both UnitInit and UnitDone events — every constructed building was counted twice.
3. **supply_costs.json path wrong**: Default path was `casehub/quarkmind-sc2/...` instead of `casehub/quarkmind/quarkmind-sc2/...`. The extractor fell back to hardcoded defaults (most units = 2 supply), causing a 2x army_supply_ratio divergence.

Also made two Java changes to enable alignment testing:
- Added `MapInfo` to `SimulatedGame` (protected field) and `IEM10JsonSimulatedGame` (extracts start positions from first base UnitBorn events, map dimensions from metadata)
- Added enemy building tracking to `IEM10JsonSimulatedGame` for UnitBorn and UnitInit events

Fixed a circular import in neocortex (`sc2egset_extractor` ↔ `feature_engineering`) by making the `feature_engineering` import lazy inside `build_samples_from_replays()`.

Player features match within 1e-3 tolerance. Opponent features have expected timing differences (logged but not asserted) due to the ~6 game-loop sampling offset between Java ticks and Python seconds.

## Immediate Next Step

Task 10: Train models and OnnxClassificationCalibrationTest. Requires Python environment with torch/ONNX. Steps:
1. Run Python test suite (`pytest`) in neocortex to verify the extractor fixes
2. Regenerate training data with enriched features (`sc2egset_extractor --zip`)
3. Run `normalize.py` to compute new norm_stats
4. Train all 3 matchup models via `run_pipeline.py`
5. Copy ONNX models + norm_stats.json to quarkmind-sc2 test resources
6. Write `OnnxClassificationCalibrationTest`

## Cross-Module

Neocortex repo: 4 commits on branch `issue-345-goal-cognition` (Tasks 6-8 from previous session + extractor bugfixes from this session). These need verification with `pytest` before training.

## References

- Design spec: `specs/issue-306-retrain-onnx-spatial/2026-09-23-onnx-feature-enrichment-design.md`
- Decisions: `specs/issue-306-retrain-onnx-spatial/decisions.md`
- Plan: `plans/2026-09-23-onnx-feature-enrichment.md`
- Garden: GE-20260923-3afc13 (binary fog-of-war masking for spatial features)
- Neocortex branch: `issue-345-goal-cognition` (Python pipeline changes)
