# HANDOFF — quarkmind

## Last Session

Completed Task 9 (FeatureAlignmentTest) and started Task 10 for #306. 9 of 10 tasks done.

### Task 9 — FeatureAlignmentTest (complete)

Found and fixed 3 Python extractor bugs via the alignment test:
1. **UnitBorn buildings uncounted**: Starting bases missed because UnitBorn handler didn't track BUILDING_IDX
2. **UnitInit+UnitDone double-counting**: Building counts incremented on both events — moved to UnitInit only
3. **supply_costs.json path wrong**: Default path missing `quarkmind/` directory — fell back to hardcoded defaults

Also fixed a circular import (`sc2egset_extractor` ↔ `feature_engineering`) by making the `feature_engineering` import lazy inside `build_samples_from_replays()`.

Java changes: added `MapInfo` field to `SimulatedGame`, populated it in `IEM10JsonSimulatedGame` (start positions from first base UnitBorn events, map dimensions from metadata), and added enemy building tracking for UnitBorn and UnitInit events.

### Task 10 — Train + Calibrate (in progress)

**Written:** `OnnxClassificationCalibrationTest.java` — compiles, awaiting retrained models.

**Blocker:** Another session switched neocortex from `issue-345-goal-cognition` to `main` during our background extraction. The extraction ran with old code (269 features) for 70/71 tournaments. Only IEM10 Taipei has correct 298-feature data. **All sc2egset data needs re-extraction from the correct branch.**

## What Task 10 Needs to Do

The ONNX strategy classifier feature vector was enriched from 269→298 per window (Tasks 1-9). Task 10 retrains the three per-matchup ONNX models (vs_terran, vs_zerg, vs_protoss) with the enriched features and adds a calibration test.

The pipeline has 6 steps, all run from neocortex with the venv:

### Step 0 — Switch neocortex to the correct branch (CRITICAL)

```bash
git -C /Users/mdproctor/claude/casehub/neocortex checkout issue-345-goal-cognition
```

Verify the 298-feature extractor code is present:
```bash
git -C /Users/mdproctor/claude/casehub/neocortex log --oneline -5
# Must show: 9261ff9e fix: sc2egset_extractor building count bugs and supply cost path
```

If this commit is missing, the extraction will produce 269-feature data and everything downstream breaks silently.

### Step 1 — Re-extract SC2EGSet data (long-running)

Re-extracts all 71 tournament ZIPs with the 298-feature pipeline. Each ZIP takes 5-30 seconds depending on size. Total time: ~10-30 minutes.

```bash
PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
/Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
-m evaluation.strategy_classifier.prepare_real_data \
--zips /Users/mdproctor/claude/casehub/neocortex/evaluation/strategy_classifier/data/sc2egset/raw/*.zip --force
```

**Verify output is 298 features:**
```bash
python3 -c "import numpy as np; d=np.load('evaluation/strategy_classifier/data/sc2egset/2016_IEM_10_Taipei/vs_terran/train.npz'); print(d['temporal'].shape)"
# Must show: (N, 10, 298) — if it shows 269, wrong branch
```

### Step 2 — Normalize

Computes mean/std across all training data (Welford's online algorithm). Writes `norm_stats.npz` in the combined output directory.

```bash
PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
/Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
-m evaluation.strategy_classifier.normalize --sources sc2egset
```

### Step 3 — Train models

Trains 3 models (vs_terran, vs_zerg, vs_protoss), calibrates temperature, exports to ONNX.

```bash
PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex \
/Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3 \
-m evaluation.strategy_classifier.run_pipeline --data combined
```

Output: `evaluation/strategy_classifier/output/strategy_vs_{terran,zerg,protoss}.onnx`

### Step 4 — Deploy to quarkmind

Copy ONNX models:
```bash
cp neocortex/evaluation/strategy_classifier/output/strategy_vs_*.onnx \
   quarkmind/quarkmind-sc2/src/test/resources/models/strategy/
```

Convert and copy norm_stats (NPZ → JSON):
```python
import numpy as np, json
d = np.load("evaluation/strategy_classifier/data/combined/norm_stats.npz")
json.dump({"mean": d["mean"].tolist(), "std": d["std"].tolist()}, open("norm_stats.json", "w"))
# Copy to quarkmind-sc2/src/main/resources/classifier/norm_stats.json
```

### Step 5 — Run calibration test

```bash
mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=OnnxClassificationCalibrationTest
```

Success criteria: >= 70% rush accuracy at minute 3. Also re-run `FeatureAlignmentTest` to confirm norm_stats didn't break anything.

### Step 6 — Commit and close

Commit ONNX models, norm_stats.json, and OnnxClassificationCalibrationTest results. Then `work end` to close the branch.

## Python Environment

Venv: `/Users/mdproctor/claude/casehub/neocortex/.venv` (created with `--system-site-packages`)
- torch 2.14.0, onnx 1.22.0, onnxruntime 1.27.0, scikit-learn 1.9.1
- Python 3.14 (Homebrew)
- Always prefix commands with: `PYTHONPATH=/Users/mdproctor/claude/casehub/neocortex /Users/mdproctor/claude/casehub/neocortex/.venv/bin/python3`

## Cross-Module

Neocortex branch `issue-345-goal-cognition` has 5 commits with all Python pipeline changes:
- `5b2b10ce` feat: sc2egset_extractor position tracking + spatial + ratio features
- `e45620b4` feat: feature_engineering two-phase window assembly (269→298)
- `7c84d40a` feat: normalize vis_idx fix + modality dropout for army_gap/has_vision
- `9261ff9e` fix: sc2egset_extractor building count bugs and supply cost path
- Plus one more for lazy import fix

These commits are NOT on main. **The branch must be checked out before any Python pipeline operation.**

## References

- Design spec: `specs/issue-306-retrain-onnx-spatial/2026-09-23-onnx-feature-enrichment-design.md`
- Plan: `plans/2026-09-23-onnx-feature-enrichment.md`
- Garden: GE-20260923-3afc13 (binary fog-of-war masking for spatial features)
- Neocortex branch: `issue-345-goal-cognition`
