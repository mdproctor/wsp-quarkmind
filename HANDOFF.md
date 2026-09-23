# HANDOFF — quarkmind

## Last Session

Completed Task 9 (FeatureAlignmentTest) and partially completed Task 10 (OnnxClassificationCalibrationTest) for #306.

### Task 9 — FeatureAlignmentTest (complete)

Found and fixed 3 Python extractor bugs via the alignment test:
1. **UnitBorn buildings uncounted**: Starting bases missed because UnitBorn handler didn't track BUILDING_IDX
2. **UnitInit+UnitDone double-counting**: Building counts incremented on both events
3. **supply_costs.json path wrong**: Default path was missing the `quarkmind/` directory; fell back to hardcoded defaults (most units = 2 supply)

Also fixed a circular import (`sc2egset_extractor` ↔ `feature_engineering`) — made the `feature_engineering` import lazy inside `build_samples_from_replays()`.

Java changes: added MapInfo to `SimulatedGame`/`IEM10JsonSimulatedGame` and enemy building tracking for UnitBorn/UnitInit.

### Task 10 — Train + Calibrate (in progress)

**Written:** `OnnxClassificationCalibrationTest.java` — compiles, awaiting retrained models.

**Background process running:** `prepare_real_data.py --force` is re-extracting all 71 SC2EGSet ZIPs with 298 features. PID 9806 (check with `ps aux | grep prepare_real_data`). IEM10 Taipei completed with 298 features; remaining 70 ZIPs are processing.

**Python venv:** `/Users/mdproctor/claude/casehub/neocortex/.venv` with `--system-site-packages`. Has torch 2.14.0, onnx 1.22.0, onnxruntime 1.27.0, scikit-learn 1.9.1.

## Immediate Next Step

When `prepare_real_data.py` finishes (check `ps aux | grep prepare_real_data`):

1. **Normalize:**
   ```bash
   PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
   /Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
   -m evaluation.strategy_classifier.normalize --sources sc2egset
   ```

2. **Train:**
   ```bash
   PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
   /Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
   -m evaluation.strategy_classifier.run_pipeline --data combined
   ```

3. **Deploy models:** Copy ONNX files from `neocortex/evaluation/strategy_classifier/output/` to `quarkmind-sc2/src/test/resources/models/strategy/`

4. **Convert norm_stats:** Copy the new `norm_stats.json` to `quarkmind-sc2/src/main/resources/classifier/norm_stats.json`

5. **Run calibration test:**
   ```bash
   mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=OnnxClassificationCalibrationTest
   ```

6. **Run alignment test** to confirm the new norm_stats still works:
   ```bash
   mvn test -pl quarkmind-sc2 -Dtest=FeatureAlignmentTest
   ```

## Cross-Module

Neocortex repo: 5 commits on branch `issue-345-goal-cognition` (Tasks 6-8 from previous session + extractor bugfixes + circular import fix). Background extraction is overwriting per-tournament data in `data/sc2egset/`.

## References

- Design spec: `specs/issue-306-retrain-onnx-spatial/2026-09-23-onnx-feature-enrichment-design.md`
- Decisions: `specs/issue-306-retrain-onnx-spatial/decisions.md`
- Plan: `plans/2026-09-23-onnx-feature-enrichment.md`
- Garden: GE-20260923-3afc13 (binary fog-of-war masking for spatial features)
- Neocortex branch: `issue-345-goal-cognition` (Python pipeline changes)
