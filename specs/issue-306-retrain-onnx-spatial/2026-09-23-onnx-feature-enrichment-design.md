# Enrich ONNX Strategy Classifier Feature Vector

**Issue:** #306
**Branch:** issue-306-retrain-onnx-spatial
**Date:** 2026-09-23
**Depends on:** #298 (UnitPositions tracker events), #300 (spatial recalibration)

## Problem

The ONNX strategy classifier predicts opponent strategy archetypes (rush, macro, air, mech, etc.) from a dual-encoder 1D-CNN trained on temporal feature windows. The feature vector is purely compositional — 134 features per player: building counts (53), unit counts (53), economic stats (13), upgrade flags (15). It encodes WHAT units exist but has zero information about WHERE they are, HOW FAST the composition is changing, or HOW MUCH resource commitment is allocated to army vs economy.

The current codebase has no spatial features. `DroolsScoutingTask.buildSnapshot()` extracts counts, stats, and upgrade flags — it never reads unit or building positions. The Python extractor (`sc2egset_extractor.py`) processes `UnitBorn`, `UnitDied`, `PlayerStats`, and `Upgrade` events — never position coordinates. Issue #306's original title ("retrain against corrected spatial features") anticipated spatial features that were never implemented. This spec defines the actual scope: adding spatial, temporal delta, and structural ratio features, then retraining.

With #298 and #300 landed, real enemy unit positions are now available during replay playback. The training data can include spatial features for the first time. More broadly, the feature vector has three information gaps that limit classification accuracy:

1. **Spatial intent** — a zergling rush at your base and zerglings sitting at home are identical in the current feature vector. A proxy gateway and a standard gateway are the same building count. The WHERE axis is the strongest discriminator between rush and non-rush strategies, and it's completely missing.

2. **Temporal dynamics** — the 10-window temporal structure captures some timing implicitly, but rate-of-change between windows isn't explicit. The model's conv layers must learn to compute deltas from raw counts, burning capacity on arithmetic instead of pattern recognition.

3. **Resource commitment ratios** — army supply ratio (the single most discriminative rush signal), worker saturation, and tech investment ratio are all derivable from raw features but require division operations that small CNNs struggle to learn.

## Solution

Add 29 features per window across three dimensions, growing the per-window vector from 269 to 298. Retrain all three per-matchup models (vs_terran, vs_zerg, vs_protoss) with the enriched features.

Features fall into two computation categories:
- **Per-tick features** (spatial + ratios): computed in `buildSnapshot()` for each game tick, averaged across ticks within a window by the accumulator. These extend `WindowSnapshot` from 134 to 144 per-player features.
- **Per-window features** (deltas + cross-player): computed at window-assembly time by the accumulator, after tick averaging. These are appended to each player block (deltas) or at the end of the window vector (army_gap, has_vision).

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
- No army units: all 7 spatial features = 0 (centroid, distances, spread, max_forward, proxy_building_score). A clean zero signal avoids leaking map geometry into a "no army" state and is trivially produced by both Java and Python for empty unit lists.
- No buildings: proxy_building_score = 0.
- No opponent units visible: opponent spatial block = zeros, `has_opponent` availability flag = 0. Model trained with modality dropout handles this.

**Fog-of-war treatment for spatial features:**

Spatial features use **binary visibility**, not continuous scaling. This differs from count features (which are scaled by continuous `scoutingVisibility`):

- **Java inference:** Spatial aggregates are computed from `GameState.enemyUnits()` and `GameState.enemyBuildings()`, which contain only visible units. When no opponent is visible, spatial features are naturally zero. The continuous `scoutingVisibility` scalar is NOT applied to opponent spatial or ratio features — only to count/stat/upgrade features (indices 0–133).
- **Python training:** The `scouting_mask` from `fog_of_war.generate_scouting_mask()` is **continuous** (ramps from 0 → 0.3–0.5 at first scout → increasing toward 1.0 with each visit). Count/stat/upgrade features (indices 0–133) continue using this continuous mask. Spatial and ratio features (indices 134–143) require a **derived binary mask**:
  ```python
  spatial_mask = (scouting_mask > 0).astype(np.float32)  # 0 or 1
  ```
  When `spatial_mask = 0`: opponent spatial/ratio features are zero. When `spatial_mask = 1`: spatial/ratio features are computed from all units (full replay data). The continuous mask is semantically correct for counts (partial visibility ≈ scaled count) but meaningless for positions (a centroid × 0.3 is a nonsensical location).

Rationale: scaling a centroid position by a continuous visibility factor (e.g., 0.3) produces a nonsensical location — 30% of the way from the origin. Binary visibility (full value or zero) is the only semantically meaningful treatment for positions.

**Position sources:**
- Java inference: `Unit.position()` from `GameState.enemyUnits()` / `GameState.myUnits()`, `Building.position()` from building lists.
- Python training: `UnitBornEvent` position + `UnitPositionsEvent` periodic updates from SC2EGSet tracker events.

### 2. Temporal Delta Features (8 per window)

Per player (4 features × 2 players = 8):

| Feature | Computation |
|---------|-------------|
| `delta_army_supply` | Sum of `SC2Data.supplyCost()` for army units this window − previous window |
| `delta_worker_count` | Worker count this window − previous window |
| `delta_production_buildings` | Production building count this window − previous window |
| `delta_tech_buildings` | Non-production, non-base building count this window − previous window |

**Computation timing:** Deltas are **per-window** features — they compare aggregate values between consecutive windows. They cannot be computed at the per-tick level because window boundaries don't exist at tick granularity. The accumulator computes deltas after averaging per-tick features for the current window, comparing with the previous window's averaged values.

**Window 0:** All deltas = 0 (no previous window).

**Why these 4:** These are aggregate counts (sums across types), so per-window averages are smooth and deltas are stable. Per-unit-type deltas would be noisy (counts of individual types change by 0 or 1 per window).

**`delta_army_supply`** uses supply-weighted counts via `SC2Data.supplyCost()`, not raw unit counts. Supply-weighted counts distinguish between a zergling (0.5 supply) and a siege tank (3 supply), making the delta a better measure of army investment rate. This aligns with `army_supply_ratio` (§3) which also uses supply costs.

**Production buildings** (buildings that produce units):
- Terran: Barracks, Factory, Starport
- Zerg: Hatchery, Lair, Hive (produce larvae, which morph into units)
- Protoss: Gateway, Robotics Facility, Stargate

**Tech buildings** (prerequisite/tech-enabling buildings): all buildings with a non-empty `SC2Data.techTier()` that are not production buildings. Examples: Spawning Pool, Roach Warren, Hydralisk Den, Spire (Zerg); Engineering Bay, Armory, Ghost Academy (Terran); Cybernetics Core, Twilight Council, Templar Archives (Protoss).

**Prerequisite fix:** `SC2Data.techTier()` currently omits `CYBERNETICS_CORE` — it falls to the default branch and returns empty. Cybernetics Core gates Stalkers, Sentries, Adepts, WarpGate research, and all Protoss T2+ paths. Without this fix, building a Cybernetics Core would not register in `delta_tech_buildings`, silently dropping the first and most significant Protoss tech transition. Fix: add `case CYBERNETICS_CORE -> OptionalInt.of(1)` to the Protoss T1 tier in `SC2Data.techTier()`. This is a prerequisite for this spec's tech building classification.

Note: Spawning Pool, Roach Warren, Hydralisk Den, and Spire are **prerequisite** buildings in Zerg — they unlock unit types but do not produce units. Larvae are produced at Hatchery/Lair/Hive. The production/tech split captures production capacity growth vs tech investment separately.

**Opponent delta noise:** Opponent count features are scaled by `scoutingVisibility` (Java) or `scouting_mask` (Python) before window averaging. When visibility changes between windows, opponent deltas capture visibility change mixed with real composition change (e.g., a scout dying between windows produces a phantom supply drop). Player deltas are unaffected (full self-visibility). Opponent deltas are therefore noisier than player deltas. The model can learn to discount this noise — opponent deltas are still informative on average — but they should not be expected to be as clean as player deltas.

### 3. Structural Ratio Features (6 per window)

Per player (3 features × 2 players = 6):

| Feature | Computation | Edge case handling |
|---------|-------------|-------------------|
| `army_supply_ratio` | Σ(`SC2Data.supplyCost(unit)` for non-worker units) ÷ `food_used` | food_used clipped to min=1 |
| `worker_saturation` | `workersActiveCount` ÷ (`base_count` × 16) | denominator clipped to min=1 |
| `gas_mineral_ratio` | `vespene_spent` ÷ (`vespene_spent` + `minerals_spent`) | denominator clipped to min=1 |

**Source specifications:**

- **`army_supply_ratio`:** Numerator: sum of `SC2Data.supplyCost(type)` for all non-worker units (`!SC2Data.isWorker(type)`). Denominator: `PlayerEconomyStats.foodUsed()` (economy stat at feature index 111 = `scoreValueFoodUsed`). In Python: `food_used` from `STAT_KEYS[5]`.
- **`worker_saturation`:** Numerator: `PlayerEconomyStats.workersActiveCount()` (economy stat at feature index 112 = `scoreValueWorkersActiveCount`). In Python: `STAT_KEYS[6]`. Denominator: `base_count` × 16, where base count uses `SC2Data.isBase(type)` to identify base buildings.
- **`gas_mineral_ratio`:** `vespene_spent` = sum of `vespeneUsedCurrentArmy` + `vespeneUsedCurrentEconomy` + `vespeneUsedCurrentTechnology` (economy stats at feature indices 116–118 = `STAT_KEYS[10:13]`). `minerals_spent` = sum of `mineralsUsedCurrentArmy` + `mineralsUsedCurrentEconomy` + `mineralsUsedCurrentTechnology` (economy stats at feature indices 113–115 = `STAT_KEYS[7:10]`).

**Domain method reuse:** `SC2Data` already provides `supplyCost(UnitType)`, `isWorker(UnitType)`, and `isBase(BuildingType)`. The Java extractor uses these directly — no duplicate lookup tables in `FeatureIndexMaps`.

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
 player_spatial(7) | player_ratios(3) | player_deltas(4) |
 opponent_buildings(53) | opponent_units(53) | opponent_economy(13) | opponent_upgrades(15) |
 opponent_spatial(7) | opponent_ratios(3) | opponent_deltas(4) |
 army_gap(1) | has_vision(1)]
```

**Constants:**
- `N_TICK_FEATURES_PER_PLAYER = 144` — per-tick features stored in `WindowSnapshot` (134 original + 7 spatial + 3 ratios)
- `N_FEATURES_PER_PLAYER = 148` — total per-player features in the window vector (144 per-tick + 4 deltas)
- `FEATURES_PER_WINDOW = 2 × 148 + 1 (army_gap) + 1 (has_vision) = 298`

**Two-phase computation:**
1. **Tick averaging (per-tick features 0–143):** The accumulator averages `WindowSnapshot.playerFeatures[0:144]` across all ticks in a window. For opponent features: count/stat/upgrade features (indices 0–133) are scaled by the continuous `scoutingVisibility`. Spatial and ratio features (indices 134–143) are NOT scaled by `scoutingVisibility` — in Java, they are computed from visible units only (inherently binary); in Python, a derived binary mask (`scouting_mask > 0`) is applied instead of the continuous mask.
2. **Window assembly (per-window features 144–147, 292–297):** After tick averaging, the accumulator computes deltas by comparing the current window's averaged counts with the previous window's. `army_gap` is computed from the averaged player and opponent centroids. `has_vision` is set based on whether any tick in the window had `scoutingVisibility > 0`.

Per-tick features (spatial + ratios) are grouped before per-window features (deltas) within each player block, so the averaging loop cleanly covers indices 0–143 without touching delta slots.

**Breaking change:** All opponent feature indices shift by 14 (from starting at 134 to starting at 148). The `has_vision` flag moves from index 268 to 297. All tests, `norm_stats.json`, and models must be updated simultaneously — there is no partial migration path.

### Model Architecture

The model uses a **dual `ConvEncoder` architecture** that splits the temporal features at the `N_PLAYER_FEATURES` boundary:

```python
class StrategyClassifier(nn.Module):
    def __init__(self, f_temporal, f_map, num_classes, hp):
        f_opponent = f_temporal - N_PLAYER_FEATURES
        self.player_enc = ConvEncoder(N_PLAYER_FEATURES, ...)   # 134 → 148
        self.opponent_enc = ConvEncoder(f_opponent, ...)         # 135 → 150

    def encode(self, temporal, map_feat):
        player = temporal[:, :, :N_PLAYER_FEATURES]             # player block
        opponent = temporal[:, :, N_PLAYER_FEATURES:]           # opponent + cross-player
```

With the enriched layout:
- `player_enc`: 134 → 148 input features (player spatial, ratios, deltas)
- `opponent_enc`: 135 → 150 input features (opponent spatial, ratios, deltas + `army_gap` + `has_vision`)

The parameter increase per encoder is ~10%, negligible overall.

**Cross-player feature routing:** `army_gap` and `has_vision` are processed by the opponent encoder. This is the existing pattern — `has_vision` has always been in the opponent encoder's input (it was the 135th feature, at `f_temporal - N_PLAYER_FEATURES = 135`). The learned gate mechanism (`self.gate = nn.Linear(2, 2)`) modulates each encoder's contribution based on availability flags, handling cases where opponent data is missing. No architecture change is required.

`MapCharacteristics` is unchanged (4 features + 2 availability flags).

### Normalization

`norm_stats.json` must be regenerated from training data with the new features. The existing mean/std entries are no longer valid — all 298 entries must be recomputed.

**Non-normalizable features:** Only `has_vision` (index 297) is non-normalizable — it is a binary flag with `mean=0.0, std=1.0`. The code identifies it by explicit index:
```python
HAS_VISION_IDX = 2 * N_PLAYER_FEATURES + 1  # 2 * 148 + 1 = 297
mean[HAS_VISION_IDX] = 0.0
std[HAS_VISION_IDX] = 1.0
```

**`army_gap` normalization:** `army_gap` (index 296) IS a continuous feature — it is z-score normalized like all other features. It receives map-relative normalization (÷ map diagonal) before z-score, same as other spatial features.

Spatial features use map-relative normalization (÷ map diagonal or [0,1] by map bounds) BEFORE z-score normalization. This prevents map-size variation from dominating the feature statistics.

### Changes Required

**Java — quarkmind-sc2 (inference path):**

1. **`WindowSnapshot`** — Expand to carry `N_TICK_FEATURES_PER_PLAYER = 144` features per player (was 134). Compute spatial aggregates and ratios directly in `DroolsScoutingTask.buildSnapshot()` and store the 7 spatial + 3 ratio scalar features in `playerFeatures[134:144]` and `opponentFeatures[134:144]`. No variable-length position arrays — aggregates are computed in-place.

2. **`FeatureIndexMaps`** — Update `N_FEATURES_PER_PLAYER` from 134 to 148. Add `N_TICK_FEATURES_PER_PLAYER = 144`. Add `FEATURES_PER_WINDOW = 298`. No new lookup tables — use `SC2Data.supplyCost()`, `SC2Data.isBase()`, `SC2Data.isWorker()`, and `SC2Data.techTier()` from the existing domain layer.

3. **`TemporalWindowAccumulator`** — Two-phase window assembly:
   - **Phase 1 (tick averaging):** Average `playerFeatures[0:144]` directly. Average opponent count/stat/upgrade features (indices 0–133) with `scoutingVisibility` scaling. Average opponent spatial/ratio features (indices 134–143) WITHOUT visibility scaling (binary visibility is inherent).
   - **Phase 2 (window assembly):** Compute player deltas (indices 144–147) and opponent deltas (indices 292–295) by comparing current window averages with previous window. Compute `army_gap` (index 296) from averaged centroids. Set `has_vision` (index 297).

4. **`StrategyFeatureExtractor`** — Update to handle 298 features per window. Normalization skips `has_vision` at explicit index 297 (replaces the current `HAS_VISION_OFFSET = FEATURES_PER_WINDOW - 1` which remains correct).

5. **`DroolsScoutingTask.buildSnapshot()`** — Extend to compute spatial aggregates from `GameState` unit/building positions and ratio features from unit counts + economy stats. Uses `SC2Data.supplyCost()` for army supply computation, `SC2Data.isBase()` for base count, `SC2Data.isWorker()` for worker identification.

**Python — neocortex (training path):**

1. **`sc2egset_extractor.py`** — Extract unit positions from `UnitBornEvent` and `UnitPositionsEvent` tracker events. Maintain a position map (unit tag → (x, y)) updated on each position event. Compute per-second spatial aggregates (centroid, spread, distances, proxy score) and ratio features. Export two constants:
   - `N_FEATURES_PER_PLAYER = 148` — the per-player block width in the window vector. This is the constant that `model.py`, `normalize.py`, and `dataset.py` import for the encoder split, normalization indices, and dropout boundaries.
   - `N_TICK_FEATURES_PER_PLAYER = 144` — the per-second feature count used by the extractor's own per-second arrays and by `feature_engineering.py` for the tick-averaging phase of window assembly.
   
   The per-second data arrays remain 144-wide. The extractor's internal loops use `N_TICK_FEATURES_PER_PLAYER`. All downstream consumers (`model.py`, `normalize.py`, `dataset.py`) continue importing `N_FEATURES_PER_PLAYER` and get 148 — the window-level constant they need for the encoder split at `temporal[:, :, :N_PLAYER_FEATURES]`.

2. **`feature_engineering.py`** — Update `build_temporal_features()` with two-phase window assembly. Imports both `N_FEATURES_PER_PLAYER` (148) and `N_TICK_FEATURES_PER_PLAYER` (144) from the extractor:
   - Average per-second features (`N_TICK_FEATURES_PER_PLAYER = 144` per player) into window means. For opponent features, apply the continuous `scouting_mask` to count/stat/upgrade features (indices 0–133) and a derived binary mask (`scouting_mask > 0`) to spatial/ratio features (indices 134–143).
   - Compute deltas (4 per player) from consecutive window means.
   - Compute `army_gap` from averaged centroids.
   - Concatenate: `[player_avg(144), player_deltas(4), opponent_avg(144), opponent_deltas(4), army_gap(1), has_vision(1)]` = 298.

3. **`normalize.py`** — `compute_stats()` handles 298 features per window. Uses explicit index `HAS_VISION_IDX = 2 * 148 + 1 = 297` for the non-normalizable flag. Replaces the current `vis_idx = 2 * N_PLAYER_FEATURES` arithmetic which would point to `army_gap` (index 296) instead of `has_vision` with the new layout.

4. **`dataset.py` — `ModalityDropoutDataset`** — When zeroing the opponent block (`temporal[:, n:2*n]`), also zero `army_gap` at index `2*n` (it depends on opponent centroid). When zeroing the player block (`temporal[:, :n]`), also zero `army_gap` at index `2*n` (it depends on player centroid). Zero `has_vision` at index `2*n+1` during opponent dropout (opponent visibility is meaningless when opponent features are dropped).

5. **`export_onnx.py`** — `f_temporal` parameter changes from 269 to 298. ONNX input shape adapts automatically via the existing dynamic axes. The `model_manifest.json` must update `"f_temporal": 298` — the Java side reads this manifest for tensor dimension validation, and a stale value of 269 would cause a validation failure.

6. **`config.py`** — No changes to hyperparameters or window structure.

**Shared data files:**

1. **`norm_stats.json`** — Regenerated with 298 entries (was 269). Placed in `quarkmind-sc2/src/main/resources/classifier/`.

2. **`strategy_vs_*.onnx`** — 3 retrained models placed in `quarkmind-sc2/src/test/resources/models/strategy/`.

3. **`supply_costs.json`** — Generated from `SC2Data.supplyCost()` (authoritative source) as a shared artifact. A generation script dumps `{unit_type_name: supply_cost}` for all `UnitType` enum values, eliminating dual-maintenance of supply cost mappings. Placed in `quarkmind-sc2/src/main/resources/classifier/supply_costs.json`. 

   **Cross-repo distribution:** `quarkmind-sc2` and `neocortex` are separate repositories — symlinks across repos don't work in CI (independent checkouts). The `FeatureAlignmentTest` setup step copies `supply_costs.json` from `quarkmind-sc2`'s classpath to the Python test fixture directory. The training pipeline reads it from a configurable path (defaulting to `../quarkmind-sc2/src/main/resources/classifier/supply_costs.json` for local development, overridable via `Paths.supply_costs` for CI). The alignment test verifies that the Python supply cost dict matches the JSON artifact.

### Alignment Verification

**Critical constraint:** Java and Python feature extractors must produce identical feature vectors for the same game state. Any divergence causes a train/inference distribution mismatch that silently degrades accuracy.

**Alignment test (new):** A plain JUnit test (`FeatureAlignmentTest`) that:
1. Loads a known replay
2. Extracts features at a fixed game time using the Java pipeline
3. Compares against a pre-computed `.npz` file generated by the Python pipeline for the same replay at the same time
4. Asserts feature-level equality within floating-point tolerance (1e-5)

The alignment test must cover all unit types present in `supply_costs.json`, not just those in a single replay. A supplementary fixture verifies that the Python supply cost dict matches the JSON artifact.

This test is the regression gate — it must pass before any ONNX model update is accepted.

### Calibration

**Drools-tier baseline (existing):** `PatternClassificationCalibrationTest` measures Drools rule-based classification accuracy using `CascadingPatternClassifier.computeAllConfidences()` and `mergeCumulative()`. This test does NOT instantiate ONNX models or call `TensorClassifier.classify()`. It is unaffected by feature vector changes since Drools uses rule evidence, not the feature tensor. Run before and after as a regression gate for the Drools tier.

**ONNX-tier calibration (new):** A new `OnnxClassificationCalibrationTest` that:
1. Loads the trained ONNX model for each matchup
2. Runs inference on the same replay set used by `PatternClassificationCalibrationTest`
3. Compares ONNX predictions against ground truth labels
4. Reports per-archetype accuracy at the 3-min, 8-min, and 15-min marks

Success criteria: >= 70% accuracy for rush and air-threat archetypes at 3-min mark (same threshold as Drools tier). Expected improvement: spatial features should improve rush detection significantly since spatial intent is the primary rush discriminator.

### What This Does NOT Change

- **Cascade architecture** — Drools → ONNX → LLM tiers unchanged.
- **ONNX model architecture** — Same dual-encoder 1D-CNN (`StrategyClassifier`), wider input per encoder.
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
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/SC2Data.java` — supply costs, base/worker identification, tech tiers
- `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py` — Python feature extraction from replays
- `neocortex/evaluation/strategy_classifier/feature_engineering.py` — Python temporal feature assembly
- `neocortex/evaluation/strategy_classifier/normalize.py` — normalization stat computation
- `neocortex/evaluation/strategy_classifier/dataset.py` — ModalityDropoutDataset (40% drop)
- `neocortex/evaluation/strategy_classifier/model.py` — dual ConvEncoder StrategyClassifier
- `neocortex/evaluation/strategy_classifier/export_onnx.py` — ONNX export and manifest
- `neocortex/evaluation/strategy_classifier/run_pipeline.py` — training pipeline orchestrator
- #298 — UnitPositions tracker events (enemy units move during replay)
- #300 — Spatial recalibration (corrected enemy positions)
- #208 — ONNX strategy classifier epic
- #212 — Three-tier confidence cascade

**Spatial calibration protocol:** The proposed spatial features use map-relative normalization (÷ map diagonal, ÷ map width/height) and threshold-free relative comparisons (`proxy_building_score` compares distances without fixed thresholds). No new spatial constants are introduced. The calibration protocol (`sc2data-spatial-constants-require-calibration`) is satisfied — all spatial values are normalized by map dimensions, making them map-size-invariant. The protocol's requirement for calibration tests applies only to fixed spatial thresholds (e.g., "within 30 game units"), which this design avoids.
