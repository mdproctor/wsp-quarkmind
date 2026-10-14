# HANDOFF — quarkmind

## Last Session (2026-10-14)

Branch: `issue-412-train-binary-onnx-detectors` (#412, on pause stack)
Paused: `issue-388-train-onnx-reconstituted-ladder` (#388, on pause stack)

### What Shipped

**Binary ONNX detectors (#412):**
- 23 binary detectors trained across 3 matchups, 3 game phases (early/mid/late)
- Average 87.7% accuracy. Best: early_cheese_vs_protoss (97.3%), early_greedy_vs_terran (98.7%)
- Weak detectors identified: rare archetypes at specific minutes lack positive samples
- New files: `binary_catalog.py`, `train_binary.py`, `compare_detectors.py`, tests
- ONNX files in `output/detectors/` (gitignored) with `detector_manifest.json`

**CI fixed (landed on main):**
- Added `drools-wiring-dynamic` dependency — resolved ~130 Drools rule unit test errors
- Updated 14 test files for stale feature counts (UpgradeType 15→82), morph events, ability profiles, round-trip mappings
- Fixed `QuarkusSC2TransportTest` race condition (async shutdown timing on slow CI)
- Tracked webui `package-lock.json` for `npm ci` in CI
- Excluded `quarkmind-chat` from default build (unpublished casehub-blocks SNAPSHOTs) — available via `-Pchat` profile
- Merged and closed PR #334 (dependency management for platform 0.2-SNAPSHOT)

**Data acquired (1.72M new replays):**
- Blizzard 4.9.2: **928,214 replays** (65 GB, ZIPs deleted)
- Blizzard 4.9.1: **641,922 replays** (~45 GB, ZIPs auto-deleted)
- SC2EGSet: download started from Zenodo (71 ZIPs, 12.5 GB) — may still be running

### Issues Resolved This Session

- #364 (Regenerate FeatureAlignmentTest fixture) — done as part of CI fix, can close
- #316 (Regenerate sc2egset intermediate data) — SC2EGSet download started, close when processed

---

## Strategic Overview — All Work Directions

QuarkMind has 30 open issues across 5 distinct workstreams. Here's the map:

### 1. ONNX Strategy Detection (active — the current focus)

The pipeline: raw replays → reconstitution → feature extraction → labelling → training → ONNX export → runtime integration.

| Stage | Status | Next step |
|-------|--------|-----------|
| **Data acquisition** | 1.72M new replays downloaded (4.9.2 + 4.9.1) | Reconstitute the new data |
| **Reconstitution** | Pipeline works for baseBuild ≤ 75689 | Run `reconstitution_pipeline.py` on new replay packs |
| **Multi-class training** (#388) | 87.8% vs_protoss, 84% vs_zerg, 69% vs_terran on 148K replays | Retrain on expanded dataset (~1.7M replays) |
| **Binary detectors** (#412) | 23 detectors trained, 87.7% avg accuracy | Retrain with expanded data — weak detectors should improve |
| **Detector library** (#411, #414) | Architecture designed (D1-D9 decisions captured) | Build after detectors are validated |
| **Gap analysis** (#413) | Not started | Run after expanded detectors are trained |

**Critical path:** Reconstitute new data → retrain → validate → build detector library.

**Issues:** #388, #412, #411, #413, #414, #357, #358, #364 (done), #316, #365

### 2. Emulated Game Physics (stable — maintenance mode)

EmulatedGame simulates SC2 mechanics for offline testing. 99.8% unit accuracy at 5-min checkpoint.

| Issue | What | Priority |
|-------|------|----------|
| #401 | Calibrate mining income from dual-run results | Low — needs live SC2 |
| #396 | Dual-run calibration framework | Low — needs live SC2 |
| #385 | Run baseline against 2025-2026 tournament replays | Low |
| #405 | Calibrated SCV/Overlord multiplication | Low |
| #402 | Extend BalanceData for behavioral parameters | Low |
| #407 | Exact replay reconstruction via building routing | Deprioritised |
| #403 | Smoke test playbook suite | Low |

**Not blocking anything.** These improve emulator fidelity but the current accuracy is sufficient.

### 3. AI-vs-AI Training Loop (future — blocked on #1 and #2)

Autonomous play + continuous ONNX improvement. Requires both the detector library and full-game EmulatedGame play.

| Issue | What | Blocked by |
|-------|------|------------|
| #344 | Phase 4 epic: AI-vs-AI training loop | Everything below |
| #359 | P4-1: EmulatedGame autonomous full-game play | EmulatedGame maturity |
| #360 | P4-2: Feature extraction from emulated games | #359 |
| #361 | P4-3: Continuous training loop | #360 |
| #362 | P4-4: Domain gap validation | #361 |

**Not actionable now.** Depends on the detector library and emulated game being production-ready.

### 4. Framework Generalisation (architectural — independent)

Extracting SC2-specific patterns into the quarkmind-core framework for reuse across game domains (QuarkVille, chat, etc.).

| Issue | What | Priority |
|-------|------|----------|
| #408 | Promote situational awareness loop to quarkmind-core | Medium — depends on #409 |
| #409 | Bidirectional alignment: SC2 ↔ quarkmind-core | Medium — architectural |

**Can be worked on independently** when there's a natural pause in ONNX work.

### 5. Coaching & Strategy (future features)

| Issue | What | Priority |
|-------|------|----------|
| #345 | Build orders as adaptive plans | Future |
| #346 | Strategy workbench UI | Future |
| #315 | Integrate starcraft2.ai community analysis | Future |

**Backlog.** These build on the detector library and coaching pipeline.

---

## How to Proceed

### Immediate (next session)

1. **Check data downloads:**
   - 4.9.2: done (928K replays in `data/replay_packs/blizzard_ladder/4.9.2/replays/`)
   - 4.9.1: done (642K replays in `data/replay_packs/blizzard_ladder/4.9.1/replays/`)
   - SC2EGSet: check `data/sc2egset/` — if empty, restart download from Zenodo
   - Disk: only ~43 GB free. No room for 4.8.6 without cleanup.

2. **Reconstitute new replays:**
   ```
   cd quarkmind-classifier
   PYTHONPATH=. .venv/bin/python3 -m src.reconstitution_pipeline \
     --recon-dir data/reconstituted --output-dir data/full --max-replays 0
   ```
   But first: the replay packs need to go through the reconstitution extraction (stripped extractor → reconstituted JSON → feature vectors). The pipeline reads from `data/reconstituted/`, so raw .SC2Replay files need to be processed into that format first via the Java `ReconstitutionExportTest` or the Python extractor.

3. **Retrain binary detectors** with expanded `data/full/` dataset:
   ```
   cd quarkmind-classifier
   PYTHONPATH=. .venv/bin/python3 -m src.train_binary --data full
   ```

4. **Close housekeeping issues:** #364 (done), #316 (in progress)

### Blizzard API credentials

OAuth client "sc2-replay-download" on https://develop.battle.net/access/clients (login as Poldy). Client ID: `159d5558be314bc9b7e0db0a47cf19c7`. Secret visible after login.

Download script: `/tmp/download_blizzard_replays.py` (not in repo — write to /tmp/). Auto-deletes ZIPs after extraction.

### Additional data sources (not yet tapped)

- **AI Arena:** needs auth token from aiarena.net/profile/token/. Same patch 4.9.3. Good for rare archetypes (bots play unconventional strategies).
- **SC2ReplayStats:** needs 10 SC2 games played + uploaded to unlock account. Parked.
- **Blizzard 4.8.6/4.8.3/4.7.1:** ~700K more replays available but need disk space (~40 GB each).
- **Excluded 4.10.1:** 2,837 replays on disk, needs ability profile bridging.

### Reconstitution constraint

Blizzard ladder replays have NO tracker events — they use `StrippedReplayFeatureExtractor`, calibrated on 4.9.3. Only patches with `baseBuild ≤ 75689` (≤ 4.10) work. `ABILITY_PROFILE_BOUNDARY = 75689` in `reconstitution_pipeline.py`.

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-412-train-binary-onnx-detectors` | #412 | Detectors trained, data acquired | `work resume` |
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
