# Temporal Strategy Evolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #471 — Dynamic Bayes net for temporal strategy evolution — model how strategies unfold over game time
**Issue group:** #471

**Goal:** Replace discrete phase-transition reinitialisation with continuous temporal inference across both DetectorOrchestrator and HypothesisTrackerFusion.

**Architecture:** Compose temporal blending (decay-based posterior carry-over) with existing EvidenceFusionEngine/HypothesisTracker. Unified archetype set eliminates phase-transition rebuilds. Transition probabilities learned offline from replay data.

**Tech Stack:** Java 21, Quarkus, neocortex inference-bayes (BayesNetwork, HypothesisTracker, EvidenceFusionEngine), JUnit 5, Jackson

## Global Constraints

- All new classes in `io.quarkmind.sc2.detector` package (quarkmind-sc2 module)
- Phase-window decay floor: 0.01 (never fully zero)
- Default temporal decay: 0.3
- Transition frequency threshold: 5% for discovered transitions
- Hierarchical smoothing α: 10.0
- Taxonomy transition prior boost: 50 additional pseudo-observations
- Replay checkpoints: 1, 3, 5, 7, 10 minutes (game loops: 1344, 4032, 6720, 9408, 13440)
- Supersedes ADR-0010's windowing choice — document in ADR update

---

## Batch 1: Phase-window decay function and transition CPT loading

### Task 1: Phase-window decay function

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/PhaseWindowDecay.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/PhaseWindowDecayTest.java`

**Interfaces:**
- Consumes: `StrategyArchetype.phase()` → `GamePhase`, `StrategyTaxonomy.lookup(archetype).phaseWindow()` → `double[2]`
- Produces: `static double weight(double gameTimeMinutes, double windowMin, double windowMax, double ramp)` — returns [0.01, 1.0]; `static Map<String, Double> computePhaseWeights(String[] archetypes, StrategyTaxonomy taxonomy, double gameTimeMinutes)` — weights per archetype label

- [ ] **Step 1: Write failing test for inside-window weight**

```java
@Test
void insideWindowReturnsOne() {
    // archetype with window [2.0, 6.0], game time at 4.0
    double w = PhaseWindowDecay.weight(4.0, 2.0, 6.0, 1.0);
    assertEquals(1.0, w, 1e-6);
}
```

- [ ] **Step 2: Run test — verify FAIL (class not found)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PhaseWindowDecayTest#insideWindowReturnsOne -q`

- [ ] **Step 3: Write failing tests for outside-window and ramp regions**

```java
@Test
void beforeWindowReturnsFloor() {
    double w = PhaseWindowDecay.weight(0.5, 2.0, 6.0, 1.0);
    assertEquals(0.01, w, 1e-6);
}

@Test
void afterWindowReturnsFloor() {
    double w = PhaseWindowDecay.weight(8.0, 2.0, 6.0, 1.0);
    assertEquals(0.01, w, 1e-6);
}

@Test
void rampBeforeWindowIsBetweenFloorAndOne() {
    // ramp region: [wMin - ramp, wMin] = [1.0, 2.0], midpoint at 1.5
    double w = PhaseWindowDecay.weight(1.5, 2.0, 6.0, 1.0);
    assertTrue(w > 0.01 && w < 1.0, "ramp weight should be between floor and 1.0, got " + w);
}

@Test
void rampAfterWindowIsBetweenFloorAndOne() {
    double w = PhaseWindowDecay.weight(6.5, 2.0, 6.0, 1.0);
    assertTrue(w > 0.01 && w < 1.0, "ramp weight should be between floor and 1.0, got " + w);
}

@Test
void floorIsNeverZero() {
    double w = PhaseWindowDecay.weight(100.0, 0.0, 4.0, 1.0);
    assertTrue(w >= 0.01, "weight should never be less than floor");
}
```

- [ ] **Step 4: Implement PhaseWindowDecay**

```java
package io.quarkmind.sc2.detector;

import io.quarkmind.agent.StrategyTaxonomy;
import java.util.LinkedHashMap;
import java.util.Map;

public final class PhaseWindowDecay {

    private static final double FLOOR = 0.01;

    private PhaseWindowDecay() {}

    public static double weight(double gameTimeMinutes, double windowMin, double windowMax, double ramp) {
        if (gameTimeMinutes < windowMin - ramp) {
            return FLOOR;
        } else if (gameTimeMinutes < windowMin) {
            return FLOOR + (1.0 - FLOOR) * sigmoid((gameTimeMinutes - windowMin + ramp / 2) / (ramp / 4));
        } else if (gameTimeMinutes <= windowMax) {
            return 1.0;
        } else if (gameTimeMinutes < windowMax + ramp) {
            return FLOOR + (1.0 - FLOOR) * sigmoid((windowMax + ramp / 2 - gameTimeMinutes) / (ramp / 4));
        } else {
            return FLOOR;
        }
    }

    public static Map<String, Double> computePhaseWeights(
            String[] archetypes, StrategyTaxonomy taxonomy, double gameTimeMinutes) {
        Map<String, Double> weights = new LinkedHashMap<>();
        for (String label : archetypes) {
            var archetype = io.quarkmind.domain.StrategyArchetype.valueOf(label);
            var entry = taxonomy.lookup(archetype);
            double[] window = entry.phaseWindow();
            weights.put(label, weight(gameTimeMinutes, window[0], window[1], 1.0));
        }
        return weights;
    }

    private static double sigmoid(double x) {
        return 1.0 / (1.0 + Math.exp(-x));
    }
}
```

- [ ] **Step 5: Run all tests — verify PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=PhaseWindowDecayTest -q`

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/PhaseWindowDecay.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/PhaseWindowDecayTest.java
git commit -m "feat(#471): phase-window decay function for temporal archetype weighting

Refs #471"
```

### Task 2: Transition CPT loader and tests

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisPriorLoader.java`
- Create: `quarkmind-sc2/src/main/resources/transitions/transition-cpts-vs_protoss.json` (stub for tests)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/StrategyTransitionCptTest.java`

**Interfaces:**
- Consumes: Classpath JSON files in `transitions/` directory
- Produces: `HypothesisPriorLoader.loadTransitionCpts(Race enemyRace)` → `Map<String, Map<String, Double>>`; `HypothesisPriorLoader.loadUnifiedPriors(Race enemyRace)` → `Map<String, Double>`

- [ ] **Step 1: Write failing test for transition CPT loading**

```java
@Test
void loadTransitionCptsReturnsValidCpt() {
    Map<String, Map<String, Double>> cpts = HypothesisPriorLoader.loadTransitionCpts(Race.PROTOSS);
    assertFalse(cpts.isEmpty(), "transition CPTs should not be empty");
    for (var row : cpts.values()) {
        double sum = row.values().stream().mapToDouble(Double::doubleValue).sum();
        assertEquals(1.0, sum, 0.01, "CPT row should sum to 1.0");
    }
}

@Test
void loadUnifiedPriorsContainsAllPhases() {
    Map<String, Double> priors = HypothesisPriorLoader.loadUnifiedPriors(Race.PROTOSS);
    boolean hasEarly = priors.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.EARLY);
    boolean hasMid = priors.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.MID);
    boolean hasLate = priors.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.LATE);
    assertTrue(hasEarly && hasMid && hasLate, "unified priors must span all phases");
}
```

- [ ] **Step 2: Run test — verify FAIL (method not found)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTransitionCptTest -q`

- [ ] **Step 3: Create stub transition CPT JSON for Protoss**

Create `quarkmind-sc2/src/main/resources/transitions/transition-cpts-vs_protoss.json` with a minimal valid CPT (3 archetypes, rows summing to 1.0). This is a seed file — the real data comes from `StrategyTransitionExportTest` in Batch 3.

```json
{
  "PROTOSS_GATEWAY_RUSH": {"PROTOSS_GATEWAY_RUSH": 0.6, "PROTOSS_COLOSSUS_PUSH": 0.3, "PROTOSS_MACRO": 0.1},
  "PROTOSS_MACRO": {"PROTOSS_MACRO": 0.4, "PROTOSS_STALKER_COLOSSUS": 0.3, "PROTOSS_AIR_SUPERIORITY": 0.3},
  "PROTOSS_STALKER_COLOSSUS": {"PROTOSS_STALKER_COLOSSUS": 0.7, "PROTOSS_CARRIER": 0.2, "PROTOSS_TEMPEST_SIEGE": 0.1}
}
```

Create corresponding stubs for Terran and Zerg.

- [ ] **Step 4: Implement loadTransitionCpts and loadUnifiedPriors on HypothesisPriorLoader**

Add to `HypothesisPriorLoader`:

```java
public static Map<String, Map<String, Double>> loadTransitionCpts(Race enemyRace) {
    String path = "transitions/transition-cpts-vs_" + enemyRace.name().toLowerCase() + ".json";
    try (InputStream in = Thread.currentThread().getContextClassLoader().getResourceAsStream(path)) {
        if (in == null) return Map.of();
        return MAPPER.readValue(in, new com.fasterxml.jackson.core.type.TypeReference<>() {});
    } catch (IOException e) {
        return Map.of();
    }
}

public static Map<String, Double> loadUnifiedPriors(Race enemyRace) {
    Map<String, Double> unified = new LinkedHashMap<>();
    for (StrategyArchetype arch : StrategyArchetype.values()) {
        if (arch.race() == enemyRace) {
            unified.put(arch.name(), 1.0);
        }
    }
    double sum = unified.size();
    unified.replaceAll((k, v) -> v / sum);
    return unified;
}
```

- [ ] **Step 5: Run all tests — verify PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=StrategyTransitionCptTest -q`

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisPriorLoader.java quarkmind-sc2/src/main/resources/transitions/ quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/StrategyTransitionCptTest.java
git commit -m "feat(#471): transition CPT loader and unified priors for all-phase archetype set

Refs #471"
```

---

## Batch 2: DetectorOrchestrator temporal upgrade

### Task 3: Replace TemporalAssessmentTracker with inline decay blending

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/DetectorOrchestrator.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/DetectorOrchestratorTest.java`
- Delete: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/TemporalAssessmentTracker.java` (use `ide_refactor_safe_delete`)
- Delete: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/TemporalAssessmentTrackerTest.java`

**Interfaces:**
- Consumes: `EvidenceFusionEngine` (unchanged), `PhaseWindowDecay` (from Task 1)
- Produces: Constructor signature changes: `int windowSize` → `double temporalDecay`. All public methods unchanged. `detectTransition()` uses decay-based comparison instead of windowed average.

- [ ] **Step 1: Write failing test for decay-based smoothing**

Add to `DetectorOrchestratorTest`:

```java
@Test
void temporalDecaySmoothsJitter() {
    var orchestrator = new DetectorOrchestrator(registry, priors, cpts, labelToArchetype, 0.3);
    orchestrator.initializeNetwork(detectorNames);

    // First classify — no previous, should return raw posterior
    var result1 = orchestrator.classify(Map.of("det1", 0.9f), List.of(), 100);
    double conf1 = result1.get(0).confidence();

    // Second classify with lower signal — decay should smooth
    var result2 = orchestrator.classify(Map.of("det1", 0.3f), List.of(), 200);
    double conf2 = result2.get(0).confidence();

    // With decay=0.3, result should be 0.7*current + 0.3*previous
    // The smoothed value should be between the two raw posteriors
    assertTrue(conf2 > 0.0, "smoothed confidence should be positive");
}
```

- [ ] **Step 2: Run test — verify FAIL (constructor signature mismatch)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DetectorOrchestratorTest#temporalDecaySmoothsJitter -q`

- [ ] **Step 3: Modify DetectorOrchestrator — replace windowSize with temporalDecay**

Changes to `DetectorOrchestrator.java`:
1. Replace `TemporalAssessmentTracker tracker` field with `double temporalDecay` and `double[] previousPosterior` and `int previousDominant = -1`
2. Update all constructors: `int windowSize` → `double temporalDecay`
3. In `classify()`: after `engine.marginalize()`, apply inline blending:
   ```java
   double[] current = engine.marginalize();
   if (previousPosterior != null && temporalDecay > 0.0) {
       for (int i = 0; i < current.length; i++) {
           current[i] = (1.0 - temporalDecay) * current[i] + temporalDecay * previousPosterior[i];
       }
   }
   previousPosterior = current.clone();
   ```
4. In `detectTransition()`: compare dominant index in `previousPosterior` vs current
5. In `reset()`: `previousPosterior = null; previousDominant = -1;`

- [ ] **Step 4: Update existing DetectorOrchestratorTest — fix constructor calls**

All existing tests that construct `DetectorOrchestrator` with `windowSize` (int) need to change to `temporalDecay` (double). Replace `5` (window) with `0.3` (decay).

- [ ] **Step 5: Run all DetectorOrchestrator tests — verify PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DetectorOrchestratorTest -q`

- [ ] **Step 6: Check for other callers of DetectorOrchestrator constructor**

Use `ide_find_references` on the constructor to find all callers. Update each one to use `double temporalDecay` instead of `int windowSize`.

- [ ] **Step 7: Delete TemporalAssessmentTracker and its test**

Use `ide_refactor_safe_delete` for `TemporalAssessmentTracker.java`. Then delete `TemporalAssessmentTrackerTest.java`.

- [ ] **Step 8: Run full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass. No references to `TemporalAssessmentTracker` remain.

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/DetectorOrchestrator.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/DetectorOrchestratorTest.java
git commit -m "feat(#471): replace TemporalAssessmentTracker with inline decay blending in DetectorOrchestrator

Supersedes ADR-0010's windowing choice — decay blending is more appropriate
for strategy evolution where recent evidence should dominate.

Refs #471"
```

---

## Batch 3: HypothesisTrackerFusion unified archetype set

### Task 4: Unified initialisation — all archetypes, no phase rebuild

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/UnifiedHypothesisTrackerFusionTest.java`

**Interfaces:**
- Consumes: `HypothesisPriorLoader.loadUnifiedPriors(Race)` (from Task 2), `HypothesisPriorLoader.loadTransitionCpts(Race)` (from Task 2), `PhaseWindowDecay` (from Task 1), `StrategyTaxonomy`
- Produces: `HypothesisTrackerFusion.initializeUnified(List<String> detectorNames, Race enemyRace, StrategyTaxonomy taxonomy)` — one-time init with all archetypes; `classify()` now takes `double gameTimeMinutes` parameter for phase-window weighting

- [ ] **Step 1: Write failing test for unified initialisation**

```java
@Test
void initializeUnifiedIncludesAllPhaseArchetypes() {
    var fusion = new HypothesisTrackerFusion(registry, learnedPriors, detectorCpts,
        observationCpts, learnedNetworks, phaseResolver);
    fusion.initializeUnified(detectorNames, Race.PROTOSS, taxonomy);

    // Should have archetypes from EARLY, MID, and LATE
    var weights = fusion.opponentWeights();
    boolean hasEarly = weights.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.EARLY);
    boolean hasMid = weights.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.MID);
    boolean hasLate = weights.keySet().stream()
        .anyMatch(k -> StrategyArchetype.valueOf(k).phase() == GamePhase.LATE);
    assertTrue(hasEarly && hasMid && hasLate);
}
```

- [ ] **Step 2: Run test — verify FAIL (method not found)**

Run: `mvn test -pl quarkmind-sc2 -Dtest=UnifiedHypothesisTrackerFusionTest#initializeUnifiedIncludesAllPhaseArchetypes -q`

- [ ] **Step 3: Write tests for no-rebuild behaviour and phase-window weighting**

```java
@Test
void classifyDoesNotRebuildAcrossPhases() {
    var fusion = new HypothesisTrackerFusion(registry, learnedPriors, detectorCpts,
        observationCpts, learnedNetworks, phaseResolver);
    fusion.initializeUnified(detectorNames, Race.PROTOSS, taxonomy);

    // Classify at EARLY time, then MID time — should NOT rebuild
    fusion.classify(detectorOutputs, List.of(), 100, 2.0);
    var weightsAtEarly = new LinkedHashMap<>(fusion.opponentWeights());

    fusion.classify(detectorOutputs, List.of(), 200, 6.0);
    var weightsAtMid = fusion.opponentWeights();

    // Same keys — no archetypes vanished or appeared
    assertEquals(weightsAtEarly.keySet(), weightsAtMid.keySet());
}

@Test
void phaseWindowReducesEarlyArchetypeAtMidTime() {
    var fusion = new HypothesisTrackerFusion(registry, learnedPriors, detectorCpts,
        observationCpts, learnedNetworks, phaseResolver);
    fusion.initializeUnified(detectorNames, Race.PROTOSS, taxonomy);

    fusion.classify(detectorOutputs, List.of(), 100, 8.0);
    var weights = fusion.opponentWeights();

    // EARLY archetypes should have low weight at minute 8
    for (var entry : weights.entrySet()) {
        var arch = StrategyArchetype.valueOf(entry.getKey());
        if (arch.phase() == GamePhase.EARLY) {
            assertTrue(entry.getValue() < 0.05,
                arch + " should have low weight at minute 8, got " + entry.getValue());
        }
    }
}
```

- [ ] **Step 4: Implement initializeUnified on HypothesisTrackerFusion**

Add new method `initializeUnified(List<String> detectorNames, Race enemyRace, StrategyTaxonomy taxonomy)`:
1. Load unified priors via `HypothesisPriorLoader.loadUnifiedPriors(enemyRace)`
2. Load transition CPTs via `HypothesisPriorLoader.loadTransitionCpts(enemyRace)`
3. Build tracker with ALL archetypes for the race (not phase-filtered) via new `SC2HypothesisNetworkBuilder.buildUnifiedOpponentTracker()` method
4. Store `taxonomy` and `transitionMatrix` as fields

Add `gameTimeMinutes` parameter to `classify()` (or overload). Before evidence fusion, apply phase weights via `PhaseWindowDecay.computePhaseWeights()`.

- [ ] **Step 5: Add buildUnifiedOpponentTracker to SC2HypothesisNetworkBuilder**

New static method that builds the network with all archetypes for a race, not phase-filtered:

```java
public static HypothesisTracker buildUnifiedOpponentTracker(
        Race enemyRace,
        Map<String, Double> unifiedPriors,
        List<String> activeDetectorNames,
        Map<String, Map<String, Double>> detectorCpts,
        ObservationCpts observationCpts) {
    String[] archetypes = filteredUnifiedArchetypes(enemyRace, unifiedPriors);
    JunctionTree intelTree = buildIntelTree(archetypes, unifiedPriors,
        activeDetectorNames, detectorCpts, observationCpts);
    NetworkComposition composition = CompositionBuilder.create("sc2-unified")
        .slot("intel", intelTree).build();
    return buildHypothesisTracker(composition, archetypes, unifiedPriors);
}
```

- [ ] **Step 6: Run all new tests — verify PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=UnifiedHypothesisTrackerFusionTest -q`

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/SC2HypothesisNetworkBuilder.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/UnifiedHypothesisTrackerFusionTest.java
git commit -m "feat(#471): unified archetype set — single tracker for all phases, no rebuild

Eliminates phase-transition reinitialisation. Phase-window decay concentrates
probability mass on in-phase archetypes.

Refs #471"
```

### Task 5: Transition-aware temporal blending in classify()

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/UnifiedHypothesisTrackerFusionTest.java`

**Interfaces:**
- Consumes: `transitionMatrix` (from Task 2 loader), `previousPosterior` (stored per tick)
- Produces: `classify()` returns temporally-smoothed assessments; `applyTransitionPrior(Map<String, Double> prev, Map<String, Map<String, Double>> transitions, double decay)` → `Map<String, Double>`

- [ ] **Step 1: Write failing test for transition prior boosting**

```java
@Test
void transitionPriorBoostsExpectedSuccessor() {
    var fusion = new HypothesisTrackerFusion(registry, learnedPriors, detectorCpts,
        observationCpts, learnedNetworks, phaseResolver);
    fusion.initializeUnified(detectorNames, Race.PROTOSS, taxonomy);

    // First classify at EARLY time — rush detector fires
    Map<String, Float> rushDetectors = Map.of("rush_det", 0.95f);
    fusion.classify(rushDetectors, List.of(), 100, 2.0);
    // Leader should be a RUSH archetype

    // Second classify at MID time — ambiguous evidence
    Map<String, Float> ambiguousDetectors = Map.of("rush_det", 0.4f);
    var result = fusion.classify(ambiguousDetectors, List.of(), 200, 5.0);

    // Transition prior should boost COLOSSUS_PUSH (taxonomy: GATEWAY_RUSH → COLOSSUS_PUSH)
    // It should appear with higher confidence than without transition blending
    boolean hasTransitionTarget = result.stream()
        .anyMatch(a -> a.archetype() == StrategyArchetype.PROTOSS_COLOSSUS_PUSH
                    && a.confidence() > 0.05);
    assertTrue(hasTransitionTarget, "transition prior should boost expected successor");
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -pl quarkmind-sc2 -Dtest=UnifiedHypothesisTrackerFusionTest#transitionPriorBoostsExpectedSuccessor -q`

- [ ] **Step 3: Implement transition-aware temporal blending**

In `HypothesisTrackerFusion.classify()`, after getting raw posteriors from tracker:

```java
Map<String, Double> rawWeights = opponentTracker.weights();
Map<String, Double> smoothed;
if (previousPosterior != null && transitionMatrix != null && !transitionMatrix.isEmpty()) {
    smoothed = applyTransitionPrior(rawWeights, previousPosterior, transitionMatrix, temporalDecay);
} else {
    smoothed = new LinkedHashMap<>(rawWeights);
}
previousPosterior = smoothed;
```

Implement `applyTransitionPrior`:

```java
private static Map<String, Double> applyTransitionPrior(
        Map<String, Double> current,
        Map<String, Double> previous,
        Map<String, Map<String, Double>> transitions,
        double decay) {
    Map<String, Double> blended = new LinkedHashMap<>();
    for (var entry : current.entrySet()) {
        String archJ = entry.getKey();
        double rawJ = entry.getValue();
        double transitionSum = 0.0;
        for (var prevEntry : previous.entrySet()) {
            String archI = prevEntry.getKey();
            double prevI = prevEntry.getValue();
            Map<String, Double> row = transitions.getOrDefault(archI, Map.of());
            double pJgivenI = row.getOrDefault(archJ, 1.0 / current.size());
            transitionSum += prevI * pJgivenI;
        }
        blended.put(archJ, (1.0 - decay) * rawJ + decay * transitionSum);
    }
    normalize(blended);
    return blended;
}
```

- [ ] **Step 4: Write test for decay=0.0 behaves like non-temporal**

```java
@Test
void zeroDecayBehavesLikeNonTemporal() {
    var fusion = new HypothesisTrackerFusion(registry, learnedPriors, detectorCpts,
        observationCpts, learnedNetworks, phaseResolver);
    fusion.initializeUnified(detectorNames, Race.PROTOSS, taxonomy);
    fusion.setTemporalDecay(0.0);

    fusion.classify(detectorOutputs, List.of(), 100, 3.0);
    var result1 = fusion.opponentWeights();

    fusion.classify(detectorOutputs, List.of(), 200, 3.0);
    var result2 = fusion.opponentWeights();

    // With zero decay, consecutive identical evidence should give identical weights
    for (var entry : result1.entrySet()) {
        assertEquals(entry.getValue(), result2.get(entry.getKey()), 0.01,
            "zero decay should give identical weights for identical evidence");
    }
}
```

- [ ] **Step 5: Run all tests — verify PASS**

Run: `mvn test -pl quarkmind-sc2 -Dtest=UnifiedHypothesisTrackerFusionTest -q`

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/HypothesisTrackerFusion.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/UnifiedHypothesisTrackerFusionTest.java
git commit -m "feat(#471): transition-aware temporal blending in HypothesisTrackerFusion

P(strategy_t | strategy_{t-1}) learned from replay data biases posteriors
toward expected successor archetypes.

Refs #471"
```

---

## Batch 4: Transition learning pipeline and integration

### Task 6: StrategyTransitionExportTest — learn transition CPTs from replays

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrategyTransitionExportTest.java`

**Interfaces:**
- Consumes: Reconstituted replay JSON files in `quarkmind-classifier/data/reconstituted/`, `StrategyTaxonomy`, `DetectorOrchestrator` for classification at checkpoints
- Produces: Writes `transitions/transition-cpts-vs_<race>.json` to `quarkmind-sc2/src/main/resources/transitions/`

- [ ] **Step 1: Create test skeleton with @Tag("report")**

```java
@Tag("report")
class StrategyTransitionExportTest {
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Path RECONSTITUTED_DIR = Path.of("../quarkmind-classifier/data/reconstituted");
    private static final Path OUTPUT_DIR = Path.of("quarkmind-sc2/src/main/resources/transitions");
    private static final int[] CHECKPOINTS = {1344, 4032, 6720, 9408, 13440}; // 1,3,5,7,10 min
    private static final double ALPHA = 10.0;
    private static final int TAXONOMY_BOOST = 50;
    private static final double DISCOVERY_THRESHOLD = 0.05;

    @Test
    void exportStrategyTransitionCpts() {
        // Implementation follows ContextResponsePriorExportTest pattern
    }
}
```

- [ ] **Step 2: Implement the export pipeline**

Following `ContextResponsePriorExportTest` pattern:
1. Scan `RECONSTITUTED_DIR` for replay JSON files by matchup subdirectory
2. For each replay, extract opponent strategy classification at each checkpoint
3. Record transitions between consecutive checkpoints
4. Aggregate counts per matchup: `Map<Race, Map<String, Map<String, Integer>>>`
5. Apply hierarchical smoothing (α=10.0, taxonomy boost=50)
6. Apply hybrid filtering (taxonomy transitions always included, discovered >5%)
7. Normalize rows to sum to 1.0
8. Write JSON files

- [ ] **Step 3: Add validation assertions**

```java
// Per-matchup validations
for (var raceEntry : transitionCpts.entrySet()) {
    var cpts = raceEntry.getValue();
    for (var row : cpts.entrySet()) {
        double sum = row.getValue().values().stream().mapToDouble(Double::doubleValue).sum();
        assertEquals(1.0, sum, 0.01, "row " + row.getKey() + " should sum to 1.0");
    }
    // Taxonomy transitions present
    for (var transition : taxonomyTransitions) {
        if (transition.from().race() == raceEntry.getKey()) {
            assertTrue(cpts.containsKey(transition.from().name()),
                "taxonomy transition source should be present");
            assertTrue(cpts.get(transition.from().name()).getOrDefault(transition.to().name(), 0.0) > 0.01,
                "taxonomy transition should have probability > 0.01");
        }
    }
    // No deterministic successors
    for (var row : cpts.values()) {
        for (double p : row.values()) {
            assertTrue(p < 0.95, "no transition should be > 0.95");
        }
    }
}
```

- [ ] **Step 4: Run with report profile (requires replay data)**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StrategyTransitionExportTest -q`
Expected: Writes transition CPT JSON files. This replaces the stub files from Task 2.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/StrategyTransitionExportTest.java quarkmind-sc2/src/main/resources/transitions/
git commit -m "feat(#471): strategy transition export pipeline — learn P(strategy_t|strategy_{t-1}) from 149K replays

5 checkpoints (1,3,5,7,10 min), hierarchical smoothing (α=10.0),
taxonomy transitions boosted, discovery threshold 5%.

Refs #471"
```

### Task 7: Integration, docs, and regression verification

**Files:**
- Modify: `docs/adr/0010-bayesian-evidence-fusion-replaces-multi-class-onnx.md` — add supersession note
- Modify: `CLAUDE.md` — register new tests
- Modify: `docs/guides/contributor-guide.md` — update detector pipeline docs

**Interfaces:**
- Consumes: All previous tasks
- Produces: Updated documentation, verified regression

- [ ] **Step 1: Update ADR-0010 with supersession note**

Add to the end of ADR-0010:

```markdown
## Superseded (partial)

The `TemporalAssessmentTracker` windowing approach has been superseded by
decay-based temporal blending (#471). Windowing was correct at the time
but transition-aware priors and phase-window decay make decay blending
more appropriate for strategy evolution.
```

- [ ] **Step 2: Register new tests in CLAUDE.md**

Add to the unit tests list:
- `PhaseWindowDecayTest`
- `StrategyTransitionCptTest`
- `UnifiedHypothesisTrackerFusionTest`

Add to the report tests section:
- `StrategyTransitionExportTest` — learns P(strategy_t|strategy_{t-1}) transition CPTs from 149K replays at 1/3/5/7/10 min checkpoints; writes transition-cpts-vs_*.json

- [ ] **Step 3: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass including existing tests that exercise classification.

- [ ] **Step 4: Run benchmark to verify latency budget**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=GameLoopBenchmarkTest -q`
Expected: Per-tick timings within previous baseline. Document results.

- [ ] **Step 5: Commit**

```bash
git add docs/adr/0010-bayesian-evidence-fusion-replaces-multi-class-onnx.md CLAUDE.md docs/guides/contributor-guide.md
git commit -m "docs(#471): update ADR-0010 supersession, register temporal strategy tests

Refs #471"
```

## References

- [2026-10-29-temporal-strategy-evolution-design.md] — design spec this plan implements
- `DetectorOrchestrator.java:37-78` — constructor signatures to change
- `DetectorOrchestrator.java:179-217` — classify() flow to modify
- `HypothesisTrackerFusion.java:24-62` — fields and constructor
- `HypothesisTrackerFusion.java:79-138` — initialize() to replace with initializeUnified()
- `HypothesisPriorLoader.java:33-131` — existing loader API to extend
- `SC2HypothesisNetworkBuilder.java:23-39` — buildOpponentTracker to model buildUnifiedOpponentTracker on
- `ContextResponsePriorExportTest.java` — pattern for transition export pipeline
- `TemporalAssessmentTracker.java` — class to delete
- `strategy-taxonomy.yaml` — transition paths and phase windows
- GitHub #471
