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
- `ourMinerals` — `gs.playerEconomy().mineralsCurrent()`
- `ourArmySupply` — `gs.playerEconomy().foodUsed() - gs.playerEconomy().workersActiveCount()`

Own-side fields only. Enemy economy data is unavailable during live games (`ObservationTranslator` sets both `playerEconomy` and `enemyEconomy` to `PlayerEconomyStats.EMPTY`). DTW between live partial-enemy data and stored full-replay data would systematically bias matching.

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
    SimilaritySpec.dtw(WarpingConstraint.sakoeChibaBand(3)),
    FeatureField.numeric("our_workers", 0, 80),
    FeatureField.numeric("our_minerals", 0, 5000),
    FeatureField.numeric("our_army_supply", 0, 200)
)
```

Sakoe-Chiba band window=3 allows ±90 seconds of temporal shift (3 time steps at 30s intervals). Covers typical SC2 timing variation without pathological alignments.

**Upgraded field — `phase_sequence`:**

Changes from `FeatureField.categoricalList("phase_sequence")` (unordered Jaccard) to:

```java
FeatureField.discreteSequence("phase_sequence",
    SimilaritySpec.editDistance(phaseSubstitutionCosts(), 1.0, 1.0))
```

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

**EnrichedGameData:** Gains `List<TimelineObservation> timeline` field. The `buildForGameEnriched` factory method adds the timeline feature.

### 6. TemporalCbrTask (plugin task)

New `@ApplicationScoped @CaseType("starcraft-game")` TaskDefinition in `io.quarkmind.agent.cbr`.

**Activation:**
- `requires()`: `GAME_STATE`, `STRATEGY_ROUTED_ARCHETYPE`
- `activateIf()`: `timelineSampler.getTimeline().size() >= 4` (≥2 min of data)

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
2. Build query phase sequence from `SC2CbrRetentionObserver`'s accumulated `phases` list (already collected via `SummarisationLifecycle.phaseBus()` subscription). Inject the observer and call a new `getPhaseSequence()` accessor that returns the current phase labels as `List<String>`.
3. Call `cbrStore.retrieveSimilar()` with temporal-heavy weights
4. Extract DTW alignment to find current game's alignment point in each past game
5. Compute lookahead predictions from past game timelines beyond alignment point
6. Aggregate across top matches, build `TemporalPrediction`
7. Write to CaseFile keys

**CaseFile keys produced:**
- `agent.temporal.prediction` — serialised `TemporalPrediction`
- `agent.temporal.similar.count` — number of temporal matches found
- `agent.temporal.similar.best.score` — best DTW match score

### 7. Prediction Extraction

Converting DTW alignment into "what happened next."

Given query timeline Q (current game, e.g. 6 steps = 3 min) matched against stored timeline S (past game, e.g. 20 steps = 10 min):

1. DTW alignment maps Q's last step to step `s_k` in S
2. **Lookahead window:** `S[k+1 .. k+4]` — next 2 minutes of the past game
3. If `k+1 >= len(S)`, past game ended shortly after — prediction is "game ending soon"

**Trend computation from lookahead:**
- Compare lookahead `our_workers` values: monotonically increasing → `GROWING`, decreasing → `DECLINING`, variance < threshold → `STABLE`, sudden jump > 2σ → `SPIKE`
- Same logic for `our_army_supply`

**Phase prediction:** Find the phase at the timestamp corresponding to `s_k` in the matched case's `phase_sequence`. The next distinct phase is `predictedNextPhase`. Time delta gives `minutesToNextTransition`.

**Confidence:** `dtwScore * (matchCount / topK)`. High DTW similarity with many agreeing matches gives high confidence. Capped at 1.0.

**Multi-case aggregation:** When ≥3 of top 5 cases agree on next phase, confidence is boosted. When cases disagree, take majority prediction with lower confidence.

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
| DTW query (per case) | O(n × m × w), n≈8, m≈30, w=3 | ~720 ops |
| DTW query (500 cases) | ~360K ops | Sub-millisecond |
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
- `ObservationTranslator.translate()` line 160 — enemy economy `EMPTY` (justifies own-side-only fields)
- Tier 2 decisions D5 — explicit deferral of temporal matching to Tier 3 (#222)
- `docs/specs/issue-215-cbr-tier2-game-experience/2026-08-10-cbr-tier2-game-experience-design.md` — predecessor spec
- `docs/specs/2026-07-17-cbr-reference-implementation-design.md` — CBR Tier 1 spec
