# Phase-Adaptive Dominance Weights Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #227 — feat: game-phase-adaptive dominance weights — shift factor weights by game phase
**Issue group:** #227

**Goal:** Replace fixed dominance weights with an SPI that resolves weights per-tick based on game frame and situational phase, with two concrete strategies (temporal interpolation, situational modulation).

**Architecture:** `DominanceWeightStrategy` SPI with `id()`-based config selection. `AnchorInterpolator` provides shared anchor-point linear interpolation. `MultiFactorDominanceAssessor` delegates weight resolution to the active strategy, subscribing to the phase bus for situational context. Config adds `weight-strategy` selector and `anchors[]` list to `MilestoneConfig.Dominance`.

**Tech Stack:** Java 21, Quarkus (SmallRye Config `@ConfigMapping`), CDI (`@ApplicationScoped`, `Instance<>`), AssertJ

## Global Constraints

- Domain types in `domain/` — no CDI, no Quarkus imports
- SPI and strategies in `agent/` — CDI beans where needed
- `DominanceWeights` sum-to-1.0 invariant (tolerance 0.001)
- Anchor frames strictly ascending (`<`, not `≤`) — division by zero otherwise
- Anchor list minimum cardinality: 1
- Phase subscription: lazy double-checked locking (same pattern as `GamePhaseTrigger`)
- `volatile` on `cachedPhase` field
- Test constructor pattern: no CDI, no bus subscription — pass strategy directly
- Factor computation methods unchanged — only weighting changes

---

### Task 1: Domain Types, SPI Interface, Config Additions

**Files:**
- Create: `src/main/java/io/quarkmind/domain/DominanceWeights.java`
- Create: `src/main/java/io/quarkmind/agent/WeightContext.java`
- Create: `src/main/java/io/quarkmind/agent/DominanceWeightStrategy.java`
- Modify: `src/main/java/io/quarkmind/agent/MilestoneConfig.java` — add `weightStrategy()`, `anchors()`, `WeightAnchor` (keep old weight fields)
- Modify: `src/main/resources/application.properties` — add strategy + anchors config (keep old weight lines)
- Test: `src/test/java/io/quarkmind/domain/DominanceWeightsTest.java`

**Interfaces:**
- Produces: `DominanceWeights(double economy, double army, double tech, double bases)` — used by all subsequent tasks
- Produces: `WeightContext(long gameFrame, String currentPhase)` — used by strategies
- Produces: `DominanceWeightStrategy { String id(); DominanceWeights resolve(WeightContext context); }` — used by strategies and assessor
- Produces: `MilestoneConfig.Dominance.WeightAnchor { long frame(); double economyWeight(); double armyWeight(); double techWeight(); double basesWeight(); }` — used by strategies
- Produces: `MilestoneConfig.Dominance.weightStrategy()` — used by assessor
- Produces: `MilestoneConfig.Dominance.anchors()` — used by strategies

- [ ] **Step 1: Write `DominanceWeightsTest`**

```java
package io.quarkmind.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class DominanceWeightsTest {

    @Test
    void validWeights_accepted() {
        var w = new DominanceWeights(0.30, 0.35, 0.20, 0.15);
        assertThat(w.economy()).isEqualTo(0.30);
        assertThat(w.army()).isEqualTo(0.35);
        assertThat(w.tech()).isEqualTo(0.20);
        assertThat(w.bases()).isEqualTo(0.15);
    }

    @Test
    void weightsSumTooHigh_rejected() {
        assertThatThrownBy(() -> new DominanceWeights(0.50, 0.50, 0.20, 0.15))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("sum to 1.0");
    }

    @Test
    void weightsSumTooLow_rejected() {
        assertThatThrownBy(() -> new DominanceWeights(0.10, 0.10, 0.10, 0.10))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("sum to 1.0");
    }

    @Test
    void equalQuarters_accepted() {
        var w = new DominanceWeights(0.25, 0.25, 0.25, 0.25);
        assertThat(w.economy()).isEqualTo(0.25);
    }

    @Test
    void allZeros_rejected() {
        assertThatThrownBy(() -> new DominanceWeights(0.0, 0.0, 0.0, 0.0))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void withinTolerance_accepted() {
        // 0.001 tolerance — 0.9995 rounds within
        assertThatCode(() -> new DominanceWeights(0.2499, 0.2501, 0.25, 0.25))
            .doesNotThrowAnyException();
    }

    @Test
    void beyondTolerance_rejected() {
        assertThatThrownBy(() -> new DominanceWeights(0.30, 0.35, 0.20, 0.16))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test — verify FAIL (class not found)**

Run: `mvn test -Dtest=DominanceWeightsTest -q`

- [ ] **Step 3: Create `DominanceWeights.java`**

Use `ide_create_file`:

```java
package io.quarkmind.domain;

public record DominanceWeights(double economy, double army, double tech, double bases) {
    public DominanceWeights {
        double sum = economy + army + tech + bases;
        if (Math.abs(sum - 1.0) > 0.001) {
            throw new IllegalArgumentException("Weights must sum to 1.0, got " + sum);
        }
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=DominanceWeightsTest -q`

- [ ] **Step 5: Create `WeightContext.java`**

Use `ide_create_file`:

```java
package io.quarkmind.agent;

public record WeightContext(long gameFrame, String currentPhase) {}
```

- [ ] **Step 6: Create `DominanceWeightStrategy.java`**

Use `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;

public interface DominanceWeightStrategy {
    String id();
    DominanceWeights resolve(WeightContext context);
}
```

- [ ] **Step 7: Add `WeightAnchor`, `weightStrategy()`, `anchors()` to `MilestoneConfig.Dominance`**

Use `ide_insert_member` to add to the `Dominance` interface inside `MilestoneConfig.java`, after `minEnemyVisibility()`:

```java
@WithName("weight-strategy")
@WithDefault("temporal")
String weightStrategy();

List<WeightAnchor> anchors();

interface WeightAnchor {
    long frame();
    @WithName("economy-weight") double economyWeight();
    @WithName("army-weight") double armyWeight();
    @WithName("tech-weight") double techWeight();
    @WithName("bases-weight") double basesWeight();
}
```

- [ ] **Step 8: Add anchor config to `application.properties`**

Add after the existing `min-enemy-visibility` line (keep old weight lines — removed in Task 5):

```properties
quarkmind.milestones.dominance.weight-strategy=temporal
quarkmind.milestones.dominance.anchors[0].frame=0
quarkmind.milestones.dominance.anchors[0].economy-weight=0.30
quarkmind.milestones.dominance.anchors[0].army-weight=0.35
quarkmind.milestones.dominance.anchors[0].tech-weight=0.20
quarkmind.milestones.dominance.anchors[0].bases-weight=0.15
```

- [ ] **Step 9: Verify build**

Run: `mvn compile -q`

- [ ] **Step 10: Commit**

```
feat(#227): domain types, SPI interface, config additions for phase-adaptive dominance

DominanceWeights record with sum-to-1.0 invariant, WeightContext signal
bundle, DominanceWeightStrategy SPI, MilestoneConfig.Dominance anchor
config. Refs #227
```

---

### Task 2: AnchorInterpolator

**Files:**
- Create: `src/main/java/io/quarkmind/agent/AnchorInterpolator.java`
- Test: `src/test/java/io/quarkmind/agent/AnchorInterpolatorTest.java`

**Interfaces:**
- Consumes: `DominanceWeights(double, double, double, double)` from Task 1
- Consumes: `MilestoneConfig.Dominance.WeightAnchor` from Task 1
- Produces: `AnchorInterpolator(List<MilestoneConfig.Dominance.WeightAnchor>)` — constructor validates ≥ 1 anchor, strictly ascending frames
- Produces: `AnchorInterpolator.interpolate(long gameFrame) → DominanceWeights` — used by both strategy implementations

- [ ] **Step 1: Write `AnchorInterpolatorTest`**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;
import static org.assertj.core.data.Offset.offset;

class AnchorInterpolatorTest {

    // --- validation ---

    @Test
    void emptyAnchorList_rejected() {
        assertThatThrownBy(() -> new AnchorInterpolator(List.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("at least one");
    }

    @Test
    void duplicateFrames_rejected() {
        assertThatThrownBy(() -> new AnchorInterpolator(List.of(
                anchor(100, 0.30, 0.35, 0.20, 0.15),
                anchor(100, 0.25, 0.25, 0.25, 0.25))))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("strictly ascending");
    }

    @Test
    void descendingFrames_rejected() {
        assertThatThrownBy(() -> new AnchorInterpolator(List.of(
                anchor(200, 0.30, 0.35, 0.20, 0.15),
                anchor(100, 0.25, 0.25, 0.25, 0.25))))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("strictly ascending");
    }

    // --- single anchor ---

    @Test
    void singleAnchor_alwaysReturnsSameWeights() {
        var interp = new AnchorInterpolator(List.of(
            anchor(0, 0.30, 0.35, 0.20, 0.15)));
        DominanceWeights w = interp.interpolate(5000);
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.35, offset(0.001));
        assertThat(w.tech()).isCloseTo(0.20, offset(0.001));
        assertThat(w.bases()).isCloseTo(0.15, offset(0.001));
    }

    // --- two anchors ---

    @Test
    void beforeFirstAnchor_returnsFirstWeights() {
        var interp = new AnchorInterpolator(List.of(
            anchor(1000, 0.40, 0.20, 0.25, 0.15),
            anchor(2000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = interp.interpolate(500);
        assertThat(w.economy()).isCloseTo(0.40, offset(0.001));
        assertThat(w.army()).isCloseTo(0.20, offset(0.001));
    }

    @Test
    void afterLastAnchor_returnsLastWeights() {
        var interp = new AnchorInterpolator(List.of(
            anchor(1000, 0.40, 0.20, 0.25, 0.15),
            anchor(2000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = interp.interpolate(5000);
        assertThat(w.economy()).isCloseTo(0.20, offset(0.001));
        assertThat(w.army()).isCloseTo(0.40, offset(0.001));
    }

    @Test
    void atExactAnchorFrame_returnsAnchorWeights() {
        var interp = new AnchorInterpolator(List.of(
            anchor(1000, 0.40, 0.20, 0.25, 0.15),
            anchor(2000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = interp.interpolate(1000);
        assertThat(w.economy()).isCloseTo(0.40, offset(0.001));
    }

    @Test
    void midpointBetweenAnchors_interpolatesLinearly() {
        var interp = new AnchorInterpolator(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = interp.interpolate(5000);
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.30, offset(0.001));
        assertThat(w.tech()).isCloseTo(0.25, offset(0.001));
        assertThat(w.bases()).isCloseTo(0.15, offset(0.001));
    }

    @Test
    void quarterPointBetweenAnchors_interpolatesLinearly() {
        var interp = new AnchorInterpolator(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = interp.interpolate(2500);
        assertThat(w.economy()).isCloseTo(0.35, offset(0.001));
        assertThat(w.army()).isCloseTo(0.25, offset(0.001));
    }

    // --- three anchors ---

    @Test
    void threeAnchors_interpolatesBetweenCorrectPair() {
        var interp = new AnchorInterpolator(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(8064, 0.30, 0.35, 0.20, 0.15),
            anchor(16128, 0.15, 0.50, 0.15, 0.20)));
        // Frame 12096 is midpoint between anchor[1] (8064) and anchor[2] (16128)
        DominanceWeights w = interp.interpolate(12096);
        assertThat(w.economy()).isCloseTo(0.225, offset(0.001));
        assertThat(w.army()).isCloseTo(0.425, offset(0.001));
    }

    @Test
    void interpolatedWeights_sumToOne() {
        var interp = new AnchorInterpolator(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.15, 0.50, 0.15, 0.20)));
        DominanceWeights w = interp.interpolate(3333);
        double sum = w.economy() + w.army() + w.tech() + w.bases();
        assertThat(sum).isCloseTo(1.0, offset(0.001));
    }

    // --- helpers ---

    private static MilestoneConfig.Dominance.WeightAnchor anchor(
            long frame, double economy, double army, double tech, double bases) {
        return new MilestoneConfig.Dominance.WeightAnchor() {
            @Override public long frame() { return frame; }
            @Override public double economyWeight() { return economy; }
            @Override public double armyWeight() { return army; }
            @Override public double techWeight() { return tech; }
            @Override public double basesWeight() { return bases; }
        };
    }
}
```

- [ ] **Step 2: Run test — verify FAIL (class not found)**

Run: `mvn test -Dtest=AnchorInterpolatorTest -q`

- [ ] **Step 3: Create `AnchorInterpolator.java`**

Use `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;

import java.util.List;

class AnchorInterpolator {

    private final List<MilestoneConfig.Dominance.WeightAnchor> anchors;

    AnchorInterpolator(List<MilestoneConfig.Dominance.WeightAnchor> anchors) {
        if (anchors.isEmpty()) {
            throw new IllegalArgumentException("Anchors must contain at least one entry");
        }
        for (int i = 1; i < anchors.size(); i++) {
            if (anchors.get(i).frame() <= anchors.get(i - 1).frame()) {
                throw new IllegalArgumentException(
                    "Anchor frames must be strictly ascending: frame[" + (i - 1) + "]="
                    + anchors.get(i - 1).frame() + " >= frame[" + i + "]=" + anchors.get(i).frame());
            }
        }
        this.anchors = List.copyOf(anchors);
    }

    DominanceWeights interpolate(long gameFrame) {
        if (anchors.size() == 1 || gameFrame <= anchors.get(0).frame()) {
            return toWeights(anchors.get(0));
        }
        if (gameFrame >= anchors.get(anchors.size() - 1).frame()) {
            return toWeights(anchors.get(anchors.size() - 1));
        }
        for (int i = 0; i < anchors.size() - 1; i++) {
            var lo = anchors.get(i);
            var hi = anchors.get(i + 1);
            if (gameFrame >= lo.frame() && gameFrame <= hi.frame()) {
                double t = (double) (gameFrame - lo.frame()) / (hi.frame() - lo.frame());
                return new DominanceWeights(
                    lerp(lo.economyWeight(), hi.economyWeight(), t),
                    lerp(lo.armyWeight(), hi.armyWeight(), t),
                    lerp(lo.techWeight(), hi.techWeight(), t),
                    lerp(lo.basesWeight(), hi.basesWeight(), t));
            }
        }
        return toWeights(anchors.get(anchors.size() - 1));
    }

    private static DominanceWeights toWeights(MilestoneConfig.Dominance.WeightAnchor a) {
        return new DominanceWeights(a.economyWeight(), a.armyWeight(), a.techWeight(), a.basesWeight());
    }

    private static double lerp(double a, double b, double t) {
        return a + (b - a) * t;
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=AnchorInterpolatorTest -q`

- [ ] **Step 5: Commit**

```
feat(#227): AnchorInterpolator — shared anchor-point linear interpolation

Package-private helper with validation (≥ 1 anchor, strictly ascending
frames) and linear interpolation per weight component. Refs #227
```

---

### Task 3: TemporalDominanceWeightStrategy

**Files:**
- Create: `src/main/java/io/quarkmind/agent/TemporalDominanceWeightStrategy.java`
- Test: `src/test/java/io/quarkmind/agent/TemporalDominanceWeightStrategyTest.java`

**Interfaces:**
- Consumes: `DominanceWeightStrategy` from Task 1
- Consumes: `DominanceWeights` from Task 1
- Consumes: `WeightContext` from Task 1
- Consumes: `AnchorInterpolator` from Task 2
- Consumes: `MilestoneConfig.Dominance.anchors()`, `MilestoneConfig.Dominance.WeightAnchor` from Task 1
- Produces: `TemporalDominanceWeightStrategy` CDI bean — `id()` returns `"temporal"`, `resolve()` delegates to `AnchorInterpolator`

- [ ] **Step 1: Write `TemporalDominanceWeightStrategyTest`**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.data.Offset.offset;

class TemporalDominanceWeightStrategyTest {

    @Test
    void id_returnsTemporal() {
        var strategy = new TemporalDominanceWeightStrategy(List.of(
            anchor(0, 0.30, 0.35, 0.20, 0.15)));
        assertThat(strategy.id()).isEqualTo("temporal");
    }

    @Test
    void resolve_delegatesToInterpolator() {
        var strategy = new TemporalDominanceWeightStrategy(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights w = strategy.resolve(new WeightContext(5000, null));
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.30, offset(0.001));
    }

    @Test
    void resolve_ignoresPhase() {
        var strategy = new TemporalDominanceWeightStrategy(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.20, 0.40, 0.25, 0.15)));
        DominanceWeights withPhase = strategy.resolve(new WeightContext(5000, "DEFENSIVE_HOLD"));
        DominanceWeights noPhase = strategy.resolve(new WeightContext(5000, null));
        assertThat(withPhase).isEqualTo(noPhase);
    }

    private static MilestoneConfig.Dominance.WeightAnchor anchor(
            long frame, double economy, double army, double tech, double bases) {
        return new MilestoneConfig.Dominance.WeightAnchor() {
            @Override public long frame() { return frame; }
            @Override public double economyWeight() { return economy; }
            @Override public double armyWeight() { return army; }
            @Override public double techWeight() { return tech; }
            @Override public double basesWeight() { return bases; }
        };
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -Dtest=TemporalDominanceWeightStrategyTest -q`

- [ ] **Step 3: Create `TemporalDominanceWeightStrategy.java`**

Use `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;

@ApplicationScoped
public class TemporalDominanceWeightStrategy implements DominanceWeightStrategy {

    private final AnchorInterpolator interpolator;

    @Inject
    TemporalDominanceWeightStrategy(MilestoneConfig config) {
        this(config.dominance().anchors());
    }

    TemporalDominanceWeightStrategy(List<MilestoneConfig.Dominance.WeightAnchor> anchors) {
        this.interpolator = new AnchorInterpolator(anchors);
    }

    @Override
    public String id() {
        return "temporal";
    }

    @Override
    public DominanceWeights resolve(WeightContext context) {
        return interpolator.interpolate(context.gameFrame());
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=TemporalDominanceWeightStrategyTest -q`

- [ ] **Step 5: Commit**

```
feat(#227): TemporalDominanceWeightStrategy — anchor-point interpolation

CDI bean that resolves dominance weights by linearly interpolating
between configured anchor points based on game frame. Refs #227
```

---

### Task 4: SituationalDominanceWeightStrategy

**Files:**
- Create: `src/main/java/io/quarkmind/agent/SituationalDominanceWeightStrategy.java`
- Test: `src/test/java/io/quarkmind/agent/SituationalDominanceWeightStrategyTest.java`

**Interfaces:**
- Consumes: `DominanceWeightStrategy` from Task 1
- Consumes: `DominanceWeights` from Task 1
- Consumes: `WeightContext` from Task 1
- Consumes: `AnchorInterpolator` from Task 2
- Consumes: `MilestoneConfig.Dominance.anchors()` from Task 1
- Produces: `SituationalDominanceWeightStrategy` CDI bean — `id()` returns `"situational"`, `resolve()` applies phase modifiers over temporal baseline

- [ ] **Step 1: Write `SituationalDominanceWeightStrategyTest`**

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.data.Offset.offset;

class SituationalDominanceWeightStrategyTest {

    private final SituationalDominanceWeightStrategy strategy =
        new SituationalDominanceWeightStrategy(List.of(
            anchor(0, 0.30, 0.35, 0.20, 0.15)));

    @Test
    void id_returnsSituational() {
        assertThat(strategy.id()).isEqualTo("situational");
    }

    @Test
    void nullPhase_returnsTemporalBaseline() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, null));
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.35, offset(0.001));
        assertThat(w.tech()).isCloseTo(0.20, offset(0.001));
        assertThat(w.bases()).isCloseTo(0.15, offset(0.001));
    }

    @Test
    void transitioning_noModifier() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "TRANSITIONING"));
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.35, offset(0.001));
    }

    @Test
    void defensiveHold_spikesArmyWeight() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "DEFENSIVE_HOLD"));
        assertThat(w.army()).isGreaterThan(0.35);
        assertThat(w.economy()).isLessThan(0.30);
    }

    @Test
    void earlyMacro_boostsEconomyWeight() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "EARLY_MACRO"));
        assertThat(w.economy()).isGreaterThan(0.30);
        assertThat(w.army()).isLessThan(0.35);
    }

    @Test
    void earlyAggression_boostsArmyWeight() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "EARLY_AGGRESSION"));
        assertThat(w.army()).isGreaterThan(0.35);
    }

    @Test
    void midSkirmish_boostsArmyWeight() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "MID_SKIRMISH"));
        assertThat(w.army()).isGreaterThan(0.35);
    }

    @Test
    void allPhases_sumToOne() {
        for (String phase : List.of("DEFENSIVE_HOLD", "EARLY_AGGRESSION", "EARLY_MACRO",
                "MID_SKIRMISH", "TRANSITIONING")) {
            DominanceWeights w = strategy.resolve(new WeightContext(5000, phase));
            double sum = w.economy() + w.army() + w.tech() + w.bases();
            assertThat(sum).as("phase=" + phase).isCloseTo(1.0, offset(0.001));
        }
    }

    @Test
    void allWeights_atLeastFloor() {
        for (String phase : List.of("DEFENSIVE_HOLD", "EARLY_AGGRESSION", "EARLY_MACRO",
                "MID_SKIRMISH", "TRANSITIONING")) {
            DominanceWeights w = strategy.resolve(new WeightContext(5000, phase));
            assertThat(w.economy()).as("economy phase=" + phase).isGreaterThanOrEqualTo(0.05);
            assertThat(w.army()).as("army phase=" + phase).isGreaterThanOrEqualTo(0.05);
            assertThat(w.tech()).as("tech phase=" + phase).isGreaterThanOrEqualTo(0.05);
            assertThat(w.bases()).as("bases phase=" + phase).isGreaterThanOrEqualTo(0.05);
        }
    }

    @Test
    void unknownPhase_treatedAsNoModifier() {
        DominanceWeights w = strategy.resolve(new WeightContext(5000, "UNKNOWN_PHASE"));
        assertThat(w.economy()).isCloseTo(0.30, offset(0.001));
        assertThat(w.army()).isCloseTo(0.35, offset(0.001));
    }

    @Test
    void phaseModifier_composesWithInterpolation() {
        var multiAnchor = new SituationalDominanceWeightStrategy(List.of(
            anchor(0, 0.40, 0.20, 0.25, 0.15),
            anchor(10000, 0.20, 0.40, 0.25, 0.15)));
        // At frame 5000: baseline is interpolated (0.30, 0.30, 0.25, 0.15)
        // DEFENSIVE_HOLD modifies and re-normalises
        DominanceWeights w = multiAnchor.resolve(new WeightContext(5000, "DEFENSIVE_HOLD"));
        assertThat(w.army()).isGreaterThan(0.30);
        double sum = w.economy() + w.army() + w.tech() + w.bases();
        assertThat(sum).isCloseTo(1.0, offset(0.001));
    }

    private static MilestoneConfig.Dominance.WeightAnchor anchor(
            long frame, double economy, double army, double tech, double bases) {
        return new MilestoneConfig.Dominance.WeightAnchor() {
            @Override public long frame() { return frame; }
            @Override public double economyWeight() { return economy; }
            @Override public double armyWeight() { return army; }
            @Override public double techWeight() { return tech; }
            @Override public double basesWeight() { return bases; }
        };
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

Run: `mvn test -Dtest=SituationalDominanceWeightStrategyTest -q`

- [ ] **Step 3: Create `SituationalDominanceWeightStrategy.java`**

Use `ide_create_file`:

```java
package io.quarkmind.agent;

import io.quarkmind.domain.DominanceWeights;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;

@ApplicationScoped
public class SituationalDominanceWeightStrategy implements DominanceWeightStrategy {

    private static final Logger log = Logger.getLogger(SituationalDominanceWeightStrategy.class);
    private static final double FLOOR = 0.05;

    private static final Map<String, double[]> PHASE_MODIFIERS = Map.of(
        "DEFENSIVE_HOLD",    new double[]{-0.10, +0.15, -0.05,  0.00},
        "EARLY_AGGRESSION",  new double[]{-0.05, +0.10, -0.05,  0.00},
        "EARLY_MACRO",       new double[]{+0.10, -0.10, +0.05, -0.05},
        "MID_SKIRMISH",      new double[]{-0.05, +0.10, -0.05,  0.00},
        "TRANSITIONING",     new double[]{ 0.00,  0.00,  0.00,  0.00}
    );

    private final AnchorInterpolator interpolator;

    @Inject
    SituationalDominanceWeightStrategy(MilestoneConfig config) {
        this(config.dominance().anchors());
    }

    SituationalDominanceWeightStrategy(List<MilestoneConfig.Dominance.WeightAnchor> anchors) {
        this.interpolator = new AnchorInterpolator(anchors);
    }

    @Override
    public String id() {
        return "situational";
    }

    @Override
    public DominanceWeights resolve(WeightContext context) {
        DominanceWeights baseline = interpolator.interpolate(context.gameFrame());
        double[] mod = context.currentPhase() != null
            ? PHASE_MODIFIERS.getOrDefault(context.currentPhase(), new double[4])
            : new double[4];

        double economy = Math.max(FLOOR, baseline.economy() + mod[0]);
        double army    = Math.max(FLOOR, baseline.army()    + mod[1]);
        double tech    = Math.max(FLOOR, baseline.tech()    + mod[2]);
        double bases   = Math.max(FLOOR, baseline.bases()   + mod[3]);

        if (economy != baseline.economy() + mod[0] || army != baseline.army() + mod[1]
                || tech != baseline.tech() + mod[2] || bases != baseline.bases() + mod[3]) {
            log.debugf("[DOMINANCE] Clamping fired: phase=%s", context.currentPhase());
        }

        double sum = economy + army + tech + bases;
        return new DominanceWeights(economy / sum, army / sum, tech / sum, bases / sum);
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

Run: `mvn test -Dtest=SituationalDominanceWeightStrategyTest -q`

- [ ] **Step 5: Commit**

```
feat(#227): SituationalDominanceWeightStrategy — temporal + phase modifiers

CDI bean that applies additive phase modifiers over temporal baseline,
re-normalises to sum-to-1.0, clamps to 0.05 floor. Refs #227
```

---

### Task 5: Assessor Migration — Config, Assessor, Properties, Tests

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/MilestoneConfig.java` — remove old flat weight fields from `Dominance`
- Modify: `src/main/java/io/quarkmind/agent/MultiFactorDominanceAssessor.java` — replace weight fields with strategy delegation, add phase subscription, game-reset, logging
- Modify: `src/main/resources/application.properties` — remove old flat weight lines
- Modify: `src/test/java/io/quarkmind/agent/MultiFactorDominanceAssessorTest.java` — adapt to new constructor, add strategy tests

**Interfaces:**
- Consumes: `DominanceWeightStrategy` from Task 1
- Consumes: `DominanceWeights` from Task 1
- Consumes: `WeightContext` from Task 1
- Consumes: `TemporalDominanceWeightStrategy` from Task 3
- Consumes: `SituationalDominanceWeightStrategy` from Task 4
- Consumes: `MilestoneConfig.Dominance.weightStrategy()`, `MilestoneConfig.Dominance.anchors()` from Task 1

- [ ] **Step 1: Remove old flat weight fields from `MilestoneConfig.Dominance`**

Use `ide_edit_member` on `MilestoneConfig.java` to replace the `Dominance` interface. Remove `economyWeight()`, `armyWeight()`, `techWeight()`, `basesWeight()`. Keep `maxExpected*`, `minEnemyVisibility`, and the new `weightStrategy()`, `anchors()`, `WeightAnchor` from Task 1.

The full `Dominance` interface after edit:

```java
interface Dominance {
    @WithName("weight-strategy")
    @WithDefault("temporal")
    String weightStrategy();

    List<WeightAnchor> anchors();

    @WithName("max-expected-economy-delta")
    @WithDefault("25.0")
    double maxExpectedEconomyDelta();
    @WithName("max-expected-army-delta")
    @WithDefault("3000")
    int maxExpectedArmyDelta();
    @WithName("max-expected-tech-delta")
    @WithDefault("2.0")
    double maxExpectedTechDelta();
    @WithName("max-expected-base-delta")
    @WithDefault("3")
    int maxExpectedBaseDelta();

    @WithName("min-enemy-visibility")
    @WithDefault("3")
    int minEnemyVisibility();

    interface WeightAnchor {
        long frame();
        @WithName("economy-weight") double economyWeight();
        @WithName("army-weight") double armyWeight();
        @WithName("tech-weight") double techWeight();
        @WithName("bases-weight") double basesWeight();
    }
}
```

- [ ] **Step 2: Remove old flat weight lines from `application.properties`**

Remove these four lines:
```
quarkmind.milestones.dominance.economy-weight=0.30
quarkmind.milestones.dominance.army-weight=0.35
quarkmind.milestones.dominance.tech-weight=0.20
quarkmind.milestones.dominance.bases-weight=0.15
```

The strategy + anchors config added in Task 1 remains.

- [ ] **Step 3: Rewrite `MultiFactorDominanceAssessor`**

Use `ide_edit_member` to replace the full class. The factor computation methods stay identical. Changes:
- Remove 4 weight fields, add `DominanceWeightStrategy strategy`
- Add `volatile GamePhase cachedPhase`, lazy phase subscription, game-reset handler
- CDI constructor: resolve strategy from `Instance<DominanceWeightStrategy>` + config
- Test constructor: takes `DominanceWeightStrategy` directly
- `assess()`: build `WeightContext`, call `strategy.resolve()`, use returned weights
- DEBUG log when weights shift

```java
@ApplicationScoped
public class MultiFactorDominanceAssessor implements DominanceAssessor {

    private static final Logger log = Logger.getLogger(MultiFactorDominanceAssessor.class);

    private final DominanceWeightStrategy strategy;
    private final double maxExpectedEconomyDelta;
    private final int maxExpectedArmyDelta;
    private final double maxExpectedTechDelta;
    private final int maxExpectedBaseDelta;
    private final int minEnemyVisibility;

    private final Instance<SummarisationLifecycle> lazyLifecycle;
    private volatile GamePhase cachedPhase;
    private volatile boolean subscribed = false;
    private DominanceWeights lastWeights;

    @Inject
    MultiFactorDominanceAssessor(
            @Any Instance<DominanceWeightStrategy> strategies,
            Instance<SummarisationLifecycle> summarisationLifecycle,
            MilestoneConfig config) {
        String selectedId = config.dominance().weightStrategy();
        this.strategy = strategies.stream()
            .filter(s -> s.id().equals(selectedId))
            .reduce((a, b) -> { throw new IllegalStateException(
                "Duplicate DominanceWeightStrategy id: " + selectedId); })
            .orElseThrow(() -> new IllegalStateException(
                "No DominanceWeightStrategy with id '" + selectedId + "'"));
        this.lazyLifecycle = summarisationLifecycle;
        this.maxExpectedEconomyDelta = config.dominance().maxExpectedEconomyDelta();
        this.maxExpectedArmyDelta = config.dominance().maxExpectedArmyDelta();
        this.maxExpectedTechDelta = config.dominance().maxExpectedTechDelta();
        this.maxExpectedBaseDelta = config.dominance().maxExpectedBaseDelta();
        this.minEnemyVisibility = config.dominance().minEnemyVisibility();
    }

    MultiFactorDominanceAssessor(
            DominanceWeightStrategy strategy,
            double maxExpectedEconomyDelta, int maxExpectedArmyDelta,
            double maxExpectedTechDelta, int maxExpectedBaseDelta,
            int minEnemyVisibility) {
        this.strategy = strategy;
        this.lazyLifecycle = null;
        this.subscribed = true;  // no bus in test mode
        this.maxExpectedEconomyDelta = maxExpectedEconomyDelta;
        this.maxExpectedArmyDelta = maxExpectedArmyDelta;
        this.maxExpectedTechDelta = maxExpectedTechDelta;
        this.maxExpectedBaseDelta = maxExpectedBaseDelta;
        this.minEnemyVisibility = minEnemyVisibility;
    }

    private void ensureSubscribed() {
        if (!subscribed) {
            synchronized (this) {
                if (!subscribed) {
                    lazyLifecycle.get().phaseBus().subscribe(
                        p -> true, e -> cachedPhase = e.payload());
                    subscribed = true;
                }
            }
        }
    }

    void onGameStarted(@Observes GameStarted event) {
        cachedPhase = null;
    }

    // assess(), economyFactor(), armyFactor(), techFactor(), basesFactor(), techScore(), clamp() ...
    // assess() changes to use strategy.resolve() — see step description
}
```

The `assess()` method body becomes:

```java
@Override
public DominanceScore assess(GameState state) {
    ensureSubscribed();
    int totalEnemyVisible = state.enemyUnits().size() + state.enemyBuildings().size();
    if (totalEnemyVisible < minEnemyVisibility) {
        return NEUTRAL;
    }

    double economy = economyFactor(state);
    double army = armyFactor(state);
    double tech = techFactor(state);
    double bases = basesFactor(state);

    GamePhase phase = cachedPhase;
    WeightContext ctx = new WeightContext(state.gameFrame(),
        phase != null ? phase.phase() : null);
    DominanceWeights weights = strategy.resolve(ctx);

    if (lastWeights != null && (
            Math.abs(weights.economy() - lastWeights.economy()) > 0.01
            || Math.abs(weights.army() - lastWeights.army()) > 0.01
            || Math.abs(weights.tech() - lastWeights.tech()) > 0.01
            || Math.abs(weights.bases() - lastWeights.bases()) > 0.01)) {
        log.debugf("[DOMINANCE] Weights shifted: economy=%.2f army=%.2f tech=%.2f bases=%.2f (strategy=%s frame=%d phase=%s)",
            weights.economy(), weights.army(), weights.tech(), weights.bases(),
            strategy.id(), state.gameFrame(), ctx.currentPhase());
    }
    lastWeights = weights;

    double overall = clamp(economy * weights.economy() + army * weights.army()
        + tech * weights.tech() + bases * weights.bases());

    Map<String, Double> factors = new LinkedHashMap<>(4);
    factors.put("economy", economy);
    factors.put("army", army);
    factors.put("tech", tech);
    factors.put("bases", bases);

    return new DominanceScore(overall, factors);
}
```

All other methods (`economyFactor`, `armyFactor`, `techFactor`, `basesFactor`, `techScore`, `clamp`) stay identical.

- [ ] **Step 4: Adapt `MultiFactorDominanceAssessorTest`**

Replace the field-level `assessor` construction. The old constructor took 9 args (4 weights + 4 max deltas + visibility). The new test constructor takes `DominanceWeightStrategy` + 4 max deltas + visibility.

Create a single-anchor temporal strategy matching the old fixed weights:

```java
private static final DominanceWeightStrategy FIXED_WEIGHTS =
    new TemporalDominanceWeightStrategy(List.of(
        new MilestoneConfig.Dominance.WeightAnchor() {
            @Override public long frame() { return 0; }
            @Override public double economyWeight() { return 0.30; }
            @Override public double armyWeight() { return 0.35; }
            @Override public double techWeight() { return 0.20; }
            @Override public double basesWeight() { return 0.15; }
        }));

private final MultiFactorDominanceAssessor assessor = new MultiFactorDominanceAssessor(
    FIXED_WEIGHTS, 25.0, 3000, 2.0, 3, 3);
```

Update `assess_overallIsWeightedSum` to get weights from the strategy rather than hardcoded values:

```java
DominanceWeights w = FIXED_WEIGHTS.resolve(new WeightContext(state.gameFrame(), null));
double expectedOverall = score.factors().get("economy") * w.economy()
    + score.factors().get("army") * w.army()
    + score.factors().get("tech") * w.tech()
    + score.factors().get("bases") * w.bases();
```

Update `assess_overallClampedToOne` similarly.

Add new tests:

```java
@Test
void assess_weightsChangeWithGameFrame() {
    var multiAnchor = new TemporalDominanceWeightStrategy(List.of(
        anchor(0, 0.40, 0.20, 0.25, 0.15),
        anchor(10000, 0.20, 0.40, 0.25, 0.15)));
    var earlyAssessor = new MultiFactorDominanceAssessor(multiAnchor, 25.0, 3000, 2.0, 3, 3);
    GameState earlyState = gameState(200, 100, 15, 10,
        List.of(probe(), probe(), probe(), probe(), probe()), List.of(nexus()),
        List.of(probe(), probe(), zealot()), List.of(nexus()), 1000);
    GameState lateState = gameState(200, 100, 15, 10,
        List.of(probe(), probe(), probe(), probe(), probe()), List.of(nexus()),
        List.of(probe(), probe(), zealot()), List.of(nexus()), 9000);
    DominanceScore earlyScore = earlyAssessor.assess(earlyState);
    DominanceScore lateScore = earlyAssessor.assess(lateState);
    // Same factor values, different weights → different overall
    assertThat(earlyScore.overall()).isNotCloseTo(lateScore.overall(), offset(0.01));
}
```

Add CDI constructor error path tests:

```java
@Test
void cdiConstructor_noMatch_throws() {
    DominanceWeightStrategy wrong = new TemporalDominanceWeightStrategy(List.of(
        anchor(0, 0.25, 0.25, 0.25, 0.25)));
    // Simulate Instance with wrong id
    assertThatThrownBy(() -> java.util.stream.Stream.of(wrong)
        .filter(s -> s.id().equals("nonexistent"))
        .reduce((a, b) -> { throw new IllegalStateException("Duplicate"); })
        .orElseThrow(() -> new IllegalStateException("No DominanceWeightStrategy with id 'nonexistent'")))
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("nonexistent");
}
```

The `gameState` helper needs an overload accepting `gameFrame`:

```java
private static GameState gameState(int minerals, int vespene, int supply, int supplyUsed,
        List<Unit> myUnits, List<Building> myBuildings,
        List<Unit> enemyUnits, List<Building> enemyBuildings, long gameFrame) {
    return new GameState(minerals, vespene, supply, supplyUsed,
        myUnits, myBuildings, enemyUnits, enemyBuildings,
        List.of(), List.of(), List.of(), gameFrame);
}
```

- [ ] **Step 5: Run all tests**

Run: `mvn test -q`

Expected: all tests pass (existing + new)

- [ ] **Step 6: Verify with `ide_diagnostics`**

Check `MultiFactorDominanceAssessor.java` and `MilestoneConfig.java` for compilation errors.

- [ ] **Step 7: Commit**

```
feat(#227): migrate assessor to phase-adaptive weight strategy SPI

MultiFactorDominanceAssessor delegates weight resolution to the active
DominanceWeightStrategy. Subscribes to phase bus for situational context.
Fail-fast CDI strategy resolution at startup. Refs #227
```

- [ ] **Step 8: Run full build verification**

Run: `mvn test -q`

Verify: all tests pass, zero compilation warnings on modified files.

---

## Update CLAUDE.md

After Task 5, add `MultiFactorDominanceAssessorTest` note about the new test constructor pattern to the Testing Patterns section if it differs from existing conventions. The existing test list already includes the test class name — no new test class to register.
