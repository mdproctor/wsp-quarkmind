# EmulatedGame Feature Extraction — Design Spec

**Issue:** #360
**Date:** 2026-10-26
**Scope:** Wire feature extraction to run on EmulatedGame output, producing ONNX-compatible training data in standard game_json format. Full round-trip validation: shape correctness, feature equivalence characterisation, and ONNX inference.
**Decisions:** See `decisions.md` in this directory for D1–D3.

---

## 1. Problem Statement

The ONNX strategy classifier trains on feature vectors extracted from replays via `TrackerEventFeatureExtractor` (ground truth) and `StrippedReplayFeatureExtractor` (reconstitution from game events). Neither path works on EmulatedGame output — there is no code that converts EmulatedGame tick state into the game_json format the Python training pipeline consumes.

EmulatedGame has full state (units, buildings, upgrades, economy) via `snapshot()` returning a `GameState` record. The runtime Java inference path already works — `DroolsScoutingTask.buildSnapshot(GameState)` is source-agnostic and produces `WindowSnapshot` from any `GameState` regardless of origin. The gap is exclusively in producing **training data** for the Python pipeline.

This is a prerequisite for the Phase 4 AI-vs-AI training loop (#344): emulated games must produce the same feature format as replays so ONNX models can be retrained on synthetic data.

## 2. Goals and Success Criteria

| Criterion | Measure |
|-----------|---------|
| Feature extraction runs on EmulatedGame output | `EmulatedGameFeatureExtractor` produces valid game_json from GameState sequences |
| Feature vectors match Phase 2 shape | 204 features per player (53 buildings + 56 units + 13 stats + 82 upgrades), 10 windows × 409 = 4090 temporal floats, 4 map features |
| Feature equivalence characterised | Per-category divergence report against oracle replay features |
| ONNX inference works on emulated features | `BinaryDetectorRunner` produces valid classifications (non-NaN logits) from emulated features |

## 3. Architecture

### 3.1 New Class: EmulatedGameFeatureExtractor

**Package:** `io.quarkmind.sc2.emulated`
**Responsibility:** Convert a sequence of EmulatedGame `GameState` snapshots into game_json format (the same `Map<String, Object>` that `TrackerEventFeatureExtractor` produces).

The extractor follows protocol PP-20260528-612dee: extraction logic lives in a dedicated class, not in EmulatedGame itself. It reads GameState via public accessors only.

### 3.2 Diff Strategy (D2)

Count-based diffs — compare unit/building type counts between consecutive GameState snapshots:

- **UnitBorn:** unit type count increased → emit N `UnitBornEvent` records (one per increase)
- **UnitDied:** unit type count decreased → emit N `UnitDiedEvent` records
- **UnitInit/UnitDone:** building type count increased → emit `UnitInitEvent` + `UnitDoneEvent` pair
- **Upgrade:** upgrade appeared in `playerUpgrades` set → emit `UpgradeEvent`
- **PlayerStats:** emitted every 160 game loops from `PlayerEconomyStats.toFeatureVector()` (D3), scaled by 1000× to match tracker event format

Count-based diffs are sufficient because the Python pipeline (`sc2egset_extractor.py`) only consumes per-type counts at each second, not individual unit identities.

### 3.3 Data Flow

```
EmulatedGame.tick() → snapshot() → GameState
                                      │
                        EmulatedGameFeatureExtractor
                         (diff consecutive snapshots)
                                      │
                            game_json (Map<String, Object>)
                              - trackerEvents: [...]
                              - ToonPlayerDescMap: {...}
                              - header: {...}
                              - metadata: {extractionMethod: "emulator"}
                                      │
                         [write to JSON file]
                                      │
                    Python reconstitution_pipeline.py
                              (unchanged)
                                      │
                         ONNX training / inference
```

### 3.4 game_json Output Format

The output matches the format produced by `TrackerEventFeatureExtractor.extract()`:

```java
Map<String, Object> gameJson = Map.of(
    "trackerEvents", List.of(
        Map.of("_event", "UnitBornEvent", "_gameloop", 1234,
               "m_unitTypeName", "Marine", "m_controlPlayerId", 1, ...),
        Map.of("_event", "PlayerStatsEvent", "_gameloop", 1600,
               "m_stats", Map.of(
                   "m_scoreValueMineralsCurrent", 450000, // ×1000
                   ...)),
        ...
    ),
    "ToonPlayerDescMap", Map.of(
        "1", Map.of("m_name", "Player1"),
        "2", Map.of("m_name", "Player2")
    ),
    "header", Map.of(
        "m_elapsedGameLoops", totalLoops
    ),
    "metadata", Map.of(
        "extractionMethod", "emulator",
        "emulatorVersion", "1.0"
    )
);
```

### 3.5 API

```java
public class EmulatedGameFeatureExtractor {

    // Batch extraction: convert a full game's snapshots to game_json
    public Map<String, Object> extract(
            List<GameState> snapshots,
            int gameLoopsPerSnapshot,
            Race playerRace,
            Race opponentRace) { ... }

    // Incremental: accumulate tick-by-tick, then finalise
    public void addSnapshot(GameState state, int gameLoop) { ... }
    public Map<String, Object> finalise(Race playerRace, Race opponentRace) { ... }
}
```

The batch API is for test/export use. The incremental API enables future integration with live EmulatedGame runs (e.g., #359 autonomous play).

## 4. Validation Plan

### 4.1 Tier 1 — Shape Validation (Controlled Scenarios)

**Test:** `EmulatedGameFeatureShapeTest` (plain JUnit)

1. Run `ScenarioLibrary.PROTOSS_4GATE` through EmulatedGame for 5 minutes (simulated)
2. Collect GameState snapshots every tick
3. Feed through `EmulatedGameFeatureExtractor` → game_json
4. Write game_json to temp file
5. Feed through Python `sc2egset_extractor.py` (subprocess call)
6. Assert feature dimensions: 204 per player, correct building/unit/stat/upgrade indices
7. Also verify: event count > 0 for each event type, PlayerStats events at 160-loop intervals, all UnitType/BuildingType values match `FeatureIndexMaps` indices

This test validates **format correctness** — that the extractor produces game_json the Python pipeline can consume without errors, and the resulting features have the right shape.

### 4.2 Tier 2 — Equivalence Characterisation (Real Replays)

**Test:** `EmulatedGameFeatureEquivalenceTest` (`@Tag("report")`)

1. For each oracle replay (118 replays with tracker events):
   a. Extract ground-truth features via `TrackerEventFeatureExtractor`
   b. Run the same replay through EmulatedGame (using `ReplayValidationHarness` to feed intents)
   c. Collect EmulatedGame snapshots → `EmulatedGameFeatureExtractor` → game_json
   d. Compare feature vectors at 1/3/5/7 min checkpoints
2. Report per-category divergence:
   - Building counts: % match per type
   - Unit counts: % match per type
   - Economy stats: MAPE per field
   - Upgrades: set difference
3. Write report to `docs/benchmarks/emulated-game-feature-equivalence.md`

This is a **characterisation test** — it measures divergence without asserting thresholds. EmulatedGame's accuracy limitations (#384 baseline: 36% units at 5-min) will cause known divergence. The report documents the current state and tracks improvement as #384 progresses.

### 4.3 Tier 3 — ONNX Inference Validation

**Test:** `EmulatedGameOnnxInferenceTest` (plain JUnit)

1. Run `ScenarioLibrary.PROTOSS_4GATE` through EmulatedGame
2. Feed GameState through the runtime path: `DroolsScoutingTask.buildSnapshot()` → `TemporalWindowAccumulator` → `StrategyFeatureExtractor.extract()` → `StrategyFeatures`
3. Feed through `BinaryDetectorRunner.runAll()` → classifications
4. Assert: non-NaN logits, valid archetype labels, confidence scores in [0,1]
5. Compare: run the same scenario through SimulatedGame, extract features via the same runtime path, compare classifications

This validates that ONNX inference produces meaningful results on emulated features — the models don't produce garbage when fed EmulatedGame-sourced data.

## 5. Scope Boundaries

**In scope:**
- `EmulatedGameFeatureExtractor` class with batch and incremental APIs
- Shape validation test (controlled scenarios)
- Equivalence characterisation test (oracle replays, `@Tag("report")`)
- ONNX inference validation test (ScenarioLibrary scenarios)

**Out of scope:**
- EmulatedGame accuracy improvements (that's #384)
- Standalone EmulatedGame mode without GT sync (that's #384 Phase 3)
- Python pipeline changes (none needed — game_json format is compatible)
- Autonomous play integration (that's #359)
- Bulk reconstitution of ladder replays (that's #384 Phase 3)

## 6. Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| EmulatedGame accuracy is too low for meaningful ONNX inference | Tier 3 test produces random classifications | This is expected and documented; the extractor is correct even if the data is noisy |
| PlayerStats ×1000 scaling mismatch | Economy features off by orders of magnitude | Validate scaling in Tier 1 against known TrackerEvent PlayerStats format |
| GameState snapshot frequency mismatch | Events at wrong timestamps | Align snapshot collection to EmulatedGame's actual tick rate; test against known scenarios |
| Python subprocess in Tier 1 test is fragile | CI failures on missing Python env | Guard with `assumeTrue(pythonAvailable())` — skip if no Python; shape can also be validated in pure Java by checking game_json structure directly |

## References

- #384 spec §6.4 — output format bridge design (preferred approach: GameState diffs → synthetic events)
- #384 spec §6.1 — standalone vs. harness mode architectural distinction
- PP-20260528-612dee — command extraction logic lives in dedicated extractor classes
- `TrackerEventFeatureExtractor.java` — gold-standard extraction format (game_json output shape)
- `StrippedReplayFeatureExtractor.java` — synthetic event emission patterns
- `EmulatedGame.java` — `snapshot()` method, GameState record shape
- `DroolsScoutingTask.java` — `buildSnapshot(GameState)` — source-agnostic feature extraction (runtime path)
- `FeatureIndexMaps.java` — building/unit/upgrade index mappings (53+56+13+82=204 per player)
- `TemporalWindowAccumulator.java` — windowed feature accumulation (10 windows × 438 features)
- `BinaryDetectorRunner.java` — `adaptFeatures()` strips spatial/ratio/delta for ONNX (204 per player → 409 per window)
- `sc2egset_extractor.py` — Python feature extraction from game_json
- `feature_engineering.py` — temporal windowing in Python
- `decisions.md` — D1 (GameState diff approach), D2 (count-based diffs), D3 (PlayerStats from PlayerEconomyStats)
