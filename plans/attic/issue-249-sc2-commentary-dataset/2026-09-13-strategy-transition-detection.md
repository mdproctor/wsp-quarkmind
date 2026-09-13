# Strategy Transition Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #260 — feat: strategy transition detection
**Issue group:** #260

**Goal:** Detect when the enemy's dominant archetype shifts (e.g., Marine Rush to Bio Timing) and emit coaching-relevant transition events consumed by coaching, advisory, workbench, and CBR pipelines.

**Architecture:** `CascadingPatternClassifier` gains a `prevDominant` field and hysteresis-gated transition detection. Raw transitions are enriched with YAML-defined coaching text by `DroolsScoutingTask`, then published through the existing scouting intel pipeline and a new CDI event. Four consumers subscribe: coaching (via moment pipeline), advisory (via moment pipeline), workbench (via CDI observer), CBR (via CDI observer).

**Tech Stack:** Java 21, Quarkus, Drools rule units, SnakeYAML, CDI events

## Global Constraints

- Domain records (`io.quarkmind.domain`) must remain plain Java — no CDI, no Quarkus imports
- `TransitionPath` carries only `displayName` + `coachingAdvice` — no `from`/`to` duplication (R1-08)
- CaseFile key uses `agent.intel.enemy.*` namespace (R1-11)
- No backward-compatibility constructor on `CascadeResult` — all callers use three-arg form (R1-03)
- CDI event firing consolidated inside `publishIntel()` (R1-07)
- `prevDominant` only updates on first-dominant or qualifying transition (R1-02)
- Switches on `GameMomentType` converted to exhaustive — no `default` arm (R1-09)

---

## Batch 1: Foundation — domain records, enum ripple, taxonomy

### Task 1: Domain records, enum values, and sealed interface extension

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyTransition.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/TransitionPath.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/StrategyTransitionPublished.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/ScoutingIntelType.java:3-10`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java:9-42`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/summarisation/GameMomentType.java:3-16`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java:35-60`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/coaching/CoachingTriggerBuilder.java:63-70`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/AdvisoryTriggerBuilder.java:61-68`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/StrategyTransitionTest.java`

**Interfaces:**
- Produces: `StrategyTransition(StrategyArchetype from, StrategyArchetype to, double fromConfidence, double toConfidence, long detectedAtFrame, TransitionPath path)`
- Produces: `TransitionPath(String displayName, String coachingAdvice)`
- Produces: `StrategyTransitionPublished(StrategyTransition transition)`
- Produces: `ScoutingIntelPayload.TransitionDetected(StrategyTransition transition)`
- Produces: `ScoutingIntelType.STRATEGY_TRANSITION`
- Produces: `GameMomentType.STRATEGY_TRANSITION`
- Produces: `QuarkMindCaseFile.STRATEGY_TRANSITION = "agent.intel.enemy.strategy.transition"`

- [ ] **Step 1: Write failing test for StrategyTransition record**

```java
// quarkmind-sc2/src/test/java/io/quarkmind/domain/StrategyTransitionTest.java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class StrategyTransitionTest {

    @Test
    void constructionWithPath() {
        var path = new TransitionPath("Marine Rush → Bio Timing", "Get Stalkers and Colossus tech");
        var t = new StrategyTransition(
            StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING,
            0.35, 0.62, 5000L, path);
        assertEquals(StrategyArchetype.TERRAN_MARINE_RUSH, t.from());
        assertEquals(StrategyArchetype.TERRAN_BIO_TIMING, t.to());
        assertEquals(0.35, t.fromConfidence(), 0.001);
        assertEquals(0.62, t.toConfidence(), 0.001);
        assertEquals(5000L, t.detectedAtFrame());
        assertNotNull(t.path());
        assertEquals("Marine Rush → Bio Timing", t.path().displayName());
    }

    @Test
    void constructionWithNullPath() {
        var t = new StrategyTransition(
            StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING,
            0.35, 0.62, 5000L, null);
        assertNull(t.path());
    }

    @Test
    void equalityAndHashCode() {
        var path = new TransitionPath("display", "advice");
        var a = new StrategyTransition(StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING, 0.3, 0.6, 100L, path);
        var b = new StrategyTransition(StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING, 0.3, 0.6, 100L, path);
        assertEquals(a, b);
        assertEquals(a.hashCode(), b.hashCode());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTransitionTest -q`
Expected: FAIL — `StrategyTransition` and `TransitionPath` do not exist

- [ ] **Step 3: Create domain records**

```java
// quarkmind-sc2/src/main/java/io/quarkmind/domain/TransitionPath.java
package io.quarkmind.domain;

public record TransitionPath(
    String displayName,
    String coachingAdvice
) {}
```

```java
// quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyTransition.java
package io.quarkmind.domain;

public record StrategyTransition(
    StrategyArchetype from,
    StrategyArchetype to,
    double fromConfidence,
    double toConfidence,
    long detectedAtFrame,
    TransitionPath path
) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTransitionTest -q`
Expected: PASS

- [ ] **Step 5: Create CDI event record**

```java
// quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/StrategyTransitionPublished.java
package io.quarkmind.agent.plugin;

import io.quarkmind.domain.StrategyTransition;

public record StrategyTransitionPublished(StrategyTransition transition) {}
```

- [ ] **Step 6: Add ScoutingIntelType.STRATEGY_TRANSITION**

Add `STRATEGY_TRANSITION` after `PATTERN_ASSESSMENT` in `ScoutingIntelType.java`:

```java
public enum ScoutingIntelType {
    THREAT_POSITION,
    POSTURE,
    TIMING_ALERT,
    ARMY_SIZE,
    BUILD_ORDER,
    PATTERN_ASSESSMENT,
    STRATEGY_TRANSITION
}
```

- [ ] **Step 7: Add TransitionDetected to ScoutingIntelPayload sealed interface**

Add the new record and update the permits clause:

```java
public sealed interface ScoutingIntelPayload
        permits ScoutingIntelPayload.ThreatPosition,
                ScoutingIntelPayload.PostureUpdate,
                ScoutingIntelPayload.TimingAlert,
                ScoutingIntelPayload.ArmySize,
                ScoutingIntelPayload.BuildOrder,
                PatternAssessmentPayload,
                ScoutingIntelPayload.TransitionDetected {
    // ... existing records unchanged ...

    record TransitionDetected(StrategyTransition transition) implements ScoutingIntelPayload {
        public ScoutingIntelType type() { return ScoutingIntelType.STRATEGY_TRANSITION; }
    }
}
```

Add import: `import io.quarkmind.domain.StrategyTransition;`

- [ ] **Step 8: Add GameMomentType.STRATEGY_TRANSITION**

Add `STRATEGY_TRANSITION` after `POSTURE_CHANGE` in `GameMomentType.java`:

```java
public enum GameMomentType {
    FIRST_CONTACT,
    BATTLE_STARTED,
    BATTLE_ENDED,
    SUPPLY_BLOCK,
    ECONOMIC_CRISIS,
    BUILDING_LOST,
    NEXUS_UNDER_ATTACK,
    SCOUT_LOST,
    TECH_TRANSITION_DETECTED,
    ARMY_SHIFT,
    POSTURE_CHANGE,
    STRATEGY_TRANSITION,
    GAME_ENDING
}
```

- [ ] **Step 9: Convert CoachingTriggerBuilder.mapMomentToTier() to exhaustive switch**

Replace the method body to add `STRATEGY_TRANSITION` and remove `default`:

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

- [ ] **Step 10: Convert AdvisoryTriggerBuilder.mapMomentTypeToTrigger() to exhaustive switch**

Replace the method body:

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

- [ ] **Step 11: Add CaseFile key constant**

Add to `QuarkMindCaseFile.java` after `ENEMY_POSTURE`:

```java
public static final String STRATEGY_TRANSITION    = "agent.intel.enemy.strategy.transition";
```

- [ ] **Step 12: Compile and run existing tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTransitionTest,CoachingTriggerBuilderTest,AdvisoryTriggerBuilderTest -q`
Expected: ALL PASS — domain records compile, exhaustive switches handle all enum values, existing tests still pass

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyTransition.java quarkmind-sc2/src/main/java/io/quarkmind/domain/TransitionPath.java quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/StrategyTransitionPublished.java quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/ScoutingIntelType.java quarkmind-sc2/src/main/java/io/quarkmind/agent/plugin/ScoutingIntelPayload.java quarkmind-sc2/src/main/java/io/quarkmind/plugin/summarisation/GameMomentType.java quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java quarkmind-sc2/src/main/java/io/quarkmind/plugin/coaching/CoachingTriggerBuilder.java quarkmind-sc2/src/main/java/io/quarkmind/agent/AdvisoryTriggerBuilder.java quarkmind-sc2/src/test/java/io/quarkmind/domain/StrategyTransitionTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: add strategy transition domain model and enum values Refs #260"
```

---

### Task 2: YAML transition paths and StrategyTaxonomy extension

**Files:**
- Modify: `quarkmind-sc2/src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/StrategyTaxonomy.java:22-213`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/StrategyTaxonomyTest.java` (extend existing)

**Interfaces:**
- Consumes: `TransitionPath(String displayName, String coachingAdvice)` from Task 1
- Consumes: `StrategyArchetype` enum from existing code
- Produces: `StrategyTaxonomy.transitionPath(StrategyArchetype from, StrategyArchetype to)` → `Optional<TransitionPath>`

- [ ] **Step 1: Write failing test for transitionPath lookup**

Add to existing `StrategyTaxonomyTest.java`:

```java
@Test
void transitionPathLookup_knownPath() {
    var taxonomy = new StrategyTaxonomy();
    taxonomy.init();
    var path = taxonomy.transitionPath(
        StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING);
    assertTrue(path.isPresent());
    assertEquals("Marine Rush → Bio Timing", path.get().displayName());
    assertFalse(path.get().coachingAdvice().isEmpty());
}

@Test
void transitionPathLookup_unknownPath() {
    var taxonomy = new StrategyTaxonomy();
    taxonomy.init();
    var path = taxonomy.transitionPath(
        StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.ZERG_BROOD_LORD);
    assertTrue(path.isEmpty());
}

@Test
void allTransitionsReferenceValidArchetypes() {
    var taxonomy = new StrategyTaxonomy();
    taxonomy.init();
    // If init() succeeds, all YAML from/to values resolved to valid enums
    // and phase ordering was validated. This test exists to document the guarantee.
    assertNotNull(taxonomy);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTaxonomyTest#transitionPathLookup_knownPath -q`
Expected: FAIL — `transitionPath` method does not exist

- [ ] **Step 3: Add transitions section to strategy-taxonomy.yaml**

Append after the closing `archetypes:` entries (at end of file):

```yaml
transitions:
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
  ZERG_LING_BANE__ZERG_ULTRALISK:
    from: ZERG_LING_BANE
    to: ZERG_ULTRALISK
    displayName: "Ling Bane → Ultralisk"
    coachingAdvice: "Ultralisk tech — they'll be tanky and fast. Get Immortals — they hard-counter Ultralisks. Avoid Zealots."
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
```

- [ ] **Step 4: Add TransitionKey record and transitionPaths map to StrategyTaxonomy**

Add fields after existing `entries` field (line 24):

```java
private record TransitionKey(StrategyArchetype from, StrategyArchetype to) {}
private final Map<TransitionKey, TransitionPath> transitionPaths = new java.util.HashMap<>();
```

- [ ] **Step 5: Add transition parsing to init()**

After the existing archetype validation loop (after line 60), add:

```java
@SuppressWarnings("unchecked")
Map<String, Map<String, Object>> transitions =
    (Map<String, Map<String, Object>>) root.get("transitions");
if (transitions != null) {
    for (var te : transitions.entrySet()) {
        Map<String, Object> td = te.getValue();
        String fromStr = (String) td.get("from");
        String toStr = (String) td.get("to");
        StrategyArchetype from, to;
        try { from = StrategyArchetype.valueOf(fromStr); }
        catch (IllegalArgumentException e) {
            throw new IllegalStateException("Transition '" + te.getKey()
                + "' has invalid 'from': " + fromStr);
        }
        try { to = StrategyArchetype.valueOf(toStr); }
        catch (IllegalArgumentException e) {
            throw new IllegalStateException("Transition '" + te.getKey()
                + "' has invalid 'to': " + toStr);
        }
        if (from.phase().ordinal() > to.phase().ordinal()) {
            throw new IllegalStateException("Transition '" + te.getKey()
                + "' goes from " + from.phase() + " to " + to.phase()
                + " — late-to-early transitions are invalid");
        }
        String displayName = (String) td.get("displayName");
        String coachingAdvice = (String) td.get("coachingAdvice");
        if (displayName == null || displayName.isEmpty()) {
            throw new IllegalStateException("Transition '" + te.getKey()
                + "' has empty displayName");
        }
        if (coachingAdvice == null || coachingAdvice.isEmpty()) {
            throw new IllegalStateException("Transition '" + te.getKey()
                + "' has empty coachingAdvice");
        }
        var key = new TransitionKey(from, to);
        if (transitionPaths.containsKey(key)) {
            throw new IllegalStateException("Duplicate transition path: "
                + from + " -> " + to);
        }
        transitionPaths.put(key, new TransitionPath(displayName, coachingAdvice));
    }
}
```

Add import: `import io.quarkmind.domain.TransitionPath;`

- [ ] **Step 6: Add transitionPath() API method**

Add after `activeSignatures()`:

```java
public Optional<TransitionPath> transitionPath(StrategyArchetype from, StrategyArchetype to) {
    return Optional.ofNullable(transitionPaths.get(new TransitionKey(from, to)));
}
```

Add import: `import java.util.Optional;`

- [ ] **Step 7: Run tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTaxonomyTest -q`
Expected: ALL PASS — YAML loads, lookup returns paths, unknown returns empty

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/resources/io/quarkmind/domain/strategy-taxonomy.yaml quarkmind-sc2/src/main/java/io/quarkmind/agent/StrategyTaxonomy.java quarkmind-sc2/src/test/java/io/quarkmind/domain/StrategyTaxonomyTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: add YAML transition paths and taxonomy lookup Refs #260"
```

---

## Batch 2: Detection + Pipeline — transitions detected and consumed end-to-end

### Task 3: CascadeResult extension and classifier transition detection

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadeResult.java:1-10`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadingPatternClassifier.java:135-228`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadingPatternClassifierTest.java` (extend existing)

**Interfaces:**
- Consumes: `StrategyTransition(from, to, fromConfidence, toConfidence, detectedAtFrame, path)` from Task 1
- Produces: `CascadeResult(List<PatternAssessment> assessments, boolean llmTriggered, StrategyTransition transition)` — three-arg, no compat constructor
- Produces: `CascadingPatternClassifier.findDominant(EnumMap<StrategyArchetype, Double>)` → `StrategyArchetype` (or null)

- [ ] **Step 1: Write failing tests for transition detection**

Add to existing `CascadingPatternClassifierTest.java`:

```java
@Test
void transitionDoesNotFire_onFirstDominant() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    var evidence = List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush signal"));
    CascadeResult result = classifier.classify(evidence, List.of(), null, Race.TERRAN, 100, 0, null, 5);
    assertNull(result.transition());
}

@Test
void transitionFires_whenDominantChangesWithSufficientGap() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    // Establish first dominant
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // New dominant with sufficient confidence and gap
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.7, "bio timing")),
        List.of(), null, Race.TERRAN, 200, 100, null, 5);
    assertNotNull(result.transition());
    assertEquals(StrategyArchetype.TERRAN_MARINE_RUSH, result.transition().from());
    assertEquals(StrategyArchetype.TERRAN_BIO_TIMING, result.transition().to());
}

@Test
void transitionDoesNotFire_whenBelowMinConfidence() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // New archetype has low confidence (below TRANSITION_MIN_CONFIDENCE = 0.4)
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.2, "weak bio")),
        List.of(), null, Race.TERRAN, 200, 100, null, 5);
    assertNull(result.transition());
}

@Test
void transitionDoesNotFire_whenGapInsufficient() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    // Both archetypes accumulate similar confidence
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.5, "rush"),
                new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.45, "bio")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // Bio timing slightly overtakes but gap < TRANSITION_GAP_MARGIN (0.15)
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.52, "bio")),
        List.of(), null, Race.TERRAN, 200, 100, null, 5);
    assertNull(result.transition());
}

@Test
void resetClearsPrevDominant() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    classifier.reset();
    // After reset, next dominant is first (no transition)
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.7, "bio")),
        List.of(), null, Race.TERRAN, 200, 0, null, 5);
    assertNull(result.transition());
}

@Test
void prevDominantUnchanged_whenBelowThreshold() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    // Establish MARINE_RUSH as dominant
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // Noise-level archetype momentarily leads — should NOT update prevDominant
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_FAST_EXPAND, 0.1, "noise")),
        List.of(), null, Race.TERRAN, 200, 100, null, 5);
    // Real transition to BIO_TIMING — "from" should be MARINE_RUSH, not FAST_EXPAND
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.8, "strong bio")),
        List.of(), null, Race.TERRAN, 300, 200, null, 5);
    assertNotNull(result.transition());
    assertEquals(StrategyArchetype.TERRAN_MARINE_RUSH, result.transition().from());
}

@Test
void cascadeResultTransitionNull_whenNoTransition() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // Same dominant — no transition
    CascadeResult result2 = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 200, 100, null, 5);
    assertNull(result2.transition());
}

@Test
void wrapperPreservesTransition() {
    var classifier = new CascadingPatternClassifier(0.3, 0.8);
    classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_MARINE_RUSH, 0.6, "rush")),
        List.of(), null, Race.TERRAN, 100, 0, null, 5);
    // Transition fires — even if wrapper replaces assessments with unknown, transition is preserved
    CascadeResult result = classifier.classify(
        List.of(new EvidenceMarker(StrategyArchetype.TERRAN_BIO_TIMING, 0.7, "bio")),
        List.of(), null, Race.TERRAN, 200, 100, null, 0);
    // enemyCount=0 so wrapper does NOT fallback (empty assessments + 0 enemies = keep empty)
    // Transition should still be present from inner classify
    // Note: with enemyCount=0 the wrapper passes through the inner result unchanged
    assertNotNull(result.transition());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CascadingPatternClassifierTest#transitionDoesNotFire_onFirstDominant -q`
Expected: FAIL — `CascadeResult` has no `transition()` method

- [ ] **Step 3: Extend CascadeResult to three-arg record**

Replace `CascadeResult.java`:

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.PatternAssessment;
import io.quarkmind.domain.StrategyTransition;
import java.util.List;

public record CascadeResult(List<PatternAssessment> assessments, boolean llmTriggered, StrategyTransition transition) {
    public CascadeResult {
        assessments = List.copyOf(assessments);
    }
}
```

- [ ] **Step 4: Update all CascadeResult constructor calls in CascadingPatternClassifier**

Update each `new CascadeResult(...)` call to pass `null` as the third argument (transition will be computed later in step 6):

Line 152: `return new CascadeResult(allAssessments(...), false, null);`
Line 163: `return new CascadeResult(allAssessments(...), false, null);`
Line 180: `return new CascadeResult(allAssessments(...), false, null);`
Line 202: `return new CascadeResult(allAssessments(...), llmTriggered, null);`
Line 215-218 (wrapper): `return new CascadeResult(List.of(new PatternAssessment(...)), result.llmTriggered(), result.transition());`

- [ ] **Step 5: Verify compilation**

Run: `mvn compile -pl quarkmind-sc2 -q`
Expected: PASS — all callers updated

- [ ] **Step 6: Add transition detection logic to CascadingPatternClassifier**

Add field after `lastProcessedLlmArchetype` (line 60):

```java
private StrategyArchetype prevDominant;
```

Add constants after `FALLBACK_CONFIDENCE` (line 37):

```java
static final double TRANSITION_MIN_CONFIDENCE = 0.4;
static final double TRANSITION_GAP_MARGIN = 0.15;
```

Add `findDominant()` static method after `allAssessments()`:

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

Add transition detection in `classify()` after line 144 (`applyRevisions(...)`) and before the LLM prior check (line 147). Compute `transition` as a local variable:

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
                prevDominant, currentDominant, prevConf, currentConf, frame, null);
            prevDominant = currentDominant;
        }
    }
}
```

Thread `transition` through all four return paths:
- Line 152: `return new CascadeResult(..., false, transition);`
- Line 163: `return new CascadeResult(..., false, transition);`
- Line 180: `return new CascadeResult(..., false, transition);`
- Line 202: `return new CascadeResult(..., llmTriggered, transition);`

Add `prevDominant = null;` to `reset()`.

Add imports: `import io.quarkmind.domain.StrategyTransition;` and `import io.quarkmind.domain.StrategyArchetype;` (if not already present).

- [ ] **Step 7: Run transition detection tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CascadingPatternClassifierTest -q`
Expected: ALL PASS

- [ ] **Step 8: Run full test suite to check for regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadeResult.java quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadingPatternClassifier.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadingPatternClassifierTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: add transition detection with hysteresis to CascadingPatternClassifier Refs #260"
```

---

### Task 4: DroolsScoutingTask integration, moment detection, and consumer wiring

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java:70-398`
- Modify: `quarkmind-sc2/src/main/resources/io/quarkmind/plugin/summarisation/MomentDetectionTask.drl:1-94`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java:154-202`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java:17-64`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java:41-106`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/TransitionPayload.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java` (extend existing)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionTaskTest.java` (extend existing)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java` (extend existing)

**Interfaces:**
- Consumes: `CascadeResult.transition()` from Task 3
- Consumes: `StrategyTaxonomy.transitionPath(from, to)` from Task 2
- Consumes: `ScoutingIntelPayload.TransitionDetected` from Task 1
- Consumes: `StrategyTransitionPublished` from Task 1
- Consumes: `QuarkMindCaseFile.STRATEGY_TRANSITION` from Task 1

- [ ] **Step 1: Write failing test for MomentDetectionTask.drl transition rule**

Add to existing `MomentDetectionTaskTest.java`:

```java
@Test
void strategyTransitionMoment_emitted() {
    var transition = new StrategyTransition(
        StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING,
        0.3, 0.6, 5000L, null);
    publishIntel(new ScoutingIntelPayload.TransitionDetected(transition));

    executeWith(5000);

    assertThat(capturedMoments.stream())
        .filteredOn(e -> e.payload().type() == GameMomentType.STRATEGY_TRANSITION)
        .hasSize(1);
    var moment = capturedMoments.stream()
        .filter(e -> e.payload().type() == GameMomentType.STRATEGY_TRANSITION)
        .findFirst().get().payload();
    assertEquals("TERRAN_MARINE_RUSH", moment.context().get("from"));
    assertEquals("TERRAN_BIO_TIMING", moment.context().get("to"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=MomentDetectionTaskTest#strategyTransitionMoment_emitted -q`
Expected: FAIL — DRL rule does not exist yet

- [ ] **Step 3: Add Strategy Transition rule to MomentDetectionTask.drl**

Add after the "Posture Change" rule (after line 81):

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

Note: The `import` for `TransitionDetected` goes at the top of the file with the other imports.

- [ ] **Step 4: Run moment detection test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=MomentDetectionTaskTest#strategyTransitionMoment_emitted -q`
Expected: PASS

- [ ] **Step 5: Add DroolsScoutingTask wiring**

Add CDI event field after `patternAssessmentPublished` (line 74):

```java
@Inject
Event<StrategyTransitionPublished> strategyTransitionPublished;
```

Add imports:
```java
import io.quarkmind.agent.plugin.StrategyTransitionPublished;
import io.quarkmind.domain.StrategyTransition;
import io.quarkmind.domain.TransitionPath;
```

In `publishIntel()`, add `instanceof` branch after the `PatternAssessmentPayload` branch (after line 397):

```java
if (payload instanceof ScoutingIntelPayload.TransitionDetected td
        && strategyTransitionPublished != null) {
    strategyTransitionPublished.fire(new StrategyTransitionPublished(td.transition()));
}
```

In `execute()`, after the existing assessment handling block (after `prevAssessments = List.of();` around line 335), add transition handling:

```java
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

Add `STRATEGY_TRANSITION` to `produces()` — add `QuarkMindCaseFile.STRATEGY_TRANSITION` to the `Set.of(...)` return.

- [ ] **Step 6: Add coaching prompt enrichment to CoachingWorkerFactory.buildUserMessage()**

After the game phase line (after line 191 `if (phase != null) ...`), add:

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

Add import: `import io.quarkmind.domain.StrategyTransition;`

- [ ] **Step 7: Write and run coaching prompt test**

Add to existing `CoachingWorkerFactoryTest.java`:

```java
@Test
void buildUserMessage_includesTransitionWithPath() {
    var taxonomy = new StrategyTaxonomy();
    taxonomy.init();
    var path = new TransitionPath("Marine Rush → Bio Timing", "Get Stalkers and Colossus tech");
    var transition = new StrategyTransition(
        StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.TERRAN_BIO_TIMING,
        0.35, 0.62, 5000L, path);
    Map<String, Object> input = new LinkedHashMap<>();
    input.put(QuarkMindCaseFile.STRATEGY_TRANSITION, transition);
    input.put(QuarkMindCaseFile.COACHING_TRIGGER, Map.of(
        "momentTypes", List.of("STRATEGY_TRANSITION"),
        "urgencyTier", "STRATEGIC"));
    String message = CoachingWorkerFactory.buildUserMessage(input, taxonomy);
    assertTrue(message.contains("STRATEGY TRANSITION:"));
    assertTrue(message.contains("TERRAN_MARINE_RUSH"));
    assertTrue(message.contains("TERRAN_BIO_TIMING"));
    assertTrue(message.contains("COACHING:"));
    assertTrue(message.contains("Get Stalkers and Colossus tech"));
}

@Test
void buildUserMessage_transitionWithoutPath_omitsCoachingLine() {
    var taxonomy = new StrategyTaxonomy();
    taxonomy.init();
    var transition = new StrategyTransition(
        StrategyArchetype.TERRAN_MARINE_RUSH, StrategyArchetype.ZERG_BROOD_LORD,
        0.3, 0.5, 5000L, null);
    Map<String, Object> input = new LinkedHashMap<>();
    input.put(QuarkMindCaseFile.STRATEGY_TRANSITION, transition);
    input.put(QuarkMindCaseFile.COACHING_TRIGGER, Map.of(
        "momentTypes", List.of("STRATEGY_TRANSITION"),
        "urgencyTier", "STRATEGIC"));
    String message = CoachingWorkerFactory.buildUserMessage(input, taxonomy);
    assertTrue(message.contains("STRATEGY TRANSITION:"));
    assertFalse(message.contains("COACHING:"));
}
```

Run: `mvn test -pl quarkmind-sc2 -Dtest=CoachingWorkerFactoryTest -q`
Expected: ALL PASS

- [ ] **Step 8: Add WorkbenchEnricher observer and TransitionPayload**

Create `TransitionPayload.java`:

```java
// quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/TransitionPayload.java
package io.quarkmind.qa.workbench;

record TransitionPayload(
    String from, String to,
    double fromConfidence, double toConfidence,
    long detectedAtFrame,
    String displayName, String coachingAdvice
) {}
```

Add observer method to `WorkbenchEnricher.java`:

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

Add import: `import io.quarkmind.agent.plugin.StrategyTransitionPublished;`

- [ ] **Step 9: Add SC2CbrRetentionObserver transition collection**

Add field after `latestArc` (line 54):

```java
private final List<StrategyTransition> transitions = new java.util.concurrent.CopyOnWriteArrayList<>();
```

Add observer method:

```java
void onStrategyTransition(@Observes StrategyTransitionPublished event) {
    transitions.add(event.transition());
}
```

Add to `onGameStarted()` (inside the method, after `latestArc.set(null);`):

```java
transitions.clear();
```

Add to `onOutcome()`, in the features-building section alongside existing enrichment:

```java
features.put("transition_count", FeatureValue.number(transitions.size()));
if (!transitions.isEmpty()) {
    List<String> transitionNames = transitions.stream()
        .map(t -> t.from().name() + "->" + t.to().name())
        .toList();
    features.put("transitions", FeatureValue.stringList(transitionNames));
}
```

Add imports:
```java
import io.quarkmind.agent.plugin.StrategyTransitionPublished;
import io.quarkmind.domain.StrategyTransition;
```

- [ ] **Step 10: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/DroolsScoutingTask.java quarkmind-sc2/src/main/resources/io/quarkmind/plugin/summarisation/MomentDetectionTask.drl quarkmind-sc2/src/main/java/io/quarkmind/plugin/coaching/CoachingWorkerFactory.java quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/TransitionPayload.java quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/DroolsScoutingTaskTest.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/summarisation/MomentDetectionTaskTest.java quarkmind-sc2/src/test/java/io/quarkmind/plugin/coaching/CoachingWorkerFactoryTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat: wire strategy transitions through scouting, coaching, workbench, and CBR pipelines Refs #260"
```

---

## References

- `2026-09-12-strategy-transition-detection-design.md` — design spec this plan implements
- `decisions.md` — 9 captured decisions (D1-D9)
- `CascadingPatternClassifier.java:52-53,135-228` — classifier with cumulative confidence and cascade tiers
- `CascadeResult.java:6-10` — current two-arg record
- `DroolsScoutingTask.java:304-335,389-398` — pattern classification block and publishIntel
- `ScoutingIntelPayload.java:9-42` — sealed interface with 6 current variants
- `ScoutingIntelType.java:3-10` — 6 current enum values
- `GameMomentType.java:3-16` — 12 current enum values
- `CoachingTriggerBuilder.java:63-70` — mapMomentToTier with default arm
- `AdvisoryTriggerBuilder.java:61-68` — mapMomentTypeToTrigger with default arm
- `CoachingWorkerFactory.java:154-202` — buildUserMessage prompt construction
- `WorkbenchEnricher.java:30-39` — existing CDI observer pattern
- `SC2CbrRetentionObserver.java:52-54,102-106` — CopyOnWriteArrayList pattern and onGameStarted
- `StrategyTaxonomy.java:22-61` — YAML loader and init pattern
- `strategy-taxonomy.yaml:1-39` — existing YAML schema
- `QuarkMindCaseFile.java:35-60` — CaseFile key constants
- `MomentDetectionTask.drl:49-57` — existing TECH_TRANSITION_DETECTED rule pattern
- GitHub #260 — focal issue
- GitHub #252 — parent epic
- GitHub #243 — strategy taxonomy infrastructure (parent spec)
