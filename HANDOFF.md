# HANDOFF — quarkmind

## Last Session

Branch `issue-317-blizzard-ladder-restore`, issue #317. Designed and began implementing a Java-native pipeline to extract ONNX classifier features from 151K stripped Blizzard ladder replays.

### What was done

1. **Brainstorming + design** — explored 3 approaches (Docker/SC2 headless, EmulatedGame physics, Java-native deterministic). Landed on Java-native with Docker oracle validation. Key insight: game events in stripped replays contain player commands, and calibrated SC2Data constants can deterministically reconstruct unit births, building placements, upgrades, and economy — no SC2 engine needed.

2. **Decision review** (standard, 3 rounds, 13 issues) — reviewer identified that AbilityMapping only covers ~24 of 53 unit types, zero buildings, zero upgrades. abilLink discovery (Phase 2) is the critical path. Also surfaced: morph-death semantics, WarpGate auto-morph, production queue tracking, building double-counting.

3. **Spec review** (standard, 3 rounds, 26 issues) — added production queue state (§3a-2), 10 morph types including Archon merge (§3a-3), WarpGate auto-morph (§3a-4), 3-tier economy model (§3b), string-name mapping layer (§3c-1).

4. **Implementation — Batch 1 complete (Docker Oracle)**
   - Fixed `docker/sc2-restore/run.sh` and `setup.sh` — paths now location-independent (relative to script dir)
   - Extracted SC2 v4.9.3 headless (Base75025, 3.9GB) from existing ZIP
   - Built `sample_oracle.py` — stratified 198 replays (33 per matchup)
   - Launched oracle restoration via Podman — **118/198 complete when session ended, container still running**

5. **Implementation — Batch 2 partial (abilLink Discovery)**
   - Created `UpgradeType` enum — 15 classifier-tracked upgrades with `pythonName` matching Python pipeline's UPGRADES list
   - Added `SC2Data.upgradeTimeInLoops()` — wiki-derived estimates, to be refined from oracle calibration
   - Built `AbilityDiscoveryCalibrationTest` — cross-references oracle game events with tracker events. Initial run on 118 replays shows strong signals (Drone=193/0, DarkTemplar=170/15) but lookback window needs refinement for lower-frequency units and upgrades

### Key validation results (from brainstorming)

- **Stripped replays parse via Scelight** — `.backup` fallback in RepParserEngine works
- **ReplayCommandExtractor works on stripped replays** — 21 intents + 24 movement orders from a 4.9.3 ladder replay
- **abilLink constants from 2023+ work on 2019 replays** — no patch version mismatch
- **Docker restoration works for 4.9.3** — 5/5 test replays restored, tracker events confirmed (50,749 bytes)
- **4.10.1 blocked** — needs Base75800, we only have Base75689 (v4.10.0)

### Oracle restoration status

Container `7b5d314c9a75` running in Podman. 118/198 complete at session end. Checkpoint-safe — already-restored replays are skipped on restart. Output: `quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored/`

To check progress:
```bash
ls quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored/ | wc -l
```

To restart if container stopped:
```bash
podman run --rm --platform linux/amd64 \
  -v $PWD/quarkmind-classifier/data/sc2_headless/4.9.3/SC2.4.9.3/StarCraftII:/opt/StarCraftII:ro \
  -v $PWD/quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/input:/data/input:ro \
  -v $PWD/quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/restored:/data/output \
  sc2-restore:latest --input /data/input --output /data/output --workers 2
```

## What's Next

| Item | Scale | Complexity | Notes |
|------|-------|------------|-------|
| Refine AbilityDiscoveryCalibrationTest lookback window | S | Med | Current modal matching catches noise — needs tighter window for buildings/upgrades, possibly filter by hasTargetPoint. Run with full 198 oracle set. |
| T5: Extend AbilityMapping with discovered abilLinks | M | Med | Add ReplayCommand variants (BuildCommand, UpgradeCommand, MorphCommand, CancelCommand), populate abilLink constants from discovery output |
| T6-T8: StrippedReplayFeatureExtractor (Batch 3) | L | High | Core extractor with production queues, morph-death semantics, WarpGate auto-morph, 3-tier economy |
| T9-T10: Validation + Bulk Processing (Batch 4) | M | Med | Oracle validation test, prepare_replay_pack.py JSON input, bulk Java runner, ONNX retrain |

## Artifacts

| Path | What |
|------|------|
| `specs/issue-317-blizzard-ladder-restore/2026-09-28-blizzard-ladder-restore-design.md` | Design spec (reviewed, 3 rounds) |
| `specs/issue-317-blizzard-ladder-restore/decisions.md` | 7 decisions (D1-D7, reviewed, 3 rounds) |
| `plans/2026-09-28-blizzard-ladder-restore.md` | Implementation plan (4 batches, 10 tasks) |
| `specs/issue-317-blizzard-ladder-restore/pipeline.state` | Brainstorming pipeline state |
| Decision review: `/Users/mdproctor/reviews/casehub-quarkmind/issue-317-decision-20260928-034545/` | |
| Spec review: `/Users/mdproctor/reviews/casehub-quarkmind/issue-317-spec-20260928-043626/` | |
