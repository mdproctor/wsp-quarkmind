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

**New (all in `quarkmind-sc2` module):**
- `UpgradeType` enum — domain type covering the 15 classifier-tracked upgrades, following `UnitType`/`BuildingType` pattern
- `SC2Data.upgradeTimeInLoops(UpgradeType)` — calibrated upgrade research times (initially estimated, refined from oracle data)
- `StrippedReplayFeatureExtractor` — reconstructs synthetic tracker events from game event commands using SC2Data timings
- `AbilityDiscoveryCalibrationTest` — cross-references oracle game events with tracker events to discover and validate abilLink mappings (including cancel abilLinks)
- `StrippedReplayValidationTest` — compares Java pipeline output against oracle ground truth, asserts zero divergence on deterministic features

**Modified (in `quarkmind-sc2` module):**
- `AbilityMapping` — extended dispatch table covering all human UI abilLinks for all 3 races (buildings, upgrades, morphs, warp-in, cancels). The existing class already handles human-format abilLinks (AI Arena bots use the same abilLinks as human play); this extends coverage from ~24 to all 134 feature-producing command types.

**Modified (in `quarkmind-classifier`):**
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

Restore ~200 replays from the 4.9.3 batch via Podman (drop-in replacement for `docker` CLI on macOS ARM64). Stratify by matchup:
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

### 2b-1. Feature coverage analysis (D6)

Systematic mapping of the 134 classifier features to their reconstruction source:

| Feature group | Count | Tracker event source | Java pipeline reconstruction | Accuracy |
|---------------|-------|---------------------|------------------------------|----------|
| Unit counts | 53 | UnitBorn + UnitDied | Train commands + SC2Data.trainTimeInLoops() | Exact for births; deaths missing (overcounts at minute 4+). Cancel commands detected and suppressed (§3a). |
| Building counts | 53 | UnitInit + UnitDone + UnitDied | Build commands + SC2Data.buildTimeInLoops() | Exact for construction starts/completions; destruction not reconstructed (overcounts when buildings destroyed). Double-counted per Python pipeline semantics (§3a-1). |
| Economy stats | 13 | PlayerStats | Derived: cumulative spending (exact) + mining model (approximate) — see §3b for per-stat breakdown | Mixed: 8 stats exact, 5 stats approximate (§3b) |
| Upgrade flags | 15 | Upgrade | Research commands + SC2Data.upgradeTimeInLoops(UpgradeType) | Exact (new UpgradeType enum + calibrated times) |

**String-name mapping:** The `game_json` output uses SC2-native string names (e.g. `"BarracksReactor"`, `"HellionTank"`, `"WarpGate"`) matching the Python pipeline's BUILDINGS/UNITS lists exactly — NOT Java enum `.name()` values. The 7 building types absent from `BuildingType` enum (BarracksReactor, BarracksTechLab, FactoryReactor, FactoryTechLab, StarportReactor, StarportTechLab, WarpGate) and the unit name divergences (Java `HELLBAT` → Python `HellionTank`, Java `VIKING` → Python `VikingFighter`) are handled by a string-name mapping table keyed by abilLink (§3c-1). `FeatureIndexMaps.java` already documents these gaps.

**After Phase 2 abilLink discovery completes:** all 134 features are reconstructable. Economy stats are the only approximate category — the Java pipeline uses EmulatedGame's mining rate model which is calibrated but not identical to SC2's internal economy simulation.

**Before Phase 2 completes (current state):** only ~24 of 53 unit types can be extracted (AbilityMapping gaps block buildings, upgrades, and unmapped units). This is why Phase 2 is the critical path.

### 2c. AbilityMapping extension

Extend the existing `AbilityMapping` class with discovered mappings. The existing class already handles human-format abilLinks (AI Arena bots produce identical abilLinks to human play — see class javadoc). The extension adds:

1. **Building abilLinks** — place commands for all 53 building types (including add-ons and WarpGate morph), mapped to SC2-native string names
2. **Upgrade abilLinks** — research commands for all 15 tracked upgrades, mapped to `UpgradeType` values
3. **Cancel abilLinks** — cancel commands for train, build, and research, mapped to the pending event they cancel
4. **Morph abilLinks** — Zerg morphs (Baneling, Ravager, Lurker, BroodLord, Lair, Hive, Overseer, GreaterSpire)
5. **WarpGate warp-in** — abilLink=170: verify whether `abilCmdIndex` distinguishes unit types during discovery. If yes, add per-index mappings. If no, design a fallback (tech tree inference from available buildings) or accept as a quantified limitation.

New dispatch entries use the same `ReplayCommand` abstraction. Existing bot replay mappings remain unchanged.

### 2d. Calibration validation

Assert that discovered abilLinks + SC2Data train/build/upgrade times produce events matching oracle tracker events within ±1 tick tolerance.

**All-race calibration:** The existing `SC2TrainTimeCalibrationTest` only covers 5 Protoss unit types (`EXPECTED_RANGES` for PROBE, ZEALOT, STALKER, IMMORTAL, OBSERVER). Phase 2 discovery extends calibration to all 3 races — Terran and Zerg train times in SC2Data are currently estimates (ticks × LOOPS_PER_TICK). The oracle set (stratified by matchup at ~33 per matchup) provides cross-race data.

**Upgrade time calibration:** Initial `upgradeTimeInLoops()` values are wiki-derived estimates. Phase 2 calibrates from oracle Upgrade tracker events cross-referenced against research command abilLinks, following the same modal-diff calibration pattern as `SC2TrainTimeCalibrationTest`.

## Phase 3: StrippedReplayFeatureExtractor

### 3a. Core extractor

A Java class (`quarkmind-sc2/.../replay/StrippedReplayFeatureExtractor`) that:
1. Accepts a stripped replay `Path`
2. Parses game events via `GameEventStream.events()`
3. Runs `AbilityMapping` (extended) to classify all command types (train, build, upgrade, morph, warp-in, cancel)
4. Simulates deterministic game state:
   - Unit births: `command_loop + SC2Data.trainTimeInLoops(unitType)`
   - Building starts: `command_loop` (UnitInit equivalent)
   - Building completions: `command_loop + SC2Data.buildTimeInLoops(buildingType)` (UnitDone equivalent)
   - Upgrades: `command_loop + SC2Data.upgradeTimeInLoops(upgradeType)`
   - Cancel commands: remove the pending scheduled event (UnitBorn, UnitDone, or Upgrade) for the cancelled action. Emit a synthetic UnitDied for buildings where UnitInit was already emitted (matching the Python pipeline's UnitInit → UnitDied cancel sequence).
   - Economy: derived from build order costs and worker count (starting economy → subtract costs → track workers) — see §3b
5. Emits `game_json` dict matching the structure `sc2reader_to_game_json()` produces, using SC2-native string names — see §3c-1

### 3a-1. Building double-counting semantic

The Python pipeline's `extract_replay()` increments `building_counts` for **both** UnitInit and UnitDone events — each completed building contributes +2 to its feature slot. UnitDied decrements by 1. The Java pipeline must replicate this exact semantic:

| Event sequence | building_counts change | Scenario |
|---------------|----------------------|----------|
| UnitInit + UnitDone | +2 | Normal building completion |
| UnitInit + UnitDied (cancel) | +1 - 1 = 0 | Building cancelled during construction |
| UnitInit + UnitDone + UnitDied | +2 - 1 = +1 | Building completed then destroyed |

The feature meaning is "building activity events" (cumulative init + done), not "building count." The Java pipeline schedules UnitInit-equivalent at `command_loop` and UnitDone-equivalent at `command_loop + buildTimeInLoops()`, both contributing +1 to the feature vector. Cancel handling (§3a point 4) prevents the UnitDone emission and adds a corrective UnitDied.

### 3b. Economy reconstruction

Economy stats (PlayerStats equivalent) fall into three accuracy tiers:

**Tier 1 — Exact from commands (8 stats):**
| Stat | Algorithm |
|------|-----------|
| `scoreValueFoodMade` | Initial supply (15 all races) + `SC2Data.supplyBonus(buildingType)` for each completed supply building (Pylon +8, SupplyDepot +8, Overlord +8, Hatchery +6, etc.) |
| `scoreValueMineralsUsedCurrentArmy` | Cumulative `SC2Data.mineralCost(unitType)` for all army unit train commands |
| `scoreValueMineralsUsedCurrentEconomy` | Cumulative mineral cost for workers + gas buildings |
| `scoreValueMineralsUsedCurrentTechnology` | Cumulative mineral cost for tech buildings + upgrades |
| `scoreValueVespeneUsedCurrentArmy` | Cumulative `SC2Data.gasCost(unitType)` for army units |
| `scoreValueVespeneUsedCurrentEconomy` | Cumulative gas cost for economy |
| `scoreValueVespeneUsedCurrentTechnology` | Cumulative gas cost for tech |

Note: SC2's `UsedCurrent` stats are cumulative totals, not net-of-refunds. Cancelled commands (detected via cancel abilLinks, §3a) are excluded from the cumulative sum.

**Tier 2 — Approximate via mining model (4 stats):**
| Stat | Algorithm | Expected divergence |
|------|-----------|-------------------|
| `scoreValueMineralsCurrent` | `INITIAL_MINERALS + Σ(miningIncome) - Σ(spending)` using `SC2Data.mineralIncomePerTick()` | ±10-20% (mining model assumes optimal saturation) |
| `scoreValueVespeneCurrent` | `INITIAL_VESPENE + Σ(gasIncome) - Σ(spending)` | ±10-20% (gas rate model approximation) |
| `scoreValueMineralsCollectionRate` | `SC2Data.mineralIncomePerTick(workerCount) × GAME_LOOPS_PER_SECOND` | ±15% (saturation model vs reality) |
| `scoreValueVespeneCollectionRate` | Gas workers × gas rate constant | ±15% |

Use the same constants as EmulatedGame's economy model for consistency.

**Tier 3 — Overcounts without death tracking (2 stats):**
| Stat | Algorithm | Divergence source |
|------|-----------|------------------|
| `scoreValueFoodUsed` | Cumulative `SC2Data.supplyCost(unitType)` for living units | Overcounts: no death subtraction. ±0 at minute 2-3, growing divergence at 4-5. |
| `scoreValueWorkersActiveCount` | Workers trained - workers lost | Overcounts: worker deaths not tracked. |

**Validation tolerance (Phase 4):** Tier 1 stats: zero divergence. Tier 2 stats: ≤20% relative divergence per stat averaged across oracle set. Tier 3 stats: report divergence magnitude, accept for early-game windows (minutes 2-3); flag but accept for later windows. If any Tier 2 stat exceeds 30% divergence on >10% of oracle replays, the mining model requires refinement.

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

### 3c-1. String-name mapping layer

The `game_json` `unitTypeName` values must exactly match the Python pipeline's BUILDINGS (53) and UNITS (53) lists. The Java `BuildingType`/`UnitType` enums diverge from these lists in two ways:

**Missing building types (7):** BarracksReactor, BarracksTechLab, FactoryReactor, FactoryTechLab, StarportReactor, StarportTechLab, WarpGate — these have no `BuildingType` enum entry. `FeatureIndexMaps.java` documents them as empty feature slots (indices 15-20, 39).

**Name divergences:**
| SC2/Python name | Java enum | Feature index |
|----------------|-----------|---------------|
| `HellionTank` | `UnitType.HELLBAT` | unit:6 |
| `VikingFighter` | `UnitType.VIKING` | unit:11 |
| `WarpGate` | `BuildingType.GATEWAY` (collapsed) | building:39 |
| `BarracksReactor` | `BuildingType.BARRACKS` (collapsed) | building:15 |
| (etc. for all 6 add-on types) | | |

**Solution:** The abilLink discovery (Phase 2) maps each abilLink to a **Python-compatible string name** directly — not to a Java enum. The `StrippedReplayFeatureExtractor` stores these as `String` values and emits them verbatim in `game_json`. This sidesteps the enum gap entirely:

```
abilLink → Map<Integer, String> pythonBuildingName  (53 entries)
abilLink → Map<Integer, String> pythonUnitName      (53 entries)
abilLink → Map<Integer, UpgradeType> upgrade        (15 entries)
```

For building add-ons, a separate abilLink triggers the add-on construction (e.g., "Build Reactor on Barracks" has a distinct abilLink from "Build Barracks"). The discovery maps this abilLink to `"BarracksReactor"` directly.

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
- **Unit birth events:** exact match on type and loop (deterministic from commands + SC2Data timings, cancel commands handled)
- **Building placement events:** exact match on type and loop (UnitInit-equivalent)
- **Building completion events:** exact match on type and loop (UnitDone-equivalent, cancel commands handled)
- **Upgrade events:** exact match on type and loop
- **Unit count vectors (minutes 2-3):** exact match expected — combat is rare before minute 3
- **Unit count vectors (minutes 4-5):** overcounted vs oracle (Java pipeline has no combat deaths). Report divergence magnitude but do not assert match — this is a known, accepted limitation
- **Building count vectors (minutes 2-3):** near-exact match expected — building destruction is uncommon in the first 3 minutes. Report any divergence (likely from proxy strategies, cannon rushes, or worker harassment killing gas buildings)
- **Building count vectors (minutes 4-5):** may diverge for games with building destruction. Report divergence magnitude; accept as a limitation analogous to unit death overcounting
- **Upgrade count vectors:** exact match at all windows (upgrades cannot be destroyed once completed)
- **Economy stats:** per §3b tolerance tiers — Tier 1 (8 stats): zero divergence. Tier 2 (4 stats): ≤20% relative divergence. Tier 3 (2 stats): report magnitude

### 4c. Divergence-driven hardening

Every divergence drives a fix in one of:
- `SC2Data` constants (train/build/upgrade times)
- `AbilityMapping` (abilLink → command mapping)
- `StrippedReplayFeatureExtractor` (simulation logic)
- `EmulatedGame` (when the fix applies to physics shared with emulation)

Re-run validation after each fix until the oracle set passes at zero divergence for deterministic features.

## Phase 5: Bulk Processing

### 5a. Java batch runner

Process all 151,477 stripped replays through the validated Java pipeline:
- Emit one `game_json` file per replay
- Parallel processing (Java thread pool, ~4-8 threads)
- Expected throughput: 100-500 replays/second (Scelight MPQ parsing involves file I/O, decompression, and binary protocol parsing — 2-10ms per replay). Total processing time: ~5-25 minutes.
- Crash-resume: skip replays that already have a corresponding `game_json` output file. On restart, only unprocessed replays are parsed. Log failures with replay path for later investigation.

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
| abilLink values differ between v4.9.3 and v4.10.1 patches | Low (validated: 2019 abilLinks match 2023+) | Cross-validation: compare abilLinks in a sample of v4.10.1 game events against the v4.9.3 discovered mapping. If any abilLink in a v4.10.1 command doesn't appear in the v4.9.3 mapping, flag for investigation before bulk processing. |
| Economy model divergence too large for classifier accuracy | Medium | Validate against oracle with per-stat tolerance tiers (§3b). If Tier 2 stats exceed 30% divergence on >10% of replays, refine mining model. |
| sc2reader can't process Docker-restored replays at load_level=4 | Confirmed | Use load_level=3 or merge metadata from gamemetadata.json |
| v4.10.1 Base75800 unavailable for Docker restoration | Confirmed | Defer 4.10.1 oracle; validate Java pipeline on 4.9.3 first, process 4.10.1 via Java-only with cross-patch abilLink validation |
| Some abilLinks are context-dependent (WarpGate vs Gateway) | Medium | Phase 2 discovery verifies whether abilCmdIndex distinguishes warp-in unit types. If yes, direct mapping. If no, fallback to tech tree inference or accept as quantified limitation. |
| Cancelled commands create overcount in Java pipeline | Low-Medium | Cancel abilLinks discovered alongside train/build/upgrade abilLinks in Phase 2. Extractor removes pending events on cancel. Phase 4 validation catches any missed cancel patterns. |

## Deferred Items

The following items are deferred and will be filed as GitHub issues:

| Item | Reason | Issue |
|------|--------|-------|
| v4.10.1 oracle restoration | Base75800 SC2 binary unavailable | To be filed: blocked on obtaining SC2 v4.10.1 headless binary |
| Combat death reconstruction for unit counts | Out of scope — acceptable for early-game classification | To be filed: enhancement for post-minute-5 accuracy |
| Building destruction reconstruction | Out of scope — analogous to combat deaths | To be filed: enhancement for building count accuracy in games with proxy/rush strategies |
| Economy model refinement beyond EmulatedGame constants | Depends on Phase 4 validation results | To be filed after Phase 4: if Tier 2 divergence exceeds threshold |

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
