# HANDOFF — quarkmind

## Last Session

Completed Task 9 (FeatureAlignmentTest) and partially completed Task 10 for #306.

### Task 9 — FeatureAlignmentTest (complete)

Found and fixed 3 Python extractor bugs via the alignment test:
1. **UnitBorn buildings uncounted**: Starting bases missed because UnitBorn handler didn't track BUILDING_IDX
2. **UnitInit+UnitDone double-counting**: Building counts incremented on both events
3. **supply_costs.json path wrong**: Default path missing `quarkmind/` directory

Also fixed a circular import (`sc2egset_extractor` ↔ `feature_engineering`).

Java changes: added MapInfo to `SimulatedGame`/`IEM10JsonSimulatedGame` and enemy building tracking.

### Task 10 — Train + Calibrate (blocked)

**Written:** `OnnxClassificationCalibrationTest.java` — compiles, awaiting retrained models.

**Problem:** Another session switched neocortex from `issue-345-goal-cognition` to `main` during our background extraction. The extraction ran with old code (269 features) for 70/71 tournaments. Only IEM10 Taipei has 298-feature data. **All sc2egset data needs re-extraction.**

## Immediate Next Step

**CRITICAL: switch neocortex back to the feature branch first.**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex checkout issue-345-goal-cognition
```

Verify our commits are present:
```bash
git -C /Users/mdproctor/claude/casehub/neocortex log --oneline -5
# Should show: 9261ff9e fix: sc2egset_extractor building count bugs and supply cost path
```

Then re-run the full pipeline:

1. **Re-extract all 71 ZIPs:**
   ```bash
   PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
   /Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
   -m evaluation.strategy_classifier.prepare_real_data \
   --zips /Users/mdproctor/claude/casehub/neocortex/evaluation/strategy_classifier/data/sc2egset/raw/*.zip --force
   ```
   (~5-10 min per ZIP, ~6 hours total. Consider running in background.)

2. **Normalize:** `python3 -m evaluation.strategy_classifier.normalize --sources sc2egset`

3. **Train:** `python3 -m evaluation.strategy_classifier.run_pipeline --data combined`

4. **Deploy models** to `quarkmind-sc2/src/test/resources/models/strategy/`

5. **Deploy norm_stats.json** to `quarkmind-sc2/src/main/resources/classifier/`

6. **Run calibration:** `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=OnnxClassificationCalibrationTest`

**Python venv:** `/Users/mdproctor/claude/casehub/neocortex/.venv` (system-site-packages). Has torch, onnx, onnxruntime, scikit-learn. But **must be on `issue-345-goal-cognition` branch** for the 298-feature extractor code.

## Cross-Module

Neocortex branch `issue-345-goal-cognition` has 5 commits with all Python pipeline changes (Tasks 6-8 + extractor bugfixes). These commits are NOT on main. Commit `9261ff9e` is the latest fix.

## References

- Design spec: `specs/issue-306-retrain-onnx-spatial/2026-09-23-onnx-feature-enrichment-design.md`
- Plan: `plans/2026-09-23-onnx-feature-enrichment.md`
- Garden: GE-20260923-3afc13 (binary fog-of-war masking for spatial features)
- Neocortex branch: `issue-345-goal-cognition` (Python pipeline changes — NOT on main)
