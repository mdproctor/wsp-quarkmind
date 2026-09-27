# HANDOFF — quarkmind

## Last Session

Closed #312 (ONNX class imbalance), #313 (ground truth alignment), #314 (eval mode export). Branch `issue-312-onnx-class-imbalance` landed on main as 2 squashed commits.

### Key change: quarkmind-classifier module

Moved the entire SC2 strategy classifier training pipeline from `neocortex/evaluation/strategy_classifier/` to `quarkmind/quarkmind-classifier/`. Zero neocortex dependencies — the pipeline was SC2-domain code misplaced in the ML repo. This eliminates cross-repo coordination for all future training work.

- Flat layout: `src/`, `tests/`, `docker/sc2-restore/`
- Imports rewritten: `evaluation.strategy_classifier.X` → `src.X`
- Config paths: relative from module root (CWD = `quarkmind-classifier/`)
- Own `pyproject.toml` and `.venv`

### Training improvements applied

- `train.py`: class weight cap 5.0 → 15.0
- `normalize.py`: oversampling floor 50 → 500 samples/class
- `run_pipeline.py`: `export_model.eval()` before ONNX export

### Ground truth fix (#313)

Terran rush threshold tightened: `marines >= 5 && < 4min` → `marines >= 8 && < 3min`. Standard bio openings no longer misclassified as rushes. Calibration gate raised 40% → 60%.

### Neocortex cleanup

`evaluation/strategy_classifier/` removed from neocortex git tracking. Committed to neocortex main (not pushed). ONNX dependency retained — used by `inference-runtime` module.

### Data incident

`rm -rf` accidentally deleted intermediate `sc2egset/` per-tournament NPZ files during the move. `combined/` training data (152M) and `replay_packs/` (13G raw replays) survived. Issue #316 tracks regeneration of intermediates. Current ONNX models are unaffected.

## What's Next

| Item | Scale | Complexity | Notes |
|------|-------|------------|-------|
| #316 — Regenerate sc2egset intermediates | S | Low | Run `prepare_real_data.py` against replay_packs/. Needed before re-normalization. |
| Acquire rare archetype training data | M | Med | SC2ReplayStats ladder data or Blizzard ladder replay restoration. Targets TECH_RUSH and AIR_SUPERIORITY gaps. |
| Push neocortex cleanup | XS | Low | `git -C neocortex push origin main` — classifier removal commit is local only. |

## Training Pipeline (new location)

All commands run from `quarkmind-classifier/`:

```bash
# Setup
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'

# Test
PYTHONPATH=. .venv/bin/python3 -m pytest tests/ -v

# Train (requires data in data/combined/)
PYTHONPATH=. .venv/bin/python3 -m src.run_pipeline --data combined
```

## References

- Design spec: `specs/issue-312-onnx-class-imbalance/2026-09-27-move-classifier-to-quarkmind-design.md`
- Plan: `plans/2026-09-27-move-classifier-to-quarkmind.md`
- Issue #316: regenerate sc2egset intermediates
- Standing rule: never `rm -rf` data directories — always `mv`
