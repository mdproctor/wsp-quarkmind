# HANDOFF — quarkmind

## Last Session (2026-10-17)

Branch: `issue-412-train-binary-onnx-detectors` — **landed on main** as `f68f47a5`
Paused: `issue-388-train-onnx-reconstituted-ladder` (#388, on pause stack)

### What Was Done

**#412 closed — 23 binary ONNX detectors trained on 1.57M expanded replays.**

Multi-session effort spanning Oct 14-17. Reconstituted 1.57M Blizzard ladder replays (4.9.1 + 4.9.2), trained 23 phase-specific binary classifiers, validated against 148K-replay baseline.

**Pipeline challenges solved:**
- Disk: 58 GB merge output filled 926 GB drive → pivoted to chunk-streaming (no merge step)
- Memory: 52 GB per matchup OOMed on 36 GB RAM → batched loading + per-chunk downsampling
- Speed: 10+ hours sequential → 45 min with parallel workers (4×), NEG_RATIO=2 (2.5×), batch_size=512 (4×)

**Results — 11/12 weak detectors fixed:**

| Detector | Before | After |
|----------|--------|-------|
| `mid_hidden_tech_vs_terran` | 23.8% | 84.8% |
| `late_ground_army_vs_terran` | 5.8% | 74.1% |
| `mid_ground_heavy_vs_protoss` | 0% | 65.7% |
| `mid_ground_heavy_vs_zerg` | 0% | 59.4% |
| `mid_air_tech_vs_protoss` | 25.5% | 79.0% |
| `late_air_superiority_vs_protoss` | 23.8% | 75.8% |

**Still broken:** `mid_ground_heavy_vs_terran` (43.8%→11.1%), `late_ground_army_vs_protoss` (16.5%→10.3%) — filed as #417 (labelling quality, not data volume).

### Landed Commit

```
f68f47a5 feat: train 23 binary ONNX detectors on 1.57M expanded replays (#412)
```

---

## What Comes Next — ONNX Detector Track

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|-----------|------------|-------|
| #414 | **Epic: ONNX Detector Library** | Epic | — | — | Parent epic for #411, #412 (done), #413, #417 |
| #411 | Detector library — phase-gated binary detectors with pluggable selection and aggregation | L | High | #412 (done) | Wire 23 ONNX models into Java game loop; pluggable selection, aggregation, phase gating |
| #413 | Combinatorial gap analysis — detector coverage audit | S | Med | #412 (done) | Audit blind spots across phase transitions and compound strategies |
| #417 | Investigate broken detectors (mid_ground_heavy_vs_terran, late_ground_army_vs_protoss) | S | Med | #412 (done) | Labelling quality investigation — MECH_PUSH/COLOSSUS_PUSH conflicting signals |
| #388 | Train multi-class ONNX classifier on reconstituted ladder data | M | Med | — | Paused on stack; perspectives format fixed, trained on 148K; retrain on expanded data |

---

### Previous Sessions

**Built and launched batch reconstitution pipeline (#412):**

The 1.57M new replays (4.9.2 + 4.9.1) can't be reconstituted to JSON all at once — 233 GB needed, only 36 GB free. Built a batch-and-recycle pipeline:

1. Added `blizzard_ladder_4.9.1` and `blizzard_ladder_4.9.2` datasets to `ReconstitutionExportTest.java`
2. Added batch support: `recon.dataset`, `recon.offset`, `recon.limit`, `recon.skip-existing` system properties
3. Added dataset filter + skip-existing to `DroolsCoarseLabelExportTest.java`
4. New `batch_reconstitute.py` — orchestrates: Java reconstitute 150K → Python extract features → delete JSON → repeat
5. New `merge_training_data.py` — combines chunk .npz outputs into unified dataset
6. New `validate_improvement.py` — baseline vs expanded side-by-side comparison
7. Fixed `prepare_replay_pack.py` to filter `.label.json` sidecars from JSON glob

**Key discovery:** The Java `DroolsCoarseLabelExportTest` labels use different archetype names than the Python pipeline expects. Bypassed this entirely by using `prepare_replay_pack.py --format json` which reads reconstituted JSON and applies its own Python rule-based labelling via `label_replay()`. This unified labelling pipeline replaced the two-step Java reconstitute + Java label approach.

**Pipeline running in background:**

```
PYTHONPATH=quarkmind-classifier .venv/bin/python3 -m src.batch_reconstitute \
  --chunk-size 150000 --workers 8 --dataset blizzard_ladder_4.9.2 --min-disk-gb 20
```

- 2/7 chunks complete when session ended (chunks 0 and 1, 300K of 928K replays)
- Chunk 2 was actively reconstituting (~18K files in)
- Each cycle: ~35 min Java reconstitution + ~1 min Python extraction + cleanup
- Total estimated: ~4-5 hours remaining for 4.9.2

**Also re-processed 4.9.3 reconstituted data:**
- Ran `prepare_replay_pack.py --format json` on existing 148K reconstituted JSONs
- Produced `data/sc2egset/blizzard_ladder_4.9.3/` (283 MB, 958K training samples)
- Uses same label pipeline as new data for consistency

**SC2EGSet download completed (from prior session):**
- 71/71 tournament ZIPs from Zenodo (12 GB, 17,930 replays)
- Located in `data/sc2egset/` alongside tournament-specific directories
- Already processed into .npz — included automatically by merge

**Baseline saved:**
- `output/detectors/detector_manifest_baseline.json` (gitignored) — the 23 detectors trained on 148K replays
- 4 catastrophically broken (0-17% accuracy), 8 very weak (<30% precision)
- All weak detectors suffer from rare archetype underrepresentation

### Commits This Session

```
ab7000c3 wip: add validate_improvement.py for baseline vs expanded comparison
1dcf8a1f wip: fix label.json glob filter, PYTHONPATH, chunk size default
7ece4abb wip: add --max-replays flag and fix merge path resolution
df7238a8 wip: add batch reconstitution pipeline for 1.57M new replays
```

---

## What Needs to Happen Next — Complete Checklist

### Path 1: Check if batch pipeline finished (do first)

```bash
# Check if the background pipeline is still running:
ps aux | grep batch_reconstitute | grep -v grep

# Check completed chunks:
ls -d quarkmind-classifier/data/sc2egset/blizzard_ladder_4.9.2_chunk_* | wc -l
# Expected: 7 chunks if 4.9.2 finished

# Check if reconstituted dir is clean:
ls quarkmind-classifier/data/reconstituted/blizzard_ladder_4.9.2/ 2>/dev/null | wc -l
# Expected: 0 or "No such file" if cleanup completed
```

**If pipeline finished for 4.9.2:** proceed to Path 2a (run 4.9.1).
**If pipeline crashed mid-chunk:** the script auto-skips completed chunks on re-run:
```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.batch_reconstitute \
  --chunk-size 150000 --workers 8 --dataset blizzard_ladder_4.9.2 --min-disk-gb 20
```
**If pipeline still running:** wait. Each chunk takes ~35 min.

### Path 2a: Run batch pipeline for 4.9.1

```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.batch_reconstitute \
  --chunk-size 150000 --workers 8 --dataset blizzard_ladder_4.9.1 --min-disk-gb 20
```

642K replays → 5 chunks → ~3 hours.

### Path 2b: Merge all training data

After both 4.9.2 AND 4.9.1 batches complete:

```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.merge_training_data
```

This merges all `data/sc2egset/blizzard_ladder_*_chunk_*/` directories (new data) plus all existing tournament data in `data/sc2egset/` into `data/full/`.

**Important:** Do NOT use `--include-combined`. The old `data/combined/` used a different labelling pipeline (Java HybridLabeller). The 4.9.3 data was re-processed through `prepare_replay_pack.py` and lives at `data/sc2egset/blizzard_ladder_4.9.3/` with consistent labels.

### Path 3: Retrain binary detectors

```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.train_binary --data full
```

This reads from `data/full/{matchup}/train.npz` and trains all 23 binary detectors. Each gets an ONNX model in `output/detectors/`.

### Path 4: Validate improvement

```bash
cd quarkmind-classifier
PYTHONPATH=. .venv/bin/python3 -m src.validate_improvement
```

Compares `output/detectors/detector_manifest.json` (new) against `output/detectors/detector_manifest_baseline.json` (old, saved this session). Prints side-by-side accuracy, precision, recall with improvement deltas. Highlights weak detectors (precision < 30%).

### Path 5: Close #412 or iterate

If weak detectors improved significantly → close #412, proceed to detector library (#411, #414).
If still weak → investigate: are the archetypes genuinely rare even in 1.7M replays? Consider targeted data acquisition (AI Arena bots for unconventional strategies).

---

## Architecture Notes for Next Session

**Pipeline flow:**
```
.SC2Replay (stripped, no tracker events)
  → Java ReconstitutionExportTest (StrippedReplayFeatureExtractor)
  → reconstituted JSON (synthetic tracker events, ~148 KB/file)
  → Python prepare_replay_pack.py --format json (rule-based labelling + feature extraction)
  → .npz training data (~283 MB per 150K replays)
  → train_binary.py → ONNX detectors
```

**Key scripts (all in quarkmind-classifier/src/):**
| Script | Purpose |
|--------|---------|
| `batch_reconstitute.py` | Orchestrates batch-and-recycle pipeline |
| `merge_training_data.py` | Combines chunk .npz into unified dataset |
| `prepare_replay_pack.py` | .SC2Replay or JSON → labelled .npz (labels via `label_replay()`) |
| `train_binary.py` | Trains binary ONNX detectors from .npz |
| `validate_improvement.py` | Baseline vs expanded comparison |
| `compare_detectors.py` | Binary vs multi-class accuracy comparison |
| `binary_catalog.py` | 9 detector definitions (early/mid/late phases) |

**Disk space:** ~31 GB free. Each 150K-replay chunk uses ~22 GB temporarily (reconstituted JSON) then reclaims after cleanup. The .npz outputs accumulate (~283 MB each, 12 chunks = ~3.4 GB total).

**Timing benchmarks:**
| Step | Speed | For 150K replays |
|------|-------|-----------------|
| Java reconstitution | 74 replays/sec | ~34 min |
| Python feature extraction | 2,293 replays/sec | ~65 sec |
| Maven startup overhead | — | ~90 sec per chunk |

---

## Strategic Overview — All Work Directions

(Unchanged from previous session — see sections below)

### 1. ONNX Strategy Detection (active)

| Stage | Status | Next step |
|-------|--------|-----------|
| **Data acquisition** | 1.72M new replays on disk + SC2EGSet (17.9K tournament) | Done |
| **Reconstitution** | Batch pipeline built, 2/7 chunks of 4.9.2 done | Complete 4.9.2, then 4.9.1 |
| **Multi-class training** (#388) | 87.8% vs_protoss, 84% vs_zerg, 69% vs_terran on 148K | Retrain on expanded dataset |
| **Binary detectors** (#412) | 23 detectors, 4 broken, 8 weak | Retrain → validate |
| **Detector library** (#411, #414) | Architecture designed | Build after detectors validated |
| **Gap analysis** (#413) | Not started | After expanded detectors trained |

### 2-5. Other workstreams

Unchanged. See previous HANDOFF for full details on Emulated Game Physics, AI-vs-AI Training Loop, Framework Generalisation, and Coaching & Strategy.

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-412-train-binary-onnx-detectors` | #412 | Batch pipeline running, 2/7 chunks done | `work continue` |
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |

## Reference

### Blizzard API credentials

OAuth client "sc2-replay-download" on https://develop.battle.net/access/clients (login as Poldy). Client ID: `159d5558be314bc9b7e0db0a47cf19c7`. Secret visible after login.

### Additional data sources (not yet tapped)

- **AI Arena:** auth token from aiarena.net/profile/token/. Good for rare archetypes.
- **Blizzard 4.8.6/4.8.3/4.7.1:** ~700K more replays, need disk space (~40 GB each).
- **Excluded 4.10.1:** 2,837 replays, needs ability profile bridging.

### Reconstitution constraint

Blizzard ladder replays have NO tracker events — `StrippedReplayFeatureExtractor` reconstructs them from CmdEvents. Only `baseBuild ≤ 75689` (≤ 4.10) works. `ABILITY_PROFILE_BOUNDARY = 75689` in `reconstitution_pipeline.py`.
