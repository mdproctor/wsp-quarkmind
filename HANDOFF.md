# HANDOFF — quarkmind

## Last Session (2026-10-14)

Branch: `issue-412-train-binary-onnx-detectors` (#412)
Paused: `issue-388-train-onnx-reconstituted-ladder` (#388, on pause stack)

### What Shipped (#412)

**23 binary ONNX detectors** trained and exported — phase-specific (early/mid/late) binary classifiers answering yes/no questions about opponent strategy. Average 87.7% accuracy across all detectors.

New files:
- `quarkmind-classifier/src/binary_catalog.py` — 9 detector definitions (early_rush, early_cheese, early_greedy, mid_air_tech, mid_hidden_tech, mid_ground_heavy, mid_timing_attack, late_air_superiority, late_ground_army) across 3 matchups
- `quarkmind-classifier/src/train_binary.py` — training pipeline with per-matchup data caching and minute inference from temporal padding
- `quarkmind-classifier/src/compare_detectors.py` — binary vs multi-class comparison
- `quarkmind-classifier/tests/test_binary_catalog.py` (8 tests), `tests/test_train_binary.py` (5 tests)
- ONNX files in `output/detectors/` (gitignored) with `detector_manifest.json`

### Key Findings

- Multi-class model generally outperforms binary detectors on accuracy (benefits from 4x more training data across all minutes)
- Binary detectors achieve higher recall for common patterns: cannon rush 97.2% vs 94.4%, ling-bane 86.7% vs 70.2%
- Weak detectors: `late_ground_army_vs_terran` (17.4% acc — catastrophic), several with 0% precision on rare classes at specific minutes
- Root cause of weak detectors: insufficient positive samples at specific minutes

### Data Acquisition — In Progress

**Downloads running in background (may have completed or timed out):**

1. **Blizzard 4.9.2 ladder replays** — downloading 222 packs (~350 MB each) to `data/replay_packs/blizzard_ladder/4.9.2/`. Script: `/tmp/download_blizzard_replays.py`. ZIPs extracted to `replays/` subdirectory, password `iagreetotheeula`. Check completion and clean up ZIPs.

2. **SC2EGSet from Zenodo** — downloading 71 tournament ZIPs (~12.5 GB) to `data/sc2egset/`. Zenodo is slow. After download, process via `sc2egset_extractor.py` / `prepare_real_data.py`.

**Blizzard API credentials (OAuth client "sc2-replay-download"):**
- Client ID: `159d5558be314bc9b7e0db0a47cf19c7`
- Secret: visible at https://develop.battle.net/access/clients (login required)
- Download script pattern: `python3 /tmp/download_blizzard_replays.py --key <key> --secret <secret> --version <ver> --output-dir <dir>`

**Available Blizzard versions (baseBuild ≤ 75689, reconstitutable):**

| Version | Packs | Est. replays | Status |
|---------|-------|-------------|--------|
| 4.9.3 | ~300 | 148K | Done |
| 4.9.2 | 222 | **928,214** | **Done** (65 GB in replays/, ZIPs deleted) |
| 4.9.1 | 210 | ~150K | Downloading (auto-deletes ZIPs) |
| 4.8.6 | 314 | ~200K | Queued |
| 4.8.3 | 454 | ~250K | Queued |
| 4.7.1 | 474 | ~250K | Queued |

**Disk space:** 158 GB free at session start. Each version is ~35-40 GB extracted (delete ZIPs after extraction). Download one at a time.

**Other data sources:**
- AI Arena: needs auth token from aiarena.net/profile/token/. Same patch 4.9.3, good for rare archetypes.
- SC2ReplayStats: needs 10 SC2 games played + uploaded to unlock. Parked.
- Excluded 4.10.1 (2,837 replays on disk): needs ability profile bridging. Parked.

### Reconstitution Constraint

Blizzard ladder replays have NO tracker events — they use the stripped fallback extractor (`StrippedReplayFeatureExtractor`), calibrated on 4.9.3. Only patches with `baseBuild ≤ 75689` (≤ 4.10) work without recalibration. `ABILITY_PROFILE_BOUNDARY = 75689` in `reconstitution_pipeline.py`.

---

## Next Session

### Immediate: Complete data acquisition

1. Check if 4.9.2 and SC2EGSet downloads completed
2. Clean up 4.9.2 ZIPs (if script didn't auto-delete — old version ran first)
3. Download 4.9.1 and 4.8.6 (one at a time, delete ZIPs after extraction)
4. Get AI Arena auth token, bulk-download bot replays
5. Process SC2EGSet through `prepare_real_data.py`

### Then: Reconstitute and retrain

6. Run reconstitution pipeline on all new replay data
7. Retrain binary detectors with expanded dataset
8. Focus on improving weak detectors (rare archetypes at specific minutes)
9. Update comparison table

### Open Issues

| # | Title | Scale | Status |
|---|-------|-------|--------|
| 412 | Train binary ONNX detectors | M | Active — detectors trained, expanding data |
| 388 | Train ONNX on reconstituted data | M | Paused (on stack) |
| 411 | ONNX detector library architecture | L | After #412 |
| 407 | Exact replay reconstruction via building routing | M | Deprioritised |
| 408 | Promote situational awareness loop to generic core | M | Depends on #409 |
| 409 | Bidirectional alignment: SC2 ↔ quarkmind-core | L | Architectural |
