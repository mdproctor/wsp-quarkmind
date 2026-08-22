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

### Data Availability (R1-02)

The 134-feature vector requires data beyond what `GameState` currently carries:

**Available now:**
- Building counts — from `GameState.buildings` / `GameState.enemyBuildings`
- Unit counts — from `GameState.units` / `GameState.enemyUnits`
- Basic economy (minerals, vespene, supply) — from `GameState`

**Requires extraction from replay data:**
- Economy stats (13) — IEM10 JSON carries all 13 fields in `PlayerStats` tracker
  events. Extend `IEM10JsonSimulatedGame.applyPlayerStats()` to extract them into
  a new `PlayerEconomyStats` record on `GameState`. For AI Arena binary replays,
  `ReplaySimulatedGame` already processes `PlayerStats` — extend similarly.
- Upgrade flags (15) — IEM10 JSON carries `UpgradeEvent` tracker events. Add
  `UpgradeEvent` processing to `IEM10JsonSimulatedGame` and track active upgrades
  in a `Set<String>` on `GameState`. For AI Arena binary replays, process
  `UpgradeCompleteEvent` from Scelight tracker events.

**Live SC2 (future, not in #213 scope):**
- Economy stats available via SC2 API `ScoreDetails` — wire through
  `ObservationTranslator` → `GameStateTranslator`. Separate issue.
- Upgrades available via SC2 API `UpgradeId` — separate issue.

**Opponent economy/upgrades in replay context:** Both players' full stats are
available in replay tracker data (no fog of war). The scouting mask still applies
to opponent features — it models what the classifier would see in a live game.

### Temporal Windowing

- 10 windows of 30 seconds each (max_windows=10, window_seconds=30)
- At minute 1: 2 windows populated, 8 zero-padded
- At minute 2: 4 windows populated, 6 zero-padded
- At minute 3: 6 windows populated, 4 zero-padded
- At minute 4: 8 windows populated, 2 zero-padded
- At minute 5: 10 windows populated
- Zero-padded windows are handled by the model's internal padding mask
- ONNX input flattened: [10, 269] → [1, 2690]

### WindowSnapshot and Accumulation (R1-03)

New record capturing a single tick's game state for temporal windowing:

```java
public record WindowSnapshot(
    float[] playerFeatures,   // 134 elements
    float[] opponentFeatures, // 134 elements
    float scoutingVisibility  // [0, 1]
) {}
```

**Accumulation mechanism:** A new `TemporalWindowAccumulator` class manages
snapshot collection and windowing:

```java
public class TemporalWindowAccumulator {
    private static final int MAX_WINDOWS = 10;
    private static final int WINDOW_SECONDS = 30;
    private static final int TICKS_PER_WINDOW = 60; // 30s / 0.5s per tick

    private final List<WindowSnapshot> tickSnapshots = new ArrayList<>();

    public void addSnapshot(WindowSnapshot snapshot);
    public List<float[]> getWindowedFeatures(); // 10 × 269, zero-padded
    public void reset();
}
```

- **Storage:** `TemporalWindowAccumulator` is a field on `DroolsScoutingTask`,
  reset on game start.
- **Tick-to-window mapping:** Index-based. Tick N falls in window
  `N / TICKS_PER_WINDOW`. Each tick produces one snapshot. Within a window,
  all tick snapshots are averaged to produce the 269-element feature vector.
- **"Per-second" vs tick granularity:** Ticks fire at ~500ms intervals. The
  training pipeline uses per-second PlayerStats. Two ticks ≈ one second — the
  averaging within windows smooths this difference. No interpolation needed.
- **Zero-padding:** `getWindowedFeatures()` always returns exactly 10 windows.
  Unpopulated windows are zeroed arrays. The ONNX model's internal padding mask
  (`temporal.abs().sum(dim=-1) == 0`) detects and ignores them.
- **Window averaging:** The extractor receives the raw tick snapshots from the
  accumulator, groups by window index, and computes the mean of player/opponent
  features within each window. Scouting mask is applied to opponent features
  before averaging. `has_vision` is 1.0 if any snapshot in the window had
  `scoutingVisibility > 0`.

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

### Label-to-StrategyArchetype Mapping (R1-05, R1-09)

The ONNX models output coarse consolidated labels (e.g., "RUSH"). The Drools tier
tracks fine-grained archetypes (e.g., `TERRAN_MARINE_RUSH`). The mapping must
resolve this without confidence fragmentation.

**Approach: ONNX labels map to the broadest existing archetype per race.** When
the ONNX tier resolves, its confidence replaces (not accumulates alongside)
any Drools confidence for archetypes in the same consolidation group. This
prevents fragmentation — a single archetype represents the classification.

**Mapping table:**

| Race | ONNX Label | Maps to StrategyArchetype | Notes |
|------|-----------|--------------------------|-------|
| Terran | RUSH | TERRAN_MARINE_RUSH | Broadest early Terran rush |
| Terran | BANSHEE_HARASS | TERRAN_BANSHEE_HARASS | Direct match |
| Terran | AIR_SUPERIORITY | TERRAN_AIR_SUPERIORITY | **New enum value** |
| Terran | MECH_PUSH | TERRAN_MECH_PUSH | **New enum value** |
| Terran | BIO_TIMING | TERRAN_BIO_TIMING | **New enum value** |
| Zerg | RUSH | ZERG_ZERGLING_RUSH | Broadest early Zerg rush |
| Zerg | ROACH_RUSH | ZERG_ROACH_RUSH | Direct match |
| Zerg | LING_BANE | ZERG_LING_BANE | **New enum value** |
| Zerg | MUTA_HARASS | ZERG_MUTA_HARASS | **New enum value** |
| Zerg | HYDRA_PUSH | ZERG_HYDRA_PUSH | **New enum value** |
| Zerg | MACRO_ECONOMY | ZERG_MACRO | Existing |
| Protoss | RUSH | PROTOSS_GATEWAY_RUSH | Broadest early Protoss rush |
| Protoss | PROXY | PROTOSS_PROXY_GATE | Existing |
| Protoss | CANNON_RUSH | PROTOSS_CANNON_RUSH | **New enum value** |
| Protoss | DT_RUSH | PROTOSS_DT_RUSH | Existing |
| Protoss | BLINK_STALKER | PROTOSS_BLINK_STALKER | **New enum value** |
| Protoss | COLOSSUS_PUSH | PROTOSS_COLOSSUS_PUSH | **New enum value** |
| Protoss | AIR_SUPERIORITY | PROTOSS_AIR_SUPERIORITY | **New enum value** |

**New StrategyArchetype enum values (10):** All tagged with their race and
`GamePhase.EARLY` or `GamePhase.MID` as appropriate, `ArchetypeCategory` matching
the strategy type. All switch expressions over `StrategyArchetype` must be updated
in the same commit (sealed-type exhaustiveness rule).

**Confidence fragmentation prevention:** When the ONNX tier fires and maps to
e.g. `TERRAN_MARINE_RUSH`, the cascade merges its confidence via `Math::max`
into the cumulative map at that key. Because both Drools and ONNX now target the
same archetype key, confidence accumulates rather than fragmenting. The Drools
rules that fire for specific rush indicators (proxy barracks, reaper harass)
may produce confidence on different fine-grained archetypes — but the ONNX
mapping targets the broadest match, which the cumulative max-merge handles
correctly.

**StrategyTaxonomy interaction (R1-14):** ONNX assessments bypass the taxonomy.
The taxonomy governs which Drools `SignatureSpec` rules are active per game phase.
The ONNX tier classifies from raw features independently — it does not consult
the taxonomy. The `PatternAssessment` output carries `AssessmentSource.ONNX` so
consumers can distinguish the source if needed.

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

### Configuration (R1-08)

Neocortex model loading via `InferenceModelProducer`:

```properties
# Model paths — resolved by neocortex inference-quarkus
casehub.inference.models.strategy-vs-terran.model-path=models/strategy/strategy_vs_terran.onnx
casehub.inference.models.strategy-vs-zerg.model-path=models/strategy/strategy_vs_zerg.onnx
casehub.inference.models.strategy-vs-protoss.model-path=models/strategy/strategy_vs_protoss.onnx
```

For `%test` profile, model paths resolve to test resources:
```properties
%test.casehub.inference.models.strategy-vs-terran.model-path=${project.basedir}/src/test/resources/models/strategy/strategy_vs_terran.onnx
%test.casehub.inference.models.strategy-vs-zerg.model-path=${project.basedir}/src/test/resources/models/strategy/strategy_vs_zerg.onnx
%test.casehub.inference.models.strategy-vs-protoss.model-path=${project.basedir}/src/test/resources/models/strategy/strategy_vs_protoss.onnx
```

Raw tensor models do not use tokenizers — `tokenizerPath` is omitted (the
`InferenceModelConfig` treats it as optional for non-NLP models).

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

`cascadingClassifier.reset()` between every game AND between modes to clear
cumulative state. The classifier has persistent state (`cumulativeConfidence`,
`lastLlmFallbackFrame`) that would contaminate subsequent games without reset.

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

**AI Arena separate reporting (R1-04):**

AI Arena replays (29 games, all PvP) are reported separately from IEM10 games.
Bot strategies are fixed build orders — accuracy should be significantly higher.

```
AI Arena Bot Validation (PvP, 29 games):
Mode         | Min 3 Acc | Min 4 Acc | Min 5 Acc
Drools-only  | ...       | ...       | ...
ONNX-only    | ...       | ...       | ...
Cascade      | ...       | ...       | ...
```

**Assertions:**
- Hard: Drools-only rush accuracy ≥ 70% at minute 3
- Hard: Drools-only air-threat accuracy ≥ 70% at minute 3 (if samples exist)
- Hard: AI Arena cascade accuracy ≥ 80% at minute 4 (bot builds are predictable)
- Informational: all other metrics including overall accuracy across all archetypes

**Deferred criterion (R1-06):** The issue's "≥ 60% top-1 accuracy across all
archetypes at minute 4" is reported informationally but not hard-asserted. The
IEM10 dataset is from 2016 — meta has shifted significantly, and the model was
trained on modern replays. Asserting cross-era accuracy is premature until the
baseline is established and analyzed.

**ReplayValidationHarness (R1-10):** The issue text mentions the harness, but
`ReplayValidationHarness` validates economic divergence (EmulatedGame vs ground
truth). Classification accuracy validation is a different concern — it compares
classifier predictions against ground-truth strategy labels. The two tests share
replay loading but serve different purposes. The harness is not used here because
classification validation doesn't need economic divergence tracking.

**Pre-integration verification (R1-13):** Before merging, run
`mvn test -pl quarkmind-sc2 -Preport` to confirm the classifier changes don't
regress replay validation accuracy.

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
- [ ] AI Arena cascade accuracy ≥ 80% at minute 4 (hard-asserted)
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
