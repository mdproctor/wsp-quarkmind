# CBR Tier 2 — Structured Game Experience Design Spec

**Issues:** #215, #217, #218, #219, #220, #221
**Epic:** #214 — CBR game experience — learning across games
**Date:** 2026-08-10
**Branch:** issue-215-cbr-tier2-game-experience

## Summary

Enrich the existing CBR Tier 1 infrastructure with structured game experience data across six domains: event hierarchy (L2/L3/L4), tactical outcomes, economic build paths, scouting effectiveness, opponent identity, and learning curve observability. Every game produces a fat `SC2GameCbrCase` capturing how the game unfolded — not just who won.

Builds on the Tier 1 foundation: `SC2CbrRetentionObserver`, `SC2GameCbrCase`, `SC2ImplementationRoutingStrategy`, `SC2StrategyRouterTask` (all from #192).

## Design Approach: Single Enriched Case with Bus-Accumulated Data

### Root insight

A game is one thing. Splitting case data across multiple case types (sc2-tactics, sc2-economics, sc2-scouting) creates a cross-type correlation problem at retrieval time — "find games similar to this one" becomes N queries joined in application code. A single enriched `sc2-strategy` case with all domains as features keeps retrieval as one `CbrQuery` call. Neocortex handles sparse features natively — old Tier 1 cases with missing enrichment fields get neutral similarity scores.

### Why bus subscription, not CaseFile keys

Tier 2 enrichment data (moment lists, phase sequences, arc narratives) lives on the event summarisation buses, not in CaseFile keys. Writing serialised lists to CaseFile keys every tick would bloat the CaseFile. Instead, the observer subscribes to `SummarisationLifecycle` buses and accumulates data during the game — the same pattern `MultiFactorDominanceAssessor` already uses for phase-aware weight adjustment. At game close, the observer has everything it needs from both the CaseFile snapshot (existing keys) and accumulated bus data (new enrichment).

## Components

### 1. SC2GameCbrCase — Enriched Feature Model

The `sc2-strategy` case grows from 4 features to ~20. Organised by issue domain.

#### Existing Tier 1 features (weights rebalanced)

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `enemy_archetype` | Categorical | 0.35 | `EnemyPatternAssessment.archetype()` |
| `enemy_race` | Categorical | 0.10 | `EnemyArchetype.race()` |
| `matchup` | Categorical | 0.10 | Derived (our race vs enemy race) |
| `assessment_confidence` | Numeric(0, 1) | 0.10 | `EnemyPatternAssessment.confidence()` |

Tier 1 total: 0.65 (reduced from 1.0). Archetype remains the primary discriminator.

#### #215 — Event hierarchy (L2/L3/L4 data)

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `phase_sequence` | CategoricalList | 0.10 | Ordered L3 posture names from phaseBus |
| `phase_count` | Numeric(0, 20) | 0.02 | Distinct L3 phases observed |
| `moment_count` | Numeric(0, 50) | 0.02 | Total L2 moments detected |
| `arc_narrative` | Text(semantic=true) | 0.05 | Latest L4 GameArc narrative |
| `game_duration_minutes` | Numeric(0, 30) | 0.03 | gameFrame / FPS / 60 |

`phase_sequence` enables retrieval like "find games that went through EARLY_MACRO → MID_SKIRMISH → DEFENSIVE_HOLD." `arc_narrative` enables semantic search on game progression descriptions.

#### #217 — Tactical outcomes (aggregate metrics)

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `battle_count` | Numeric(0, 20) | 0.03 | Count of BATTLE_STARTED moments |
| `dominance_army` | Numeric(-1, 1) | 0.04 | Army factor from final DominanceScore |
| `dominance_overall` | Numeric(-1, 1) | 0.05 | Overall dominance at close |

Per-engagement outcomes (won/lost, unit trade ratio) deferred to #270 — requires BATTLE_ENDED detection and unit loss tracking that don't exist yet.

#### #218 — Economic build paths (summary metrics)

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `expansion_count` | Numeric(0, 8) | 0.04 | Count of base buildings at close |
| `worker_count_final` | Numeric(0, 80) | 0.03 | Workers alive at close |
| `dominance_economy` | Numeric(-1, 1) | 0.04 | Economy factor from final DominanceScore |
| `supply_block_count` | Numeric(0, 20) | 0.02 | Count of SUPPLY_BLOCK moments |

`supply_block_count` requires a new Drools rule (see Section 4).

#### #219 — Scouting effectiveness (summary metrics)

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `first_contact_minute` | Numeric(0, 15) | 0.03 | FIRST_CONTACT moment frame / FPS / 60 |
| `scout_dispatch_minute` | Numeric(0, 10) | 0.02 | New CaseFile key written by ScoutingTask |
| `archetype_confidence` | Numeric(0, 1) | 0.03 | Final pattern assessment confidence |

Posture correctness (was early assessment accurate at game end) deferred to #269 — needs retrospective evaluation logic.

#### #220 — Opponent model

| Feature | FeatureField type | Weight | Source |
|---------|------------------|--------|--------|
| `opponent_id` | Categorical | 0.0 | CaseFile key `game.opponent.id` |

Weight 0.0 — does not drive similarity scoring. Used exclusively via `CbrQuery.withFilter("opponent_id", Contains(...))` for per-opponent retrieval. In emulated mode, the value is `EnemyStrategy.name()` (e.g., `"ZERG_ROACH_RUSH"`). In mock mode, `"mock-opponent"`. SC2 mode deferred to #271.

#### Weight budget

| Domain | Weight |
|--------|--------|
| Tier 1 (archetype, race, matchup, confidence) | 0.65 |
| Event hierarchy (#215) | 0.22 |
| Tactical (#217) | 0.12 |
| Economics (#218) | 0.13 |
| Scouting (#219) | 0.08 |
| Opponent (#220) | 0.00 |
| **Total** | **1.20** |

Weights intentionally exceed 1.0 — neocortex normalises the weighted similarity sum. Over-weighting slightly is preferable to under-representing Tier 2 features, and the exact balance will be calibrated via replay validation after implementation.

### 2. SC2GameCbrCase — Factory Method

New static factory alongside the existing `buildForGame()`, using a parameter record to avoid a 20+ parameter method:

```java
record EnrichedGameData(
    // #215 — Event hierarchy
    List<String> phaseSequence, int momentCount, String arcNarrative,
    double gameDurationMinutes,
    // #217 — Tactical
    int battleCount, double dominanceArmy, double dominanceOverall,
    // #218 — Economics
    int expansionCount, int workerCountFinal,
    double dominanceEconomy, int supplyBlockCount,
    // #219 — Scouting
    OptionalDouble firstContactMinute, OptionalDouble scoutDispatchMinute,
    double archetypeConfidence,
    // #220 — Opponent
    String opponentId
) {}

static SC2GameCbrCase buildForGameEnriched(
    String archetype, String raceName, String matchup,
    double assessmentConfidence, String strategyId,
    EnrichedGameData enrichment
)
```

`OptionalDouble` for timing features that may not have occurred — the factory omits these from the feature map when empty (see below).

Returns `SC2GameCbrCase` with all available features populated via `FeatureValue.string()`, `FeatureValue.number()`, `FeatureValue.stringList()`. Features that are unavailable (e.g., `first_contact_minute` when no contact occurred, `scout_dispatch_minute` when no scout was sent) are **omitted from the feature map entirely** — neocortex excludes absent features from the similarity score, which is correct behaviour (no information = no signal). Do not use sentinel values (-1.0, empty string) for missing data.

Package: `io.quarkmind.agent.cbr`

### 3. SC2CbrRetentionObserver — Bus Subscriptions and Accumulation

#### New injections

```java
@Inject SummarisationLifecycle summarisationLifecycle;
@Inject MomentBroker momentBroker;
@Inject MultiFactorDominanceAssessor dominanceAssessor;
```

#### Bus subscriptions (added in `@PostConstruct`)

```java
momentBroker.momentBus().subscribe(e -> true, this::collectMoment);
summarisationLifecycle.phaseBus().subscribe(e -> true, this::collectPhase);
summarisationLifecycle.arcBus().subscribe(e -> true, this::collectArc);
```

#### Accumulated state

```java
private final List<GameMoment> moments = new CopyOnWriteArrayList<>();
private final List<TacticalPosture> phases = new CopyOnWriteArrayList<>();
private final AtomicReference<GameArc> latestArc = new AtomicReference<>();
```

Cleared on `@Observes GameStarted` — same lifecycle as `AdvisoryInvocationCounter`.

#### Enriched onOutcome() flow

After existing Tier 1 extraction (archetype, strategy, confidence):

1. Read `GAME_STATE` from CaseFile snapshot → `GameState`
2. Call `dominanceAssessor.assess(gameState)` → `DominanceScore`
3. Compute aggregate metrics from accumulated data:
   - `phaseSequence` = `phases.stream().map(TacticalPosture::posture).toList()`
   - `momentCount` = `moments.size()`
   - `battleCount` = `moments.stream().filter(m -> m.type() == BATTLE_STARTED).count()`
   - `supplyBlockCount` = `moments.stream().filter(m -> m.type() == SUPPLY_BLOCK).count()`
   - `firstContactMinute` = first FIRST_CONTACT moment's `gameFrame / FPS / 60.0`, or `OptionalDouble.empty()` if none
   - `scoutDispatchMinute` = CaseFile key `game.scouting.dispatch.frame` / FPS / 60.0, or `OptionalDouble.empty()`
   - `expansionCount` = `gameState.myBuildings().stream().filter(b -> isBase(b.type())).count()`
   - `workerCountFinal` = `gameState.myUnits().stream().filter(u -> u.type().isWorker()).count()`
   - `opponentId` = CaseFile key `game.opponent.id`, or `"unknown"`
   - `arcNarrative` = `latestArc.get().narrative()`, or empty string
   - `gameDurationMinutes` = `gameState.gameTimeMinutes()`
4. Call `SC2GameCbrCase.buildForGameEnriched(...)` with all fields
5. Store + recordOutcome (unchanged flow)

#### Base building detection helper

```java
private static boolean isBase(BuildingType type) {
    return type == BuildingType.NEXUS
        || type == BuildingType.HATCHERY
        || type == BuildingType.LAIR
        || type == BuildingType.HIVE
        || type == BuildingType.COMMAND_CENTER
        || type == BuildingType.ORBITAL_COMMAND
        || type == BuildingType.PLANETARY_FORTRESS;
}
```

### 4. New Instrumentation

#### 4a. Supply Block Drools Rule

New rule in `MomentDetectionTask.drl`:

```drl
rule "Supply Block"
    salience 130
when
    $e : ScoutingIntelPayload() from entry-point intelEvents
    eval(supplyUsed > 0 && supplyUsed >= supplyCap)
then
    detectedMoments.add(new GameMoment(
        GameMomentType.SUPPLY_BLOCK, currentFrame, Map.of()));
end
```

`supplyUsed` and `supplyCap` need to be exposed to the rule unit. Two options:
- Add `supplyUsed` and `supplyCap` fields to `MomentDetectionRuleUnit` (populated from CaseContext in `MomentDetectionTask.execute()`)
- Introduce a `SupplySnapshot` fact inserted into the DataStore

Recommended: add fields to `MomentDetectionRuleUnit` — simpler, consistent with `previousArmyValue` and `previousPosture` patterns.

Supply block deduplication: only fire once per contiguous block period. Track `lastSupplyBlockFrame` in `MomentDetectionTask` and suppress if within 224 frames (10 seconds) of the last one.

#### 4b. Scout Dispatch CaseFile Key

New key in `QuarkMindCaseFile`:

```java
public static final String SCOUTING_DISPATCH_FRAME = "game.scouting.dispatch.frame";
```

Written by `DroolsScoutingTask` (or `BasicScoutingTask`) when it first dispatches a scout unit. The task already runs per tick — add a `scoutDispatched` boolean flag, write the frame on first dispatch, skip on subsequent ticks. Clear on `GameStarted`.

Added to `ALL_KEYS`.

#### 4c. Opponent ID CaseFile Key

New key in `QuarkMindCaseFile`:

```java
public static final String OPPONENT_ID = "game.opponent.id";
```

Written at game start:
- **Emulated mode:** `EmulatedGame` or startup bean writes `EnemyStrategy.name()`
- **Mock mode:** `MockStartupBean` writes `"mock-opponent"`
- **SC2 mode:** writes `"unknown"` (deferred to #271)

Added to `ALL_KEYS`.

#### 4d. Retrieval Influence CaseFile Key

New key in `QuarkMindCaseFile`:

```java
public static final String CBR_INFLUENCED_SELECTION = "agent.strategy.cbr.influenced";
```

Written by `SC2StrategyRouterTask` when it makes a routing decision. After blending trust + CBR scores, compare the winner against what trust-only would have selected. If they differ, write `true`; otherwise `false`.

```java
// In SC2StrategyRouterTask, after select():
String trustOnlyWinner = candidates.stream()
    .max(Comparator.comparingDouble(c -> trustScores.get(c.bindingName())))
    .map(ImplementationCandidate::bindingName).orElse(fallback);
boolean cbrInfluenced = !winner.equals(trustOnlyWinner);
ctx.set(QuarkMindCaseFile.CBR_INFLUENCED_SELECTION, cbrInfluenced);
```

Added to `ALL_KEYS`.

### 5. SC2CbrSchemaRegistrar — Extended Schema

The `sc2-strategy` schema registration extends with all new fields:

```java
// Existing Tier 1
FeatureField.categorical("enemy_archetype"),
FeatureField.categorical("enemy_race"),
FeatureField.categorical("matchup"),
FeatureField.numeric("assessment_confidence", 0.0, 1.0),

// #215 — Event hierarchy
FeatureField.categoricalList("phase_sequence"),
FeatureField.numeric("phase_count", 0, 20),
FeatureField.numeric("moment_count", 0, 50),
FeatureField.text("arc_narrative", true),  // semantic=true
FeatureField.numeric("game_duration_minutes", 0, 30),

// #217 — Tactical
FeatureField.numeric("battle_count", 0, 20),
FeatureField.numeric("dominance_army", -1.0, 1.0),
FeatureField.numeric("dominance_overall", -1.0, 1.0),

// #218 — Economics
FeatureField.numeric("expansion_count", 0, 8),
FeatureField.numeric("worker_count_final", 0, 80),
FeatureField.numeric("dominance_economy", -1.0, 1.0),
FeatureField.numeric("supply_block_count", 0, 20),

// #219 — Scouting
FeatureField.numeric("first_contact_minute", 0, 15),
FeatureField.numeric("scout_dispatch_minute", 0, 10),
FeatureField.numeric("archetype_confidence", 0, 1.0),

// #220 — Opponent
FeatureField.categorical("opponent_id")
```

### 6. QA Endpoints — Learning Curve Observability (#221)

New JAX-RS resource: `CbrLearningCurveEndpoint`

```java
@Path("/qa/cbr")
@UnlessBuildProfile("prod")
@ApplicationScoped
public class CbrLearningCurveEndpoint {

    @Inject CbrCaseMemoryStore cbrStore;
```

#### GET /qa/cbr/learning-curve

Scans all `sc2-strategy` cases, sorts by `storedAt`, computes:

```json
{
  "totalGames": 25,
  "overallWinRate": 0.60,
  "last10WinRate": 0.80,
  "last20WinRate": 0.70,
  "trend": "IMPROVING",
  "perMatchup": {
    "PvZ": {"games": 10, "winRate": 0.70},
    "PvT": {"games": 8, "winRate": 0.50},
    "PvP": {"games": 7, "winRate": 0.57}
  }
}
```

Trend: `IMPROVING` if last-10 > first-10 by > 0.1, `DECLINING` if lower by > 0.1, else `STABLE`.

#### GET /qa/cbr/strategy-evolution

Groups cases by `solution` (strategy ID), computes per-strategy stats:

```json
{
  "strategies": [
    {
      "strategyId": "strategy.early-pressure",
      "selectionRate": 0.48,
      "winRate": 0.75,
      "games": 12,
      "perArchetype": {
        "ZERG_ROACH_RUSH": {"games": 5, "winRate": 0.80},
        "TERRAN_2RAX_MARINE": {"games": 4, "winRate": 0.75}
      }
    }
  ]
}
```

#### GET /qa/cbr/case-stats

Lightweight diagnostics:

```json
{
  "totalCases": 25,
  "casesByType": {"sc2-strategy": 25, "sc2-advisory": 40},
  "tier2Coverage": 0.80,
  "retrievalInfluenceRate": 0.32,
  "perOpponent": {
    "ZERG_ROACH_RUSH": 5,
    "TERRAN_BIO_PUSH": 8
  }
}
```

`tier2Coverage`: fraction of cases with at least one Tier 2 feature populated.
`retrievalInfluenceRate`: fraction of cases where `CBR_INFLUENCED_SELECTION` was true — the key validation metric proving CBR adds value beyond trust-only routing.

### 7. New CaseFile Keys

| Key | Constant | Type | Written by | Read by |
|-----|----------|------|-----------|---------|
| `game.opponent.id` | `OPPONENT_ID` | String | Engine startup | `SC2CbrRetentionObserver` |
| `game.scouting.dispatch.frame` | `SCOUTING_DISPATCH_FRAME` | Long | `DroolsScoutingTask` | `SC2CbrRetentionObserver` |
| `agent.strategy.cbr.influenced` | `CBR_INFLUENCED_SELECTION` | Boolean | `SC2StrategyRouterTask` | `CbrLearningCurveEndpoint` |

All added to `QuarkMindCaseFile.ALL_KEYS`.

## Testing

### Unit tests (plain JUnit)

| Test | Coverage |
|------|----------|
| `SC2GameCbrCaseTest` (extend) | `buildForGameEnriched()` factory — all features populated, sparse features handled, `withOutcome()` preserves enrichment, feature map contains all expected keys |
| `SC2CbrRetentionObserverTest` (extend) | Bus subscription wiring, moment/phase/arc accumulation, `GameStarted` clears accumulators, aggregate metric computation (battle count, expansion count, worker count, phase sequence, first contact minute, supply blocks), enriched case feature extraction, opponent_id from CaseFile, dominance score integration, scout dispatch minute extraction |
| `CbrLearningCurveEndpointTest` | Win rate computation from mock scan results (overall, rolling window, per-matchup), empty case base returns zeros, strategy evolution grouping and per-archetype breakdown, case stats with tier2Coverage and retrievalInfluenceRate |
| `SC2StrategyRouterTaskTest` (extend) | CBR_INFLUENCED_SELECTION written correctly — same winner as trust-only → false, different winner → true |

### Integration tests (@QuarkusTest)

| Test | Coverage |
|------|----------|
| `SC2CbrRetentionIT` (extend) | Full game lifecycle → verify enriched case stored with L2/L3/L4 features, dominance factors, opponent_id, expansion count, worker count, supply blocks |
| `CbrLearningCurveIT` | Store 5+ cases with varying outcomes and strategies → hit all three QA endpoints → verify JSON response structure and computed metrics |

### Benchmark

Run `GameLoopBenchmarkTest` before/after — verify bus subscriptions in the observer add no measurable tick latency (observer only accumulates per tick, no computation).

## Deletions

None. This is purely additive.

## Deferred — Follow-Up Issues

| Issue | What | Why deferred |
|-------|------|-------------|
| #269 | Posture correctness evaluation | Needs definition of "correct" and retrospective comparison logic |
| #270 | Per-engagement outcome tracking | BATTLE_ENDED detection and unit loss tracking don't exist |
| #271 | SC2 mode opponent ID | Protocol layer changes, needs live SC2 testing |

## Garden References

None surfaced — garden MCP unavailable during this session.

## Protocol References

- **PP-20260610-3c3e89** — Inject concrete StrategyTask type in @QuarkusTest (relevant for SC2StrategyRouterTask test extension)
- **PP-20260603-cefed9** — activateIf() must only declare CDI-injected gates (not affected by this work)
