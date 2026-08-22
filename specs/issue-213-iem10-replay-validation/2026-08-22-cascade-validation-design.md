# IEM10 Replay Validation & Accuracy Benchmarking — Design Spec

**Issue:** #213
**Epic:** #208 (ONNX-based learned strategy classifier)
**Branch:** `issue-213-iem10-replay-validation`

## Summary

Wire the ONNX strategy classifier models into QuarkMind's three-tier cascade
(Drools → ONNX → LLM) and validate accuracy against replay datasets. Three
deliverables: feature extractor rewrite + race-aware cascade routing, cascade
accuracy validation test, and cascade latency benchmark.

## Deliverables

1. **Feature extractor rewrite + race-aware cascade routing** — rewrite
   `StrategyFeatureExtractor` to produce the 269-feature temporal windows + 6
   map features expected by the ONNX models. Update `CascadingPatternClassifier`
   to route to per-race models.

2. **Cascade accuracy validation** — `CascadeValidationCalibrationTest` runs all
   59 replays through the cascade at minutes 1–5, in three modes (Drools-only,
   ONNX-only, cascade). Reports per-matchup accuracy, tier hit rates, and
   comparison baselines.

3. **Cascade latency benchmark** — `CascadeLatencyBenchmarkTest` measures
   end-to-end classification latency (feature extraction + ONNX inference).

---

## 1. StrategyFeatureExtractor Rewrite

### API

```java
public class StrategyFeatureExtractor {

    public record StrategyFeatures(
        Map<String, float[][]> tensors  // "temporal" → [1][2690], "map" → [1][6]
    ) {}

    public StrategyFeatures extract(
        List<WindowSnapshot> windows,
        MapCharacteristics map,
        boolean hasVision
    );
}
```

### Feature Vector Layout

Matches the neocortex training pipeline (`sc2egset_extractor.py`).

**Per-player block (134 features):**

| Range | Count | Content |
|-------|-------|---------|
| [0..52] | 53 | Building counts (live count) |
| [53..105] | 53 | Unit counts (live count) |
| [106..118] | 13 | Economy stats (value / 1000.0) |
| [119..133] | 15 | Upgrade flags (0/1) |

Building list (53): CommandCenter, OrbitalCommand, PlanetaryFortress, Barracks,
Factory, Starport, EngineeringBay, Armory, GhostAcademy, FusionCore, Bunker,
MissileTurret, SensorTower, SupplyDepot, Refinery, BarracksReactor,
BarracksTechLab, FactoryReactor, FactoryTechLab, StarportReactor,
StarportTechLab, Hatchery, Lair, Hive, SpawningPool, BanelingNest,
RoachWarren, EvolutionChamber, Extractor, HydraliskDen, SpineCrawler,
SporeCrawler, Spire, GreaterSpire, InfestationPit, NydusNetwork,
UltraliskCavern, Nexus, Gateway, WarpGate, CyberneticsCore, Forge,
Assimilator, Pylon, PhotonCannon, ShieldBattery, RoboticsFacility,
RoboticsBay, Stargate, FleetBeacon, TwilightCouncil, TemplarArchive,
DarkShrine.

Unit list (53): SCV, Marine, Marauder, Reaper, Ghost, Hellion, HellionTank,
SiegeTank, Cyclone, Thor, Medivac, VikingFighter, VikingAssault, Liberator,
Banshee, Raven, WidowMine, Drone, Zergling, Baneling, Roach, Ravager, Queen,
Mutalisk, Corruptor, BroodLord, Hydralisk, Lurker, Infestor, SwarmHost,
Ultralisk, Viper, Overlord, Overseer, Probe, Zealot, Stalker, Sentry, Adept,
HighTemplar, DarkTemplar, Archon, Immortal, Colossus, Disruptor, WarpPrism,
Phoenix, Oracle, VoidRay, Carrier, Tempest, Mothership, Observer.

Economy stats (13): scoreValueMineralsCurrent, scoreValueVespeneCurrent,
scoreValueMineralsCollectionRate, scoreValueVespeneCollectionRate,
scoreValueFoodMade, scoreValueFoodUsed, scoreValueWorkersActiveCount,
scoreValueMineralsUsedCurrentArmy, scoreValueMineralsUsedCurrentEconomy,
scoreValueMineralsUsedCurrentTechnology, scoreValueVespeneUsedCurrentArmy,
scoreValueVespeneUsedCurrentEconomy, scoreValueVespeneUsedCurrentTechnology.

Upgrades (15): Stimpack, ShieldWall, PunisherGrenades, BansheeCloak,
TerranVehicleWeaponsLevel1, PersonalCloaking, DrillClaws,
zerglingmovementspeed, GlialReconstitution, CentrificalHooks, Burrow,
WarpGateResearch, BlinkTech, Charge, AdeptPiercingAttack.

**Per-window (269 features):**

| Range | Count | Content |
|-------|-------|---------|
| [0..133] | 134 | Player features (mean over 30-second window) |
| [134..267] | 134 | Opponent features (scouting-masked, then mean) |
| [268] | 1 | has_vision flag (1.0 if scouting in window) |

### Temporal Windowing

- 10 windows of 30 seconds each (max_windows=10, window_seconds=30)
- At minute 2: 4 windows populated, 6 zero-padded
- At minute 3: 6 windows populated, 4 zero-padded
- At minute 4: 8 windows populated, 2 zero-padded
- At minute 5: 10 windows populated
- Zero-padded windows are handled by the model's internal padding mask
- ONNX input flattened: [10, 269] → [1, 2690]

### WindowSnapshot

New record capturing per-second game state within a 30-second window.
Accumulated by the classification loop in `DroolsScoutingTask` each tick.

```java
public record WindowSnapshot(
    float[] playerFeatures,   // 134 elements
    float[] opponentFeatures, // 134 elements
    float scoutingVisibility  // [0, 1]
) {}
```

The extractor averages per-second snapshots within each window, applies the
scouting mask to opponent features, and sets the has_vision flag.

### Building/Unit Name Mapping

Static lookup maps from `UnitType`/`BuildingType` enum → feature index, matching
the exact ordering from `sc2egset_extractor.py`. Units/buildings not in the
feature list are ignored (the Java enums have types the model wasn't trained on).

### Scouting Mask

In replays, true fog-of-war data is unavailable. The training pipeline uses a
synthetic cumulative ramp. For replay validation: a float in [0,1] that increases
as enemy units are first observed. Count of unique enemy unit types seen / total
expected types gives the visibility fraction. The `EnemyUnitFirstSeen` events
already track first observations.

### Normalization

The model was trained with mean/std normalization. Export `norm_stats.npz` values
as a JSON resource file at `src/main/resources/classifier/norm_stats.json`.
`StrategyFeatureExtractor` applies z-score normalization before producing the
final tensor. The has_vision flag at index 268 is not normalized (mean=0, std=1).

### Map Features (6)

```java
public record MapCharacteristics(
    float rushDistance,  // short=0.0, medium=0.5, long=1.0
    float expansions,   // count / 10.0
    float size,         // small=0.0, medium=0.5, large=1.0
    float choke         // wall_off=1.0, open=0.0
) {
    public float[] toArray(boolean hasPlayer, boolean hasOpponent) {
        return new float[] {
            rushDistance, expansions, size, choke,
            hasPlayer ? 1.0f : 0.0f,
            hasOpponent ? 1.0f : 0.0f
        };
    }
}
```

Static catalog of 15 known maps (matching neocortex `MAP_CATALOG`). Unknown maps
get medium defaults (0.5, 0.5, 0.5, 0.5).

---

## 2. Race-Aware Cascade Routing

### Per-Race Model Injection

Replace single `@Inference("strategy-classifier")` with three:

```java
@Inference("strategy-vs-terran")  Instance<InferenceModel> terranModel
@Inference("strategy-vs-zerg")    Instance<InferenceModel> zergModel
@Inference("strategy-vs-protoss") Instance<InferenceModel> protossModel
```

Each wraps in a `TensorClassifier` with the correct post-consolidation labels.
If any model is unavailable, that race's ONNX tier is skipped.

### Post-Consolidation Class Labels

| Matchup | Classes | Labels |
|---------|---------|--------|
| vs Terran | 5 | RUSH, BANSHEE_HARASS, AIR_SUPERIORITY, MECH_PUSH, BIO_TIMING |
| vs Zerg | 6 | RUSH, ROACH_RUSH, LING_BANE, MUTA_HARASS, HYDRA_PUSH, MACRO_ECONOMY |
| vs Protoss | 7 | RUSH, PROXY, CANNON_RUSH, DT_RUSH, BLINK_STALKER, COLOSSUS_PUSH, AIR_SUPERIORITY |

### Label-to-StrategyArchetype Mapping

Static map per race. Model labels are race-generic (e.g., "RUSH"); archetype
enum values are race-specific (e.g., `TERRAN_MARINE_RUSH`). Add new
`StrategyArchetype` values where no direct mapping exists for consolidated labels.

### classify() Signature Change

```java
public CascadeResult classify(
    List<EvidenceMarker> evidence,
    List<ConfidenceRevision> revisions,
    StrategyFeatures onnxFeatures,  // was Map<String, float[][]>
    Race enemyRace,                  // NEW
    long frame, long prevFrame,
    CaseContext ctx)
```

The cascade selects the `TensorClassifier` for the given enemy race. If no model
for that race, ONNX tier is skipped.

### DroolsScoutingTask Changes

- Pass enemy race (from `CaseContext` key `game.enemy.race` or `GameState`) into
  `cascadingClassifier.classify()`
- Accumulate `WindowSnapshot` instances each tick for temporal windowing
- Pass `StrategyFeatures` (two-tensor map) instead of the old single-key map

### Configuration

```properties
quarkmind.classifier.onnx.model.vs-terran=strategy-vs-terran
quarkmind.classifier.onnx.model.vs-zerg=strategy-vs-zerg
quarkmind.classifier.onnx.model.vs-protoss=strategy-vs-protoss
```

---

## 3. Cascade Accuracy Validation

### Shared Infrastructure

`ReplayClassificationTestSupport` — plain utility class extracted from
`PatternClassificationCalibrationTest`:

- `deriveGroundTruth(Map<UnitType, Integer> enemyCounts, double gameTimeMin, String matchup)` → `StrategyArchetype`
- `loadIEM10Games(Path zipPath)` → `List<GameEntry>`
- `loadAIArenaGames(Path dir)` → `List<GameEntry>` (all PvP)
- `ClassificationResult` record (groundTruth, prediction, correct, confidence)

`PatternClassificationCalibrationTest` refactored to use this utility. No
behavioral change.

### CascadeValidationCalibrationTest

`@QuarkusTest`, `@Tag("benchmark")`.

**Validation loop:** For each of 59 replay games, run classification at minutes
1, 2, 3, 4, 5 in three configurations:

1. **Drools-only** — `droolsThreshold=0.0`, ONNX + LLM disabled. Drools always
   resolves.
2. **ONNX-only** — `droolsThreshold=1.0`, `onnxThreshold=0.0`, LLM disabled.
   ONNX always resolves. Skipped if model unavailable.
3. **Cascade** — default thresholds (Drools 0.7, ONNX 0.5). Production routing.

`cascadingClassifier.reset()` between modes to clear cumulative state.

**Output tables:**

Per-mode accuracy:
```
Mode: DROOLS_ONLY
Minute | PvT Acc | PvZ Acc | PvP Acc | Overall | Samples
  1    | ...     | ...     | ...     | ...     | 59
  ...
  5    | ...     | ...     | ...     | ...     | 59
```

Tier hit rates (cascade mode, per-minute):
```
Tier Hit Rates (minute 4):
  Drools resolved:  N% (n/59)
  ONNX resolved:    N% (n/59)
  LLM triggered:    N% (n/59)
  Default fallback: N% (n/59)
```

Comparison baselines:
```
Accuracy at minute 4:
  Drools-only: N%
  ONNX-only:   N%
  Cascade:     N%
  Δ Cascade vs Drools-only: +N%
```

**Assertions:**
- Hard: Drools-only rush accuracy ≥ 70% at minute 3
- Hard: Drools-only air-threat accuracy ≥ 70% at minute 3 (if samples exist)
- Informational: all other metrics

### Output

- Stdout tables during test run
- Dated snapshot committed to `docs/benchmarks/YYYY-MM-DD-cascade-validation.md`

---

## 4. Cascade Latency Benchmark

### CascadeLatencyBenchmarkTest

Plain JUnit (no `@QuarkusTest`), `@Tag("benchmark")`.

**Setup:**
- Construct `CascadingPatternClassifier` directly (test constructor)
- Load one ONNX model from test resources
- Prepare realistic feature input from a single replay run to minute 3

**Measurement:**
- 100 warmup iterations
- 1000 measured iterations at nanosecond granularity
- Three components measured separately:
  - Feature extraction alone (`extract()`)
  - ONNX inference alone (`TensorClassifier.classify()`)
  - Full cascade (`classify()`)

**Output:**
```
Cascade Latency Benchmark (vs_terran model, minute 3 features):
  Component          | Mean    | p50     | p95     | p99     | Max
  Feature extraction | ...     | ...     | ...     | ...     | ...
  ONNX inference     | ...     | ...     | ...     | ...     | ...
  Full cascade       | ...     | ...     | ...     | ...     | ...
```

**Assertions:**
- p99 < 10ms for full cascade
- p99 < 1ms for feature extraction alone

Results also written to `target/cascade-latency-benchmark.txt` and included in
the dated `docs/benchmarks/YYYY-MM-DD-cascade-validation.md` snapshot.

---

## Testing

### New test classes
- `StrategyFeatureExtractorTest` — unit test for the rewritten extractor. Verifies
  feature vector dimensions, building/unit index mapping, temporal windowing,
  scouting mask application, normalization, map feature encoding.
- `CascadeValidationCalibrationTest` — accuracy validation (@QuarkusTest, @Tag("benchmark"))
- `CascadeLatencyBenchmarkTest` — latency benchmark (@Tag("benchmark"))

### Modified test classes
- `PatternClassificationCalibrationTest` — refactored to use `ReplayClassificationTestSupport`
- `CascadingPatternClassifierTest` — updated for new `classify()` signature (Race parameter)
- `DroolsScoutingTaskTest` / `DroolsScoutingTaskIT` — updated for Race parameter plumbing

### Run commands
- `mvn test -pl quarkmind-sc2` — unit tests (includes StrategyFeatureExtractorTest)
- `mvn test -pl quarkmind-sc2 -Pbenchmark` — all benchmarks including new cascade tests

---

## ONNX Model Artifacts

Copy the three `.onnx` files from neocortex into quarkmind-sc2 test resources:

```
quarkmind-sc2/src/test/resources/models/strategy/
  strategy_vs_terran.onnx
  strategy_vs_zerg.onnx
  strategy_vs_protoss.onnx
```

For production, models are resolved via CDI `@Inference` qualifier — the
`inference-quarkus` module handles model loading from configured paths.

---

## Acceptance Criteria

From the issue, adjusted for current state:

- [ ] StrategyFeatureExtractor produces 269×10+6 feature tensors matching neocortex training pipeline
- [ ] CascadingPatternClassifier routes to per-race ONNX models
- [ ] Drools-only rush accuracy ≥ 70% at minute 3 (hard-asserted)
- [ ] Drools-only air-threat accuracy ≥ 70% at minute 3 (hard-asserted, if samples exist)
- [ ] ONNX-only and cascade accuracy reported per matchup at minutes 1–5
- [ ] Tier hit rate report showing classification resolution distribution across tiers
- [ ] Latency benchmark: < 10ms p99 for full cascade on CPU
- [ ] Comparison report: Drools-only vs ONNX-only vs cascade accuracy
- [ ] Dated results snapshot committed to docs/benchmarks/YYYY-MM-DD-cascade-validation.md

---

## References

- `CascadingPatternClassifier.java` — three-tier cascade implementation
- `StrategyFeatureExtractor.java` — current flat-vector extractor (to be rewritten)
- `PatternClassificationCalibrationTest.java` — existing calibration test (refactored)
- `neocortex/evaluation/strategy_classifier/sc2egset_extractor.py` — feature definitions
- `neocortex/evaluation/strategy_classifier/feature_engineering.py` — temporal windowing
- `neocortex/evaluation/strategy_classifier/map_features.py` — map feature catalog
- `neocortex/evaluation/strategy_classifier/normalize.py` — normalization stats
- `neocortex/inference-runtime/.../StrategyClassifierOnnxTest.java` — model input shapes
- `neocortex/evaluation/strategy_classifier/output/model_manifest.json` — model manifest
- GE-20260528-6ebb38 — SC2EGSet JSON extra UnitBorn at loop 0 gotcha
