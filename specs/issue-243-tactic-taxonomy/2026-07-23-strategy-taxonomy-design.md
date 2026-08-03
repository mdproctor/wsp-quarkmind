# Strategy Taxonomy Design — #243

**Issue:** casehubio/quarkmind#243
**Epic:** #252 (SC2 Strategy Knowledge Infrastructure)
**Branch:** issue-243-tactic-taxonomy
**Date:** 2026-07-23

## Problem

The current `EnemyArchetype` enum covers 10 early-game patterns only. The coaching, advisory, and strategy pipelines need tactical awareness across all game phases to provide meaningful real-time advice. A coach that can only detect early-game patterns is limited to the first 3-5 minutes.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Knowledge architecture | Hybrid — structured YAML for data, hand-authored Drools for complex CEP | Simple signatures are data; counter-indications and tech transitions need multi-fact reasoning |
| Naming | `StrategyArchetype` (replaces `EnemyArchetype`) | Player-agnostic — the same taxonomy applies to either player or replay analysis |
| Taxonomy structure | Flat enum with metadata + YAML authoring surface | EnumMap approach from #233 makes adding values cheap; compile-time exhaustiveness checking preserved |
| Phase model | `PhaseResolver` interface, time-based default | Extensible to dynamic state-based detection without consumer changes |
| Counter encoding | Structured data in YAML, injected into LLM prompt AND available as standalone API | Grounds coaching advice in concrete counters; standalone API enables future visualizer overlay |
| Scope | Infrastructure + ~20-25 representative archetypes | Validate pipeline works across all phases; remaining 30-60 archetypes are pure data authoring tracked under epic #252 |
| Neocortex RAG | Excluded | Counter data is structured, finite, and deterministic — a hashmap lookup serves it with zero latency. RAG adds retrieval latency and corpus maintenance overhead for a dataset fully enumerable at startup |
| Counter scope | Protoss perspective | QuarkMind is a Protoss agent; counters describe Protoss responses to each archetype. Multi-race counters deferred to follow-up under epic #252 |

## Section 1: Domain Model

### Rename

- `EnemyArchetype` → `StrategyArchetype`
- `EnemyPatternAssessment` → `PatternAssessment`
- `io.quarkmind.plugin.summarisation.GamePhase` → `TacticalPosture` (existing record carries tactical state names like `"DEFENSIVE_HOLD"`, `"EARLY_AGGRESSION"` — not temporal phases; renaming resolves the collision with the new `GamePhase` enum)
  - Record component rename: `phase` → `posture` — i.e., `record TacticalPosture(String posture, long sinceFrame, String rationale)`. Without this, `tacticalPosture.phase()` reads as "the phase of this posture" rather than "the posture name." All accessor call sites (`e.payload().phase()`, `latestPhase.phase()`, DRL `.phase().equals(...)`) migrate to `.posture()`.

Full codebase migration via IDE refactor. Pre-release — no deprecation shim. `.drl` files require manual update (IDE refactor doesn't cover Drools rule text).

### StrategyArchetype enum

```java
public enum StrategyArchetype {
    TERRAN_MARINE_RUSH(Race.TERRAN, GamePhase.EARLY, ArchetypeCategory.RUSH),
    TERRAN_BIO_TIMING(Race.TERRAN, GamePhase.MID, ArchetypeCategory.TIMING),
    TERRAN_MECH_PUSH(Race.TERRAN, GamePhase.MID, ArchetypeCategory.TIMING),
    TERRAN_BANSHEE_HARASS(Race.TERRAN, GamePhase.EARLY, ArchetypeCategory.HARASS),
    TERRAN_BATTLE_MECH(Race.TERRAN, GamePhase.MID, ArchetypeCategory.COMPOSITION),
    // ... ~20-25 total in this pass
    ;
    private final Race race;
    private final GamePhase phase;
    private final ArchetypeCategory category;

    StrategyArchetype(Race race, GamePhase phase, ArchetypeCategory category) {
        this.race = race;
        this.phase = phase;
        this.category = category;
    }

    public Race race() { return race; }
    public GamePhase phase() { return phase; }
    public ArchetypeCategory category() { return category; }
}
```

### New enums

```java
public enum GamePhase { EARLY, MID, LATE }

public enum ArchetypeCategory { RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION }
```

Both in `io.quarkmind.domain`.

### PatternAssessment

```java
public record PatternAssessment(
    StrategyArchetype archetype,
    double confidence,
    long detectedAtFrame,
    String rationale
) {}
```

### Ripple

**EnemyArchetype → StrategyArchetype, EnemyPatternAssessment → PatternAssessment:**
- `PatternClassification.drl` — enum references in evidence and revision rules
- `DominanceWeightAdjustment.drl` — enum references + ALL `.name().contains()` conversions:
  - `.name().contains("RUSH")` → `.category() == ArchetypeCategory.RUSH` (rush response, combined rush rules)
  - `.name().contains("MACRO")` → `.category() == ArchetypeCategory.MACRO` (macro response, combined macro rule)
- `StarCraftStrategy.drl` — imports `EnemyPatternAssessment`, rule "Strategy: Defend — Rush Detected" uses `.archetype().name().contains("RUSH")` → `.archetype().category() == ArchetypeCategory.RUSH`
- `PatternClassifier` — type references (`EnemyArchetype` → `StrategyArchetype`, `EnemyPatternAssessment` → `PatternAssessment`)
- `DroolsScoutingTask` — `EnumMap<StrategyArchetype, Double>`, `List<PatternAssessment>`
- `DominanceWeightRuleUnit` — `DataStore<EnemyPatternAssessment>` → `DataStore<PatternAssessment>`
- `StrategyRuleUnit` — `DataStore<EnemyPatternAssessment>` → `DataStore<PatternAssessment>`
- `ScoutingIntelPayload` — inner record `PatternAssessment` → `PatternAssessmentPayload` (avoids collision with domain `PatternAssessment`)
- `CoachingWorkerFactory` — prompt building
- CBR layer (`SC2GameCbrCase`, `SC2CbrRetentionObserver`, `SC2ImplementationRoutingStrategy`)
- All test classes

**GamePhase record → TacticalPosture (summarisation layer):**
- `GamePhase.java` → `TacticalPosture.java` (record definition in `io.quarkmind.plugin.summarisation`)
- `GamePhaseSummariser` — return type references, constructor call `new GamePhase(...)` → `new TacticalPosture(...)`
- `GamePhaseTrigger` — parameter and field references
- `SummarisationLifecycle` — `EventStreamBus<GamePhase>` → `EventStreamBus<TacticalPosture>` (phaseBus field, line 49), `SummarisationRunner<GameMoment, GamePhase>` → `SummarisationRunner<GameMoment, TacticalPosture>` (phaseRunner, line 52), `SummarisationRunner<GamePhase, GameArc>` → `SummarisationRunner<TacticalPosture, GameArc>` (arcRunner, line 53), `phaseBus()` return type (line 75)
- `GameArcSummariser` — `Summariser<GamePhase, GameArc>` → `Summariser<TacticalPosture, GameArc>` (class declaration), `List<LevelEvent<GamePhase>>` → `List<LevelEvent<TacticalPosture>>` in `summarise()` and `doSummarise()` parameters. Accessor calls `.phase()` → `.posture()`
- `NarrativeContextHolder` — `volatile GamePhase latestPhase` → `volatile TacticalPosture latestPosture` (line 37), `EventStreamBus<GamePhase>` → `EventStreamBus<TacticalPosture>` (constructor param line 46, field line 52), return type `GamePhase latestPhase()` → `TacticalPosture latestPosture()` (line 103), setter `setLatestPhase(GamePhase)` → `setLatestPosture(TacticalPosture)` (line 109). Bridges summarisation to coaching via `snapshot()`.
- `StarCraftStrategy.drl` — `import io.quarkmind.plugin.summarisation.GamePhase` → `TacticalPosture`, `/phaseStore[this.phase().equals("MID_SKIRMISH")]` → `/tacticalPostureStore[this.posture().equals("MID_SKIRMISH")]`
- `DominanceWeightAdjustment.drl` — `/phaseStore[this == "..."]` → `/tacticalPostureStore[this.posture().equals("...")]` (Phase modifier rules and combined signal rules reference the store by field name). Note: this requires the `DominanceWeightRuleUnit.phaseStore` type upgrade below.

**Rule unit field renames and type corrections (resolves naming collision + type inconsistency):**

*StrategyRuleUnit:*
- `DataStore<GamePhase> phaseStore` → `DataStore<TacticalPosture> tacticalPostureStore` (line 76)
- `DataStore<String> postureStore` → `DataStore<String> enemyPostureStore` (line 60)
- Getter renames: `getPhaseStore()` → `getTacticalPostureStore()`, `getPostureStore()` → `getEnemyPostureStore()`

*DominanceWeightRuleUnit:*
- `DataStore<String> phaseStore` → `DataStore<TacticalPosture> tacticalPostureStore` (line 15). **Type upgrade:** the current `DataStore<String>` holding bare posture names (`"DEFENSIVE_HOLD"`, `"EARLY_AGGRESSION"`) is upgraded to `DataStore<TacticalPosture>` for consistency with StrategyRuleUnit. Eliminates the type inconsistency where the same conceptual data (L3 tactical posture) is represented as `String` in one rule unit and `GamePhase` record in another.
- Getter rename: `getPhaseStore()` → `getTacticalPostureStore()`
- Populator update: whoever adds to this store (traced from `DroolsStrategyTask` or similar) passes full `TacticalPosture` records instead of raw strings. Mechanical change.

*DRL updates (manual — IDE refactor doesn't cover DRL field paths):*
- `StarCraftStrategy.drl`: `/phaseStore` → `/tacticalPostureStore`, `/postureStore` → `/enemyPostureStore`
- `DominanceWeightAdjustment.drl`: `/phaseStore` → `/tacticalPostureStore`, pattern matching updates from `this == "DEFENSIVE_HOLD"` to `this.posture().equals("DEFENSIVE_HOLD")` (now correct because the store holds `TacticalPosture` records, not strings)
- All callers of both rule units that populate these stores update field names and types

- `StrategyRuleUnit` — `DataStore<GamePhase>` → `DataStore<TacticalPosture>` (covered by field rename above)
- Associated test classes (`GamePhaseSummariserTest`, `GamePhaseTriggerTest`, `NarrativeContextHolderTest` if exists, `SummarisationLifecycleTest` if exists)

## Section 2: Structured Taxonomy (YAML)

### File location

`src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml`

### Schema

```yaml
archetypes:
  TERRAN_MARINE_RUSH:
    displayName: "Marine Rush"
    race: TERRAN
    phase: EARLY
    category: RUSH
    phaseWindow: [0.0, 5.0]
    handAuthored: true
    signature:
      units:
        - type: MARINE
          minCount: 5
      noExpansion: true
    strongCounters:
      - units: [STALKER, SENTRY]
        action: "Wall off natural, build Stalkers and a Sentry for Force Field"
    weakCounters:
      - units: [ZEALOT]
        action: "Zealots alone trade poorly — need ranged support"
    detectionSignals:
      - "5+ Marines before 4 minutes"
      - "No expansion with Marines"
```

Each entry fields:
- `displayName` — human-readable name for coaching prompts
- `race`, `phase`, `category` — must match the enum value's constructor args (validated by test)
- `phaseWindow` — `[startMin, endMin]` time range for data-driven detection
- `signature` — unit types, counts, and conditions that signal this archetype
- `strongCounters` / `weakCounters` — units + natural-language action descriptions
- `detectionSignals` — human-readable detection descriptions for coaching context

**Signature semantics:** When `signature.units` contains multiple entries, each generates an independent `SignatureFact` contributing separately to noisy-OR confidence combination. Multi-unit conjunctions (e.g., Marines AND Medivac simultaneously) require hand-authored CEP rules in Layer 2. The `signature.units` list documents all contributing unit types, but the generic rules treat each independently.

**Counter scope:** All counter entries are from the Protoss player perspective, consistent with QuarkMind's identity as a Protoss agent. The coaching pipeline always advises from this perspective.

### StrategyTaxonomy loader

```java
@ApplicationScoped
public class StrategyTaxonomy {
    // loaded from YAML at startup via @PostConstruct

    public ArchetypeEntry lookup(StrategyArchetype archetype) { ... }
    public List<ArchetypeEntry> forPhase(GamePhase phase) { ... }
    public CounterInfo countersFor(StrategyArchetype archetype) { ... }
    public List<SignatureSpec> signaturesForPhase(GamePhase phase) { ... }
}
```

### Counter records

```java
public record CounterInfo(
    List<CounterEntry> strongCounters,
    List<CounterEntry> weakCounters
) {}

public record CounterEntry(
    List<UnitType> units,
    String action
) {}
```

In `io.quarkmind.domain` — plain Java, no framework dependencies.

### Validation

`StrategyTaxonomy` uses SnakeYAML (via Quarkus `@ConfigMapping` or direct `Yaml` parser). `@PostConstruct` loading fails fast at startup on:
- Missing required fields (`race`, `phase`, `category`, `signature`, `strongCounters`)
- Unknown enum values (`UnitType`, `Race`, `ArchetypeCategory`)
- Invalid `phaseWindow` (start > end, negative values)
- YAML keys that don't map to a valid `StrategyArchetype` enum value

Fail-fast is correct — a broken taxonomy at startup is better than silent runtime failures in detection or coaching.

## Section 3: Detection Pipeline (Hybrid)

### Layer 1 — Data-driven generic rules

`PatternClassificationRuleUnit` gains a new `DataStore<SignatureFact>`:

```java
private final DataStore<SignatureFact> signatureStore = DataSource.createStore();
```

`SignatureFact` record:

```java
public record SignatureFact(
    StrategyArchetype archetype,
    UnitType unitType,
    int minCount,
    double windowStart,
    double windowEnd,
    double weight,
    boolean noExpansion,
    Race race
) {}
```

Generic Drools rules in `PatternClassification.drl`:

```drl
rule "Generic: unit count threshold"
when
    $sig: /signatureStore
    accumulate(/unitEvents[this.type() == $sig.unitType()];
               $count: count(); $count >= $sig.minCount())
    /gameTimeStore[this >= $sig.windowStart(), this <= $sig.windowEnd()]
then
    evidence.add(new EvidenceMarker($sig.archetype(), $sig.weight(),
        $count + " " + $sig.unitType() + " in window"));
end

rule "Generic: no expansion gate"
when
    $sig: /signatureStore[noExpansion == true]
    not /expansionEvents
    exists /unitEvents[this.type().race() == $sig.race()]
then
    evidence.add(new EvidenceMarker($sig.archetype(), 0.3,
        "No expansion with " + $sig.race() + " units"));
end
```

### Layer 2 — Hand-authored CEP rules

Existing counter-indication rules stay hand-authored, migrated from `EnemyArchetype` to `StrategyArchetype`:

- Expansion vs rush suppression
- Tech transition signals (Brood Lord → suppress rush, Colossus → suppress gateway rush)
- Prediction window expiry (past 5 min with no attack → suppress rush)

New hand-authored rules added as needed for complex mid/late-game patterns that can't be expressed as simple unit-count-in-time-window signatures.

### SignatureFactBuilder

```java
@ApplicationScoped
public class SignatureFactBuilder {
    @Inject StrategyTaxonomy taxonomy;

    public List<SignatureFact> buildFacts(double gameTimeMinutes) {
        // converts taxonomy signature entries into SignatureFact instances
        // filtered by time-window overlap: includes any signature whose
        // [windowStart, windowEnd] overlaps with the current game time
        // (NOT filtered by categorical phase — avoids false negatives at transitions)
    }
}
```

**Wiring:** `DroolsScoutingTask` injects `SignatureFactBuilder`. In the pattern classification block, after `sessionManager.buildPatternRuleUnit(gameTimeMin)` returns the rule unit, the task calls `signatureFactBuilder.buildFacts(gameTimeMin)` and adds each resulting `SignatureFact` to `patternData.getSignatureStore()` before firing rules.

### Migration strategy — Layer 1/Layer 2 coexistence

The 10 existing archetypes retain their hand-authored evidence rules in `PatternClassification.drl`. Their YAML entries include `signature` sections for documentation and counter data, but `SignatureFactBuilder` does NOT generate generic facts for archetypes that have hand-authored evidence rules — preventing double-fire and confidence inflation.

New archetypes (~10-15 mid/late-game) use the generic pipeline by default. Hand-authored CEP rules are added only for patterns that require multi-unit conjunctions, tech-transition detection, or counter-indication logic beyond simple unit-count-in-window.

The `StrategyTaxonomy` YAML entry gains an optional `handAuthored: true` flag. When set, `SignatureFactBuilder` skips generic fact generation for that archetype. Default is `false` (generic pipeline).

### PatternClassifier

Unchanged — aggregates `EvidenceMarker` and `ConfidenceRevision` from both layers identically. Noisy-OR combination, exponential decay, cumulative confidence tracking. Renamed type references only.

## Section 4: Phase Resolution

### PhaseResolver interface

```java
public interface PhaseResolver {
    GamePhase resolve(double gameTimeMinutes);
}
```

In `io.quarkmind.domain`. Takes a domain-level input — the caller in `DroolsScoutingTask` extracts game time from `CaseContext` and passes it. This keeps the interface genuinely framework-free and testable with plain `new`.

### TimeBasedPhaseResolver

```java
@ApplicationScoped
public class TimeBasedPhaseResolver implements PhaseResolver {
    static final double EARLY_END = 5.0;
    static final double MID_END = 12.0;

    @Override
    public GamePhase resolve(double gameTimeMinutes) {
        if (gameTimeMinutes < EARLY_END) return GamePhase.EARLY;
        if (gameTimeMinutes < MID_END) return GamePhase.MID;
        return GamePhase.LATE;
    }
}
```

### Integration

`DroolsScoutingTask` injects `PhaseResolver`. Each tick, it computes `gameTimeMin` from the game frame and calls `phaseResolver.resolve(gameTimeMin)`. The resolved phase is stored via `QuarkMindCaseFile.GAME_PHASE` on the context. `SignatureFactBuilder` receives `gameTimeMin` directly and filters by time-window overlap — it does not use the categorical phase for filtering.

**New CaseFile constant:** `QuarkMindCaseFile.GAME_PHASE = "agent.intel.game.phase"` — added to `ALL_KEYS`. Namespace: `agent.intel.*` because this is agent-derived data (resolved by a plugin), not raw game observation.

Future dynamic resolver: CDI `@Alternative` or `@ConfigProperty` selector — same interface, richer logic (expansion count, tech buildings, supply). No consumer changes.

## Section 5: Coaching Pipeline Integration

### Prompt enrichment

`CoachingWorkerFactory.buildUserMessage()` expanded. When pattern assessment is present:

```
ENEMY PATTERN: TERRAN_BIO_TIMING (confidence: 0.72)
GAME PHASE: MID
STRONG COUNTERS:
  - Stalker + Colossus: "Get Colossus tech — splash damage shreds bio balls"
  - High Templar: "Storm is devastating against clumped bio"
WEAK COUNTERS:
  - Pure Zealot: "Zealots get kited by stimmed Marines"
```

`CoachingWorkerFactory` injects `StrategyTaxonomy` and calls `countersFor(archetype)` to build the counter section.

### Standalone counter API

`StrategyTaxonomy.countersFor(StrategyArchetype)` returns `CounterInfo` — available to any consumer without LLM round-trip. The visualizer overlay (#248 or new issue), advisory pipeline, and future UI can query counters directly.

### Game phase in prompt

The resolved `GamePhase` is included in the coaching prompt context, giving the LLM phase-appropriate advice framing.

## Section 6: Calibration and Validation

### Multi-window calibration

Extend `PatternClassificationCalibrationTest` to evaluate at three checkpoints:

| Checkpoint | Game time | Phase | Accuracy threshold |
|------------|-----------|-------|-------------------|
| 3 min | 3:00 | EARLY | ≥ 70% (existing) |
| 8 min | 8:00 | MID | ≥ 60% |
| 15 min | 15:00 | LATE | ≥ 60% |

Mid and late thresholds start lower — less training data, harder ground truth derivation. Raised as the taxonomy matures.

### Expand deriveGroundTruth()

Extend to recognise mid-game signatures (bio timing = Marines + Medivacs after 5 min, mech = Siege Tanks + Thors) and late-game compositions (Brood Lord/Corruptor, Carrier deathball). The taxonomy's own signature data informs ground truth — if the replay shows the signature units in the signature window, that's the label.

### Per-category accuracy reporting

Report accuracy per `ArchetypeCategory` (rush, timing, harass, macro, tech, composition) rather than just aggregate rush/air-threat. Identifies where detection is weak for targeted hand-authored CEP rules.

### Taxonomy coverage test

Unit test asserting:
- Every `StrategyArchetype` enum value has a YAML entry with non-empty `signature`, `strongCounters`, and `detectionSignals`
- Every YAML key maps to a valid `StrategyArchetype` enum value
- **`handAuthored` safety check:** for each archetype with `handAuthored: true`, verify that `PatternClassification.drl` contains at least one rule whose condition references that archetype enum value. Simple text scan of the DRL file content — prevents silent detection blindness if a hand-authored rule is accidentally deleted during refactoring while the `handAuthored` flag remains set.
- **Generic pipeline validation:** for each archetype with `handAuthored: false` (or absent), verify that the YAML signature has non-empty `units` with valid `UnitType` values and a valid `phaseWindow` — ensuring the generic pipeline can actually generate `SignatureFact` instances for it.

Catches drift between code, data, and DRL rules in all directions.

### Existing test migration

- `EnemyArchetypeTest` → `StrategyArchetypeTest` — validate race, phase, category fields
- `PatternClassificationRuleUnitTest`, `DroolsScoutingTaskTest`, `DominanceWeightRuleUnitTest` — rename references, logic unchanged
- `PatternClassificationCalibrationTest` — rename + multi-window extension

## Section 7: Scope and Issue Tracking

### This branch (#243) delivers

1. Rename `EnemyArchetype` → `StrategyArchetype`, `EnemyPatternAssessment` → `PatternAssessment`
2. `GamePhase`, `ArchetypeCategory` enums
3. `PhaseResolver` interface + `TimeBasedPhaseResolver`
4. YAML taxonomy file + `StrategyTaxonomy` loader
5. `SignatureFact`, `SignatureFactBuilder`, generic data-driven Drools rules
6. `CounterInfo` / `CounterEntry` records + `countersFor()` API
7. Coaching prompt enrichment with counters and game phase
8. ~20-25 archetypes (existing 10 migrated + ~5 mid-game + ~5 late-game across races)
9. Multi-window calibration test (3/8/15 min)
10. Taxonomy coverage validation test

### Follow-up issues under epic #252

Filed as GitHub issues (not just spec notes):
- **Fill out taxonomy to 50-90 archetypes** — pure YAML authoring + enum values, no infrastructure changes. One issue per race or per phase.
- **Dynamic PhaseResolver** — state-based phase detection (expansion count, tech buildings, supply).
- **Visualizer counter overlay** — consume `CounterInfo` API in visualizer UI.
- **Strategy transitions** — model archetype transitions (e.g., Marine Rush → Bio Timing, Gateway Rush → Colossus tech). Requires temporal sequencing of archetype detections — a distinct layer beyond per-tick classification. Coaching-relevant for advice like "they're switching to mech — get Immortals."
- **Multi-race counters** — matchup-keyed counter data for non-Protoss perspectives. Required when/if the agent supports multiple player races or generic replay analysis tools consume the taxonomy.

## Package placement

| Component | Package | Rationale |
|-----------|---------|-----------|
| `StrategyArchetype`, `GamePhase`, `ArchetypeCategory` | `io.quarkmind.domain` | Plain Java domain model |
| `PatternAssessment` | `io.quarkmind.domain` | Plain Java record |
| `CounterInfo`, `CounterEntry` | `io.quarkmind.domain` | Plain Java records |
| `PhaseResolver` | `io.quarkmind.domain` | Interface — no framework deps |
| `TimeBasedPhaseResolver` | `io.quarkmind.agent` | CDI bean, game-loop integration |
| `StrategyTaxonomy` | `io.quarkmind.agent` | CDI bean, YAML loader — not in `domain/` because it has framework deps |
| `SignatureFact`, `SignatureFactBuilder` | `io.quarkmind.plugin.scouting` | Detection pipeline internal |
| `strategy-taxonomy.yaml` | `resources/io/quarkmind/domain/` | Loaded by `StrategyTaxonomy` |