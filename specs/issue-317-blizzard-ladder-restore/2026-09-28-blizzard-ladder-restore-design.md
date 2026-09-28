# Blizzard Ladder Replay Restoration — Design Spec

**Issue:** #317
**Date:** 2026-09-28
**Branch:** issue-317-blizzard-ladder-restore

## Problem

151,477 Blizzard ladder replays (2,837 v4.10.1 + 148,640 v4.9.3) in `quarkmind-classifier/data/replay_packs/blizzard_ladder/` lack tracker events. Tracker events (UnitBorn, UnitDied, UnitInit, UnitDone, PlayerStats, Upgrade, UnitPositions) are required by the ONNX classifier training pipeline — they provide 100% of the 134 features per player (53 buildings + 53 units + 13 economic stats + 15 upgrades).

Without tracker events, these replays cannot feed through `prepare_replay_pack.py` → `extract_replay()` → `build_temporal_features()`. This is the largest untapped dataset for improving classifier coverage, especially for rare archetypes that tournament replays underrepresent.

## Strategy

**Two parallel tracks with a shared oracle:**

1. **Docker/SC2 headless restoration** — restore a small oracle set (~200 replays) locally via Podman emulation. This provides ground truth tracker events for calibration and validation.

2. **Java-native command-to-features pipeline** — extend `AbilityMapping` to cover all human replay abilLinks (buildings, upgrades, all races), then build `StrippedReplayFeatureExtractor` that reconstructs game state from game event commands using calibrated SC2Data constants. Validated against the oracle set. Processes 151K replays in minutes once validated.

The Java pipeline is not just a workaround for slow emulation — it's foundational infrastructure. Full abilLink coverage benefits EmulatedGame, coaching mode, replay validation, and all future human replay processing.

## Architecture

### Data flow

```
Stripped .SC2Replay (game events only, no tracker events)
         │
         ├── Docker/SC2 headless ──→ Restored .SC2Replay (with tracker events)
         │   (oracle set only)           │
         │                               ├── sc2reader → game_json → extract_replay()
         │                               │   (existing Python pipeline, unchanged)
         │                               │
         │                               └── Oracle ground truth for validation
         │
         └── Scelight parser ──→ GameEventStream ──→ ReplayCommandExtractor
             (RepContent.GAME_EVENTS)                      │
             (.backup fallback works)              TimedIntents + UnitOrders
                                                           │
                                                   StrippedReplayFeatureExtractor
                                                   (SC2Data timings + abilLinks)
                                                           │
                                                       game_json
                                                           │
                                              prepare_replay_pack.py pipeline
                                              (existing, unchanged)
```

### Key components

**Existing (no changes needed):**
- `RepParserEngine` — parses stripped replays via `.backup` fallback (lines 104-114)
- `GameEventStream` — extracts game events from any parseable replay
- `ReplayCommandExtractor` — produces `ReplayCommandStream` (TimedIntents + UnitOrders)
- `SC2Data` — calibrated train/build times, unit costs (validated by SC2TrainTimeCalibrationTest)
- `prepare_replay_pack.py` — Python feature engineering pipeline (consumes game_json)
- Docker tooling — `docker/sc2-restore/` (Dockerfile, restore_tracker.py, run.sh, setup.sh)

**New:**
- `HumanAbilityMapping` — extended AbilityMapping covering all human UI abilLinks for all 3 races (buildings, upgrades, morphs, warp-in)
- `StrippedReplayFeatureExtractor` — reconstructs synthetic tracker events from game event commands using SC2Data timings
- `AbilityDiscoveryCalibrationTest` — cross-references oracle game events with tracker events to discover and validate abilLink mappings
- `StrippedReplayValidationTest` — compares Java pipeline output against oracle ground truth, asserts zero divergence on deterministic features

**Modified:**
- `docker/sc2-restore/run.sh` — update paths from neocortex to quarkmind-classifier
- `docker/sc2-restore/setup.sh` — update paths from neocortex to quarkmind-classifier

## Phase 1: Oracle Restoration (~3 hours)

### 1a. Fix Docker tooling paths

Update `run.sh` and `setup.sh` to reference `quarkmind-classifier/data/` instead of `neocortex/evaluation/strategy_classifier/data/`.

### 1b. Complete SC2 headless extraction

Re-extract SC2 binaries and game data from existing ZIPs in `/tmp/`:
- v4.9.3: Extract `SC2.4.9.3/StarCraftII/Versions/Base75025/SC2_x64` + SC2Data + `.build.info` + `Battle.net/` from `/tmp/SC2.4.9.3.zip`. Set executable permission.
- v4.10.1: Base75800 not available in `/tmp/SC2.4.10.zip` (only contains Base75689 = v4.10.0). Defer 4.10.1 oracle until Base75800 is obtained.

### 1c. Restore oracle set

Restore ~200 replays from the 4.9.3 batch via Podman. Stratify by matchup:
- Sample from the 148,640 replays, ~33 per matchup (PvT, PvZ, PvP, TvZ, TvT, ZvZ)
- Use `gamemetadata.json` to filter by race before restoration (avoid restoring replays that end up being wrong matchup)
- Estimated time: ~200 × 50s/replay ÷ 2 workers ≈ 2.8 hours

Output: `quarkmind-classifier/data/replay_packs/blizzard_ladder/4.9.3_oracle/`

### 1d. Validate oracle replays

Verify restored replays have tracker events and can feed through the Python pipeline:
- mpyq: confirm `replay.tracker.events` present
- sc2reader load_level=3: confirm event type counts match expectations
- `prepare_replay_pack.py --dir 4.9.3_oracle/ --workers 4`: confirm feature extraction produces training data

**Known issue:** sc2reader load_level=4 crashes on re-recorded replays (`NoneType.pid` in context plugin). load_level=3 provides raw tracker events which `sc2reader_to_game_json()` may need to handle. If the existing Python pipeline can't process restored replays, a pre-processing step that merges player metadata from `gamemetadata.json` may be needed.

## Phase 2: abilLink Discovery

### 2a. Discovery test

Build `AbilityDiscoveryCalibrationTest` following the pattern of `SC2TrainTimeCalibrationTest`:
- For each oracle replay, parse both game events (CmdEvent with abilLink) and tracker events (UnitBorn, UnitInit, Upgrade)
- Cross-reference: for each tracker event, find the game event command that caused it (same player, closest preceding loop within expected range)
- Build the abilLink → unit/building/upgrade mapping table

### 2b. Coverage targets

The classifier uses 134 features. The discovery must map abilLinks for:

| Category | Count | Currently mapped | Gap |
|----------|-------|-----------------|-----|
| Units (train) | 53 types | ~24 (Protoss gateway/robo/stargate, some Terran) | ~29 |
| Buildings (place) | 53 types | 0 (bot replays use Smart) | 53 |
| Upgrades (research) | 15 tracked | 0 | 15 |
| Morphs (Zerg) | ~8 types | 0 (Baneling, Ravager, Lurker, BroodLord, Lair, Hive, etc.) | ~8 |
| WarpGate warp-in | 5+ types | Partially (abilLink=170 known, unit selection unknown) | ~5 |

### 2c. AbilityMapping extension

Extend `AbilityMapping` (or create `HumanAbilityMapping`) with the discovered mappings. Support both bot and human abilLink vocabularies — the existing bot mappings remain valid for AI Arena replays.

### 2d. Calibration validation

Assert that discovered abilLinks + SC2Data train/build times produce unit birth loops matching oracle tracker events within ±1 tick tolerance.

## Phase 3: StrippedReplayFeatureExtractor

### 3a. Core extractor

A Java class that:
1. Accepts a stripped replay `Path`
2. Parses game events via `GameEventStream.events()`
3. Runs `HumanAbilityMapping` to produce `TimedIntent`s for all command types (train, build, upgrade, morph, warp-in)
4. Simulates deterministic game state:
   - Unit births: `command_loop + SC2Data.trainTimeInLoops(unitType)`
   - Building starts: `command_loop` (UnitInit equivalent)
   - Building completions: `command_loop + SC2Data.buildTimeInLoops(buildingType)` (UnitDone equivalent)
   - Upgrades: `command_loop + SC2Data.upgradeTimeInLoops(upgradeType)`
   - Economy: derived from build order costs and worker count (starting economy → subtract costs → track workers)
5. Emits `game_json` dict matching the structure `sc2reader_to_game_json()` produces

### 3b. Economy reconstruction

Economy stats (PlayerStats equivalent) require:
- Starting resources per race (50 minerals for all)
- Worker count over time (initial workers + trained probes/SCVs/drones - deaths)
- Mining rate model (workers × rate, capped by saturation)
- Build expenditure (subtract unit/building/upgrade costs at command time)

The mining rate model is an approximation — income depends on worker-to-base assignment, mineral patch distances, and saturation. Use the same constants as EmulatedGame's economy model for consistency.

Combat deaths are NOT reconstructed — unit counts are strictly additive (births only). This is acceptable for early-game classification (minutes 2-5) where combat is minimal. For later time windows, unit counts will be higher than reality (no deaths subtracted). The labelling pipeline (which determines strategy archetype from build order, not unit counts) is unaffected.

### 3c. Output format

Emit JSON matching `sc2reader_to_game_json()` structure:
```json
{
  "ToonPlayerDescMap": { "1": {"playerID": 1, "race": "Protoss", "result": "Win"}, ... },
  "trackerEvents": [
    {"evtTypeName": "UnitBorn", "loop": 268, "controlPlayerId": 1, "unitTypeName": "Probe", ...},
    {"evtTypeName": "PlayerStats", "loop": 160, "controlPlayerId": 1, "stats": {...}},
    ...
  ],
  "header": {"elapsedGameLoops": 14544},
  "metadata": {"mapName": "..."}
}
```

Player metadata (race, result, map name) sourced from `gamemetadata.json` in the MPQ archive.

## Phase 4: Validation Loop

### 4a. Per-replay comparison

For each oracle replay, compare:
- Java pipeline `game_json` vs Python pipeline `game_json` (from sc2reader on restored replay)
- Assert: unit birth events match (same types, same loops ±1 tick)
- Assert: building events match (same types, same loops ±1 tick)
- Assert: upgrade events match (same types, same loops ±1 tick)
- Report: economy divergence (expected — mining model approximation)

### 4b. Feature-level comparison

Compare extracted temporal features at each time window (minutes 2, 3, 4, 5):
- Unit count vectors: exact match (no combat = no deaths = same counts)
- Building count vectors: exact match
- Upgrade flags: exact match
- Economy stats: within tolerance (mining model approximation)

### 4c. Divergence-driven hardening

Every divergence drives a fix in one of:
- `SC2Data` constants (train/build/upgrade times)
- `HumanAbilityMapping` (abilLink → command mapping)
- `StrippedReplayFeatureExtractor` (simulation logic)
- `EmulatedGame` (when the fix applies to physics shared with emulation)

Re-run validation after each fix until the oracle set passes at zero divergence for deterministic features.

## Phase 5: Bulk Processing

### 5a. Java batch runner

Process all 151,477 stripped replays through the validated Java pipeline:
- Emit one `game_json` file per replay
- Parallel processing (Java thread pool, ~4-8 threads)
- Expected throughput: thousands of replays per second (native ARM64, no SC2)

### 5b. Python training pipeline

Feed Java-emitted `game_json` files through the existing pipeline:
```bash
python3 -m src.prepare_replay_pack --dir <game_json_dir> --name blizzard_ladder --batch-size 5000 --workers 10
```

### 5c. ONNX retrain

Merge with existing training data and retrain:
```bash
python3 -m src.normalize --source blizzard_ladder
python3 -m src.run_pipeline --data combined
```

## Testing Strategy

| Test | Type | What it validates |
|------|------|-------------------|
| `StrippedReplayParseTest` | Unit | Scelight parses stripped replay game events |
| `AbilityDiscoveryCalibrationTest` | Calibration | Human abilLink mappings match oracle tracker events |
| `SC2TrainTimeCalibrationTest` (existing) | Calibration | Train times match real SC2 |
| `StrippedReplayValidationTest` | Integration | Java pipeline output matches oracle ground truth |
| `StrippedReplayFeatureComparisonTest` | Integration | Feature vectors match at each time window |

## Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| abilLink values differ between v4.9.3 and v4.10.1 patches | Low (validated: 2019 abilLinks match 2023+) | Build version-aware mapping if needed |
| Economy model divergence too large for classifier accuracy | Medium | Validate against oracle; the classifier may tolerate ±10% economy divergence |
| sc2reader can't process Docker-restored replays at load_level=4 | Confirmed | Use load_level=3 or merge metadata from gamemetadata.json |
| v4.10.1 Base75800 unavailable for Docker restoration | Confirmed | Defer 4.10.1 oracle; validate Java pipeline on 4.9.3 first, process 4.10.1 via Java-only |
| Some abilLinks are context-dependent (WarpGate vs Gateway) | Medium | Track WarpGate research status per player during command replay |

## References

- `sc2egset_extractor.py` — 134 features: BUILDINGS (53), UNITS (53), STAT_KEYS (13), UPGRADES (15)
- `AbilityMapping.java` — current abilLink coverage (lines 21-33 comment documents bot/human difference)
- `SC2TrainTimeCalibrationTest.java` — calibration pattern for cross-referencing game events with tracker events
- `RepParserEngine.java:104-114` — `.backup` suffix fallback in Scelight parser
- `ReplayCommandExtractor.java` — existing command extraction pipeline
- `ReplaySimulatedGame.java` — tracker-event-driven state machine (reference for event application logic)
- `prepare_replay_pack.py` — Python feature engineering pipeline
- `StrippedReplayParseTest.java` — validates stripped replay parsing (created this session)
- `docker/sc2-restore/restore_tracker.py` — Docker restoration pipeline
- Decision review: `/Users/mdproctor/reviews/casehub-quarkmind/issue-317-decision-20260928-034545/`
