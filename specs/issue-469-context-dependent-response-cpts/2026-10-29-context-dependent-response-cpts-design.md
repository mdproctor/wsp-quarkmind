# Phase 5: Context-Dependent Response CPTs

## Problem

The response selection network (Phase 4, #460) recommends a strategy archetype
given an opponent archetype, but ignores game state context. The CPT is tiled —
`buildTiledResponseCpt()` copies the unconditional distribution into all 27
context combinations via `System.arraycopy`. A player who is economically
behind gets the same recommendation as one who is ahead.

The network structure already has context variables (`our_economy_state`,
`our_army_state`, `our_tech_state`) as root nodes with uniform 1/3 priors.
`deriveGameStateContext()` computes these from live game state during play.
Only the CPT values need updating — no structural changes.

## Goal

Learn `P(our_strategy | enemy_strategy, economy, army, tech)` from the 149K
labelled replay dataset and replace the tiled CPT with context-dependent values.
Sparse cells regress toward the Phase 4 unconditional prior via hierarchical
smoothing.

## Data Model

### ContextResponseOutcome record

New record in `PriorExportPipeline`:

```java
public record ContextResponseOutcome(
    String ourStrategy,
    String opponentStrategy,
    String matchup,
    boolean won,
    String economyState,   // "behind" | "even" | "ahead"
    String armyState,      // "smaller" | "even" | "larger"
    String techState       // "behind" | "even" | "ahead"
) {}
```

### Context key format

Composite string: `"<economy>:<army>:<tech>"` — e.g. `"behind:even:ahead"`.
27 possible keys (3 × 3 × 3).

### Discretization thresholds

Same as `DroolsScoutingTask.deriveGameStateContext()`:

| Variable | Metric | States | Thresholds |
|----------|--------|--------|------------|
| economy | income ratio (our/enemy collection rates) | behind / even / ahead | < 0.8 / 0.8–1.2 / > 1.2 |
| army | unit count ratio (our/enemy) | smaller / even / larger | < 0.7 / 0.7–1.3 / > 1.3 |
| tech | tech building count comparison | behind / even / ahead | fewer / equal / more |

Static method `discretizeContext()` computes this from raw PlayerStats and
accumulated UnitBorn/UnitDied data.

Tech buildings are the 24 types in `DroolsScoutingTask.TECH_BUILDINGS`:
all non-supply, non-base, non-gas production and research buildings across
all three races. For replay extraction, map `UnitBorn`/`UnitInit` event
`unitTypeName` values to this same set — the names match directly (e.g.
`"CyberneticsCore"`, `"TwilightCouncil"`, `"RoboticsFacility"`).

## Extraction Pipeline

### Checkpoint selection

Each replay contributes snapshots at two checkpoints:
- 3-min: game loop 4032
- 5-min: game loop 6720

Both checkpoints contribute training data. A player "behind" at 3-min but
"ahead" at 5-min contributes to both context cells.

### Extraction steps

For each reconstituted replay JSON:

1. Read both players' tracker events (keyed by `controlPlayerId`)
2. At each checkpoint loop (4032, 6720):
   a. Find nearest `PlayerStats` event within 160-loop window for each player
   b. Extract `scoreValueMineralsCollectionRate` and
      `scoreValueVespeneCollectionRate` (divide by 1000 — values are ×1000
      encoded in tracker events)
   c. Accumulate `UnitBorn` minus `UnitDied` up to that loop per player
      for army unit counts and tech building counts
   d. Discretize to `(economy, army, tech)` using thresholds
3. For each perspective (player 1 and 2): read win/loss from
   `ToonPlayerDescMap` result field, read strategy from `.label.json`
   sidecar coarseLabel
4. If the player won, emit a `ContextResponseOutcome`

### Report test

`ContextResponsePriorExportTest` (`@Tag("report")`) processes all
reconstituted replays from `quarkmind-classifier/data/reconstituted/`.
Outputs to `src/main/resources/priors/`:

- `response-cpts-context-vs_protoss.json`
- `response-cpts-context-vs_terran.json`
- `response-cpts-context-vs_zerg.json`

Prints distribution statistics: per-cell win counts, sparsity report
(cells below α threshold), total outcomes per matchup.

## Hierarchical Smoothing

### Method signature

```java
public static Map<String, Map<String, Map<String, Double>>> learnContextResponsePriors(
    List<ContextResponseOutcome> outcomes,
    Set<String> allOurArchetypes,
    Map<String, Map<String, Double>> marginalPriors,
    double alpha
)
```

### Algorithm

For each `(opponentStrategy, contextKey)` cell:

1. Count wins by our strategy with Laplace smoothing (SMOOTHING = 1.0)
2. Normalize to cell-specific distribution `P_cell`
3. Look up unconditional distribution `P_marginal` for that opponent from
   Phase 4 response priors
4. Blend: `P_smoothed = (n / (n + α)) * P_cell + (α / (n + α)) * P_marginal`
5. Re-normalize (blending can shift the sum slightly)

Where `n` = total winning outcomes in that cell.

### Concentration parameter

Default `α = 10.0`. With ~120 average wins per cell (50K replays per matchup
/ 405 cells × 2 checkpoints ≈ 247, filtered to wins ≈ 120):

| Cell wins (n) | Cell weight | Marginal weight |
|---------------|-------------|-----------------|
| 120 (average) | 92% | 8% |
| 50 | 83% | 17% |
| 10 | 50% | 50% |
| 1 | 9% | 91% |

Sparse cells regress toward Phase 4 behavior. Data-rich cells express
the learned context-dependent signal.

### Export JSON format

```json
{
  "alpha": 10.0,
  "priors": {
    "MARINE_RUSH": {
      "behind:smaller:behind": { "GATEWAY_RUSH": 0.42, "MACRO": 0.31, ... },
      "even:even:even": { "GATEWAY_RUSH": 0.38, ... },
      ...
    }
  }
}
```

The `alpha` value is recorded for reproducibility. Missing context keys
(no wins at all for that cell) are omitted — the loader falls back to
the unconditional prior.

## Network Builder Integration

### New method

Replace `buildTiledResponseCpt()` with `buildContextResponseCpt()` in
`SC2HypothesisNetworkBuilder`:

```java
static double[] buildContextResponseCpt(
    String[] enemyArchetypes, String[] ourStrategies,
    Map<String, Map<String, Map<String, Double>>> contextPriors,
    Map<String, Map<String, Double>> fallbackPriors,
    List<ExpertOverride> expertOverrides)
```

### Context combination ordering

The 27 context combinations are enumerated as:
`economy × army × tech` where each iterates `[state_0, state_1, state_2]`
matching the network variable state ordering:
- economy: behind(0), even(1), ahead(2)
- army: smaller(0), even(1), larger(2)
- tech: behind(0), even(1), ahead(2)

Combination index = `e * 9 + a * 3 + t` where e/a/t ∈ {0,1,2}.

### Fallback

For each `(enemy, contextKey)`:
1. Look up in `contextPriors` → use learned CPT row
2. If missing → use `fallbackPriors` (Phase 4 unconditional for that enemy)
3. Apply expert overrides (unchanged from Phase 4)
4. Clamp to [0.01, 0.99], normalize

### Loader

`HypothesisPriorLoader.loadContextResponsePriors()` reads the new JSON
format. Returns `Map<String, Map<String, Map<String, Double>>>` (opponent →
contextKey → strategy → probability).

`buildTracker()` in `SC2HypothesisNetworkBuilder` calls
`buildContextResponseCpt()` when context priors are available,
`buildTiledResponseCpt()` otherwise (backward compatibility).

## Testing

### Unit tests (plain JUnit)

**`PriorExportPipelineTest`** — new tests:
- `discretizeContext_thresholds` — verifies all threshold boundaries
- `learnContextResponsePriors_hierarchicalSmoothing` — sparse cell regresses
  toward marginal, data-rich cell diverges
- `learnContextResponsePriors_emptyCell_returnsMarginal` — zero-win cell
  returns pure marginal prior
- `learnContextResponsePriors_uniformContext_matchesUnconditional` — when
  all cells have equal data, result approximates the unconditional prior

**`SC2HypothesisNetworkBuilderTest`** — new tests:
- `buildContextResponseCpt_differentContexts_differentRows` — behind vs
  ahead produce distinct CPT rows
- `buildContextResponseCpt_missingContext_fallsBackToUnconditional` —
  absent context key uses Phase 4 prior
- `buildContextResponseCpt_expertOverrides_applied` — overrides still
  work with context-dependent base

### Report test (`@Tag("report")`)

**`ContextResponsePriorExportTest`**:
- Processes all reconstituted replays at 3-min and 5-min checkpoints
- Writes context-dependent CPT JSON files
- Prints per-matchup statistics: total outcomes, per-cell distribution,
  cells below α threshold, top context shifts vs unconditional

### Calibration test (`@Tag("benchmark")`)

**`ResponseSelectionCalibrationTest`** — update existing test:
- `responseSelection_contextVariables_shiftRecommendation` changes from
  diagnostic print to **assertion**: behind and ahead contexts must produce
  different top-1 recommendations for at least one opponent archetype.
  This is the acceptance criterion from the issue.

## Scope boundaries

### In scope
- `ContextResponseOutcome` record and `discretizeContext()` method
- `learnContextResponsePriors()` with hierarchical smoothing
- `ContextResponsePriorExportTest` report test
- `buildContextResponseCpt()` replacing tiled version
- `loadContextResponsePriors()` loader
- Unit tests and calibration assertion
- Export JSON resource files

### Out of scope
- Changes to `deriveGameStateContext()` thresholds (pinned by issue)
- Network structure changes (context variables already exist as root nodes)
- Expert override format changes (applied identically, per opponent)
- Changes to the intel (opponent detection) slot
- Live `ScoutingObservationExtractor` changes (extracts enemy observations,
  not own game state — context comes from `deriveGameStateContext()`)

## References

- `SC2HypothesisNetworkBuilder.java:194-235` — current `buildTiledResponseCpt()`
- `PriorExportPipeline.java:37-60` — current `learnResponsePriors()`
- `DroolsScoutingTask.java:680-707` — `deriveGameStateContext()` thresholds
- `ResponseSelectionCalibrationTest.java:119-155` — Phase 4 uniform tiling confirmation
- `TrackerEventFeatureExtractor.java:97-119` — PlayerStats extraction
- `response-priors-vs_*.json` — Phase 4 unconditional priors (marginal for smoothing)
- `response-overrides.yaml` — expert overrides
- Design spec Phase 5: `wksp/specs/issue-456-sc2-replay-scouting-pipeline/2026-10-26-hypothesis-tracker-integration-design.md:293-304`
- Issue #469: acceptance criteria
