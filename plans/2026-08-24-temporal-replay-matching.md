# Temporal Game Replay Matching — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #222 — temporal game replay matching
**Issue group:** #222

**Goal:** Wire existing QuarkMind temporal data to neocortex's sequence similarity infrastructure — store game timelines, query with DTW, predict "what happened next" from past games.

**Architecture:** New `TimelineSampler` samples GameState every 30s during play. At game close, `SC2CbrRetentionObserver` stores the timeline as a `TimeSeries` feature. A new `TemporalCbrTask` runs periodic live queries with DTW, extracts predictions from alignment paths, and writes them to CaseFile.

**Tech Stack:** Quarkus CDI, neocortex `memory-api` (DTW, EditDistance, TimeSeries, DiscreteSequence), existing SC2 plugin architecture

## Global Constraints

- Domain records (`TimelineObservation`, `TemporalPrediction`) go in `io.quarkmind.domain` — no CDI, no framework deps
- CDI beans go in `io.quarkmind.agent.cbr` — alongside existing CBR components
- Unit tests use plain JUnit — no `@QuarkusTest` unless CDI context is needed
- All CaseFile key constants go in `QuarkMindCaseFile` — no raw strings
- `SC2GameCbrCase.CBR_TYPE` = `"sc2-strategy"` for all schema/retention operations
- DTW warping constraint: `SakoeChibaBand(3)` — ±90s temporal shift
- Timeline sample interval: 672 game loops (~30 game-seconds)
- Own-side fields only (minerals via `gs.minerals()`, supply via `gs.supplyUsed()`, workers via `gs.myUnits()` filter) — `PlayerEconomyStats` is `EMPTY` during live games. Issue #222 mentions enemy economy curves, but live-game fog-of-war makes them unreliable (D2 decision). A follow-up issue should add enemy-side temporal matching for replay-only retrieval mode.
- `TemporalCbrTask` ID is `"temporal-cbr.predict"` — not in `QuarkMindCaseHub.PHASE_ORDER`, so it runs after all known phases (scouting → strategy-routing → strategy → tactics → economics → summarisation → temporal-cbr). Predictions arrive for the *next* tick's consumers.
- No consumer reads temporal predictions in this iteration. A follow-up issue will wire at least one consumer (strategy adaptation or coaching). This is acceptable for Tier 3 validation.
- Follow existing import patterns from `SC2StrategyRouterTask.java` — packages are `io.casehub.neocortex.memory.cbr.*` (not `.api.cbr`), `io.casehub.annotation.CaseType`, `io.quarkmind.agency.task.TaskDefinition`
- `CbrQuery` uses `CbrQuery.of()` factory + `.withWeights()` / `.withMinSimilarity()` chaining — no builder pattern
- `SC2CbrRetentionObserver` uses constructor injection exclusively — add new dependencies as constructor parameters, not `@Inject` fields

---

## Batch 1: Sampling Foundation

After this batch: game state is sampled every 30s during play, with unit tests passing.

### Task 1: TimelineObservation domain record

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/TimelineObservation.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/TimelineObservationTest.java`

**Interfaces:**
- Consumes: `GameState` record (`minerals()`, `supplyUsed()`, `myUnits()`, `gameFrame()`)
- Produces: `TimelineObservation(double minute, int ourWorkers, int ourMinerals, int ourArmySupply)` with `static TimelineObservation from(GameState gs)`

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TimelineObservationTest {

    @Test
    void fromGameState_extractsCorrectValues() {
        // Build a GameState with known values:
        // minerals=350, supplyUsed=44, 16 workers in myUnits, gameFrame=15120 (≈11.25 min)
        var workers = buildWorkers(16);
        var army = buildArmyUnits(5); // 5 non-worker units
        var gs = buildGameState(350, 44, workers, army, 15120);

        var obs = TimelineObservation.from(gs);

        assertEquals(15120.0 / SC2Data.GAME_LOOPS_PER_SECOND / 60.0, obs.minute(), 0.001);
        assertEquals(16, obs.ourWorkers());
        assertEquals(350, obs.ourMinerals());
        assertEquals(44 - 16, obs.ourArmySupply()); // supplyUsed - workers
    }

    @Test
    void fromGameState_zeroWorkers() {
        var gs = buildGameState(50, 0, List.of(), List.of(), 0);
        var obs = TimelineObservation.from(gs);

        assertEquals(0, obs.ourWorkers());
        assertEquals(50, obs.ourMinerals());
        assertEquals(0, obs.ourArmySupply());
    }

    @Test
    void fromGameState_noArmy_onlyWorkers() {
        var workers = buildWorkers(12);
        var gs = buildGameState(200, 12, workers, List.of(), 672);
        var obs = TimelineObservation.from(gs);

        assertEquals(12, obs.ourWorkers());
        assertEquals(0, obs.ourArmySupply()); // all supply is workers
    }
}
```

Use `SimulatedGame` or build `GameState` directly with known values. Check how existing tests like `GameStateTest` construct `GameState` instances for the helper method patterns.

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TimelineObservationTest -q`
Expected: compilation error — `TimelineObservation` does not exist

- [ ] **Step 3: Implement TimelineObservation**

```java
package io.quarkmind.domain;

public record TimelineObservation(
        double minute,
        int ourWorkers,
        int ourMinerals,
        int ourArmySupply
) {
    public static TimelineObservation from(GameState gs) {
        int workers = (int) gs.myUnits().stream()
                .filter(u -> u.type().isWorker())
                .count();
        return new TimelineObservation(
                gs.gameFrame() / SC2Data.GAME_LOOPS_PER_SECOND / 60.0,
                workers,
                gs.minerals(),
                gs.supplyUsed() - workers
        );
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TimelineObservationTest -q`
Expected: all 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/domain/TimelineObservation.java quarkmind-sc2/src/test/java/io/quarkmind/domain/TimelineObservationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): TimelineObservation domain record — 30s GameState sampling Refs #222"
```

### Task 2: TimelineSampler CDI bean + GameTickExecutor integration

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/TimelineSampler.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java:59` (add `timelineSampler.tick(gameState)` after `engine.observe()`)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TimelineSamplerTest.java`

**Interfaces:**
- Consumes: `TimelineObservation.from(GameState)`, `GameStarted` CDI event
- Produces: `void tick(GameState gs)`, `List<TimelineObservation> getTimeline()` (immutable copy)

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.agent.cbr;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.TimelineObservation;
import io.quarkmind.sc2.GameStarted;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TimelineSamplerTest {

    private TimelineSampler sampler;

    @BeforeEach
    void setUp() {
        sampler = new TimelineSampler();
    }

    @Test
    void samplesAtCorrectInterval() {
        // First tick at frame 0 should sample
        sampler.tick(gameStateAtFrame(0));
        assertEquals(1, sampler.getTimeline().size());

        // Tick at frame 671 — not yet 672 interval
        sampler.tick(gameStateAtFrame(671));
        assertEquals(1, sampler.getTimeline().size());

        // Tick at frame 672 — exactly at interval
        sampler.tick(gameStateAtFrame(672));
        assertEquals(2, sampler.getTimeline().size());
    }

    @Test
    void firstTickAlwaysSamples() {
        sampler.tick(gameStateAtFrame(100));
        assertEquals(1, sampler.getTimeline().size());
    }

    @Test
    void getTimelineReturnsImmutableCopy() {
        sampler.tick(gameStateAtFrame(0));
        List<TimelineObservation> timeline = sampler.getTimeline();
        assertThrows(UnsupportedOperationException.class, () -> timeline.clear());
    }

    @Test
    void clearsOnGameStarted() {
        sampler.tick(gameStateAtFrame(0));
        sampler.tick(gameStateAtFrame(672));
        assertEquals(2, sampler.getTimeline().size());

        sampler.onGameStarted(new GameStarted("Zerg", "Computer", "Medium", "ai-1"));
        assertEquals(0, sampler.getTimeline().size());

        // Next tick should sample again
        sampler.tick(gameStateAtFrame(1000));
        assertEquals(1, sampler.getTimeline().size());
    }

    @Test
    void multipleWindowsAccumulate() {
        for (int i = 0; i < 5; i++) {
            sampler.tick(gameStateAtFrame(i * 672L));
        }
        assertEquals(5, sampler.getTimeline().size());
    }
}
```

Build `gameStateAtFrame(long frame)` helper using the same pattern as `TimelineObservationTest`. Check how `GameStateTest` or `GameTickExecutorMigrationTest` construct `GameState` — reuse or adapt their builders.

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TimelineSamplerTest -q`
Expected: compilation error — `TimelineSampler` does not exist

- [ ] **Step 3: Implement TimelineSampler**

```java
package io.quarkmind.agent.cbr;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.TimelineObservation;
import io.quarkmind.sc2.GameStarted;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@ApplicationScoped
public class TimelineSampler {

    static final long SAMPLE_INTERVAL = 672; // ~30 game-seconds at 22.4 loops/s

    private final List<TimelineObservation> timeline = new CopyOnWriteArrayList<>();
    private long lastSampleFrame = -SAMPLE_INTERVAL; // ensures first tick always samples

    public void tick(GameState gs) {
        if (gs.gameFrame() - lastSampleFrame >= SAMPLE_INTERVAL) {
            timeline.add(TimelineObservation.from(gs));
            lastSampleFrame = gs.gameFrame();
        }
    }

    public List<TimelineObservation> getTimeline() {
        return List.copyOf(timeline);
    }

    void onGameStarted(@Observes GameStarted event) {
        timeline.clear();
        lastSampleFrame = -SAMPLE_INTERVAL;
    }
}
```

- [ ] **Step 4: Run TimelineSampler tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TimelineSamplerTest -q`
Expected: all 5 tests PASS

- [ ] **Step 5: Wire into GameTickExecutor**

Modify `GameTickExecutor.java`. At line 59, after `var gameState = engine.observe();`, add `timelineSampler.tick(gameState);`. Inject `TimelineSampler` as a field.

At the top of the class, add the injection:
```java
@Inject TimelineSampler timelineSampler;
```

After line 59 (`var gameState = engine.observe();`), insert:
```java
timelineSampler.tick(gameState);
```

- [ ] **Step 6: Run existing GameTickExecutor tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=GameTickExecutorMigrationTest -q`
Expected: existing tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/TimelineSampler.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TimelineSamplerTest.java quarkmind-sc2/src/main/java/io/quarkmind/agent/GameTickExecutor.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): TimelineSampler — 30s periodic GameState sampling + GameTickExecutor wiring Refs #222"
```

---

## Batch 2: Storage Pipeline

After this batch: game timelines are stored as TimeSeries features in CBR cases at game close.

### Task 3: CaseFile keys + schema extensions + phase_sequence upgrade

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java:57` (add 3 temporal key constants)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java:22-50` (add timeline TimeSeries field, upgrade phase_sequence to DiscreteSequence)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrarTest.java` (new or extend existing)

**Interfaces:**
- Consumes: neocortex `FeatureField.timeSeries()`, `FeatureField.discreteSequence()`, `SimilaritySpec.DtwSpec`, `SimilaritySpec.EditDistanceSpec`, `WarpingConstraint.SakoeChibaBand`
- Produces: `QuarkMindCaseFile.TEMPORAL_PREDICTION`, `TEMPORAL_SIMILAR_COUNT`, `TEMPORAL_SIMILAR_BEST_SCORE` constants; updated `sc2-strategy` schema with `timeline` TimeSeries field and `phase_sequence` DiscreteSequence field

- [ ] **Step 1: Add CaseFile key constants**

Add to `QuarkMindCaseFile.java` after the last constant (around line 57, after `SCOUTING_FINAL_ASSESSMENT`):

```java
// Temporal CBR (#222)
public static final String TEMPORAL_PREDICTION = "agent.temporal.prediction";
public static final String TEMPORAL_SIMILAR_COUNT = "agent.temporal.similar.count";
public static final String TEMPORAL_SIMILAR_BEST_SCORE = "agent.temporal.similar.best.score";
```

- [ ] **Step 2: Write failing schema test**

```java
package io.quarkmind.agent.cbr;

import io.casehub.neocortex.memory.api.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.api.cbr.FeatureField;
import io.casehub.neocortex.memory.api.cbr.SimilaritySpec;
import io.casehub.neocortex.memory.api.cbr.WarpingConstraint;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SC2CbrSchemaRegistrarTest {

    @Test
    void strategySchema_hasTimelineTimeSeries() {
        var schema = SC2CbrSchemaRegistrar.buildStrategySchema();
        var timelineField = schema.fields().stream()
                .filter(f -> f.name().equals("timeline"))
                .findFirst()
                .orElseThrow(() -> new AssertionError("timeline field not found"));

        assertInstanceOf(FeatureField.TimeSeries.class, timelineField);
        var ts = (FeatureField.TimeSeries) timelineField;
        assertEquals("minute", ts.timestampField());
        assertInstanceOf(SimilaritySpec.DtwSpec.class, ts.similaritySpec());
        var dtw = (SimilaritySpec.DtwSpec) ts.similaritySpec();
        assertInstanceOf(WarpingConstraint.SakoeChibaBand.class, dtw.constraint());
        assertEquals(3, ((WarpingConstraint.SakoeChibaBand) dtw.constraint()).windowSize());
        // 4 inner fields: minute, our_workers, our_minerals, our_army_supply
        assertEquals(4, ts.innerFields().size());
    }

    @Test
    void strategySchema_phaseSequenceIsDiscreteSequence() {
        var schema = SC2CbrSchemaRegistrar.buildStrategySchema();
        var phaseField = schema.fields().stream()
                .filter(f -> f.name().equals("phase_sequence"))
                .findFirst()
                .orElseThrow(() -> new AssertionError("phase_sequence field not found"));

        assertInstanceOf(FeatureField.DiscreteSequence.class, phaseField);
        var ds = (FeatureField.DiscreteSequence) phaseField;
        assertInstanceOf(SimilaritySpec.EditDistanceSpec.class, ds.similaritySpec());
        var ed = (SimilaritySpec.EditDistanceSpec) ds.similaritySpec();
        // Verify substitution costs exist for known phases
        assertNotNull(ed.substitutionSimilarities());
        assertTrue(ed.substitutionSimilarities().containsKey("EARLY_MACRO"));
        assertTrue(ed.substitutionSimilarities().containsKey("MID_SKIRMISH"));
    }
}
```

- [ ] **Step 3: Run test — verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2CbrSchemaRegistrarTest -q`
Expected: FAIL — `buildStrategySchema()` method does not exist

- [ ] **Step 4: Refactor SC2CbrSchemaRegistrar — extract buildStrategySchema()**

The current `register()` method both builds and registers schemas. Extract the schema construction into a package-private `static CbrFeatureSchema buildStrategySchema()` method so tests can inspect the schema without CDI.

Then modify the schema:

1. Change `phase_sequence` from `FeatureField.categoricalList("phase_sequence")` to:
```java
FeatureField.discreteSequence("phase_sequence",
        new SimilaritySpec.EditDistanceSpec(phaseSubstitutionCosts(), 1.0, 1.0))
```

2. Add `timeline` TimeSeries field after the existing fields:
```java
FeatureField.timeSeries("timeline", "minute",
        new SimilaritySpec.DtwSpec(new WarpingConstraint.SakoeChibaBand(3)),
        FeatureField.numeric("minute", 0, 30),
        FeatureField.numeric("our_workers", 0, 80),
        FeatureField.numeric("our_minerals", 0, 5000),
        FeatureField.numeric("our_army_supply", 0, 200))
```

3. Add the `phaseSubstitutionCosts()` static method:
```java
private static Map<String, Map<String, Double>> phaseSubstitutionCosts() {
    var costs = new HashMap<String, Map<String, Double>>();
    costs.put("EARLY_MACRO", Map.of(
            "TRANSITIONING", 0.6, "MID_SKIRMISH", 0.2,
            "EARLY_AGGRESSION", 0.1, "DEFENSIVE_HOLD", 0.2));
    costs.put("TRANSITIONING", Map.of(
            "EARLY_MACRO", 0.6, "MID_SKIRMISH", 0.5,
            "EARLY_AGGRESSION", 0.4, "DEFENSIVE_HOLD", 0.4));
    costs.put("MID_SKIRMISH", Map.of(
            "EARLY_MACRO", 0.2, "TRANSITIONING", 0.5,
            "EARLY_AGGRESSION", 0.6, "DEFENSIVE_HOLD", 0.5));
    costs.put("EARLY_AGGRESSION", Map.of(
            "EARLY_MACRO", 0.1, "TRANSITIONING", 0.4,
            "MID_SKIRMISH", 0.6, "DEFENSIVE_HOLD", 0.4));
    costs.put("DEFENSIVE_HOLD", Map.of(
            "EARLY_MACRO", 0.2, "TRANSITIONING", 0.4,
            "MID_SKIRMISH", 0.5, "EARLY_AGGRESSION", 0.4));
    return Map.copyOf(costs);
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2CbrSchemaRegistrarTest -q`
Expected: both tests PASS

- [ ] **Step 6: Run existing CBR tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2CbrRetentionObserverTest -q`
Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2GameCbrCaseTest -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseFile.java quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrar.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/SC2CbrSchemaRegistrarTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): TimeSeries timeline + DiscreteSequence phase_sequence schema Refs #222"
```

### Task 4: Retention extensions — store timeline at game close

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java:54-57` (add overloaded `buildForGameEnriched` with timeline parameter)
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java:197-199` (inject TimelineSampler, pass timeline to new overload)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java` (extend with timeline verification)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrRetentionIT.java` (new integration test)

**Interfaces:**
- Consumes: `TimelineSampler.getTimeline()`, `TimelineObservation`, `FeatureValue.of()`, `SC2GameCbrCase.buildForGameEnriched()`
- Produces: Stored CBR cases with `timeline` feature as `StructListVal`

- [ ] **Step 1: Write failing test for buildForGameEnriched overload**

Add to `SC2GameCbrCaseTest.java`:

```java
@Test
void buildForGameEnriched_withTimeline_includesTimelineFeature() {
    var timeline = List.of(
            new TimelineObservation(0.5, 12, 50, 0),
            new TimelineObservation(1.0, 14, 100, 4),
            new TimelineObservation(1.5, 16, 150, 8));
    var enrichment = buildDefaultEnrichment();

    var cbrCase = SC2GameCbrCase.buildForGameEnriched(
            "PROTOSS_4GATE", "Protoss", "PvZ",
            0.85, "strategy.early-pressure", enrichment, timeline);

    assertTrue(cbrCase.features().containsKey("timeline"));
    var timelineVal = cbrCase.features().get("timeline");
    assertInstanceOf(FeatureValue.StructListVal.class, timelineVal);
    var observations = ((FeatureValue.StructListVal) timelineVal).items();
    assertEquals(3, observations.size());
    // Verify first observation
    var first = observations.get(0);
    assertEquals(0.5, ((FeatureValue.NumberVal) first.get("minute")).value());
    assertEquals(12, ((FeatureValue.NumberVal) first.get("our_workers")).value().intValue());
}
```

Use the existing `buildDefaultEnrichment()` helper if one exists, or build an `EnrichedGameData` with test defaults.

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2GameCbrCaseTest#buildForGameEnriched_withTimeline_includesTimelineFeature -q`
Expected: FAIL — overload does not exist

- [ ] **Step 3: Add buildForGameEnriched overload with timeline**

Add to `SC2GameCbrCase.java` after the existing `buildForGameEnriched` method:

```java
public static SC2GameCbrCase buildForGameEnriched(
        String archetypeName, String raceName, String matchup,
        double assessmentConfidence, String strategyId,
        EnrichedGameData e, List<TimelineObservation> timeline) {
    // Call existing method to get base case
    SC2GameCbrCase base = buildForGameEnriched(
            archetypeName, raceName, matchup,
            assessmentConfidence, strategyId, e);
    if (timeline == null || timeline.isEmpty()) {
        return base;
    }
    // Add timeline feature
    var features = new java.util.HashMap<>(base.features());
    List<Map<String, FeatureValue>> observations = timeline.stream()
            .map(t -> Map.<String, FeatureValue>of(
                    "minute", FeatureValue.of(t.minute()),
                    "our_workers", FeatureValue.of(t.ourWorkers()),
                    "our_minerals", FeatureValue.of(t.ourMinerals()),
                    "our_army_supply", FeatureValue.of(t.ourArmySupply())))
            .toList();
    features.put("timeline", FeatureValue.of(observations));
    return base.withFeatures(Map.copyOf(features));
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2GameCbrCaseTest#buildForGameEnriched_withTimeline_includesTimelineFeature -q`
Expected: PASS

- [ ] **Step 5: Write failing retention observer test**

Add to `SC2CbrRetentionObserverTest.java`:

```java
@Test
void onOutcome_storesTimeline() {
    // Simulate timeline samples
    timelineSampler.tick(gameStateAtFrame(0));
    timelineSampler.tick(gameStateAtFrame(672));
    timelineSampler.tick(gameStateAtFrame(1344));

    // Trigger outcome
    var event = buildOutcomeEvent("WIN", "PROTOSS_4GATE", "strategy.early-pressure");
    observer.onOutcome(event);

    // Verify stored case has timeline
    var storedCase = capturedCase();
    assertTrue(storedCase.features().containsKey("timeline"));
    var timeline = (FeatureValue.StructListVal) storedCase.features().get("timeline");
    assertEquals(3, timeline.value().size());
}
```

This requires injecting `TimelineSampler` into the test — the observer already has it injected. Adapt the existing test setup to construct and inject a `TimelineSampler`.

- [ ] **Step 6: Wire TimelineSampler into SC2CbrRetentionObserver**

Add `TimelineSampler timelineSampler` as a constructor parameter to `SC2CbrRetentionObserver` (which uses constructor injection exclusively — see existing `@Inject` constructor at line 59). Store as a field.

In `onOutcome()`, after the `EnrichedGameData enrichment = ...` construction (around line 195), replace the `buildForGameEnriched` call:

```java
// Before:
SC2GameCbrCase cbrCase = SC2GameCbrCase.buildForGameEnriched(
        archetype, raceName, matchup,
        confidence != null ? confidence : 0.0, strategyId, enrichment);

// After:
SC2GameCbrCase cbrCase = SC2GameCbrCase.buildForGameEnriched(
        archetype, raceName, matchup,
        confidence != null ? confidence : 0.0, strategyId, enrichment,
        timelineSampler.getTimeline());
```

- [ ] **Step 7: Run retention observer tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2CbrRetentionObserverTest -q`
Expected: all tests PASS (existing + new)

- [ ] **Step 8: Write integration test — TemporalCbrRetentionIT**

```java
package io.quarkmind.agent.cbr;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TemporalCbrRetentionIT {

    @Inject TimelineSampler timelineSampler;
    @Inject SC2CbrRetentionObserver observer;
    // Inject CbrCaseMemoryStore to verify stored case

    @Test
    void fullGameLifecycle_storesTimelineFeature() {
        // 1. Simulate GameStarted (clears sampler)
        // 2. Feed 5 ticks at 672-frame intervals via timelineSampler.tick()
        // 3. Fire CaseOutcomeEvent with WIN outcome
        // 4. Retrieve the stored case from CbrCaseMemoryStore
        // 5. Assert timeline feature exists as StructListVal with 5 observations
        // 6. Assert phase_sequence is stored as StringListVal
    }
}
```

Adapt the pattern from existing `SC2CbrRetentionIT.java`. The test verifies the full store path including CDI wiring.

- [ ] **Step 9: Run integration test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalCbrRetentionIT -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2GameCbrCase.java quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserver.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/SC2GameCbrCaseTest.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/SC2CbrRetentionObserverTest.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrRetentionIT.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): store game timeline in CBR cases at game close Refs #222"
```

---

## Batch 3: Live Query + Prediction

After this batch: temporal CBR queries run periodically during live games, producing predictions to CaseFile.

### Task 5: TemporalPrediction domain record + trend/confidence logic

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/domain/TemporalPrediction.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/domain/TemporalPredictionTest.java`

**Interfaces:**
- Consumes: `TimelineObservation` (for lookahead trend computation)
- Produces: `TemporalPrediction(String predictedNextPhase, Trend economyTrend, Trend armyTrend, double minutesToNextTransition, double confidence, int matchCount, double bestMatchScore)` with `enum Trend { GROWING, STABLE, DECLINING, SPIKE, UNKNOWN }` and static factory methods for trend computation and confidence calculation

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import java.util.List;
import static io.quarkmind.domain.TemporalPrediction.Trend.*;
import static org.junit.jupiter.api.Assertions.*;

class TemporalPredictionTest {

    @Test
    void computeTrend_growing() {
        var observations = List.of(
                new TimelineObservation(1.0, 12, 100, 5),
                new TimelineObservation(1.5, 14, 150, 8),
                new TimelineObservation(2.0, 16, 200, 12),
                new TimelineObservation(2.5, 18, 250, 16));
        assertEquals(GROWING, TemporalPrediction.computeEconomyTrend(observations));
        assertEquals(GROWING, TemporalPrediction.computeArmyTrend(observations));
    }

    @Test
    void computeTrend_declining() {
        var observations = List.of(
                new TimelineObservation(1.0, 16, 200, 20),
                new TimelineObservation(1.5, 14, 150, 15),
                new TimelineObservation(2.0, 12, 100, 10),
                new TimelineObservation(2.5, 10, 50, 5));
        assertEquals(DECLINING, TemporalPrediction.computeEconomyTrend(observations));
        assertEquals(DECLINING, TemporalPrediction.computeArmyTrend(observations));
    }

    @Test
    void computeTrend_stable() {
        var observations = List.of(
                new TimelineObservation(1.0, 16, 200, 20),
                new TimelineObservation(1.5, 16, 205, 20),
                new TimelineObservation(2.0, 16, 198, 21),
                new TimelineObservation(2.5, 16, 202, 20));
        assertEquals(STABLE, TemporalPrediction.computeEconomyTrend(observations));
        assertEquals(STABLE, TemporalPrediction.computeArmyTrend(observations));
    }

    @Test
    void computeTrend_spike() {
        var observations = List.of(
                new TimelineObservation(1.0, 16, 200, 10),
                new TimelineObservation(1.5, 16, 200, 10),
                new TimelineObservation(2.0, 16, 200, 40),  // army spike
                new TimelineObservation(2.5, 16, 200, 42));
        assertEquals(SPIKE, TemporalPrediction.computeArmyTrend(observations));
    }

    @Test
    void computeTrend_unknown_tooFewObservations() {
        var observations = List.of(
                new TimelineObservation(1.0, 16, 200, 20));
        assertEquals(UNKNOWN, TemporalPrediction.computeEconomyTrend(observations));
    }

    @Test
    void computeConfidence_consensus() {
        // 3 of 5 agree: base * 1.3
        double confidence = TemporalPrediction.computeConfidence(0.8, 3, 5);
        assertEquals(Math.min(1.0, 0.8 * (3.0 / 5.0) * 1.3), confidence, 0.001);
    }

    @Test
    void computeConfidence_unanimousConsensus() {
        // 5 of 5 agree: base * 1.5
        double confidence = TemporalPrediction.computeConfidence(0.9, 5, 5);
        assertEquals(Math.min(1.0, 0.9 * (5.0 / 5.0) * 1.5), confidence, 0.001);
    }

    @Test
    void computeConfidence_noConsensus() {
        // 2 of 5 agree (no majority): base * 0.5
        double confidence = TemporalPrediction.computeConfidence(0.8, 2, 5);
        assertEquals(0.8 * (2.0 / 5.0) * 0.5, confidence, 0.001);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalPredictionTest -q`
Expected: compilation error — `TemporalPrediction` does not exist

- [ ] **Step 3: Implement TemporalPrediction**

```java
package io.quarkmind.domain;

import java.util.List;

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

    public static Trend computeEconomyTrend(List<TimelineObservation> lookahead) {
        return computeTrend(lookahead.stream().mapToInt(TimelineObservation::ourMinerals).toArray());
    }

    public static Trend computeArmyTrend(List<TimelineObservation> lookahead) {
        return computeTrend(lookahead.stream().mapToInt(TimelineObservation::ourArmySupply).toArray());
    }

    static Trend computeTrend(int[] values) {
        if (values.length < 2) return Trend.UNKNOWN;

        double mean = 0;
        for (int v : values) mean += v;
        mean /= values.length;
        if (mean == 0) return Trend.STABLE;

        // Check for spike: any single-step delta > 50% of mean
        for (int i = 1; i < values.length; i++) {
            if (Math.abs(values[i] - values[i - 1]) > 0.5 * mean) return Trend.SPIKE;
        }

        // Check monotonic
        boolean growing = true, declining = true;
        for (int i = 1; i < values.length; i++) {
            if (values[i] < values[i - 1]) growing = false;
            if (values[i] > values[i - 1]) declining = false;
        }
        if (growing) return Trend.GROWING;
        if (declining) return Trend.DECLINING;

        // Check stable: range < 10% of mean
        int min = values[0], max = values[0];
        for (int v : values) { min = Math.min(min, v); max = Math.max(max, v); }
        if ((max - min) < 0.1 * mean) return Trend.STABLE;

        return Trend.UNKNOWN;
    }

    public static double computeConfidence(double dtwScore, int agreeingCount, int topK) {
        double base = dtwScore * ((double) agreeingCount / topK);
        double boost;
        if (agreeingCount >= topK) {
            boost = 1.5;
        } else if (agreeingCount >= 3) {
            boost = 1.3;
        } else {
            boost = 0.5;
        }
        return Math.min(1.0, base * boost);
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalPredictionTest -q`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/domain/TemporalPrediction.java quarkmind-sc2/src/test/java/io/quarkmind/domain/TemporalPredictionTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): TemporalPrediction domain record — trend + confidence logic Refs #222"
```

### Task 6: TemporalCbrTask plugin task

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/TemporalCbrTask.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrTaskTest.java`

**Interfaces:**
- Consumes: `TimelineSampler.getTimeline()`, `CbrCaseMemoryStore.retrieveSimilar()`, `DtwSimilarity.compute()`, `TemporalPrediction.computeConfidence()`, `SummarisationLifecycle.phaseBus()`, CaseFile read/write
- Produces: Writes `TEMPORAL_PREDICTION`, `TEMPORAL_SIMILAR_COUNT`, `TEMPORAL_SIMILAR_BEST_SCORE` to CaseFile; `getId()` returns `"temporal-cbr.predict"`

- [ ] **Step 1: Write failing tests**

```java
package io.quarkmind.agent.cbr;

import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.domain.TemporalPrediction;
import io.quarkmind.domain.TimelineObservation;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class TemporalCbrTaskTest {

    // Test with mock CbrCaseMemoryStore and mock TimelineSampler

    @Test
    void activateIf_requiresMinimumTimelineSize() {
        // Timeline with 3 samples → false (need ≥4)
        // Timeline with 4 samples → true
    }

    @Test
    void activateIf_requiresRoutedArchetype() {
        // CaseFile without STRATEGY_ROUTED_ARCHETYPE → false
        // CaseFile with it → true (assuming timeline sufficient)
    }

    @Test
    void execute_throttles_skipsIfTooSoon() {
        // First call: executes (lastQueryFrame = -1)
        // Second call 100 frames later: skips
        // Third call 5000 frames later: executes
    }

    @Test
    void execute_buildsCorrectQueryWeights() {
        // Verify query has timeline=0.5, phase_sequence=0.3, enemy_archetype=0.1, matchup=0.1
    }

    @Test
    void execute_writesPredictionToCaseFile() {
        // Mock retrieveSimilar to return known cases with timelines
        // Verify TEMPORAL_PREDICTION, TEMPORAL_SIMILAR_COUNT, TEMPORAL_SIMILAR_BEST_SCORE written
    }

    @Test
    void execute_handlesEmptyRetrievalResults() {
        // Mock retrieveSimilar to return empty list
        // Verify no exception, no prediction written
    }

    @Test
    void produces_declaresAllKeys() {
        var task = new TemporalCbrTask(/* mocks */);
        var produced = task.produces();
        assertTrue(produced.contains(QuarkMindCaseFile.TEMPORAL_PREDICTION));
        assertTrue(produced.contains(QuarkMindCaseFile.TEMPORAL_SIMILAR_COUNT));
        assertTrue(produced.contains(QuarkMindCaseFile.TEMPORAL_SIMILAR_BEST_SCORE));
    }

    @Test
    void getId_returnsTemporalCbrPrefix() {
        var task = new TemporalCbrTask(/* mocks */);
        assertEquals("temporal-cbr.predict", task.getId());
    }
}
```

Build concrete test implementations using the same mock/test patterns as `SC2StrategyRouterTaskTest` — check how it constructs `MapCaseContext`, mocks `CbrCaseMemoryStore`, and verifies CaseFile writes.

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalCbrTaskTest -q`
Expected: compilation error — `TemporalCbrTask` does not exist

- [ ] **Step 3: Implement TemporalCbrTask**

```java
package io.quarkmind.agent.cbr;

import io.casehub.annotation.CaseType;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.quarkmind.agency.task.TaskDefinition;
import io.quarkmind.agent.QuarkMindCaseFile;
import io.quarkmind.domain.*;
import io.quarkmind.plugin.summarisation.SummarisationLifecycle;
import io.quarkmind.plugin.summarisation.TacticalPosture;
import io.quarkmind.sc2.GameStarted;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import java.nio.file.Path;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Predicate;
import java.util.stream.Collectors;

@ApplicationScoped
@CaseType("starcraft-game")
public class TemporalCbrTask implements TaskDefinition {

    static final long QUERY_INTERVAL_FRAMES = 2688; // ~2 min at 22.4 loops/s
    private static final int TOP_K = 5;
    private static final double MIN_SIMILARITY = 0.3;
    private static final MemoryDomain DOMAIN = new MemoryDomain("quarkmind");

    @Inject TimelineSampler timelineSampler;
    @Inject CbrCaseMemoryStore cbrStore;
    @Inject SummarisationLifecycle summarisationLifecycle;

    private final List<TacticalPosture> phases = new CopyOnWriteArrayList<>();
    private long lastQueryFrame = -QUERY_INTERVAL_FRAMES;

    @PostConstruct
    void subscribeToPhaseBus() {
        if (summarisationLifecycle.phaseBus() != null) {
            summarisationLifecycle.phaseBus().subscribe(e -> true, phases::add);
        }
    }

    void onGameStarted(@Observes GameStarted event) {
        phases.clear();
        lastQueryFrame = -QUERY_INTERVAL_FRAMES;
    }

    @Override public String getId() { return "temporal-cbr.predict"; }
    @Override public String getName() { return "Temporal CBR Prediction"; }

    @Override
    public Set<String> requires() {
        return Set.of(QuarkMindCaseFile.GAME_STATE, QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE);
    }

    @Override
    public Set<String> produces() {
        return Set.of(
                QuarkMindCaseFile.TEMPORAL_PREDICTION,
                QuarkMindCaseFile.TEMPORAL_SIMILAR_COUNT,
                QuarkMindCaseFile.TEMPORAL_SIMILAR_BEST_SCORE);
    }

    @Override
    public Predicate<CaseContext> activateIf() {
        return ctx -> timelineSampler.getTimeline().size() >= 4;
    }

    @Override
    public void execute(CaseContext context) {
        var gameState = (GameState) context.get(QuarkMindCaseFile.GAME_STATE);
        if (gameState.gameFrame() - lastQueryFrame < QUERY_INTERVAL_FRAMES) return;
        lastQueryFrame = gameState.gameFrame();

        var timeline = timelineSampler.getTimeline();
        var phaseSequence = phases.stream().map(TacticalPosture::posture).toList();
        var archetype = (String) context.get(QuarkMindCaseFile.STRATEGY_ROUTED_ARCHETYPE);
        // Derive matchup from enemy race — same pattern as SC2StrategyRouterTask
        var enemyRace = (String) context.get(QuarkMindCaseFile.ENEMY_RACE);
        var matchup = enemyRace != null ? "Pv" + enemyRace.charAt(0) : null;

        // Build query — use CbrQuery.of() factory, not builder
        var queryFeatures = new HashMap<String, FeatureValue>();
        queryFeatures.put("timeline", toStructListVal(timeline));
        queryFeatures.put("phase_sequence", FeatureValue.of(phaseSequence));
        if (archetype != null) queryFeatures.put("enemy_archetype", FeatureValue.of(archetype));
        if (matchup != null) queryFeatures.put("matchup", FeatureValue.of(matchup));

        var query = CbrQuery.of("default", DOMAIN, Path.of("quarkmind/strategy/cases"),
                        SC2GameCbrCase.CBR_TYPE, Map.copyOf(queryFeatures), TOP_K)
                .withWeights(Map.of("timeline", 0.50, "phase_sequence", 0.30,
                        "enemy_archetype", 0.10, "matchup", 0.10))
                .withMinSimilarity(MIN_SIMILARITY);

        var results = cbrStore.retrieveSimilar(query, SC2GameCbrCase.class);
        if (results.isEmpty()) return;

        var prediction = extractPrediction(timeline, results, phaseSequence);
        if (prediction != null) {
            context.set(QuarkMindCaseFile.TEMPORAL_PREDICTION, prediction);
            context.set(QuarkMindCaseFile.TEMPORAL_SIMILAR_COUNT, results.size());
            context.set(QuarkMindCaseFile.TEMPORAL_SIMILAR_BEST_SCORE, results.get(0).score());
        }
    }

    TemporalPrediction extractPrediction(
            List<TimelineObservation> queryTimeline,
            List<ScoredCbrCase<SC2GameCbrCase>> results,
            List<String> queryPhases) {

        // Build query observations as List<Map<String,FeatureValue>> for DtwSimilarity
        var queryObs = queryTimeline.stream()
                .map(t -> Map.<String, FeatureValue>of(
                        "minute", FeatureValue.of(t.minute()),
                        "our_workers", FeatureValue.of(t.ourWorkers()),
                        "our_minerals", FeatureValue.of(t.ourMinerals()),
                        "our_army_supply", FeatureValue.of(t.ourArmySupply())))
                .toList();

        // Get the TimeSeries schema field for DtwSimilarity.compute()
        var schema = SC2CbrSchemaRegistrar.buildStrategySchema();
        var tsField = (FeatureField.TimeSeries) schema.fields().stream()
                .filter(f -> f.name().equals("timeline")).findFirst().orElseThrow();

        // Re-run DTW on each result to get alignment paths
        Map<String, Integer> phaseVotes = new HashMap<>();
        List<Double> transitionTimes = new ArrayList<>();
        List<List<TimelineObservation>> lookaheads = new ArrayList<>();

        for (var scored : results) {
            var caseFeatures = scored.cbrCase().features();
            if (!caseFeatures.containsKey("timeline")) continue;

            var caseTimeline = ((FeatureValue.StructListVal) caseFeatures.get("timeline")).items();
            // DtwSimilarity.compute() is a static method — no instance needed
            var dtwResult = DtwSimilarity.compute(
                    queryObs, caseTimeline, tsField,
                    new WarpingConstraint.SakoeChibaBand(3), Double.MAX_VALUE);

            // Find alignment point: last query step maps to step s_k in past game
            var alignment = dtwResult.alignment();
            if (alignment.isEmpty()) continue;
            int sk = alignment.get(alignment.size() - 1).caseIndex();

            // Lookahead: past game steps after s_k (next 4 steps = ~2 min)
            int lookaheadEnd = Math.min(caseTimeline.size(), sk + 5);
            if (sk + 1 >= caseTimeline.size()) continue; // past game ended at alignment

            var lookahead = new ArrayList<TimelineObservation>();
            for (int i = sk + 1; i < lookaheadEnd; i++) {
                var obs = caseTimeline.get(i);
                lookahead.add(new TimelineObservation(
                        ((FeatureValue.NumberVal) obs.get("minute")).value().doubleValue(),
                        ((FeatureValue.NumberVal) obs.get("our_workers")).value().intValue(),
                        ((FeatureValue.NumberVal) obs.get("our_minerals")).value().intValue(),
                        ((FeatureValue.NumberVal) obs.get("our_army_supply")).value().intValue()));
            }
            lookaheads.add(lookahead);

            // Phase prediction from matched case's phase_sequence
            if (caseFeatures.containsKey("phase_sequence")) {
                var casePhases = ((FeatureValue.StringListVal) caseFeatures.get("phase_sequence")).value();
                // Find phase at alignment point timestamp and the next distinct phase
                double alignMinute = ((FeatureValue.NumberVal) caseTimeline.get(sk).get("minute")).value().doubleValue();
                // Walk phase sequence to find transition after alignment
                for (int i = 0; i < casePhases.size() - 1; i++) {
                    if (!casePhases.get(i).equals(casePhases.get(i + 1))) {
                        String nextPhase = casePhases.get(i + 1);
                        phaseVotes.merge(nextPhase, 1, Integer::sum);
                        break;
                    }
                }
            }
        }

        if (lookaheads.isEmpty()) return null;

        // Aggregate trends from all lookaheads (use first non-empty)
        var bestLookahead = lookaheads.get(0);
        var economyTrend = TemporalPrediction.computeEconomyTrend(bestLookahead);
        var armyTrend = TemporalPrediction.computeArmyTrend(bestLookahead);

        // Majority phase vote
        var bestPhase = phaseVotes.entrySet().stream()
                .max(Map.Entry.comparingByValue())
                .map(Map.Entry::getKey)
                .orElse("UNKNOWN");
        int agreeCount = phaseVotes.getOrDefault(bestPhase, 0);

        double confidence = TemporalPrediction.computeConfidence(
                results.get(0).score(), agreeCount, results.size());

        return new TemporalPrediction(
                bestPhase, economyTrend, armyTrend,
                bestLookahead.isEmpty() ? 0 :
                        bestLookahead.get(bestLookahead.size() - 1).minute() - queryTimeline.get(queryTimeline.size() - 1).minute(),
                confidence, results.size(), results.get(0).score());
    }

    private FeatureValue toStructListVal(List<TimelineObservation> timeline) {
        return FeatureValue.of(timeline.stream()
                .map(t -> Map.<String, FeatureValue>of(
                        "minute", FeatureValue.of(t.minute()),
                        "our_workers", FeatureValue.of(t.ourWorkers()),
                        "our_minerals", FeatureValue.of(t.ourMinerals()),
                        "our_army_supply", FeatureValue.of(t.ourArmySupply())))
                .toList());
    }
}
```

Key API details verified against `SC2StrategyRouterTask.java`:
- `CbrQuery.of("default", DOMAIN, Path.of(...), caseType, features, topK)` + `.withWeights()` + `.withMinSimilarity()`
- `MemoryDomain` constructed as `new MemoryDomain("quarkmind")`
- `activateIf()` returns `Predicate<CaseContext>`, not `boolean`
- `DtwSimilarity.compute()` is a static method returning `DtwResult(score, List<AlignmentPair>)`
- `ScoredCbrCase.cbrCase().features()` gives access to stored case features
- `FeatureValue.StructListVal.items()` (not `.value()`) for the observation list

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalCbrTaskTest -q`
Expected: all 8 tests PASS

- [ ] **Step 5: Run full module test suite — verify no regressions**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS (if `ClassTooLargeException`, run `mvn clean` first)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/main/java/io/quarkmind/agent/cbr/TemporalCbrTask.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrTaskTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): TemporalCbrTask — periodic DTW query + predictive CBR Refs #222"
```

### Task 7: Integration + calibration tests

**Files:**
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrQueryIT.java` (new)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalMatchingCalibrationTest.java` (new, `@Tag("benchmark")`)

**Interfaces:**
- Consumes: all components from Tasks 1-6; replay dataset for calibration
- Produces: validated end-to-end pipeline; performance baseline

- [ ] **Step 1: Write TemporalCbrQueryIT**

```java
package io.quarkmind.agent.cbr;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TemporalCbrQueryIT {

    @Inject CbrCaseMemoryStore cbrStore;
    @Inject TimelineSampler timelineSampler;
    @Inject TemporalCbrTask temporalCbrTask;

    @Test
    void queryWithPartialTimeline_retrievesSimilarGames() {
        // 1. Store 3 cases with distinct economy curves:
        //    - Case A: fast expansion (workers grow quickly, minerals stay low)
        //    - Case B: rush (workers stay low, army grows quickly)
        //    - Case C: turtle (workers moderate, minerals accumulate)
        // 2. Build a query timeline matching Case A's first 3 minutes
        // 3. Execute temporal query
        // 4. Verify Case A is the best match (highest DTW score)
        // 5. Verify prediction is written to CaseFile
    }

    @Test
    void queryWithPhaseSequence_editDistanceAffectsRanking() {
        // 1. Store 2 cases: same economy curves, different phase sequences
        //    - Case D: EARLY_MACRO → MID_SKIRMISH → DEFENSIVE_HOLD
        //    - Case E: EARLY_MACRO → EARLY_AGGRESSION → MID_SKIRMISH
        // 2. Query with phase sequence [EARLY_MACRO, MID_SKIRMISH]
        // 3. Verify Case D ranks higher (closer phase match via edit distance)
    }
}
```

Follow `SC2CbrRoutingIT` patterns for storing cases and building query contexts.

- [ ] **Step 2: Run integration test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TemporalCbrQueryIT -q`
Expected: PASS

- [ ] **Step 3: Write TemporalMatchingCalibrationTest**

```java
package io.quarkmind.agent.cbr;

import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@Tag("benchmark")
class TemporalMatchingCalibrationTest {

    @Test
    void selfRetrieval_sameGameInTop3() {
        // 1. Load 10 replays from AI Arena dataset
        // 2. For each replay, extract timeline (using SimulatedGame or ReplayEngine)
        // 3. Store all 10 as CBR cases
        // 4. For each stored case, query with first 3 minutes of its own timeline
        // 5. Assert the same game appears in top-3 results (self-retrieval sanity)
    }

    @Test
    void queryLatency_under1Second_500Cases() {
        // 1. Generate 500 synthetic cases with realistic timelines
        //    (random economy curves with controlled variance)
        // 2. Time a single temporal query
        // 3. Assert total time < 1000ms
    }
}
```

Run with: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=TemporalMatchingCalibrationTest`

- [ ] **Step 4: Run calibration test**

Run: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=TemporalMatchingCalibrationTest -q`
Expected: PASS — self-retrieval works, latency under 1s

- [ ] **Step 5: Run full module test suite — final regression check**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalCbrQueryIT.java quarkmind-sc2/src/test/java/io/quarkmind/agent/cbr/TemporalMatchingCalibrationTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "feat(#222): temporal CBR integration + calibration tests Refs #222"
```

---

## References

- [2026-08-24-temporal-replay-matching-design.md] — design spec this plan implements
- [SC2GameCbrCase.java:54-57] — existing `buildForGameEnriched` signature
- [SC2CbrRetentionObserver.java:72-73] — phaseBus subscription pattern
- [SC2CbrRetentionObserver.java:197-199] — buildForGameEnriched call site
- [SC2CbrSchemaRegistrar.java:22-50] — existing schema registration
- [GameTickExecutor.java:59] — `engine.observe()` integration point
- [QuarkMindCaseFile.java:57] — last CaseFile constant (insert after)
- [QuarkMindCaseHub.java:107-114] — PHASE_ORDER list
- [GamePhaseSummariser.java:34-52] — 5 TacticalPosture phases
- [SummarisationLifecycle.java:75] — `phaseBus()` return type
- [GameStarted.java:3-4] — CDI event record
- [EnrichedGameData.java:6-27] — 20-parameter record (unchanged)
- GitHub #222 — focal issue
- GitHub #214 — parent epic (CBR Game Experience)
