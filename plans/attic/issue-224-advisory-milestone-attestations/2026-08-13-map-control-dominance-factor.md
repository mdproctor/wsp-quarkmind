# Map Control Dominance Factor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #228 — feat: map control dominance factor — expansion coverage relative to available bases
**Issue group:** #228

**Goal:** Add a 5th dominance factor (`mapControl`) that measures expansion-site occupation relative to available base locations on the map.

**Architecture:** Extend `DominanceWeights`, `WeightModifier`, and `MilestoneConfig` with a 5th field. Add `mapControlFactor()` to `MultiFactorDominanceAssessor` using `GameState.mapInfo().expansions()` and a proximity threshold (`EXPANSION_CONTROL_RADIUS = 10.0`). Update all 3 weight strategies and all Drools rules mechanically. Write a calibration test for the proximity threshold.

**Tech Stack:** Java 21, Quarkus, Drools Rule Units, JUnit 5 + AssertJ

## Global Constraints

- `EXPANSION_CONTROL_RADIUS = 10.0` — uncalibrated starting value, requires calibration test per `sc2data-spatial-constants-require-calibration` protocol
- `maxExpectedMapControlDelta` default: 4
- Default weights: economy=0.30, army=0.35, tech=0.20, bases=0.05, mapControl=0.10
- All existing Drools rules get `mapControlDelta = 0.00` (neutral)
- Fog-of-war guard: `enemyBuildings.isEmpty() → 0.0`

---

### Task 1: Expand DominanceWeights and WeightModifier records

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/DominanceWeights.java`
- Modify: `src/main/java/io/quarkmind/agent/WeightModifier.java`
- Modify: `src/test/java/io/quarkmind/domain/DominanceWeightsTest.java`

**Interfaces:**
- Produces: `DominanceWeights(double economy, double army, double tech, double bases, double mapControl)` — 5-field record, sum-to-1.0 invariant
- Produces: `WeightModifier(double economyDelta, double armyDelta, double techDelta, double basesDelta, double mapControlDelta, String reason)` — 5-field delta record

- [ ] **Step 1: Update DominanceWeightsTest for 5-field construction**

Add `mapControl` to all constructor calls and assertions. Update sum-to-1.0 tests.

```java
// DominanceWeightsTest.java — update validWeights_accepted:
@Test
void validWeights_accepted() {
    var w = new DominanceWeights(0.30, 0.35, 0.20, 0.05, 0.10);
    assertThat(w.economy()).isEqualTo(0.30);
    assertThat(w.army()).isEqualTo(0.35);
    assertThat(w.tech()).isEqualTo(0.20);
    assertThat(w.bases()).isEqualTo(0.05);
    assertThat(w.mapControl()).isEqualTo(0.10);
}

// Update weightsSumTooHigh_rejected:
@Test
void weightsSumTooHigh_rejected() {
    assertThatThrownBy(() -> new DominanceWeights(0.50, 0.50, 0.20, 0.15, 0.10))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("sum to 1.0");
}

// Update weightsSumTooLow_rejected:
@Test
void weightsSumTooLow_rejected() {
    assertThatThrownBy(() -> new DominanceWeights(0.10, 0.10, 0.10, 0.10, 0.10))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("sum to 1.0");
}

// Update equalQuarters_accepted → equalFifths:
@Test
void equalFifths_accepted() {
    var w = new DominanceWeights(0.20, 0.20, 0.20, 0.20, 0.20);
    assertThat(w.economy()).isEqualTo(0.20);
}

// Update allZeros_rejected:
@Test
void allZeros_rejected() {
    assertThatThrownBy(() -> new DominanceWeights(0.0, 0.0, 0.0, 0.0, 0.0))
        .isInstanceOf(IllegalArgumentException.class);
}

// Update withinTolerance_accepted:
@Test
void withinTolerance_accepted() {
    assertThatCode(() -> new DominanceWeights(0.1999, 0.2001, 0.20, 0.20, 0.20))
        .doesNotThrowAnyException();
}

// Update beyondTolerance_rejected:
@Test
void beyondTolerance_rejected() {
    assertThatThrownBy(() -> new DominanceWeights(0.30, 0.35, 0.20, 0.06, 0.10))
        .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=DominanceWeightsTest -q`
Expected: FAIL — `DominanceWeights` constructor only takes 4 args

- [ ] **Step 3: Update DominanceWeights record**

```java
// DominanceWeights.java
public record DominanceWeights(double economy, double army, double tech, double bases, double mapControl) {
    public DominanceWeights {
        double sum = economy + army + tech + bases + mapControl;
        if (Math.abs(sum - 1.0) > 0.001) {
            throw new IllegalArgumentException("Weights must sum to 1.0, got " + sum);
        }
    }
}
```

- [ ] **Step 4: Update WeightModifier record**

```java
// WeightModifier.java
public record WeightModifier(
    double economyDelta,
    double armyDelta,
    double techDelta,
    double basesDelta,
    double mapControlDelta,
    String reason
) {}
```

- [ ] **Step 5: Run DominanceWeightsTest to verify it passes**

Run: `mvn test -Dtest=DominanceWeightsTest -q`
Expected: PASS

Note: The project will not compile yet — many callers of `DominanceWeights` and `WeightModifier` constructors still pass 4 args. This is expected. Task 2 fixes them.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/domain/DominanceWeights.java src/main/java/io/quarkmind/agent/WeightModifier.java src/test/java/io/quarkmind/domain/DominanceWeightsTest.java
git commit -m "feat(#228): expand DominanceWeights and WeightModifier to 5 fields"
```

---

### Task 2: Update weight strategies and interpolator

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/MilestoneConfig.java:70-76` (WeightAnchor interface)
- Modify: `src/main/java/io/quarkmind/agent/AnchorInterpolator.java:25-49`
- Modify: `src/main/java/io/quarkmind/agent/TemporalDominanceWeightStrategy.java`
- Modify: `src/main/java/io/quarkmind/agent/SituationalDominanceWeightStrategy.java:16-21,41-59`
- Modify: `src/main/java/io/quarkmind/agent/DroolsDominanceWeightStrategy.java:60-84`
- Modify: `src/main/resources/application.properties:189-199`
- Modify: `src/test/java/io/quarkmind/agent/AnchorInterpolatorTest.java`
- Modify: `src/test/java/io/quarkmind/agent/TemporalDominanceWeightStrategyTest.java`
- Modify: `src/test/java/io/quarkmind/agent/SituationalDominanceWeightStrategyTest.java`
- Modify: `src/test/java/io/quarkmind/agent/DroolsDominanceWeightStrategyTest.java`

**Interfaces:**
- Consumes: `DominanceWeights(economy, army, tech, bases, mapControl)` from Task 1
- Consumes: `WeightModifier(economyDelta, armyDelta, techDelta, basesDelta, mapControlDelta, reason)` from Task 1
- Produces: `MilestoneConfig.Dominance.WeightAnchor.mapControlWeight()` — config property
- Produces: `MilestoneConfig.Dominance.maxExpectedMapControlDelta()` — config property

- [ ] **Step 1: Update AnchorInterpolatorTest helper and tests**

Update the `anchor()` helper to accept 5 weights and update all test calls:

```java
// AnchorInterpolatorTest.java

static MilestoneConfig.Dominance.WeightAnchor anchor(
        long frame, double economy, double army, double tech, double bases, double mapControl) {
    return new MilestoneConfig.Dominance.WeightAnchor() {
        @Override public long frame() { return frame; }
        @Override public double economyWeight() { return economy; }
        @Override public double armyWeight() { return army; }
        @Override public double techWeight() { return tech; }
        @Override public double basesWeight() { return bases; }
        @Override public double mapControlWeight() { return mapControl; }
    };
}
```

Update all `anchor(...)` calls in tests from 4-weight to 5-weight format. Example pattern — the old default `anchor(0, 0.30, 0.35, 0.20, 0.15)` becomes `anchor(0, 0.30, 0.35, 0.20, 0.05, 0.10)`.

Also update assertions that check `w.bases()` — the value changes from 0.15 to 0.05. Add `assertThat(w.mapControl()).isCloseTo(0.10, offset(0.001))` where individual fields are checked.

Update `interpolatedWeights_sumToOne` to sum all 5 fields:
```java
double sum = w.economy() + w.army() + w.tech() + w.bases() + w.mapControl();
```

- [ ] **Step 2: Update MilestoneConfig**

Add to `Dominance` interface:
```java
@WithName("max-expected-map-control-delta")
@WithDefault("4")
int maxExpectedMapControlDelta();
```

Add to `WeightAnchor` interface:
```java
@WithName("map-control-weight") double mapControlWeight();
```

- [ ] **Step 3: Update AnchorInterpolator**

In `interpolate()`, add mapControl to the 5-field `DominanceWeights` construction:
```java
return new DominanceWeights(
    lerp(lo.economyWeight(), hi.economyWeight(), t),
    lerp(lo.armyWeight(), hi.armyWeight(), t),
    lerp(lo.techWeight(), hi.techWeight(), t),
    lerp(lo.basesWeight(), hi.basesWeight(), t),
    lerp(lo.mapControlWeight(), hi.mapControlWeight(), t));
```

In `toWeights()`:
```java
private static DominanceWeights toWeights(MilestoneConfig.Dominance.WeightAnchor a) {
    return new DominanceWeights(a.economyWeight(), a.armyWeight(), a.techWeight(),
        a.basesWeight(), a.mapControlWeight());
}
```

- [ ] **Step 4: Update SituationalDominanceWeightStrategy**

Update `PHASE_MODIFIERS` from `double[4]` to `double[5]` — all 5th values are `0.00`:
```java
private static final Map<String, double[]> PHASE_MODIFIERS = Map.of(
    "DEFENSIVE_HOLD",    new double[]{-0.10, +0.15, -0.05, 0.00, 0.00},
    "EARLY_AGGRESSION",  new double[]{-0.05, +0.10, -0.05, 0.00, 0.00},
    "EARLY_MACRO",       new double[]{+0.10, -0.10, +0.05, -0.05, 0.00},
    "MID_SKIRMISH",      new double[]{-0.05, +0.10, -0.05, 0.00, 0.00},
    "TRANSITIONING",     new double[]{0.00,  0.00,  0.00,  0.00, 0.00}
);
```

Update `resolve()` — add mapControl processing:
```java
public DominanceWeights resolve(WeightContext context) {
    DominanceWeights baseline = interpolator.interpolate(context.gameFrame());
    double[] mod = context.currentPhase() != null
                   ? PHASE_MODIFIERS.getOrDefault(context.currentPhase(), new double[5])
                   : new double[5];

    double economy    = Math.max(MINIMUM_WEIGHT, baseline.economy() + mod[0]);
    double army       = Math.max(MINIMUM_WEIGHT, baseline.army() + mod[1]);
    double tech       = Math.max(MINIMUM_WEIGHT, baseline.tech() + mod[2]);
    double bases      = Math.max(MINIMUM_WEIGHT, baseline.bases() + mod[3]);
    double mapControl = Math.max(MINIMUM_WEIGHT, baseline.mapControl() + mod[4]);

    if (economy != baseline.economy() + mod[0] || army != baseline.army() + mod[1]
        || tech != baseline.tech() + mod[2] || bases != baseline.bases() + mod[3]
        || mapControl != baseline.mapControl() + mod[4]) {
        log.debugf("[DOMINANCE] Clamping fired: phase=%s", context.currentPhase());
    }

    double sum = economy + army + tech + bases + mapControl;
    return new DominanceWeights(economy / sum, army / sum, tech / sum, bases / sum, mapControl / sum);
}
```

- [ ] **Step 5: Update DroolsDominanceWeightStrategy.applyModifiers()**

```java
static DominanceWeights applyModifiers(DominanceWeights baseline,
                                       List<WeightModifier> modifiers) {
    if (modifiers.isEmpty()) return baseline;

    double economy    = baseline.economy();
    double army       = baseline.army();
    double tech       = baseline.tech();
    double bases      = baseline.bases();
    double mapControl = baseline.mapControl();

    for (WeightModifier mod : modifiers) {
        economy    += mod.economyDelta();
        army       += mod.armyDelta();
        tech       += mod.techDelta();
        bases      += mod.basesDelta();
        mapControl += mod.mapControlDelta();
    }

    economy    = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, economy);
    army       = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, army);
    tech       = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, tech);
    bases      = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, bases);
    mapControl = Math.max(DominanceWeightStrategy.MINIMUM_WEIGHT, mapControl);

    double sum = economy + army + tech + bases + mapControl;
    return new DominanceWeights(
        economy / sum, army / sum, tech / sum, bases / sum, mapControl / sum);
}
```

- [ ] **Step 6: Update application.properties**

```properties
quarkmind.milestones.dominance.max-expected-map-control-delta=4
quarkmind.milestones.dominance.anchors[0].economy-weight=0.30
quarkmind.milestones.dominance.anchors[0].army-weight=0.35
quarkmind.milestones.dominance.anchors[0].tech-weight=0.20
quarkmind.milestones.dominance.anchors[0].bases-weight=0.05
quarkmind.milestones.dominance.anchors[0].map-control-weight=0.10
```

- [ ] **Step 7: Update Drools DRL — all 19 rules**

Every `new WeightModifier(a, b, c, d, "reason")` becomes `new WeightModifier(a, b, c, d, 0.00, "reason")` in `DominanceWeightAdjustment.drl`. The 5th arg (`mapControlDelta`) is `0.00` for all rules.

- [ ] **Step 8: Update weight strategy tests**

Update all `anchor(...)` calls in `TemporalDominanceWeightStrategyTest`, `SituationalDominanceWeightStrategyTest`, and `DroolsDominanceWeightStrategyTest` from 4-weight to 5-weight format.

Update `DroolsDominanceWeightStrategyTest` assertions for `applyModifiers()` to include 5-field `DominanceWeights` and `WeightModifier` construction.

Update `MultiFactorDominanceAssessorTest`:
- `FIXED_WEIGHTS` anchor: `anchor(0, 0.30, 0.35, 0.20, 0.05, 0.10)`
- Constructor: `new MultiFactorDominanceAssessor(FIXED_WEIGHTS, 25.0, 3000, 2.0, 3, 4, 3, null)` (adds `maxExpectedMapControlDelta=4`)
- All `anchor(...)` calls in `assess_weightsChangeWithGameFrame` test

- [ ] **Step 9: Run full test suite**

Run: `mvn test -q`
Expected: Compile and pass — all 4-arg callers updated to 5-arg

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#228): update weight strategies, interpolator, config, and Drools rules for 5-field weights"
```

---

### Task 3: Implement mapControlFactor in MultiFactorDominanceAssessor

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java`
- Modify: `src/main/java/io/quarkmind/agent/MilestoneConfig.java:46-58` (Dominance interface)
- Modify: `src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java`

**Interfaces:**
- Consumes: `DominanceWeights.mapControl()` from Task 1
- Consumes: `MilestoneConfig.Dominance.maxExpectedMapControlDelta()` from Task 2
- Consumes: `GameState.mapInfo().expansions()` — existing API
- Consumes: `Building.position()`, `ExpansionLocation.position()`, `Point2d.distanceTo()` — existing API
- Produces: `DominanceScore` with `"mapControl"` in factors map

- [ ] **Step 1: Write failing tests for mapControlFactor**

Add to `MultiFactorDominanceAssessorTest.java`. First, add a helper to construct `GameState` with `MapInfo`:

```java
private static GameState gameStateWithMap(int minerals, int vespene, int supply, int supplyUsed,
        List<Unit> myUnits, List<Building> myBuildings,
        List<Unit> enemyUnits, List<Building> enemyBuildings, MapInfo mapInfo) {
    return new GameState(minerals, vespene, supply, supplyUsed,
        myUnits, myBuildings, enemyUnits, enemyBuildings,
        List.of(), List.of(), List.of(), 5000, mapInfo);
}

private static MapInfo mapWithExpansions(List<ExpansionLocation> expansions) {
    return new MapInfo(new Point2d(30, 30), new Point2d(130, 130),
        160, 160, expansions, List.of(), List.of());
}

private static Building nexusAt(Point2d pos) {
    return new Building("tag-nexus-" + pos, BuildingType.NEXUS, pos, 1000, 1000, true);
}

private static ExpansionLocation expansion(int ordinal, double x, double y) {
    return new ExpansionLocation(ordinal, new Point2d(x, y));
}
```

Then add the map control tests:

```java
@Test
void mapControl_ownControlsMore_positive() {
    var expansions = List.of(
        expansion(0, 30, 30), expansion(1, 60, 60), expansion(2, 90, 90));
    MapInfo map = mapWithExpansions(expansions);
    GameState state = gameStateWithMap(200, 100, 15, 10,
        List.of(probe(), probe(), probe()),
        List.of(nexusAt(new Point2d(30, 30)), nexusAt(new Point2d(60, 60))),
        List.of(probe(), probe(), probe()),
        List.of(nexusAt(new Point2d(90, 90))), map);
    DominanceScore score = assessor.assess(state);
    assertThat(score.factors().get("mapControl")).isGreaterThan(0.0);
}

@Test
void mapControl_noExpansions_returnsZero() {
    MapInfo map = mapWithExpansions(List.of());
    GameState state = gameStateWithMap(200, 100, 15, 10,
        List.of(probe(), probe(), probe()), List.of(nexus()),
        List.of(probe(), probe(), probe()), List.of(nexus()), map);
    DominanceScore score = assessor.assess(state);
    assertThat(score.factors().get("mapControl")).isEqualTo(0.0);
}

@Test
void mapControl_enemyBuildingsEmpty_returnsZero() {
    var expansions = List.of(expansion(0, 30, 30), expansion(1, 60, 60));
    MapInfo map = mapWithExpansions(expansions);
    GameState state = gameStateWithMap(200, 100, 15, 10,
        List.of(probe(), probe(), probe()),
        List.of(nexusAt(new Point2d(30, 30))),
        List.of(probe(), probe(), probe()),
        List.of(), map);
    // 3 enemy units + 0 buildings = 3 >= threshold, but enemyBuildings empty → fog guard
    DominanceScore score = assessor.assess(state);
    assertThat(score.factors().get("mapControl")).isEqualTo(0.0);
}

@Test
void mapControl_equalControl_zero() {
    var expansions = List.of(
        expansion(0, 30, 30), expansion(1, 90, 90));
    MapInfo map = mapWithExpansions(expansions);
    GameState state = gameStateWithMap(200, 100, 15, 10,
        List.of(probe(), probe(), probe()),
        List.of(nexusAt(new Point2d(30, 30))),
        List.of(probe(), probe(), probe()),
        List.of(nexusAt(new Point2d(90, 90))), map);
    DominanceScore score = assessor.assess(state);
    assertThat(score.factors().get("mapControl")).isCloseTo(0.0, offset(0.001));
}
```

Update the existing factor-key test:
```java
@Test
void assess_containsAllFiveFactors() {
    GameState state = gameState(200, 100, 15, 10,
        List.of(probe(), probe(), probe()), List.of(nexus()),
        List.of(probe(), probe(), probe()), List.of(nexus()));
    DominanceScore score = assessor.assess(state);
    assertThat(score.factors()).containsOnlyKeys("economy", "army", "tech", "bases", "mapControl");
}
```

Update `assess_overallIsWeightedSum`:
```java
double expectedOverall = score.factors().get("economy") * w.economy()
    + score.factors().get("army") * w.army()
    + score.factors().get("tech") * w.tech()
    + score.factors().get("bases") * w.bases()
    + score.factors().get("mapControl") * w.mapControl();
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -Dtest=MultiFactorDominanceAssessorTest -q`
Expected: FAIL — no `mapControl` key in factors map

- [ ] **Step 3: Implement mapControlFactor**

Add to `MultiFactorDominanceAssessor`:

```java
static final double EXPANSION_CONTROL_RADIUS = 10.0;

private final int maxExpectedMapControlDelta;
```

Add to both constructors — CDI constructor reads `config.dominance().maxExpectedMapControlDelta()`, test constructor takes `int maxExpectedMapControlDelta` parameter (inserted after `maxExpectedBaseDelta`).

Add the factor method:
```java
private double mapControlFactor(GameState state) {
    if (state.enemyBuildings().isEmpty()) return 0.0;
    List<ExpansionLocation> expansions = state.mapInfo() != null
        ? state.mapInfo().expansions() : List.of();
    if (expansions.isEmpty()) return 0.0;

    long ownControlled = expansions.stream()
        .filter(exp -> controlledBy(state.myBuildings(), exp)).count();
    long enemyControlled = expansions.stream()
        .filter(exp -> controlledBy(state.enemyBuildings(), exp)).count();
    return clamp((double) (ownControlled - enemyControlled) / maxExpectedMapControlDelta);
}

private static boolean controlledBy(List<Building> buildings, ExpansionLocation exp) {
    return buildings.stream().anyMatch(b ->
        b.isComplete() && SC2Data.isBase(b.type())
        && b.position().distanceTo(exp.position()) <= EXPANSION_CONTROL_RADIUS);
}
```

Update `assess()`:
```java
double mapControl = mapControlFactor(state);

// In weighted sum:
double overall = clamp(economy * weights.economy() + army * weights.army()
    + tech * weights.tech() + bases * weights.bases()
    + mapControl * weights.mapControl());

// In factors map (change capacity from 4 to 5):
Map<String, Double> factors = new LinkedHashMap<>(5);
factors.put("economy", economy);
factors.put("army", army);
factors.put("tech", tech);
factors.put("bases", bases);
factors.put("mapControl", mapControl);
```

Update `NEUTRAL`:
```java
private static final DominanceScore NEUTRAL = new DominanceScore(0.0,
    Map.of("economy", 0.0, "army", 0.0, "tech", 0.0, "bases", 0.0, "mapControl", 0.0));
```

Update weight-shift logging to include mapControl:
```java
if (lastWeights != null && (
        Math.abs(weights.economy() - lastWeights.economy()) > 0.01
        || Math.abs(weights.army() - lastWeights.army()) > 0.01
        || Math.abs(weights.tech() - lastWeights.tech()) > 0.01
        || Math.abs(weights.bases() - lastWeights.bases()) > 0.01
        || Math.abs(weights.mapControl() - lastWeights.mapControl()) > 0.01)) {
    log.debugf("[DOMINANCE] Weights shifted: economy=%.2f army=%.2f tech=%.2f bases=%.2f mapControl=%.2f (strategy=%s frame=%d phase=%s)",
        weights.economy(), weights.army(), weights.tech(), weights.bases(), weights.mapControl(),
        strategy.id(), state.gameFrame(), ctx.currentPhase());
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -Dtest=MultiFactorDominanceAssessorTest -q`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: PASS — all callers updated

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#228): implement mapControlFactor in MultiFactorDominanceAssessor"
```

---

### Task 4: MapControl calibration test

**Files:**
- Create: `src/test/java/io/quarkmind/agent/MapControlCalibrationTest.java`

**Interfaces:**
- Consumes: `ExpansionLocation.fromResources()` — existing API
- Consumes: `MultiFactorDominanceAssessor.EXPANSION_CONTROL_RADIUS` — from Task 3
- Consumes: Replay datasets (IEM10 + AI Arena) via `ReplayValidationHarness`

- [ ] **Step 1: Write calibration test**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.mock.SimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("benchmark")
class MapControlCalibrationTest {

    @Test
    void expansionControlRadius_basesWithinThreshold() {
        // Validate that EXPANSION_CONTROL_RADIUS=10.0 correctly matches
        // base buildings to expansion locations across replay datasets.
        //
        // For each replay: extract resources → compute expansion locations →
        // find base buildings → measure distance from each base to nearest
        // expansion centroid → assert all bases are within EXPANSION_CONTROL_RADIUS.
        //
        // This uses the same replay loading pattern as ExpansionLocationCalibrationTest.

        double radius = MultiFactorDominanceAssessor.EXPANSION_CONTROL_RADIUS;

        // Load all replay datasets and measure base-to-centroid distances
        var replayFiles = ReplayTestSupport.allReplayFiles();
        assertThat(replayFiles).isNotEmpty();

        int totalBases = 0;
        int matched = 0;

        for (var replayFile : replayFiles) {
            var game = ReplayTestSupport.loadReplay(replayFile);
            List<Resource> minerals = game.minerals();
            List<Resource> geysers = game.geysers();
            Point2d playerStart = game.playerStart();

            List<ExpansionLocation> expansions =
                ExpansionLocation.fromResources(minerals, geysers, playerStart);

            for (Building base : game.bases()) {
                totalBases++;
                boolean withinRadius = expansions.stream()
                    .anyMatch(exp -> base.position().distanceTo(exp.position()) <= radius);
                if (withinRadius) matched++;
            }
        }

        System.out.printf("MapControl calibration: %d/%d bases within %.1f of expansion centroid (%.1f%%)%n",
            matched, totalBases, radius, 100.0 * matched / totalBases);

        // At least 90% of bases should match — some edge cases (proxy bases,
        // misplaced expansions) are expected outliers
        assertThat(matched).as("bases within EXPANSION_CONTROL_RADIUS")
            .isGreaterThanOrEqualTo((int) (totalBases * 0.90));
    }
}
```

Note: The exact replay loading API (`ReplayTestSupport`) should be modeled on the existing `ExpansionLocationCalibrationTest` pattern — read that test for the precise helper methods available.

- [ ] **Step 2: Run calibration test**

Run: `mvn test -Pbenchmark -Dtest=MapControlCalibrationTest -q`
Expected: PASS — 10.0 is wide enough for standard base placement

- [ ] **Step 3: Commit**

```bash
git add src/test/java/io/quarkmind/agent/MapControlCalibrationTest.java
git commit -m "test(#228): add MapControlCalibrationTest — validate EXPANSION_CONTROL_RADIUS against replays"
```

---

### Task 5: Update CLAUDE.md and config documentation

**Files:**
- Modify: `CLAUDE.md` — add `MapControlCalibrationTest` to unit test listing
- Modify: `src/main/resources/application.properties` — already done in Task 2, verify

**Interfaces:** None — documentation only

- [ ] **Step 1: Update CLAUDE.md test listing**

Add `MapControlCalibrationTest` to the unit tests list (alphabetical position near `MilestoneSessionTest`).

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#228): add MapControlCalibrationTest to CLAUDE.md test listing Refs #228"
```
