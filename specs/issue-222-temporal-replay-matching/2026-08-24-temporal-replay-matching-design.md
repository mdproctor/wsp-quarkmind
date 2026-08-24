# Temporal Game Replay Matching — Design Spec

**Issue:** casehubio/quarkmind#222
**Epic:** casehubio/quarkmind#214 (CBR Game Experience — Tier 3)
**Date:** 2026-08-24
**Branch:** `issue-222-temporal-replay-matching`

## Summary

Add temporal sequence similarity to the CBR pipeline — match games by how they unfolded over time, not just end-state snapshots. Three new components wire existing QuarkMind temporal data to the existing neocortex sequence similarity infrastructure (DTW, EditDistance, TimeSeries/DiscreteSequence field types).

No new algorithms. `DtwSimilarity`, `EditDistanceSimilarity`, warping constraints (`SakoeChibaBand`), and `TrendAnalyzer` are fully built and tested in neocortex `memory-api`. This is a wiring task.

## Components

### 1. TimelineObservation (domain record)

Plain Java record in `io.quarkmind.domain`. Captures a single temporal sample from `GameState`.

```java
public record TimelineObservation(
    double minute,
    int ourWorkers,
    int ourMinerals,
    int ourArmySupply
) {
    public static TimelineObservation from(GameState gs) { ... }
}
```

**Fields:**
- `minute` — `gameFrame / SC2Data.GAME_LOOPS_PER_SECOND / 60.0`
- `ourWorkers` — `(int) gs.myUnits().stream().filter(u -> u.type().isWorker()).count()`
- `ourMinerals` — `gs.minerals()` (from `PlayerCommon.getMinerals()`, always populated)
- `ourArmySupply` — `gs.supplyUsed() - (int) gs.myUnits().stream().filter(u -> u.type().isWorker()).count()`

Own-side fields only, using top-level `GameState` accessors (`minerals()`, `supplyUsed()`). `PlayerEconomyStats` is `EMPTY` during live games (`ObservationTranslator` line 137) — economy fields there are zeros. The top-level fields are populated from `PlayerCommon` and are always available. Enemy economy data is unavailable during live games — DTW between live partial-enemy data and stored full-replay data would systematically bias matching.

### 2. TemporalPrediction (domain record)

Plain Java record in `io.quarkmind.domain`. The structured prediction written to CaseFile.

```java
public record TemporalPrediction(
    String predictedNextPhase,
    Trend economyTrend,
    Trend armyTrend,
    double minutesToNextTransition,
    double confidence,
    int matchCount,
    double bestMatchScore
) {
    public enum Trend { GROWING, STABLE, DECLINING, SPIKE, UNKNOWN }
}
```

### 3. TimelineSampler (CDI bean)

`@ApplicationScoped` in `io.quarkmind.agent.cbr`. Injected into `GameTickExecutor`, called every tick with full `GameState`. Internally samples every ~30 game-seconds.

- **Sample interval:** 672 game loops (30s at 22.4 loops/s)
- **Accumulator:** `CopyOnWriteArrayList<TimelineObservation>`
- **Reset:** `@Observes GameStarted` clears accumulated samples
- **API:** `tick(GameState)` to feed data, `getTimeline()` returns immutable copy

**GameTickExecutor integration:** One line after `engine.observe()`:

```java
var gameState = engine.observe();
timelineSampler.tick(gameState);
```

### 4. Schema Extensions

`SC2CbrSchemaRegistrar` adds one new field and upgrades one existing field in the `sc2-strategy` schema.

**New TimeSeries field — `timeline`:**

```java
FeatureField.timeSeries("timeline", "minute",
    new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
    FeatureField.numeric("minute", 0, 30),
    FeatureField.numeric("our_workers", 0, 80),
    FeatureField.numeric("our_minerals", 0, 5000),
    FeatureField.numeric("our_army_supply", 0, 200)
)
```

The `minute` timestamp field must be included in `innerFields` — the `TimeSeries` constructor validates this at startup.

Sakoe-Chiba band window=3 allows ±90 seconds of temporal shift (3 time steps at 30s intervals). Covers typical SC2 timing variation without pathological alignments.

**Upgraded field — `phase_sequence`:**

Changes from `FeatureField.categoricalList("phase_sequence")` (unordered Jaccard) to:

```java
FeatureField.discreteSequence("phase_sequence",
    new SimilaritySpec.EditDistanceSpec(phaseSubstitutionCosts(), 1.0, 1.0))
```

Data format is unchanged (`StringListVal` → `List<String>`) — the schema upgrade only changes how similarity is scored. Existing stored cases remain queryable.

The strategy router does not query `phase_sequence` — its weights are `enemy_archetype=0.5, enemy_race=0.15, matchup=0.15, assessment_confidence=0.2`. This upgrade only affects the new temporal query (D10) and any future queries that weight `phase_sequence`.

**Phase substitution similarity matrix:**

| | EARLY_MACRO | TRANSITIONING | MID_SKIRMISH | EARLY_AGGRESSION | DEFENSIVE_HOLD |
|---|---|---|---|---|---|
| EARLY_MACRO | 1.0 | 0.6 | 0.2 | 0.1 | 0.2 |
| TRANSITIONING | 0.6 | 1.0 | 0.5 | 0.4 | 0.4 |
| MID_SKIRMISH | 0.2 | 0.5 | 1.0 | 0.6 | 0.5 |
| EARLY_AGGRESSION | 0.1 | 0.4 | 0.6 | 1.0 | 0.4 |
| DEFENSIVE_HOLD | 0.2 | 0.4 | 0.5 | 0.4 | 1.0 |

Same-family substitutions (combat: MID_SKIRMISH↔EARLY_AGGRESSION=0.6) are cheaper than cross-family (eco↔combat: EARLY_MACRO↔EARLY_AGGRESSION=0.1). TRANSITIONING has moderate similarity to everything.

### 5. Retention Extensions

`SC2CbrRetentionObserver.onOutcome()` stores the accumulated timeline at game close.

**Timeline storage:** Inject `TimelineSampler`, convert `getTimeline()` to `StructListVal`:

```java
List<TimelineObservation> timeline = timelineSampler.getTimeline();
if (!timeline.isEmpty()) {
    List<Map<String, FeatureValue>> observations = timeline.stream()
        .map(t -> Map.<String, FeatureValue>of(
            "minute", FeatureValue.of(t.minute()),
            "our_workers", FeatureValue.of(t.ourWorkers()),
            "our_minerals", FeatureValue.of(t.ourMinerals()),
            "our_army_supply", FeatureValue.of(t.ourArmySupply())))
        .toList();
    features.put("timeline", FeatureValue.of(observations));
}
```

**Phase sequence:** Already stored as `StringListVal`. The schema upgrade from `CategoricalList` to `DiscreteSequence` changes how similarity is scored, not how data is stored.

**EnrichedGameData** is unchanged. `EnrichedGameData` holds end-game aggregate metrics (phase count, battle count, dominance scores). The timeline is a temporal series — different concern. Instead, `SC2GameCbrCase.buildForGameEnriched()` gains a new overload that accepts an additional `List<TimelineObservation> timeline` parameter and adds the timeline feature alongside the enriched aggregate features. This avoids mixing aggregates and temporal series in one record.

### 6. TemporalCbrTask (plugin task)

New `@ApplicationScoped @CaseType("starcraft-game")` TaskDefinition in `io.quarkmind.agent.cbr`.

**Task ID:** `"temporal-cbr.predict"`. This prefix is not in `QuarkMindCaseHub.PHASE_ORDER`, so the task runs after all known phases (scouting → strategy-routing → strategy → tactics → economics → summarisation → temporal-cbr). Temporal predictions arrive for the *next* tick's consumers, not the current tick. This is correct — the prediction is advisory lookahead, not reactive input. Current-tick strategy/tactics decisions are based on current state; temporal predictions inform the next planning cycle.

**Phase accumulator:** Subscribes directly to `SummarisationLifecycle.phaseBus()` and maintains its own `CopyOnWriteArrayList<TacticalPosture>`. Clears on `@Observes GameStarted`. Self-contained — no dependency on `SC2CbrRetentionObserver`'s internal state.

**Activation:**
- `requires()`: `GAME_STATE`, `STRATEGY_ROUTED_ARCHETYPE`
- `activateIf()`: `timelineSampler.getTimeline().size() >= 4` (≥2 min of data)
- `produces()`: `TEMPORAL_PREDICTION`, `TEMPORAL_SIMILAR_COUNT`, `TEMPORAL_SIMILAR_BEST_SCORE`

**Throttling:** Tracks `lastQueryFrame`. Skips execution if fewer than ~240 ticks (~2 minutes) since last query. Activates on every orchestrator tick but only does real work periodically.

**Query weights:**

| Feature | Weight | Similarity |
|---|---|---|
| `timeline` | 0.50 | DTW (Sakoe-Chiba w=3) |
| `phase_sequence` | 0.30 | Edit distance (semantic matrix) |
| `enemy_archetype` | 0.10 | Categorical |
| `matchup` | 0.10 | Categorical |

`topK=5`, `minSimilarity=0.3`. Strategy router's existing scalar-only query is unchanged.

**Execution flow:**
1. Build query timeline from `timelineSampler.getTimeline()` as `StructListVal`
2. Build query phase sequence from own accumulated phases as `StringListVal`
3. Call `cbrStore.retrieveSimilar()` with temporal-heavy weights — returns `List<ScoredCbrCase>` with similarity scores
4. For each top-K result, extract the stored timeline (`StructListVal`) from the case's features and re-run `DtwSimilarity.compute()` directly to obtain the `DtwResult` with alignment path. This re-runs DTW on only the top-5 cases (not all 500), adding ~5 × O(n×m×w) ≈ 3600 operations — negligible.
5. Use the alignment path to find the current game's alignment point (`s_k`) in each past game's timeline
6. Compute lookahead predictions from past game timelines beyond `s_k`
7. Aggregate across top matches, build `TemporalPrediction`
8. Write to CaseFile keys

**CaseFile keys produced:**
- `agent.temporal.prediction` (`TEMPORAL_PREDICTION`) — serialised `TemporalPrediction`
- `agent.temporal.similar.count` (`TEMPORAL_SIMILAR_COUNT`) — number of temporal matches found
- `agent.temporal.similar.best.score` (`TEMPORAL_SIMILAR_BEST_SCORE`) — best DTW match score

**Consumer note:** No existing plugin reads `agent.temporal.prediction` in this iteration. The prediction is written to CaseFile for future consumers (strategy adaptation, coaching, commentary). A follow-up issue should wire at least one consumer to close the feedback loop. This is acceptable for Tier 3 validation — the acceptance criterion "usable as predictive context" is met by the CaseFile contract: any plugin can read the key, and the data is structured and documented.

### 7. Prediction Extraction

Converting DTW alignment into "what happened next."

Given query timeline Q (current game, e.g. 6 steps = 3 min) matched against stored timeline S (past game, e.g. 20 steps = 10 min):

1. DTW alignment maps Q's last step to step `s_k` in S
2. **Lookahead window:** `S[k+1 .. k+4]` — next 2 minutes of the past game
3. If `k+1 >= len(S)`, past game ended shortly after — prediction is "game ending soon"

**Trend computation from lookahead:**

Neocortex's `TrendAnalyzer`/`TrendSpec` computes trends on stored timelines at retention time (aggregate trend over the full game). Prediction trends are different — they describe the lookahead window of a specific past game (what happened *after* the alignment point). These are computed at query time, not retention time, so `TrendSpec` does not apply here.

Trend logic for the lookahead window (4 observations):
- **GROWING:** monotonically increasing (each value ≥ previous)
- **DECLINING:** monotonically decreasing (each value ≤ previous)
- **STABLE:** range (max - min) < 10% of the mean value
- **SPIKE:** any single-step delta > 50% of the mean value
- **UNKNOWN:** fewer than 2 lookahead observations, or no clear pattern

Applied independently to `our_workers` (→ `economyTrend`) and `our_army_supply` (→ `armyTrend`).

**Phase prediction:** Find the phase at the timestamp corresponding to `s_k` in the matched case's `phase_sequence`. The next distinct phase is `predictedNextPhase`. Time delta gives `minutesToNextTransition`.

**Confidence formula:**

Base: `dtwScore × (matchCount / topK)`. Range: [0.0, 1.0].

Consensus boost: if ≥3 of top-5 cases agree on `predictedNextPhase`, multiply by 1.3. If all 5 agree, multiply by 1.5. Capped at 1.0 after boosting.

Dissent: if no majority (no phase has ≥3 votes), take the plurality prediction and multiply confidence by 0.5.

This gives a clear signal: high confidence means strong DTW match + consensus; low confidence means either weak match or disagreement among similar games.

## Component Placement

| Component | Package | Type |
|---|---|---|
| `TimelineObservation` | `io.quarkmind.domain` | Plain record |
| `TemporalPrediction` | `io.quarkmind.domain` | Plain record |
| `TimelineSampler` | `io.quarkmind.agent.cbr` | `@ApplicationScoped` CDI bean |
| `TemporalCbrTask` | `io.quarkmind.agent.cbr` | `@ApplicationScoped @CaseType` TaskDefinition |
| Phase substitution matrix | `SC2CbrSchemaRegistrar` | Static method |

No new packages. All components fit existing structure.

## Testing Strategy

### Unit tests (plain JUnit)

- **`TimelineObservationTest`** — `from(GameState)` extracts correct values; handles edge cases (zero workers, no army, zero minerals)
- **`TimelineSamplerTest`** — samples at correct intervals; `getTimeline()` returns immutable copy; clears on `GameStarted`; no sample when interval not reached; handles first tick
- **`TemporalPredictionTest`** — trend computation from lookahead observations; multi-case aggregation; confidence calculation; edge case when past game ends at alignment point
- **`TemporalCbrTaskTest`** — activation gates (needs ≥4 samples + routed archetype); throttling (skips if <2 min since last query); builds correct query weights; writes prediction to CaseFile; handles empty retrieval results
- **`SC2CbrSchemaRegistrarTest`** — verify `timeline` registered as `TimeSeries` with `SakoeChibaBand(3)`; verify `phase_sequence` registered as `DiscreteSequence` with substitution matrix

### Integration tests (`@QuarkusTest`)

- **`TemporalCbrRetentionIT`** — full game lifecycle: tick sampler through game, trigger outcome, verify stored case has `timeline` feature as `StructListVal` with correct observation count and field values; verify `phase_sequence` stored as `StringListVal`
- **`TemporalCbrQueryIT`** — store 3-5 cases with known timelines, run `TemporalCbrTask` with partial query timeline, verify DTW retrieval returns cases ranked by temporal similarity, verify prediction written to CaseFile

### Calibration test (`@Tag("benchmark")`)

- **`TemporalMatchingCalibrationTest`** — run against replay dataset, store game timelines, query with partial timelines from known games, verify same game in top-3 matches (self-retrieval sanity check). Measure query latency over N stored cases, assert < 1s for 500 cases.

## Performance

| Operation | Cost | Budget |
|---|---|---|
| Sampling | 1 record copy / 30s | Negligible |
| Storage | ~20-40 observations × 4 fields per case | ~160-320 feature values |
| DTW retrieval (500 cases) | O(n × m × w) per case, n≈8, m≈30, w=3 | ~360K ops, sub-ms |
| DTW alignment re-run (top 5) | 5 × O(n × m × w) | ~3600 ops, negligible |
| Edit distance (per case) | O(n × m), n,m ≈ 5-10 | Trivial |
| Total query (500 cases) | | Well under 1s target |

LbKeogh pruning available but not needed at this scale.

## Acceptance Criteria Mapping

| Criterion | How met |
|---|---|
| Game timeline sampled and stored as temporal case segments at game close | `TimelineSampler` accumulates, `SC2CbrRetentionObserver` stores as `StructListVal` `timeline` feature |
| DTW similarity query returns games with similar economy/army trajectories | `TemporalCbrTask` queries with `timeline` weight=0.50, `DtwSpec(SakoeChibaBand(3))` |
| Phase sequence edit distance query returns games with similar phase progression | `phase_sequence` upgraded to `DiscreteSequence` with semantic substitution matrix, weight=0.30 |
| Retrieved temporal matches usable as predictive context | `TemporalPrediction` record written to CaseFile key `agent.temporal.prediction` |
| Performance: temporal query over 500 cases < 1 second | DTW at 500 cases is sub-millisecond. Validated by `TemporalMatchingCalibrationTest` |

## References

- `DtwSimilarity`, `EditDistanceSimilarity`, `LbKeogh` — neocortex `memory-api` (sequence similarity algorithms)
- `FeatureField.TimeSeries`, `FeatureField.DiscreteSequence` — neocortex schema types
- `WarpingConstraint.SakoeChibaBand`, `SimilaritySpec.DtwSpec`, `SimilaritySpec.EditDistanceSpec` — neocortex similarity configuration
- `SC2GameCbrCase` — current flat-feature case representation (Tier 1+2)
- `SC2CbrRetentionObserver` — game-close case retention pipeline
- `SC2CbrSchemaRegistrar` — CBR schema registration
- `SC2StrategyRouterTask` — CBR-based strategy routing (unchanged by this work)
- `GameTickExecutor.execute()` — periodic tick with GameState access
- `GamePhaseSummariser` — produces the 5 `TacticalPosture` phases used in substitution matrix
- `ObservationTranslator.translate()` line 137 — `PlayerEconomyStats.EMPTY` for both sides (justifies using top-level `GameState` accessors for own-side fields)
- Tier 2 decisions D5 — explicit deferral of temporal matching to Tier 3 (#222)
- `docs/specs/issue-215-cbr-tier2-game-experience/2026-08-10-cbr-tier2-game-experience-design.md` — predecessor spec
- `docs/specs/2026-07-17-cbr-reference-implementation-design.md` — CBR Tier 1 spec
