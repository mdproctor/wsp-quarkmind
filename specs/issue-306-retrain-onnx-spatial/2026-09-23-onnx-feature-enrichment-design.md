# Enrich ONNX Strategy Classifier Feature Vector

**Issue:** #306
**Branch:** issue-306-retrain-onnx-spatial
**Date:** 2026-09-23
**Depends on:** #298 (UnitPositions tracker events), #300 (spatial recalibration)

## Problem

The ONNX strategy classifier predicts opponent strategy archetypes (rush, macro, air, mech, etc.) from a 1D-CNN trained on temporal feature windows. The feature vector is purely compositional — 134 features per player: building counts (53), unit counts (53), economic stats (13), upgrade flags (15). It encodes WHAT units exist but has zero information about WHERE they are, HOW FAST the composition is changing, or HOW MUCH resource commitment is allocated to army vs economy.

With #298 and #300 landed, real enemy unit positions are now available during replay playback. The training data can include spatial features for the first time. More broadly, the feature vector has three information gaps that limit classification accuracy:

1. **Spatial intent** — a zergling rush at your base and zerglings sitting at home are identical in the current feature vector. A proxy gateway and a standard gateway are the same building count. The WHERE axis is the strongest discriminator between rush and non-rush strategies, and it's completely missing.

2. **Temporal dynamics** — the 10-window temporal structure captures some timing implicitly, but rate-of-change between windows isn't explicit. The model's conv layers must learn to compute deltas from raw counts, burning capacity on arithmetic instead of pattern recognition.

3. **Resource commitment ratios** — army supply ratio (the single most discriminative rush signal), worker saturation, and tech investment ratio are all derivable from raw features but require division operations that small CNNs struggle to learn.

## Solution

Add 29 features per window across three dimensions, growing the per-window vector from 269 to 298. Retrain all three per-matchup models (vs_terran, vs_zerg, vs_protoss) with the enriched features.

### 1. Spatial Features (15 per window)

Per player (7 features × 2 players = 14):

| Feature | Computation | Normalization |
|---------|-------------|---------------|
| `army_centroid_x` | Mean X of all army unit positions | [0, 1] by map width |
| `army_centroid_y` | Mean Y of all army unit positions | [0, 1] by map height |
| `army_dist_own_base` | Euclidean distance from army centroid to own start location | ÷ map diagonal |
| `army_dist_enemy_base` | Euclidean distance from army centroid to enemy start location | ÷ map diagonal |
| `army_spread` | √(var_x + var_y) of army unit positions | ÷ map diagonal |
| `max_forward_dist` | Distance of furthest army unit toward enemy base | ÷ map diagonal |
| `proxy_building_score` | Buildings closer to enemy base than own ÷ total buildings | [0, 1] naturally |

Inter-player (1 feature):

| Feature | Computation | Normalization |
|---------|-------------|---------------|
| `army_gap` | Euclidean distance between player and opponent army centroids | ÷ map diagonal |

**Edge cases:**
- No army units: centroid = own start location, spread = 0, max_forward = 0, distances = 0.
- No buildings: proxy_building_score = 0.
- No opponent units visible: opponent spatial block = zeros, `has_opponent` availability flag = 0. Model trained with modality dropout handles this.

**Position sources:**
- Java inference: `Unit.position()` from `GameState.enemyUnits()` / `GameState.myUnits()`, `Building.position()` from building lists.
- Python training: `UnitBornEvent` position + `UnitPositionsEvent` periodic updates from SC2EGSet tracker events.

### 2. Temporal Delta Features (8 per window)

Per player (4 features × 2 players = 8):

| Feature | Computation |
|---------|-------------|
| `delta_army_supply` | Total army unit count this window − previous window |
| `delta_worker_count` | Worker count this window − previous window |
| `delta_production_buildings` | Production building count this window − previous window |
| `delta_tech_buildings` | Non-production building count this window − previous window |

**Window 0:** All deltas = 0 (no previous window).

**Why these 4:** These are aggregate counts (sums across types), so per-window averages are smooth and deltas are stable. Per-unit-type deltas would be noisy (counts of individual types change by 0 or 1 per window).

**Production buildings:** Barracks, Factory, Starport (Terran); Spawning Pool, Roach Warren, Hydralisk Den, Spire (Zerg); Gateway, Robotics Facility, Stargate (Protoss). Count = sum of all production-capable buildings.

**Tech buildings:** All other non-production buildings in the feature index that indicate tech investment.

### 3. Structural Ratio Features (6 per window)

Per player (3 features × 2 players = 6):

| Feature | Computation | Edge case handling |
|---------|-------------|-------------------|
| `army_supply_ratio` | Σ(army unit food) ÷ food_used | food_used clipped to min=1 |
| `worker_saturation` | worker_count ÷ (base_count × 16) | denominator clipped to min=1 |
| `gas_mineral_ratio` | vespene_spent ÷ (vespene_spent + minerals_spent) | denominator clipped to min=1 |

**Army supply computation:** Sum food cost of all non-worker units. Food costs are fixed per unit type (e.g., Marine=1, Marauder=2, Siege Tank=3). Requires a food-cost lookup table in both Java and Python extractors.

**Base count:** Sum of active town-hall buildings (Command Center + Orbital Command + Planetary Fortress for Terran; Hatchery + Lair + Hive for Zerg; Nexus for Protoss).

### Feature Vector Layout

Current layout per window (269 features):
```
[player_buildings(53) | player_units(53) | player_economy(13) | player_upgrades(15) |
 opponent_buildings(53) | opponent_units(53) | opponent_economy(13) | opponent_upgrades(15) |
 has_vision(1)]
```

New layout per window (298 features):
```
[player_buildings(53) | player_units(53) | player_economy(13) | player_upgrades(15) |
 player_spatial(7) | player_deltas(4) | player_ratios(3) |
 opponent_buildings(53) | opponent_units(53) | opponent_economy(13) | opponent_upgrades(15) |
 opponent_spatial(7) | opponent_deltas(4) | opponent_ratios(3) |
 army_gap(1) | has_vision(1)]
```

New features are appended after each player block. The `army_gap` inter-player feature and `has_vision` flag are at the end. This preserves the existing feature indices for backward compatibility during development (existing tests don't break until the model is retrained).

### Model Architecture

No architecture change. The 1D-CNN (`StrategyClassifier`) takes `(batch, max_windows, f_temporal)` for temporal features and `(batch, f_map)` for map features. `f_temporal` increases from 269 to 298. The first conv layer's input channels widen from 269 to 298 — a trivial change (~11% more parameters in the first layer, negligible overall).

`MapCharacteristics` is unchanged (4 features + 2 availability flags).

### Normalization

`norm_stats.json` must be regenerated from training data with the new features. The existing 269 mean/std entries remain valid for their features; 29 new entries are appended. The vision flag (`has_vision`) and availability flags keep std=1.0 (not normalized).

Spatial features use map-relative normalization (÷ map diagonal or [0,1] by map bounds) BEFORE z-score normalization. This prevents map-size variation from dominating the feature statistics.

### Changes Required

**Java — quarkmind-sc2 (inference path):**

1. **`WindowSnapshot`** — Add `float[] playerPositionsX`, `float[] playerPositionsY`, `float[] opponentPositionsX`, `float[] opponentPositionsY`, `float[] playerBuildingPositionsX`, `float[] playerBuildingPositionsY`, `float[] opponentBuildingPositionsX`, `float[] opponentBuildingPositionsY` (raw position arrays for aggregation).

   Alternative (preferred): compute spatial aggregates directly in `DroolsScoutingTask.buildSnapshot()` and store the 7+7+1 = 15 scalar spatial features in `WindowSnapshot`. This avoids passing variable-length position arrays through the accumulator.

2. **`FeatureIndexMaps`** — Add `PRODUCTION_BUILDINGS` set, `TECH_BUILDINGS` set, `FOOD_COSTS` map (unit type → supply cost), `BASE_BUILDINGS` set. Update `N_FEATURES_PER_PLAYER` from 134 to 148 (134 + 7 spatial + 4 deltas + 3 ratios). Add `FEATURES_PER_WINDOW` = 2 × 148 + 1 (army_gap) + 1 (has_vision) = 298.

3. **`TemporalWindowAccumulator`** — Update `FEATURES_PER_WINDOW` constant. Add delta computation: when building windowed features, compute deltas from the previous window's aggregate counts.

4. **`StrategyFeatureExtractor`** — Update to handle the new feature vector size. Normalization applies to all 298 features using the regenerated `norm_stats.json`.

5. **`DroolsScoutingTask.buildSnapshot()`** — Extend to compute spatial aggregates from `GameState` unit/building positions and to pass the additional data through `WindowSnapshot`.

**Python — neocortex (training path):**

1. **`sc2egset_extractor.py`** — Extract unit positions from `UnitBornEvent` and `UnitPositionsEvent` tracker events. Maintain a position map (unit tag → (x, y)) updated on each position event. Compute per-second spatial aggregates: centroid, spread, distances, proxy score.

2. **`feature_engineering.py`** — Update `build_temporal_features()` to include spatial features, deltas, and ratios in each window. Update `F_TEMPORAL` constant.

3. **`normalize.py`** — `compute_stats()` must handle 298 features per window. Spatial features get double normalization (map-relative first, then z-score). `N_FEATURES_PER_PLAYER` updated.

4. **`config.py`** — No changes to hyperparameters or window structure.

5. **`export_onnx.py`** — `f_temporal` parameter changes; ONNX input shape adapts automatically via the existing dynamic axes.

**Shared data files:**

1. **`norm_stats.json`** — Regenerated with 298 entries (was 269). Placed in `quarkmind-sc2/src/main/resources/classifier/`.

2. **`strategy_vs_*.onnx`** — 3 retrained models placed in `quarkmind-sc2/src/test/resources/models/strategy/`.

3. **Food cost lookup** — A shared reference for unit supply costs. In Java: a static `Map<UnitType, Integer>` in `FeatureIndexMaps`. In Python: a dict in `sc2egset_extractor.py`. Must match exactly.

### Alignment Verification

**Critical constraint:** Java and Python feature extractors must produce identical feature vectors for the same game state. Any divergence causes a train/inference distribution mismatch that silently degrades accuracy.

**Alignment test (new):** A plain JUnit test (`FeatureAlignmentTest`) that:
1. Loads a known replay
2. Extracts features at a fixed game time using the Java pipeline
3. Compares against a pre-computed `.npz` file generated by the Python pipeline for the same replay at the same time
4. Asserts feature-level equality within floating-point tolerance (1e-5)

This test is the regression gate — it must pass before any ONNX model update is accepted.

### Calibration

Run `PatternClassificationCalibrationTest` before and after:
- Before: baseline accuracy with composition-only features (current model)
- After: accuracy with enriched features (retrained model)

Success criteria: >= 70% accuracy for rush and air-threat archetypes at 3-min mark (same threshold as current). Expected improvement: spatial features should improve rush detection significantly since spatial intent is the primary rush discriminator.

### What This Does NOT Change

- **Cascade architecture** — Drools → ONNX → LLM tiers unchanged.
- **ONNX model architecture** — Same 1D-CNN (StrategyClassifier), just wider input.
- **Window structure** — 10 windows of 30 seconds, covering 5 minutes. Finer resolution is a follow-up.
- **Label set** — Same archetypes per matchup (OnnxLabelMapping unchanged).
- **Map features** — MapCharacteristics (4 features + 2 availability flags) unchanged.
- **Drools rules** — Pattern classification rules unchanged.
- **LLM fallback** — LlmPatternClassifierWorkerFactory unchanged.

## References

- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:483` — `buildSnapshot()` feature extraction
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/FeatureIndexMaps.java` — feature index alignment with Python
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/TemporalWindowAccumulator.java` — windowed feature aggregation
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/StrategyFeatureExtractor.java` — normalization and tensor assembly
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadingPatternClassifier.java` — three-tier cascade
- `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/OnnxLabelMapping.java` — per-race label mapping
- `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py` — Python feature extraction from replays
- `neocortex/evaluation/strategy_classifier/feature_engineering.py` — Python temporal feature assembly
- `neocortex/evaluation/strategy_classifier/normalize.py` — normalization stat computation
- `neocortex/evaluation/strategy_classifier/run_pipeline.py` — training pipeline orchestrator
- `neocortex/evaluation/strategy_classifier/export_onnx.py` — ONNX export
- `neocortex/evaluation/strategy_classifier/dataset.py` — ModalityDropoutDataset (40% drop)
- `docs/protocols/sc2data-spatial-constants-require-calibration.md` — spatial calibration protocol
- #298 — UnitPositions tracker events (enemy units move during replay)
- #300 — Spatial recalibration (corrected enemy positions)
- #208 — ONNX strategy classifier epic
- #212 — Three-tier confidence cascade
