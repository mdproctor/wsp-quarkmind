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
    StrategyArchetype from,
    StrategyArchetype to,
    String displayName,
    String coachingAdvice
) {}
```

In `io.quarkmind.domain`. Loaded from YAML at startup. Represents a known, expected transition path with curated coaching text.

### StrategyTransitionPublished

```java
public record StrategyTransitionPublished(StrategyTransition transition) {}
```

In `io.quarkmind.agent.plugin`. CDI event record — parallel to `PatternAssessmentPublished`. Fired by `DroolsScoutingTask` when `CascadeResult` contains a transition.

### New CaseFile key

```java
public static final String STRATEGY_TRANSITION = "agent.intel.strategy.transition";
```

Added to `QuarkMindCaseFile`. Namespace `agent.intel.*` — agent-derived intelligence data, consistent with `GAME_PHASE`, `ENEMY_POSTURE`, etc.

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
- `from` and `to` must be valid `StrategyArchetype` enum values
- `from.phase().ordinal()` must be `<=` `to.phase().ordinal()` (no late→early transitions)
- `displayName` and `coachingAdvice` must be non-empty
- No duplicate `from→to` pairs

Fail-fast on invalid data — consistent with existing archetype validation.

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

Added to `classify()` after `mergeCumulative()` and `applyRevisions()` complete — the cumulative confidence map is fully updated for this tick.

```java
StrategyArchetype currentDominant = findDominant(cumulativeConfidence);
StrategyTransition transition = null;

if (currentDominant != null && currentDominant != prevDominant) {
    double currentConf = cumulativeConfidence.getOrDefault(currentDominant, 0.0);
    double prevConf = prevDominant != null
        ? cumulativeConfidence.getOrDefault(prevDominant, 0.0) : 0.0;

    if (prevDominant != null
            && currentConf >= TRANSITION_MIN_CONFIDENCE
            && currentConf > prevConf + TRANSITION_GAP_MARGIN) {
        transition = new StrategyTransition(
            prevDominant, currentDominant,
            prevConf, currentConf,
            frame, null);
    }
    prevDominant = currentDominant;
}
```

Key behaviours:
- **First dominant** (`prevDominant == null`): sets `prevDominant` but does NOT fire a transition. Initial classification is not a transition event.
- **Same dominant**: no-op.
- **Different dominant, below threshold**: updates `prevDominant` without firing. The new archetype becomes dominant but without enough confidence to be coaching-relevant.
- **Different dominant, above threshold with gap**: fires transition.

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

    public CascadeResult(List<PatternAssessment> assessments, boolean llmTriggered) {
        this(assessments, llmTriggered, null);
    }
}
```

The two-arg constructor preserves binary compatibility with existing callers (tests, non-transition code paths). The `transition` field is `null` when no transition occurred this tick.

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

After getting `CascadeResult` from the classifier (line ~320), check for a transition:

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
    if (strategyTransitionPublished != null) {
        strategyTransitionPublished.fire(new StrategyTransitionPublished(enriched));
    }
}
```

New CDI event field in `DroolsScoutingTask`:

```java
@Inject
Event<StrategyTransitionPublished> strategyTransitionPublished;
```

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

**CoachingTriggerBuilder.mapMomentToTier()** — add to switch:

```java
case STRATEGY_TRANSITION -> CoachingUrgencyTier.STRATEGIC;
```

**AdvisoryTriggerBuilder.mapMomentTypeToTrigger()** — add to switch:

```java
case STRATEGY_TRANSITION -> STRATEGIC_TRIGGER;
```

**CoachingWorkerFactory.buildUserMessage()** — when the input map contains a transition (via CaseContext snapshot), append a transition section:

```
STRATEGY TRANSITION: Marine Rush → Bio Timing (confidence: 0.35 → 0.62)
COACHING: They're adding Medivacs to their Marines — transition from rush to bio timing.
         Get Stalkers and Colossus tech to splash bio balls before they mass up.
COUNTERS FOR NEW STRATEGY:
  - Stalker + Colossus: "Get Colossus tech — splash damage shreds bio balls"
```

The coaching advice comes from `TransitionPath.coachingAdvice()` when the path is known. For unknown transitions (path is null), the section omits the COACHING line and the LLM receives only the from/to archetypes with counter data for the `to` archetype — consistent with the existing counter-enrichment pattern.

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
private final List<StrategyTransition> transitions = new ArrayList<>();

void onStrategyTransition(@Observes StrategyTransitionPublished event) {
    transitions.add(event.transition());
}
```

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

**Switch exhaustiveness:** All switch expressions over `ScoutingIntelType` and `GameMomentType` must add the new enum values. Known locations:
- `CoachingTriggerBuilder.mapMomentToTier()` — add `STRATEGY_TRANSITION`
- `AdvisoryTriggerBuilder.mapMomentTypeToTrigger()` — add `STRATEGY_TRANSITION`
- `ScoutingIntelBroker` (if it switches on `ScoutingIntelType`)

**CascadeResult callers:** Existing callers use the two-arg constructor and never read `transition()`. The compatibility constructor preserves this. New code in `DroolsScoutingTask` uses the three-arg result.

## References

- #243 spec — strategy taxonomy infrastructure (parent, transitions deferred to this issue)
- `CascadingPatternClassifier.java:52-53` — `cumulativeConfidence` EnumMap
- `CascadingPatternClassifier.java:55-60` — existing temporal state (LLM fallback tracking)
- `CascadingPatternClassifier.java:294-305` — `mergeCumulative()` decay + update (uniform decay)
- `CascadeResult.java:6` — existing record shape
- `DroolsScoutingTask.java:304-335` — pattern classification block (transition wiring point)
- `DroolsScoutingTask.java:389-398` — `publishIntel()` method
- `ScoutingIntelPayload.java:9-42` — sealed interface with existing variants
- `ScoutingIntelType.java:3-10` — existing enum values
- `MomentDetectionTask.drl:49-57` — existing `TECH_TRANSITION_DETECTED` rule (complementary signal)
- `CoachingTriggerBuilder.java:56-60` — `canFire()` cooldown preventing rapid-fire coaching
- `CoachingTriggerBuilder.java:63-70` — moment-to-tier mapping
- `AdvisoryTriggerBuilder.java:61-68` — moment-to-trigger mapping
- `WorkbenchEnricher.java:30-39` — existing CDI observer pattern
- `SC2CbrRetentionObserver.java:84-94` — existing event collection pattern
- `SC2GameCbrCase.java:44-87` — enriched builder with features map
- `SC2StrategyRouterTask.java:117-216` — pivot detection (parallel, independent concern)
- `StrategyTaxonomy.java:22-213` — existing loader and API
- `strategy-taxonomy.yaml:1-39` — existing YAML schema
- `QuarkMindCaseFile.java:35-60` — existing CaseFile key constants
