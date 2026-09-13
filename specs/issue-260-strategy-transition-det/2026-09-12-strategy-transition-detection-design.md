# Strategy Transition Detection Design — #260

**Issue:** casehubio/quarkmind#260
**Epic:** #252 (SC2 Strategy Knowledge Infrastructure)
**Branch:** issue-260-strategy-transition-det
**Date:** 2026-09-12

## Problem

Archetypes are detected independently per tick via `CascadingPatternClassifier`. There is no model for temporal archetype sequencing: "archetype A was dominant, now archetype B has overtaken it — this is a transition from A to B." Transitions are coaching-relevant moments: "they're switching to mech — get Immortals." Detecting the transition itself (not just the new archetype) enables proactive rather than reactive coaching.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Consumer scope | All four — coaching + advisory + workbench + CBR | Transition event carries full metadata; each consumer is a thin observer. Avoids follow-up issues for mechanical hookup. |
| Detection location | `CascadingPatternClassifier` | Already owns cumulative confidence per archetype. Detection is co-located with the confidence data it depends on. |
| Transition path model | YAML-defined in `strategy-taxonomy.yaml` | Grounds coaching advice in curated domain knowledge. Unknown transitions still detected but lack curated text. |
| Trigger mechanics | Dominant-swap with hysteresis | Prevents flip-flopping. Requires new dominant to exceed min confidence AND gap margin over old dominant. |
| Initial YAML coverage | All three matchups (PvT, PvZ, PvP) | ~15-24 paths. YAML authoring is mechanical once the schema is defined. |

SETTLED: Strong consistency for `prevDominant` tracking — only updated on first-dominant or when a qualifying transition fires (from R1-02).
SETTLED: No backward-compatibility constructor on `CascadeResult` — all callers use three-arg form (from R1-03).
SETTLED: CDI event firing inside `publishIntel()` for all intel-derived events (from R1-07).
SETTLED: `TransitionPath` carries only `displayName` + `coachingAdvice` — no `from`/`to` duplication (from R1-08).
SETTLED: CaseFile key is `agent.intel.enemy.strategy.transition` under the enemy-intel namespace (from R1-11).

## Section 1: Domain Model

### StrategyTransition

```java
public record StrategyTransition(
    StrategyArchetype from,
    StrategyArchetype to,
    double fromConfidence,
    double toConfidence,
    long detectedAtFrame,
    TransitionPath path
) {}
```

In `io.quarkmind.domain`. Plain Java — no framework dependencies. `path` is `null` for unknown transitions (transitions not defined in the YAML taxonomy).

### TransitionPath

```java
public record TransitionPath(
    String displayName,
    String coachingAdvice
) {}
```

In `io.quarkmind.domain`. Loaded from YAML at startup. Represents a known, expected transition with curated coaching text. The `from` and `to` archetypes are the map key in `StrategyTaxonomy.transitionPaths` and are carried by the parent `StrategyTransition` — `TransitionPath` carries only the value-add fields to avoid duplication and divergence.

### StrategyTransitionPublished

```java
public record StrategyTransitionPublished(StrategyTransition transition) {}
```

In `io.quarkmind.agent.plugin`. CDI event record — parallel to `PatternAssessmentPublished`. Fired by `publishIntel()` when a `TransitionDetected` payload is dispatched.

### New CaseFile key

```java
public static final String STRATEGY_TRANSITION = "agent.intel.enemy.strategy.transition";
```

Added to `QuarkMindCaseFile`. Namespace `agent.intel.enemy.*` — enemy-derived intelligence data, consistent with `ENEMY_ARMY_SIZE` (`agent.intel.enemy.army.size`), `ENEMY_BUILD_ORDER` (`agent.intel.enemy.build`), `ENEMY_POSTURE` (`agent.intel.enemy.posture`), etc. Placed under `agent.intel.enemy.strategy.*` to distinguish unambiguously from `agent.strategy.*` (our strategy decisions).

### New ScoutingIntelPayload variant

```java
record TransitionDetected(StrategyTransition transition) implements ScoutingIntelPayload {
    public ScoutingIntelType type() { return ScoutingIntelType.STRATEGY_TRANSITION; }
}
```

Added to the `ScoutingIntelPayload` sealed interface permits clause. Requires new `ScoutingIntelType.STRATEGY_TRANSITION` enum value.

### New GameMomentType

```java
STRATEGY_TRANSITION  // added to GameMomentType enum
```

Emitted by `MomentDetectionTask.drl` when a `TransitionDetected` scouting intel event is observed.

## Section 2: YAML Taxonomy Extension

### Schema

A top-level `transitions:` section in `strategy-taxonomy.yaml`, parallel to `archetypes:`:

```yaml
transitions:
  # PvT
  TERRAN_MARINE_RUSH__TERRAN_BIO_TIMING:
    from: TERRAN_MARINE_RUSH
    to: TERRAN_BIO_TIMING
    displayName: "Marine Rush → Bio Timing"
    coachingAdvice: "They're adding Medivacs to their Marines — transition from rush to bio timing. Get Stalkers and Colossus tech to splash bio balls before they mass up."

  TERRAN_BIO_TIMING__TERRAN_MARINE_TANK:
    from: TERRAN_BIO_TIMING
    to: TERRAN_MARINE_TANK
    displayName: "Bio Timing → Marine Tank"
    coachingAdvice: "Siege Tanks joining the bio — this is a positional switch. Get Immortals to break tank lines, keep Stalkers for the bio."

  TERRAN_FAST_EXPAND__TERRAN_MECH_PUSH:
    from: TERRAN_FAST_EXPAND
    to: TERRAN_MECH_PUSH
    displayName: "Fast Expand → Mech Push"
    coachingAdvice: "Economy into Mech — expect Siege Tanks and Hellions. Get Immortals early and avoid head-on engagements until you have splash."

  TERRAN_FAST_EXPAND__TERRAN_BIO_TIMING:
    from: TERRAN_FAST_EXPAND
    to: TERRAN_BIO_TIMING
    displayName: "Fast Expand → Bio Timing"
    coachingAdvice: "Economic opening into bio timing push — they'll have more units than a pure rush. Match with Stalker-Sentry and hold the natural."

  # PvZ
  ZERG_ZERGLING_RUSH__ZERG_ROACH_HYDRA:
    from: ZERG_ZERGLING_RUSH
    to: ZERG_ROACH_HYDRA
    displayName: "Zergling Rush → Roach Hydra"
    coachingAdvice: "Transitioning from ling aggression to Roach Hydra composition. Get Immortals — they hard-counter Roaches. Keep Sentries for Force Fields."

  ZERG_MACRO__ZERG_ROACH_HYDRA:
    from: ZERG_MACRO
    to: ZERG_ROACH_HYDRA
    displayName: "Macro → Roach Hydra"
    coachingAdvice: "Greedy economy into standard Roach Hydra. They'll have high drone count — match with your own economy and get Immortal-Archon."

  ZERG_ROACH_HYDRA__ZERG_BROOD_LORD:
    from: ZERG_ROACH_HYDRA
    to: ZERG_BROOD_LORD
    displayName: "Roach Hydra → Brood Lords"
    coachingAdvice: "Tech switch to Brood Lords — this is a late-game pivot. Get Tempests or mass Stalker-Blink to snipe them before they mass up."

  ZERG_MACRO__ZERG_MUTALISK_HARASS:
    from: ZERG_MACRO
    to: ZERG_MUTALISK_HARASS
    displayName: "Macro → Mutalisk Harass"
    coachingAdvice: "Mutalisks incoming — get Phoenix or mass Stalkers. Spread pylons and protect mineral lines."

  # PvP
  PROTOSS_GATEWAY_RUSH__PROTOSS_COLOSSUS_PUSH:
    from: PROTOSS_GATEWAY_RUSH
    to: PROTOSS_COLOSSUS_PUSH
    displayName: "Gateway Rush → Colossus Push"
    coachingAdvice: "Gateway aggression transitioning to Colossus tech. Get your own Robotics — either mirror Colossus or get Immortals to break them."

  PROTOSS_MACRO__PROTOSS_STALKER_COLOSSUS:
    from: PROTOSS_MACRO
    to: PROTOSS_STALKER_COLOSSUS
    displayName: "Macro → Stalker Colossus"
    coachingAdvice: "Standard macro into Stalker-Colossus. Mirror with your own Colossus or tech to Immortal-Archon with Prism harass."

  PROTOSS_MACRO__PROTOSS_AIR_SUPERIORITY:
    from: PROTOSS_MACRO
    to: PROTOSS_AIR_SUPERIORITY
    displayName: "Macro → Air Superiority"
    coachingAdvice: "Air tech switch — Void Rays or Phoenix coming. Get Stalkers and consider Archons for anti-air splash."

  # Late-game transitions (cross-matchup)
  TERRAN_MARINE_TANK__TERRAN_BC_TRANSITION:
    from: TERRAN_MARINE_TANK
    to: TERRAN_BC_TRANSITION
    displayName: "Marine Tank → Battlecruiser"
    coachingAdvice: "Tech switch to Battlecruisers — get Tempests or Void Rays. Keep ground pressure while they tech up."

  TERRAN_MECH_PUSH__TERRAN_MECH_LATE:
    from: TERRAN_MECH_PUSH
    to: TERRAN_MECH_LATE
    displayName: "Mech Push → Late Mech"
    coachingAdvice: "Entrenching into late-game mech — Thors, Siege Tanks, Hellbats. Get Immortals and Tempests. Avoid frontal engagements — harass expansions."

  ZERG_LING_BANE__ZERG_ULTRALISK:
    from: ZERG_LING_BANE
    to: ZERG_ULTRALISK
    displayName: "Ling Bane → Ultralisk"
    coachingAdvice: "Ultralisk tech — they'll be tanky and fast. Get Immortals — they hard-counter Ultralisks. Avoid Zealots."
```

The remaining ~10 paths follow the same pattern — EARLY→MID and MID→LATE within each race. Additional paths are pure YAML authoring with no code changes.

### Validation

`StrategyTaxonomy.init()` validates each transition entry at startup:
- YAML `from` and `to` must be valid `StrategyArchetype` enum values
- `from.phase().ordinal()` must be `<=` `to.phase().ordinal()` (no late→early transitions)
- `displayName` and `coachingAdvice` must be non-empty
- No duplicate `from→to` pairs

Fail-fast on invalid data — consistent with existing archetype validation. The validated `from`/`to` are used to construct the `TransitionKey` map key; `TransitionPath` receives only `displayName` and `coachingAdvice`.

### StrategyTaxonomy API addition

```java
public Optional<TransitionPath> transitionPath(StrategyArchetype from, StrategyArchetype to) {
    return Optional.ofNullable(transitionPaths.get(new TransitionKey(from, to)));
}
```

Internal `TransitionKey` record (private to `StrategyTaxonomy`):

```java
private record TransitionKey(StrategyArchetype from, StrategyArchetype to) {}
```

The `transitionPaths` map (`Map<TransitionKey, TransitionPath>`) is populated in `init()` alongside the existing `entries` map.

## Section 3: Detection in CascadingPatternClassifier

### New state

```java
private StrategyArchetype prevDominant;
```

Initially `null`. Cleared by `reset()`.

### Constants

```java
static final double TRANSITION_MIN_CONFIDENCE = 0.4;
static final double TRANSITION_GAP_MARGIN = 0.15;
```

`TRANSITION_MIN_CONFIDENCE` — the new dominant must have at least this cumulative confidence before a transition fires. Prevents transitions from being triggered by low-confidence noise.

`TRANSITION_GAP_MARGIN` — the new dominant must exceed the old dominant's confidence by this margin. Prevents flip-flopping between closely-matched archetypes.

### Detection logic

Added to `classify()` after `mergeCumulative()` and `applyRevisions()` complete. Detection runs after rule-based evidence accumulation but before ONNX inference — transitions are driven by stable cumulative evidence rather than single-tick model predictions. ONNX contributions from prior ticks are already reflected in cumulative via the merge+decay cycle.

```java
StrategyArchetype currentDominant = findDominant(cumulativeConfidence);
StrategyTransition transition = null;

if (currentDominant != null && currentDominant != prevDominant) {
    if (prevDominant == null) {
        prevDominant = currentDominant;
    } else {
        double currentConf = cumulativeConfidence.getOrDefault(currentDominant, 0.0);
        double prevConf = cumulativeConfidence.getOrDefault(prevDominant, 0.0);
        if (currentConf >= TRANSITION_MIN_CONFIDENCE
                && currentConf > prevConf + TRANSITION_GAP_MARGIN) {
            transition = new StrategyTransition(
                prevDominant, currentDominant,
                prevConf, currentConf,
                frame, null);
            prevDominant = currentDominant;
        }
    }
}
```

Key behaviours:
- **First dominant** (`prevDominant == null`): sets `prevDominant` but does NOT fire a transition. Initial classification is not a transition event.
- **Same dominant**: no-op.
- **Different dominant, below threshold**: `prevDominant` unchanged. The new archetype must prove itself via a qualifying transition before becoming the reference point. This prevents noise-level dominant fluctuations from corrupting the "from" field of future transitions.
- **Different dominant, above threshold with gap**: fires transition, updates `prevDominant`.

`findDominant()` — static helper returning the archetype with the highest cumulative confidence, or `null` if the map is empty or all confidences are at/below `NOISE_FLOOR`:

```java
static StrategyArchetype findDominant(EnumMap<StrategyArchetype, Double> cumulative) {
    StrategyArchetype best = null;
    double bestConf = NOISE_FLOOR;
    for (var e : cumulative.entrySet()) {
        if (e.getValue() > bestConf) {
            bestConf = e.getValue();
            best = e.getKey();
        }
    }
    return best;
}
```

### Transition threading through classify()

The `transition` variable is computed once (after `mergeCumulative()` and `applyRevisions()`), then threaded through every return path in `classify()`:

- LLM prior result return: `new CascadeResult(assessments, false, transition)`
- Tier 1 Drools return: `new CascadeResult(assessments, false, transition)`
- Tier 2 ONNX return: `new CascadeResult(assessments, false, transition)`
- Final return (Tier 3 LLM trigger): `new CascadeResult(assessments, llmTriggered, transition)`

The wrapper `classify(…, enemyCount)` preserves transition when constructing the unknown-archetype fallback:

```java
return new CascadeResult(List.of(unknownAssessment), result.llmTriggered(), result.transition());
```

There is no two-arg constructor — the compiler enforces that every `CascadeResult` construction explicitly passes a transition value (`null` when no transition occurred this tick).

### CascadeResult extension

```java
public record CascadeResult(
    List<PatternAssessment> assessments,
    boolean llmTriggered,
    StrategyTransition transition
) {
    public CascadeResult {
        assessments = List.copyOf(assessments);
    }
}
```

All callers use the three-arg constructor. Existing non-transition code paths pass `null` for `transition`. There is no backward-compatibility constructor — breaking existing callers forces explicit transition handling at every call site.

### reset()

```java
public void reset() {
    cumulativeConfidence.clear();
    prevDominant = null;
    // existing reset logic unchanged
}
```

### Evidence disappearance mitigation

When enemy visibility is lost (scout dies), two mechanisms prevent false transitions: (1) `mergeCumulative` applies uniform decay to ALL archetypes — if no new evidence arrives, all decay at the same rate and relative ordering is preserved; (2) `TRANSITION_MIN_CONFIDENCE = 0.4` ensures the "to" archetype needs substantial cumulative evidence, which noise alone cannot produce.

## Section 4: DroolsScoutingTask Integration

### Transition handling

After getting `CascadeResult` from the classifier, check for a transition:

```java
CascadeResult cascadeResult = cascadingClassifier.classify(...);

// Existing assessment handling unchanged ...

if (cascadeResult.transition() != null) {
    StrategyTransition raw = cascadeResult.transition();
    TransitionPath path = taxonomy.transitionPath(raw.from(), raw.to()).orElse(null);
    StrategyTransition enriched = new StrategyTransition(
        raw.from(), raw.to(), raw.fromConfidence(), raw.toConfidence(),
        raw.detectedAtFrame(), path);

    ctx.set(QuarkMindCaseFile.STRATEGY_TRANSITION, enriched);
    publishIntel(new ScoutingIntelPayload.TransitionDetected(enriched));
}
```

### publishIntel() extension

CDI event firing for `StrategyTransitionPublished` is consolidated inside `publishIntel()`, parallel to the existing `PatternAssessmentPublished` pattern. This maintains a single pattern for "dispatch an intel payload and also fire a CDI event for observers":

```java
private void publishIntel(ScoutingIntelPayload payload) {
    if (broker.isSubscribed(payload.type())) {
        broker.update(payload);
    }
    broker.level1Bus().publish(new LevelEvent<>(payload, lastFrame, LEVEL_1, "default"));
    dispatchToAdvisory(payload);
    if (payload instanceof PatternAssessmentPayload pa && patternAssessmentPublished != null) {
        patternAssessmentPublished.fire(new PatternAssessmentPublished(pa.assessments()));
    }
    if (payload instanceof ScoutingIntelPayload.TransitionDetected td
            && strategyTransitionPublished != null) {
        strategyTransitionPublished.fire(new StrategyTransitionPublished(td.transition()));
    }
}
```

New CDI event field in `DroolsScoutingTask`:

```java
@Inject
Event<StrategyTransitionPublished> strategyTransitionPublished;
```

### produces() declaration

`STRATEGY_TRANSITION` added to the `produces()` set, declaring this task writes the key:

```java
@Override
public Set<String> produces() {
    return Set.of(
        QuarkMindCaseFile.ENEMY_ARMY_SIZE,
        QuarkMindCaseFile.ENEMY_BUILD_ORDER,
        QuarkMindCaseFile.TIMING_ATTACK_INCOMING,
        QuarkMindCaseFile.ENEMY_POSTURE,
        QuarkMindCaseFile.GAME_PHASE,
        QuarkMindCaseFile.SCOUTING_FINAL_ASSESSMENT,
        QuarkMindCaseFile.STRATEGY_TRANSITION);
}
```

### Broker subscription

No `ScoutingIntelConsumer` subscribes to `STRATEGY_TRANSITION`. This is by design — transition data flows to consumers via:
1. **CaseContext** (`ctx.set`) — for task pipeline reads
2. **L1 bus** — for `MomentDetectionTask` Drools rules → coaching pipeline
3. **CDI events** (`StrategyTransitionPublished`) — for CBR and workbench observers
4. **Advisory dispatch** — for real-time advisory messages

The broker's `latest` map (synchronous read via `broker.current()`) is not needed because no downstream task reads transitions synchronously. This differs from `PATTERN_ASSESSMENT`, where `SC2StrategyRouterTask` reads the latest assessment from the broker for strategy routing. If a future consumer needs synchronous access, it can implement `ScoutingIntelConsumer` and add `STRATEGY_TRANSITION` to its `subscribedIntelTypes()`.

### Why enrichment happens here, not in the classifier

The classifier returns a raw `StrategyTransition` with `path = null`. `DroolsScoutingTask` enriches it by looking up the `TransitionPath` from `StrategyTaxonomy`. This keeps the classifier free of CDI dependencies — it remains a testable POJO with constructor injection for thresholds.

### Relationship to SC2StrategyRouterTask pivot detection

`SC2StrategyRouterTask` independently reads the current best pattern assessment from `ScoutingIntelBroker`, detects archetype changes, and increments `pivotCount` for strategy routing. This is strategy selection — "which counter-strategy should WE play?" The transition event drives coaching — "the opponent is switching to mech, here's advice." A single archetype change produces one router pivot (via pattern assessment) and one transition event (via transition detection). They serve different downstream effects and should remain independent.

### Relationship to TECH_TRANSITION_DETECTED

`TECH_TRANSITION_DETECTED` fires from `MomentDetectionTask.drl` when a `BuildOrder` intel event has a non-UNKNOWN value — an early warning from what they're building. `STRATEGY_TRANSITION` fires when the dominant archetype shifts in cumulative classification — a confirmed transition from what they have. Build order detection fires earlier (Factory scouted), composition classification fires later (enough Tanks observed). Both are coaching-relevant and both map to `CoachingUrgencyTier.STRATEGIC`. The existing `CoachingTriggerBuilder.canFire()` cooldown prevents rapid-fire coaching for closely-spaced moments of the same tier.

## Section 5: Consumer Wiring

### 5a. Moment Detection → Coaching Pipeline

**MomentDetectionTask.drl** — new rule:

```drl
import io.quarkmind.agent.plugin.ScoutingIntelPayload.TransitionDetected;

rule "Strategy Transition"
    salience 155
when
    $transition : /intelEvents[this instanceof TransitionDetected]
then
    detectedMoments.add(new GameMoment(
        GameMomentType.STRATEGY_TRANSITION, currentFrame,
        Map.of("from", ((TransitionDetected) $transition).transition().from().name(),
               "to",   ((TransitionDetected) $transition).transition().to().name())));
end
```

Salience 155 places it between "Tech Transition Detected" (160) and "Army Shift" (150) — strategy-level transitions are more significant than army count shifts but at the same level as build order changes.

**CoachingTriggerBuilder.mapMomentToTier()** — convert to exhaustive switch, add `STRATEGY_TRANSITION`:

```java
static CoachingUrgencyTier mapMomentToTier(GameMomentType type) {
    return switch (type) {
        case NEXUS_UNDER_ATTACK, BATTLE_STARTED, BUILDING_LOST -> CoachingUrgencyTier.CRISIS;
        case TECH_TRANSITION_DETECTED, ARMY_SHIFT, POSTURE_CHANGE,
             FIRST_CONTACT, STRATEGY_TRANSITION -> CoachingUrgencyTier.STRATEGIC;
        case ECONOMIC_CRISIS, SUPPLY_BLOCK -> CoachingUrgencyTier.ECONOMIC;
        case BATTLE_ENDED, SCOUT_LOST, GAME_ENDING -> null;
    };
}
```

**AdvisoryTriggerBuilder.mapMomentTypeToTrigger()** — convert to exhaustive switch, add `STRATEGY_TRANSITION`:

```java
private static String mapMomentTypeToTrigger(GameMomentType type) {
    return switch (type) {
        case NEXUS_UNDER_ATTACK, BATTLE_STARTED -> CRISIS_TRIGGER;
        case TECH_TRANSITION_DETECTED, STRATEGY_TRANSITION -> STRATEGIC_TRIGGER;
        case ECONOMIC_CRISIS, SUPPLY_BLOCK -> ECONOMIC_TRIGGER;
        case ARMY_SHIFT, POSTURE_CHANGE, FIRST_CONTACT, BUILDING_LOST,
             BATTLE_ENDED, SCOUT_LOST, GAME_ENDING -> null;
    };
}
```

Both switches are converted from `default -> null` to exhaustive form (no `default` arm). The compiler will enforce that any future `GameMomentType` value is explicitly handled — new values produce a compile error rather than silently falling into a default.

**CoachingWorkerFactory.buildUserMessage()** — add transition section after existing trigger handling, before game state:

```java
Object transitionObj = input.get(QuarkMindCaseFile.STRATEGY_TRANSITION);
if (transitionObj instanceof StrategyTransition transition) {
    sb.append("\nSTRATEGY TRANSITION: ")
      .append(transition.from().name()).append(" → ").append(transition.to().name())
      .append(" (confidence: ")
      .append(String.format("%.2f", transition.fromConfidence()))
      .append(" → ").append(String.format("%.2f", transition.toConfidence()))
      .append(")\n");
    if (transition.path() != null) {
        sb.append("COACHING: ").append(transition.path().coachingAdvice()).append("\n");
    }
    if (taxonomy != null) {
        var counters = taxonomy.countersFor(transition.to());
        if (counters != null) {
            appendCounters(sb, "COUNTERS FOR NEW STRATEGY", counters.strongCounters());
        }
    }
}
```

The transition section appends alongside the existing pattern assessment section — both are relevant context for the LLM. When the transition path is unknown (`path == null`), the COACHING line is omitted and the LLM receives only the from/to archetypes with counter data for the `to` archetype. Counter enrichment uses `taxonomy.countersFor(transition.to())`, consistent with the existing counter-enrichment pattern in the trigger's `patternAssessment` handling. If the `to` archetype has no counters for the player's race, the counters section is omitted (same as existing behaviour for unknown archetypes).

The `buildUserMessage()` method receives `Map<String, Object> input` — a snapshot of `CaseContext` values. `STRATEGY_TRANSITION` is set on the context by `DroolsScoutingTask.execute()` before the coaching worker runs, so it is available in the input map as a `StrategyTransition` record.

### 5b. Workbench

**WorkbenchEnricher** — new observer:

```java
void onStrategyTransition(@Observes StrategyTransitionPublished event) {
    var t = event.transition();
    broadcaster.broadcast(new WorkbenchEvent("transition",
        new TransitionPayload(
            t.from().name(), t.to().name(),
            t.fromConfidence(), t.toConfidence(),
            t.detectedAtFrame(),
            t.path() != null ? t.path().displayName() : null,
            t.path() != null ? t.path().coachingAdvice() : null)));
}
```

`TransitionPayload` — new record in `io.quarkmind.qa.workbench`:

```java
record TransitionPayload(
    String from, String to,
    double fromConfidence, double toConfidence,
    long detectedAtFrame,
    String displayName, String coachingAdvice
) {}
```

### 5c. CBR

**SC2CbrRetentionObserver** — collect transitions alongside moments:

```java
private final List<StrategyTransition> transitions = new CopyOnWriteArrayList<>();

void onStrategyTransition(@Observes StrategyTransitionPublished event) {
    transitions.add(event.transition());
}
```

Thread-safe via `CopyOnWriteArrayList`, consistent with the existing `moments` and `phases` collections (`SC2CbrRetentionObserver.java:52-53`). The CDI `@Observes` callback runs on the CDI event thread while `onOutcome()` runs on the game-end thread — concurrent access requires a thread-safe collection.

Clear in `onGameStarted()`. In `onOutcome()`, add transition features to `EnrichedGameData`:

```java
features.put("transition_count", FeatureValue.number(transitions.size()));
if (!transitions.isEmpty()) {
    List<String> transitionNames = transitions.stream()
        .map(t -> t.from().name() + "->" + t.to().name())
        .toList();
    features.put("transitions", FeatureValue.stringList(transitionNames));
}
```

This enriches the existing `SC2GameCbrCase` features map — no schema change to the `SC2GameCbrCase` record itself. Transition data is stored as feature values, consistent with how other enrichment data (phases, moments, timeline) is stored.

## Section 6: Testing

### Unit tests

**`StrategyTransitionTest`** — new test class in `io.quarkmind.domain`:
- Record construction, equality, toString
- Transition with and without path
- Null path handling

**`CascadingPatternClassifierTest`** — extend existing test class:
- `transitionFires_whenDominantChangesWithSufficientGap` — set up cumulative confidence for archetype A, advance ticks until archetype B overtakes with gap
- `transitionDoesNotFire_whenBelowMinConfidence` — new dominant has high gap but confidence below 0.4
- `transitionDoesNotFire_whenGapInsufficient` — new dominant above threshold but gap below 0.15
- `transitionDoesNotFire_onFirstDominant` — prevDominant is null, no transition event
- `noFlipFlop_whenConfidencesOscillate` — alternating evidence for A and B doesn't produce spurious transitions
- `resetClearsPrevDominant` — after reset, next dominant is treated as first (no transition)
- `cascadeResultCarriesTransition` — verify CascadeResult.transition() is non-null when transition fires
- `cascadeResultTransitionNull_whenNoTransition` — verify null on normal ticks
- `prevDominantUnchanged_whenBelowThreshold` — verify that a noise-level dominant change does NOT update prevDominant, so subsequent real transitions have the correct "from" field
- `wrapperPreservesTransition` — verify `classify(…, enemyCount)` unknown-archetype fallback preserves transition from inner result

**`StrategyTaxonomyTest`** — extend existing test class:
- `transitionPathLookup_knownPath` — returns TransitionPath with coaching advice
- `transitionPathLookup_unknownPath` — returns empty Optional
- `transitionYamlValidation_invalidFrom` — startup fails fast
- `transitionYamlValidation_invalidPhaseOrder` — late→early fails fast
- `allTransitionsReferenceValidArchetypes` — coverage test: every YAML `from`/`to` is a valid enum value

**`DroolsScoutingTaskTest`** — extend existing test class:
- `transitionPublishedWhenClassifierDetectsTransition` — verify CDI event fires
- `transitionEnrichedWithPath` — verify taxonomy lookup fills TransitionPath
- `transitionWithoutKnownPath_pathIsNull` — unknown transition still published

### Integration tests

**`MomentDetectionTaskTest`** — extend existing test class:
- `strategyTransitionMoment_emitted` — publish a `TransitionDetected` intel event, verify `STRATEGY_TRANSITION` moment is emitted with from/to in metadata

### Calibration

Extend `PatternClassificationCalibrationTest`:
- At the 8-min checkpoint, verify that replays with known mid-game archetype shifts (e.g., Marine Rush → Bio Timing in IEM10 TvP games) produce at least one transition event
- Accuracy threshold: ≥ 50% of expected transitions detected (lower than archetype detection because transition timing is harder to derive from replays)

## Section 7: Package Placement

| Component | Package | Rationale |
|-----------|---------|-----------|
| `StrategyTransition` | `io.quarkmind.domain` | Plain Java record — domain model |
| `TransitionPath` | `io.quarkmind.domain` | Plain Java record — domain model |
| `StrategyTransitionPublished` | `io.quarkmind.agent.plugin` | CDI event — parallel to `PatternAssessmentPublished` |
| `TransitionDetected` | `io.quarkmind.agent.plugin.ScoutingIntelPayload` | Sealed interface variant — consistent with other payloads |
| `STRATEGY_TRANSITION` | `io.quarkmind.agent.plugin.ScoutingIntelType` | Enum value |
| `STRATEGY_TRANSITION` | `io.quarkmind.plugin.summarisation.GameMomentType` | Enum value |
| `STRATEGY_TRANSITION` | `io.quarkmind.agent.QuarkMindCaseFile` | CaseFile key constant |
| `TransitionPayload` | `io.quarkmind.qa.workbench` | Workbench WebSocket payload |
| `transitionPath()` | `io.quarkmind.agent.StrategyTaxonomy` | API method on existing CDI bean |
| Detection logic | `io.quarkmind.plugin.scouting.CascadingPatternClassifier` | Co-located with confidence tracking |
| Transition wiring | `io.quarkmind.plugin.scouting.DroolsScoutingTask` | Existing scouting orchestrator |

### Ripple

**Sealed interface update:** `ScoutingIntelPayload` permits clause must add `TransitionDetected`.

**Switch exhaustiveness:** `CoachingTriggerBuilder.mapMomentToTier()` and `AdvisoryTriggerBuilder.mapMomentTypeToTrigger()` currently use `default -> null`, which silently swallows new enum values. Both switches are converted to exhaustive form (no `default` arm, all unhandled cases explicitly enumerated as `-> null`) as part of this change — see §5a. This ensures the compiler enforces explicit handling of every `GameMomentType` value, including `STRATEGY_TRANSITION`. `ScoutingIntelBroker` uses a map, not a switch, and requires no changes.

**CascadeResult callers:** All existing callers of `CascadeResult` (four return paths in `classify()`, one in the wrapper `classify(…, enemyCount)`, plus tests) must be updated to use the three-arg constructor. There is no backward-compatibility constructor — the compiler catches every call site.

## References

- #243 spec — strategy taxonomy infrastructure (parent, transitions deferred to this issue)
- `CascadingPatternClassifier.java:52-53` — `cumulativeConfidence` EnumMap
- `CascadingPatternClassifier.java:55-60` — existing temporal state (LLM fallback tracking)
- `CascadingPatternClassifier.java:294-305` — `mergeCumulative()` decay + update (uniform decay)
- `CascadeResult.java:6` — existing record shape
- `DroolsScoutingTask.java:304-335` — pattern classification block (transition wiring point)
- `DroolsScoutingTask.java:389-398` — `publishIntel()` method
- `DroolsScoutingTask.java:351-359` — `produces()` method
- `ScoutingIntelPayload.java:9-42` — sealed interface with existing variants
- `ScoutingIntelType.java:3-10` — existing enum values
- `MomentDetectionTask.drl:49-57` — existing `TECH_TRANSITION_DETECTED` rule (complementary signal)
- `CoachingTriggerBuilder.java:56-60` — `canFire()` cooldown preventing rapid-fire coaching
- `CoachingTriggerBuilder.java:63-70` — moment-to-tier mapping (converted to exhaustive)
- `AdvisoryTriggerBuilder.java:61-68` — moment-to-trigger mapping (converted to exhaustive)
- `CoachingWorkerFactory.java:159-186` — `buildUserMessage()` method
- `WorkbenchEnricher.java:30-39` — existing CDI observer pattern
- `SC2CbrRetentionObserver.java:52-53` — existing thread-safe collections (`CopyOnWriteArrayList`)
- `SC2CbrRetentionObserver.java:84-94` — existing event collection pattern
- `SC2GameCbrCase.java:44-87` — enriched builder with features map
- `SC2StrategyRouterTask.java:117-216` — pivot detection (parallel, independent concern)
- `StrategyTaxonomy.java:22-213` — existing loader and API
- `strategy-taxonomy.yaml:1-39` — existing YAML schema
- `QuarkMindCaseFile.java:35-60` — existing CaseFile key constants
