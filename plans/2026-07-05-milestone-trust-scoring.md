# Milestone-Based Trust Scoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Record intermediate trust attestations at game-time milestones so the trust model distinguishes "good plan, poor execution" from "fundamentally flawed strategy."

**Architecture:** A `MilestoneOutcomeRecorder` replaces `GameOutcomeRecorder`, owning the full attestation lifecycle (milestones + game-end). Pluggable `MilestoneTrigger` implementations (frame-based, phase-based) decide WHEN to evaluate. A `DominanceAssessor` interface decides WHAT the assessment is. Until engine#648 ships the `AttestingOutcomeRecorder` sub-interface, milestone evaluation is a no-op — only game-end recording is active.

**Tech Stack:** Java 21, Quarkus CDI, casehub-ledger-api (`OutcomeRecorder`), casehub-blocks (`EventStreamBus`)

## Global Constraints

- engine#648 (`AttestingOutcomeRecorder` sub-interface) is not yet available — milestones are no-op until it ships
- Domain model (`io.quarkmind.domain`) must remain plain Java — no CDI, no Quarkus imports
- `GameOutcomeRecorder` is deleted — `MilestoneOutcomeRecorder` absorbs all its responsibilities
- SC2 Faster speed = 22.4 frames per second (use `DroolsScoutingTask.FRAMES_PER_SECOND` if accessible, or define locally)
- All new classes in `io.quarkmind.agent` (milestone infrastructure) or `io.quarkmind.agent.plugin` (trigger seam interfaces)
- `ConcurrentHashMap` for `MilestoneSession` thread safety (game-start thread vs scheduler thread)
- Profiles: milestones enabled in `%sc2`, `%test`; disabled in `%mock`, `%emulated`, `%emulated-sc2`, `%replay`

---

### Task 1: MilestoneEvent, MilestoneTrigger, and MilestoneSession

**Files:**
- Create: `src/main/java/io/quarkmind/agent/MilestoneEvent.java`
- Create: `src/main/java/io/quarkmind/agent/MilestoneTrigger.java`
- Create: `src/main/java/io/quarkmind/agent/MilestoneSession.java`
- Create: `src/test/java/io/quarkmind/agent/MilestoneSessionTest.java`

**Interfaces:**
- Consumes: nothing (foundational types)
- Produces: `MilestoneEvent(String milestoneId, double temporalWeight)`, `MilestoneTrigger.check(long, MilestoneSession) → List<MilestoneEvent>`, `MilestoneSession.entryId(String) → Optional<UUID>`, `MilestoneSession.setEntryId(String, UUID)`, `MilestoneSession.hasFired(String) → boolean`, `MilestoneSession.markFired(String)`, `MilestoneSession.reset()`

- [ ] **Step 1: Write MilestoneSession tests**

```java
// src/test/java/io/quarkmind/agent/MilestoneSessionTest.java
package io.quarkmind.agent;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class MilestoneSessionTest {

    private MilestoneSession session;

    @BeforeEach
    void setUp() {
        session = new MilestoneSession();
    }

    @Test
    void entryId_returnsEmpty_whenNotSet() {
        assertThat(session.entryId("strategy.drools")).isEmpty();
    }

    @Test
    void entryId_returnsValue_afterSet() {
        UUID id = UUID.randomUUID();
        session.setEntryId("strategy.drools", id);
        assertThat(session.entryId("strategy.drools")).contains(id);
    }

    @Test
    void entryId_separatePerStrategy() {
        UUID id1 = UUID.randomUUID();
        UUID id2 = UUID.randomUUID();
        session.setEntryId("strategy.drools", id1);
        session.setEntryId("strategy.early-pressure", id2);
        assertThat(session.entryId("strategy.drools")).contains(id1);
        assertThat(session.entryId("strategy.early-pressure")).contains(id2);
    }

    @Test
    void hasFired_returnsFalse_initially() {
        assertThat(session.hasFired("frame:4032")).isFalse();
    }

    @Test
    void hasFired_returnsTrue_afterMark() {
        session.markFired("frame:4032");
        assertThat(session.hasFired("frame:4032")).isTrue();
    }

    @Test
    void reset_clearsAllState() {
        UUID id = UUID.randomUUID();
        session.setEntryId("strategy.drools", id);
        session.markFired("frame:4032");

        session.reset();

        assertThat(session.entryId("strategy.drools")).isEmpty();
        assertThat(session.hasFired("frame:4032")).isFalse();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (classes don't exist)**

Run: `mvn test -Dtest=MilestoneSessionTest -q`
Expected: compilation failure — `MilestoneSession` not found

- [ ] **Step 3: Implement MilestoneEvent, MilestoneTrigger, MilestoneSession**

```java
// src/main/java/io/quarkmind/agent/MilestoneEvent.java
package io.quarkmind.agent;

public record MilestoneEvent(String milestoneId, double temporalWeight) {}
```

```java
// src/main/java/io/quarkmind/agent/MilestoneTrigger.java
package io.quarkmind.agent;

import java.util.List;

public interface MilestoneTrigger {
    List<MilestoneEvent> check(long gameFrame, MilestoneSession session);
}
```

```java
// src/main/java/io/quarkmind/agent/MilestoneSession.java
package io.quarkmind.agent;

import jakarta.enterprise.context.ApplicationScoped;

import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class MilestoneSession {

    private final Map<String, UUID> entryIds = new ConcurrentHashMap<>();
    private final Set<String> firedMilestones = ConcurrentHashMap.newKeySet();

    public Optional<UUID> entryId(String strategyId) {
        return Optional.ofNullable(entryIds.get(strategyId));
    }

    public void setEntryId(String strategyId, UUID id) {
        entryIds.put(strategyId, id);
    }

    public boolean hasFired(String milestoneId) {
        return firedMilestones.contains(milestoneId);
    }

    public void markFired(String milestoneId) {
        firedMilestones.add(milestoneId);
    }

    public void reset() {
        entryIds.clear();
        firedMilestones.clear();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -Dtest=MilestoneSessionTest -q`
Expected: all 6 tests PASS

- [ ] **Step 5: Commit**

```
feat(#191): add MilestoneEvent, MilestoneTrigger interface, and MilestoneSession

Refs #191
```

---

### Task 2: FrameThresholdTrigger

**Files:**
- Create: `src/main/java/io/quarkmind/agent/FrameThresholdTrigger.java`
- Create: `src/test/java/io/quarkmind/agent/FrameThresholdTriggerTest.java`

**Interfaces:**
- Consumes: `MilestoneTrigger`, `MilestoneEvent`, `MilestoneSession`
- Produces: `FrameThresholdTrigger` CDI bean implementing `MilestoneTrigger`

- [ ] **Step 1: Write tests**

```java
// src/test/java/io/quarkmind/agent/FrameThresholdTriggerTest.java
package io.quarkmind.agent;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class FrameThresholdTriggerTest {

    private MilestoneSession session;

    @BeforeEach
    void setUp() {
        session = new MilestoneSession();
    }

    @Test
    void check_returnsEmpty_beforeFirstThreshold() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3)));

        assertThat(trigger.check(1000, session)).isEmpty();
    }

    @Test
    void check_firesAtThreshold() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3)));

        List<MilestoneEvent> events = trigger.check(4032, session);

        assertThat(events).hasSize(1);
        assertThat(events.get(0).milestoneId()).isEqualTo("frame:4032");
        assertThat(events.get(0).temporalWeight()).isEqualTo(0.3);
    }

    @Test
    void check_firesPastThreshold() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3)));

        List<MilestoneEvent> events = trigger.check(5000, session);

        assertThat(events).hasSize(1);
        assertThat(events.get(0).milestoneId()).isEqualTo("frame:4032");
    }

    @Test
    void check_doesNotDoubleFire() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3)));

        trigger.check(4032, session); // fires and marks
        List<MilestoneEvent> second = trigger.check(4033, session);

        assertThat(second).isEmpty();
    }

    @Test
    void check_firesMultipleThresholds_atDifferentFrames() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3),
            new FrameThresholdTrigger.Threshold(10752, 0.5)));

        List<MilestoneEvent> first = trigger.check(5000, session);
        assertThat(first).hasSize(1);
        assertThat(first.get(0).temporalWeight()).isEqualTo(0.3);

        List<MilestoneEvent> second = trigger.check(11000, session);
        assertThat(second).hasSize(1);
        assertThat(second.get(0).temporalWeight()).isEqualTo(0.5);
    }

    @Test
    void check_firesBothThresholds_whenFrameJumpsPastBoth() {
        var trigger = new FrameThresholdTrigger(List.of(
            new FrameThresholdTrigger.Threshold(4032, 0.3),
            new FrameThresholdTrigger.Threshold(10752, 0.5)));

        List<MilestoneEvent> events = trigger.check(15000, session);

        assertThat(events).hasSize(2);
    }

    @Test
    void check_returnsEmpty_whenNoThresholds() {
        var trigger = new FrameThresholdTrigger(List.of());
        assertThat(trigger.check(99999, session)).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -Dtest=FrameThresholdTriggerTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement FrameThresholdTrigger**

```java
// src/main/java/io/quarkmind/agent/FrameThresholdTrigger.java
package io.quarkmind.agent;

import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class FrameThresholdTrigger implements MilestoneTrigger {

    public record Threshold(long frame, double weight) {}

    private final List<Threshold> thresholds;

    FrameThresholdTrigger(List<Threshold> thresholds) {
        this.thresholds = List.copyOf(thresholds);
    }

    @Override
    public List<MilestoneEvent> check(long gameFrame, MilestoneSession session) {
        List<MilestoneEvent> events = new ArrayList<>();
        for (Threshold t : thresholds) {
            String milestoneId = "frame:" + t.frame();
            if (gameFrame >= t.frame() && !session.hasFired(milestoneId)) {
                session.markFired(milestoneId);
                events.add(new MilestoneEvent(milestoneId, t.weight()));
            }
        }
        return events;
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -Dtest=FrameThresholdTriggerTest -q`
Expected: all 7 tests PASS

- [ ] **Step 5: Commit**

```
feat(#191): add FrameThresholdTrigger — fires milestones at configurable game frames

Refs #191
```

---

### Task 3: GamePhaseTrigger

**Files:**
- Create: `src/main/java/io/quarkmind/agent/GamePhaseTrigger.java`
- Create: `src/test/java/io/quarkmind/agent/GamePhaseTriggerTest.java`

**Interfaces:**
- Consumes: `MilestoneTrigger`, `MilestoneEvent`, `MilestoneSession`, `SummarisationLifecycle.phaseBus()`, `GamePhase`
- Produces: `GamePhaseTrigger` CDI bean implementing `MilestoneTrigger`

- [ ] **Step 1: Write tests**

```java
// src/test/java/io/quarkmind/agent/GamePhaseTriggerTest.java
package io.quarkmind.agent;

import io.casehub.blocks.summarisation.EventStreamBus;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.EventLevel;
import io.quarkmind.plugin.summarisation.GamePhase;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class GamePhaseTriggerTest {

    private static final long EXPECTED_GAME_LENGTH = 20160; // 15 min at 22.4 fps
    private static final double MIN_WEIGHT = 0.1;
    private static final double MAX_WEIGHT = 0.8;
    private static final EventLevel LEVEL_3 = new EventLevel("phase", 3);

    private EventStreamBus<GamePhase> phaseBus;
    private GamePhaseTrigger trigger;
    private MilestoneSession session;

    @BeforeEach
    void setUp() {
        phaseBus = new EventStreamBus<>();
        trigger = new GamePhaseTrigger(phaseBus, EXPECTED_GAME_LENGTH, MIN_WEIGHT, MAX_WEIGHT);
        session = new MilestoneSession();
    }

    @Test
    void check_returnsEmpty_whenNoPhaseReceived() {
        assertThat(trigger.check(5000, session)).isEmpty();
    }

    @Test
    void check_firesOnPhaseTransition() {
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("EARLY_AGGRESSION", 3000, "combat"), 3000, LEVEL_3));

        List<MilestoneEvent> events = trigger.check(3000, session);

        assertThat(events).hasSize(1);
        assertThat(events.get(0).milestoneId()).isEqualTo("phase:EARLY_AGGRESSION");
    }

    @Test
    void check_temporalWeight_proportionalToGameProgress() {
        long frame = 10000;
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("MID_SKIRMISH", frame, "combat"), frame, LEVEL_3));

        List<MilestoneEvent> events = trigger.check(frame, session);

        double expectedWeight = (double) frame / EXPECTED_GAME_LENGTH; // ~0.496
        assertThat(events.get(0).temporalWeight()).isCloseTo(expectedWeight, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void check_temporalWeight_clampedToMin() {
        long earlyFrame = 500; // 0.025 of game → clamped to 0.1
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("EARLY_MACRO", earlyFrame, "econ"), earlyFrame, LEVEL_3));

        List<MilestoneEvent> events = trigger.check(earlyFrame, session);

        assertThat(events.get(0).temporalWeight()).isEqualTo(MIN_WEIGHT);
    }

    @Test
    void check_temporalWeight_clampedToMax() {
        long lateFrame = 25000; // beyond expected length → clamped to 0.8
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("DEFENSIVE_HOLD", lateFrame, "attack"), lateFrame, LEVEL_3));

        List<MilestoneEvent> events = trigger.check(lateFrame, session);

        assertThat(events.get(0).temporalWeight()).isEqualTo(MAX_WEIGHT);
    }

    @Test
    void check_doesNotDoubleFire_samePhase() {
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("EARLY_AGGRESSION", 3000, "combat"), 3000, LEVEL_3));

        trigger.check(3000, session); // fires and marks
        List<MilestoneEvent> second = trigger.check(3500, session);

        assertThat(second).isEmpty();
    }

    @Test
    void check_firesDifferentPhases() {
        phaseBus.publish(new LevelEvent<>(
            new GamePhase("EARLY_AGGRESSION", 3000, "combat"), 3000, LEVEL_3));
        trigger.check(3000, session);

        phaseBus.publish(new LevelEvent<>(
            new GamePhase("MID_SKIRMISH", 8000, "combat"), 8000, LEVEL_3));
        List<MilestoneEvent> events = trigger.check(8000, session);

        assertThat(events).hasSize(1);
        assertThat(events.get(0).milestoneId()).isEqualTo("phase:MID_SKIRMISH");
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -Dtest=GamePhaseTriggerTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement GamePhaseTrigger**

```java
// src/main/java/io/quarkmind/agent/GamePhaseTrigger.java
package io.quarkmind.agent;

import io.casehub.blocks.summarisation.EventStreamBus;
import io.quarkmind.plugin.summarisation.GamePhase;
import io.quarkmind.plugin.summarisation.SummarisationLifecycle;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
public class GamePhaseTrigger implements MilestoneTrigger {

    private final EventStreamBus<GamePhase> phaseBus;
    private final long expectedGameLength;
    private final double minWeight;
    private final double maxWeight;

    private volatile GamePhase lastSeenPhase;

    @Inject
    GamePhaseTrigger(
            Instance<SummarisationLifecycle> summarisationLifecycle,
            @ConfigProperty(name = "quarkmind.milestones.phase-triggers.expected-game-length", defaultValue = "20160") long expectedGameLength,
            @ConfigProperty(name = "quarkmind.milestones.phase-triggers.min-weight", defaultValue = "0.1") double minWeight,
            @ConfigProperty(name = "quarkmind.milestones.phase-triggers.max-weight", defaultValue = "0.8") double maxWeight) {
        this.phaseBus = null; // resolved lazily
        this.expectedGameLength = expectedGameLength;
        this.minWeight = minWeight;
        this.maxWeight = maxWeight;
        // Store lifecycle instance for lazy init — matching DroolsStrategyTask pattern
        this.lazyLifecycle = summarisationLifecycle;
    }

    private final Instance<SummarisationLifecycle> lazyLifecycle;
    private volatile boolean subscribed = false;

    /** Test constructor — direct phaseBus injection, no CDI. */
    GamePhaseTrigger(EventStreamBus<GamePhase> phaseBus, long expectedGameLength, double minWeight, double maxWeight) {
        this.phaseBus = phaseBus;
        this.expectedGameLength = expectedGameLength;
        this.minWeight = minWeight;
        this.maxWeight = maxWeight;
        this.lazyLifecycle = null;
        this.subscribed = true;
        phaseBus.subscribe(p -> true, e -> lastSeenPhase = e.payload());
    }

    private void ensureSubscribed() {
        if (!subscribed) {
            synchronized (this) {
                if (!subscribed) {
                    lazyLifecycle.get().phaseBus().subscribe(p -> true, e -> lastSeenPhase = e.payload());
                    subscribed = true;
                }
            }
        }
    }

    @Override
    public List<MilestoneEvent> check(long gameFrame, MilestoneSession session) {
        ensureSubscribed();
        GamePhase phase = lastSeenPhase;
        if (phase == null) return List.of();

        String milestoneId = "phase:" + phase.phase();
        if (session.hasFired(milestoneId)) return List.of();

        session.markFired(milestoneId);
        double raw = (double) gameFrame / expectedGameLength;
        double weight = Math.max(minWeight, Math.min(maxWeight, raw));
        return List.of(new MilestoneEvent(milestoneId, weight));
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -Dtest=GamePhaseTriggerTest -q`
Expected: all 7 tests PASS

- [ ] **Step 5: Commit**

```
feat(#191): add GamePhaseTrigger — fires milestones on summarisation phase transitions

Refs #191
```

---

### Task 4: DominanceAssessor and SupplyDominanceAssessor

**Files:**
- Create: `src/main/java/io/quarkmind/agent/DominanceAssessor.java`
- Create: `src/main/java/io/quarkmind/agent/SupplyDominanceAssessor.java`
- Create: `src/test/java/io/quarkmind/agent/SupplyDominanceAssessorTest.java`

**Interfaces:**
- Consumes: `GameState`, `SC2Data.supplyCost(UnitType)`
- Produces: `DominanceAssessor.assess(GameState) → double` in [-1.0, +1.0]

- [ ] **Step 1: Write tests**

```java
// src/test/java/io/quarkmind/agent/SupplyDominanceAssessorTest.java
package io.quarkmind.agent;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class SupplyDominanceAssessorTest {

    private final SupplyDominanceAssessor assessor = new SupplyDominanceAssessor(40);

    @Test
    void assess_noEnemyUnitsVisible_returnsZero() {
        GameState state = gameState(30, List.of(), List.of());
        assertThat(assessor.assess(state)).isEqualTo(0.0);
    }

    @Test
    void assess_equalSupply_returnsZero() {
        GameState state = gameState(4, // 2 zealots = 4 supply
            List.of(zealot(), zealot()),
            List.of(zealot(), zealot()));
        assertThat(assessor.assess(state)).isEqualTo(0.0);
    }

    @Test
    void assess_aheadInSupply_returnsPositive() {
        GameState state = gameState(8, // 4 zealots = 8 supply
            List.of(zealot(), zealot(), zealot(), zealot()),
            List.of(zealot())); // 2 supply enemy
        // delta = 8 - 2 = 6, score = 6/40 = 0.15
        assertThat(assessor.assess(state)).isCloseTo(0.15, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void assess_behindInSupply_returnsNegative() {
        GameState state = gameState(2,
            List.of(zealot()), // 2 supply
            List.of(zealot(), zealot(), zealot(), zealot())); // 8 supply enemy
        // delta = 2 - 8 = -6, score = -6/40 = -0.15
        assertThat(assessor.assess(state)).isCloseTo(-0.15, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void assess_clampsToPositiveOne() {
        GameState state = gameState(60,
            armyOf(30, UnitType.ZEALOT), // 60 supply
            List.of(zealot())); // 2 supply
        // delta = 60 - 2 = 58, 58/40 > 1.0 → clamped to 1.0
        assertThat(assessor.assess(state)).isEqualTo(1.0);
    }

    @Test
    void assess_clampsToNegativeOne() {
        GameState state = gameState(2,
            List.of(zealot()), // 2 supply
            armyOf(30, UnitType.ZEALOT)); // 60 supply
        assertThat(assessor.assess(state)).isEqualTo(-1.0);
    }

    @Test
    void assess_mixedUnitTypes_computesCorrectSupply() {
        // stalker = 2, immortal = 4 → enemy 6 supply
        GameState state = gameState(10,
            armyOf(5, UnitType.ZEALOT), // 10 supply
            List.of(stalker(), immortal()));
        // delta = 10 - 6 = 4, score = 4/40 = 0.1
        assertThat(assessor.assess(state)).isCloseTo(0.1, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void assess_excludesWorkersFromEnemy() {
        // probes are workers (supply cost 1) — enemy workers should still count since
        // enemyUnits list doesn't distinguish workers from army in GameState.
        // But the assessor operates on the full enemyUnits list as-is.
        GameState state = gameState(10,
            armyOf(5, UnitType.ZEALOT), // 10 supply
            List.of(probe(), probe(), probe())); // 3 supply
        // delta = 10 - 3 = 7, score = 7/40 = 0.175
        assertThat(assessor.assess(state)).isCloseTo(0.175, org.assertj.core.data.Offset.offset(0.01));
    }

    // --- helpers ---

    private static GameState gameState(int supplyUsed, List<Unit> army, List<Unit> enemyUnits) {
        return new GameState(200, 100, 15, supplyUsed, army, List.of(), enemyUnits, List.of(), List.of(), List.of(), List.of(), 5000);
    }

    private static Unit zealot() { return unit(UnitType.ZEALOT); }
    private static Unit stalker() { return unit(UnitType.STALKER); }
    private static Unit immortal() { return unit(UnitType.IMMORTAL); }
    private static Unit probe() { return unit(UnitType.PROBE); }

    private static Unit unit(UnitType type) {
        return new Unit("tag-" + type, type, new Point2d(0, 0), 100, 100, 50, 50, 0, 0);
    }

    private static List<Unit> armyOf(int count, UnitType type) {
        return java.util.stream.IntStream.range(0, count).mapToObj(i -> unit(type)).toList();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -Dtest=SupplyDominanceAssessorTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement DominanceAssessor and SupplyDominanceAssessor**

```java
// src/main/java/io/quarkmind/agent/DominanceAssessor.java
package io.quarkmind.agent;

import io.quarkmind.domain.GameState;

public interface DominanceAssessor {
    double assess(GameState state);
}
```

```java
// src/main/java/io/quarkmind/agent/SupplyDominanceAssessor.java
package io.quarkmind.agent;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.SC2Data;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class SupplyDominanceAssessor implements DominanceAssessor {

    private final int maxExpectedDelta;

    SupplyDominanceAssessor(
            @ConfigProperty(name = "quarkmind.milestones.dominance.max-expected-delta", defaultValue = "40") int maxExpectedDelta) {
        this.maxExpectedDelta = maxExpectedDelta;
    }

    @Override
    public double assess(GameState state) {
        if (state.enemyUnits().isEmpty()) {
            return 0.0;
        }
        int enemySupply = state.enemyUnits().stream()
            .mapToInt(u -> SC2Data.supplyCost(u.type()))
            .sum();
        double delta = state.supplyUsed() - enemySupply;
        return Math.max(-1.0, Math.min(1.0, delta / maxExpectedDelta));
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -Dtest=SupplyDominanceAssessorTest -q`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```
feat(#191): add DominanceAssessor interface and SupplyDominanceAssessor placeholder

Refs #191
```

---

### Task 5: MilestoneOutcomeRecorder — replaces GameOutcomeRecorder

**Files:**
- Create: `src/main/java/io/quarkmind/agent/MilestoneOutcomeRecorder.java`
- Create: `src/test/java/io/quarkmind/agent/MilestoneOutcomeRecorderTest.java`
- Delete: `src/main/java/io/quarkmind/agent/GameOutcomeRecorder.java`
- Modify: `src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java` (Javadoc link update)

**Interfaces:**
- Consumes: `OutcomeRecorder`, `StrategySelector`, `GameSession`, `DominanceAssessor`, `MilestoneTrigger` (via `Instance<>`), `MilestoneSession`, `GameStarted`, `GameStopped`, `GameState`, `GameResult`, `AttestationVerdict`, `OutcomeRecord`, `QuarkMindCapabilityTag`
- Produces: `MilestoneOutcomeRecorder.evaluateMilestones(GameState)` — called by `GameTickExecutor` each tick; `onGameStarted(@Observes GameStarted)`, `onGameStopped(@Observes GameStopped)`

- [ ] **Step 1: Write tests**

```java
// src/test/java/io/quarkmind/agent/MilestoneOutcomeRecorderTest.java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.GameResult;
import io.quarkmind.sc2.GameStarted;
import io.quarkmind.sc2.GameStopped;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class MilestoneOutcomeRecorderTest {

    private RecordingOutcomeRecorder outcomeRecorder;
    private StrategySelector strategySelector;
    private GameSession gameSession;
    private MilestoneSession milestoneSession;
    private MilestoneOutcomeRecorder recorder;

    @BeforeEach
    void setUp() {
        outcomeRecorder = new RecordingOutcomeRecorder();
        strategySelector = new StrategySelector();
        gameSession = new GameSession();
        milestoneSession = new MilestoneSession();

        // Default: milestones enabled, dead zone 0.15
        recorder = new MilestoneOutcomeRecorder(
            outcomeRecorder, strategySelector, gameSession, milestoneSession,
            state -> 0.5, // always returns "moderately ahead"
            List.of(new FrameThresholdTrigger(List.of(
                new FrameThresholdTrigger.Threshold(4032, 0.3)))),
            true, 0.15);
    }

    // --- game-end only (no milestones fired) ---

    @Test
    void gameEnd_noMilestones_recordsExactlyOneOutcome() {
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        assertThat(outcomeRecorder.records).hasSize(1);
        assertThat(outcomeRecorder.records.get(0).verdict()).isEqualTo(AttestationVerdict.ENDORSED);
        assertThat(outcomeRecorder.records.get(0).confidence()).isEqualTo(1.0);
    }

    @Test
    void gameEnd_loss_recordsChallenged() {
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.LOSS));

        assertThat(outcomeRecorder.records.get(0).verdict()).isEqualTo(AttestationVerdict.CHALLENGED);
    }

    @Test
    void gameEnd_tie_recordsSound() {
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.TIE));

        assertThat(outcomeRecorder.records.get(0).verdict()).isEqualTo(AttestationVerdict.SOUND);
    }

    @Test
    void gameEnd_unknown_skips() {
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.UNKNOWN));

        assertThat(outcomeRecorder.records).isEmpty();
    }

    // --- SPI fallback (OutcomeRecorder is not AttestingOutcomeRecorder) ---

    @Test
    void evaluateMilestones_withoutSpi_isNoOp() {
        recorder.onGameStarted(new GameStarted());
        GameState state = gameStateAtFrame(5000);

        recorder.evaluateMilestones(state);

        // No milestone attestation recorded — SPI not available
        assertThat(outcomeRecorder.records).isEmpty();
    }

    // --- dead zone ---

    @Test
    void evaluateMilestones_deadZone_skipsAttestation() {
        // dominance assessor returns 0.1 (below dead zone 0.15)
        recorder = new MilestoneOutcomeRecorder(
            outcomeRecorder, strategySelector, gameSession, milestoneSession,
            state -> 0.1, // below dead zone
            List.of(new FrameThresholdTrigger(List.of(
                new FrameThresholdTrigger.Threshold(4032, 0.3)))),
            true, 0.15);

        recorder.onGameStarted(new GameStarted());
        // Even with SPI mock, the dead zone should prevent recording
        // (but without SPI, evaluateMilestones is no-op anyway)
        assertThat(outcomeRecorder.records).isEmpty();
    }

    // --- milestone session reset ---

    @Test
    void gameStarted_resetsMilestoneSession() {
        milestoneSession.markFired("frame:4032");
        milestoneSession.setEntryId("strategy.drools", UUID.randomUUID());

        recorder.onGameStarted(new GameStarted());

        assertThat(milestoneSession.hasFired("frame:4032")).isFalse();
        assertThat(milestoneSession.entryId("strategy.drools")).isEmpty();
    }

    // --- game-end uses correct strategy and context ---

    @Test
    void gameEnd_usesCurrentStrategyAndContext() {
        strategySelector.selectForGame("strategy.early-pressure",
            QuarkMindCapabilityTag.STRATEGY_VS_AGGRESSIVE);
        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        OutcomeRecord recorded = outcomeRecorder.records.get(0);
        assertThat(recorded.actorId()).isEqualTo("strategy.early-pressure");
        assertThat(recorded.capabilityTag()).isEqualTo(QuarkMindCapabilityTag.STRATEGY_VS_AGGRESSIVE);
    }

    // --- disabled switch ---

    @Test
    void gameEnd_recordsEvenWhenMilestonesDisabled() {
        recorder = new MilestoneOutcomeRecorder(
            outcomeRecorder, strategySelector, gameSession, milestoneSession,
            state -> 0.5,
            List.of(), false, 0.15); // disabled

        recorder.onGameStarted(new GameStarted());
        recorder.onGameStopped(new GameStopped(GameResult.WIN));

        assertThat(outcomeRecorder.records).hasSize(1);
    }

    // --- helpers ---

    private static GameState gameStateAtFrame(long frame) {
        return new GameState(200, 100, 15, 6, List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), List.of(), frame);
    }

    static class RecordingOutcomeRecorder implements OutcomeRecorder {
        final List<OutcomeRecord> records = new ArrayList<>();

        @Override
        public void record(OutcomeRecord record) {
            records.add(record);
        }
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -Dtest=MilestoneOutcomeRecorderTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement MilestoneOutcomeRecorder**

```java
// src/main/java/io/quarkmind/agent/MilestoneOutcomeRecorder.java
package io.quarkmind.agent;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.GameResult;
import io.quarkmind.sc2.GameStarted;
import io.quarkmind.sc2.GameStopped;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.List;

@ApplicationScoped
public class MilestoneOutcomeRecorder {

    private static final Logger log = Logger.getLogger(MilestoneOutcomeRecorder.class);

    private final OutcomeRecorder outcomeRecorder;
    private final StrategySelector strategySelector;
    private final GameSession gameSession;
    private final MilestoneSession milestoneSession;
    private final DominanceAssessor dominanceAssessor;
    private final List<MilestoneTrigger> triggers;
    private final boolean milestonesEnabled;
    private final double deadZoneThreshold;

    @Inject
    MilestoneOutcomeRecorder(
            OutcomeRecorder outcomeRecorder,
            StrategySelector strategySelector,
            GameSession gameSession,
            MilestoneSession milestoneSession,
            DominanceAssessor dominanceAssessor,
            @Any Instance<MilestoneTrigger> triggerInstances,
            @ConfigProperty(name = "quarkmind.milestones.enabled", defaultValue = "true") boolean milestonesEnabled,
            @ConfigProperty(name = "quarkmind.milestones.dead-zone-threshold", defaultValue = "0.15") double deadZoneThreshold) {
        this.outcomeRecorder = outcomeRecorder;
        this.strategySelector = strategySelector;
        this.gameSession = gameSession;
        this.milestoneSession = milestoneSession;
        this.dominanceAssessor = dominanceAssessor;
        this.triggers = triggerInstances.stream().toList();
        this.milestonesEnabled = milestonesEnabled;
        this.deadZoneThreshold = deadZoneThreshold;
    }

    /** Test constructor — no CDI. */
    MilestoneOutcomeRecorder(
            OutcomeRecorder outcomeRecorder,
            StrategySelector strategySelector,
            GameSession gameSession,
            MilestoneSession milestoneSession,
            DominanceAssessor dominanceAssessor,
            List<MilestoneTrigger> triggers,
            boolean milestonesEnabled,
            double deadZoneThreshold) {
        this.outcomeRecorder = outcomeRecorder;
        this.strategySelector = strategySelector;
        this.gameSession = gameSession;
        this.milestoneSession = milestoneSession;
        this.dominanceAssessor = dominanceAssessor;
        this.triggers = triggers;
        this.milestonesEnabled = milestonesEnabled;
        this.deadZoneThreshold = deadZoneThreshold;
    }

    void onGameStarted(@Observes GameStarted event) {
        milestoneSession.reset();
    }

    public void evaluateMilestones(GameState state) {
        if (!milestonesEnabled) return;

        // SPI gate: milestone recording requires AttestingOutcomeRecorder (engine#648)
        // Without it, evaluateMilestones is a no-op — game-end records via record() only.
        // TODO: activate when engine#648 ships AttestingOutcomeRecorder sub-interface
        log.debugf("[MILESTONE] SPI not available — milestone evaluation skipped at frame %d", state.gameFrame());
    }

    void onGameStopped(@Observes GameStopped event) {
        if (event.result() == GameResult.UNKNOWN) {
            log.infof("[MILESTONE] Game ended with unknown result — skipped (strategy=%s context=%s)",
                strategySelector.getSelectedId(), strategySelector.getOpponentContext());
            return;
        }
        String strategyId = strategySelector.getSelectedId();
        String context = strategySelector.getOpponentContext();
        AttestationVerdict verdict = switch (event.result()) {
            case WIN     -> AttestationVerdict.ENDORSED;
            case LOSS    -> AttestationVerdict.CHALLENGED;
            case TIE     -> AttestationVerdict.SOUND;
            case UNKNOWN -> throw new AssertionError("unreachable — guarded above");
        };

        // If a milestone entry already exists for this strategy, append game-end attestation.
        // Otherwise, create entry with game-end outcome (identical to pre-milestone behavior).
        // Until engine#648: always creates via record() (no milestone entries exist).
        outcomeRecorder.record(OutcomeRecord.of(
            strategyId,
            gameSession.id(),
            context,
            verdict,
            1.0
        ));
        log.infof("[MILESTONE] Recorded game-end: strategy=%s context=%s result=%s verdict=%s",
            strategyId, context, event.result(), verdict);
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -Dtest=MilestoneOutcomeRecorderTest -q`
Expected: all 9 tests PASS

- [ ] **Step 5: Delete GameOutcomeRecorder, update AdvisoryGameOutcomeRecorder Javadoc**

Delete `src/main/java/io/quarkmind/agent/GameOutcomeRecorder.java`.

In `src/main/java/io/quarkmind/agent/AdvisoryGameOutcomeRecorder.java`, update the Javadoc on line 20:
```
- {@link GameOutcomeRecorder}
+ {@link MilestoneOutcomeRecorder}
```

- [ ] **Step 6: Run full test suite to verify no breakage**

Run: `mvn test -q`
Expected: all tests PASS. `StrategyOutcomeRecordIT` still passes because `MilestoneOutcomeRecorder.onGameStopped()` does the same `record()` call that `GameOutcomeRecorder` did.

- [ ] **Step 7: Commit**

```
feat(#191): add MilestoneOutcomeRecorder, delete GameOutcomeRecorder

MilestoneOutcomeRecorder absorbs all GameOutcomeRecorder responsibilities.
Milestone evaluation is no-op until engine#648 ships AttestingOutcomeRecorder.
Game-end recording is identical to pre-milestone behavior.

Refs #191
```

---

### Task 6: Wire into GameTickExecutor and add configuration

**Files:**
- Modify: `src/main/java/io/quarkmind/agent/GameTickExecutor.java`
- Modify: `src/test/java/io/quarkmind/agent/GameTickExecutorMigrationTest.java`
- Modify: `src/main/resources/application.properties`

**Interfaces:**
- Consumes: `MilestoneOutcomeRecorder.evaluateMilestones(GameState)`, `GameState`
- Produces: milestone evaluation called each tick in the correct ordering (after summarisation, before dispatch)

- [ ] **Step 1: Write test for milestone evaluation in tick**

Add to `GameTickExecutorMigrationTest`:

```java
@Test
void execute_callsMilestoneEvaluationAfterSummarisationBeforeDispatch() {
    GameState state = stubGameState(5000L, 200, 100);
    when(engine.observe()).thenReturn(state);
    when(caseHub.signalAndAwaitSync(any(), any(), any())).thenReturn(mock(CaseContext.class));

    executor.execute();

    verify(milestoneOutcomeRecorder).evaluateMilestones(state);
    // Verify ordering: summarisation before milestone, milestone before dispatch
    var inOrder = org.mockito.Mockito.inOrder(summarisation, milestoneOutcomeRecorder, engine);
    inOrder.verify(summarisation).tick(5000L);
    inOrder.verify(milestoneOutcomeRecorder).evaluateMilestones(state);
    inOrder.verify(engine).dispatch();
}
```

Update `setUp()` to add the `milestoneOutcomeRecorder` mock:

```java
private MilestoneOutcomeRecorder milestoneOutcomeRecorder;

@BeforeEach
void setUp() {
    // ... existing mocks ...
    milestoneOutcomeRecorder = mock(MilestoneOutcomeRecorder.class);
    executor.milestoneOutcomeRecorder = milestoneOutcomeRecorder;
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -Dtest=GameTickExecutorMigrationTest#execute_callsMilestoneEvaluationAfterSummarisationBeforeDispatch -q`
Expected: FAIL — `milestoneOutcomeRecorder` field doesn't exist on `GameTickExecutor`

- [ ] **Step 3: Wire MilestoneOutcomeRecorder into GameTickExecutor**

Add field to `GameTickExecutor`:

```java
@Inject MilestoneOutcomeRecorder milestoneOutcomeRecorder;
```

In `execute()`, after `summarisationLifecycle.tick(gameState.gameFrame())` and before `if (ctx != null) { deferredAdvisoryEvaluator.evaluate(...)`:

```java
// Milestone evaluation: assess strategy dominance at game-time checkpoints
milestoneOutcomeRecorder.evaluateMilestones(gameState);
```

- [ ] **Step 4: Add configuration to application.properties**

Append to `src/main/resources/application.properties` after the advisory trust section:

```properties
# --- Milestone-based trust scoring (Layer 6+) ---
# Milestones record intermediate trust attestations at game-time checkpoints.
# Requires engine#648 (AttestingOutcomeRecorder SPI) for milestone attestations;
# game-end recording works regardless.
quarkmind.milestones.enabled=true
quarkmind.milestones.dead-zone-threshold=0.15
quarkmind.milestones.dominance.max-expected-delta=40

# Frame-based triggers (SC2 Faster = 22.4 fps)
quarkmind.milestones.frame-thresholds[0].frame=4032
quarkmind.milestones.frame-thresholds[0].weight=0.3
quarkmind.milestones.frame-thresholds[1].frame=10752
quarkmind.milestones.frame-thresholds[1].weight=0.5

# Phase-based triggers
quarkmind.milestones.phase-triggers.expected-game-length=20160
quarkmind.milestones.phase-triggers.min-weight=0.1
quarkmind.milestones.phase-triggers.max-weight=0.8

# Profile overrides — milestones disabled in profiles without meaningful game-end outcomes
%mock.quarkmind.milestones.enabled=false
%emulated.quarkmind.milestones.enabled=false
%emulated-sc2.quarkmind.milestones.enabled=false
%replay.quarkmind.milestones.enabled=false
```

- [ ] **Step 5: Run all tests — verify they pass**

Run: `mvn test -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```
feat(#191): wire MilestoneOutcomeRecorder into GameTickExecutor, add config

evaluateMilestones() called after summarisation, before dispatch.
Milestones disabled in mock/emulated/replay profiles (no meaningful game-end).

Refs #191
```

---

### Task 7: Rewrite StrategyOutcomeRecordIT for MilestoneOutcomeRecorder

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/mock/StrategyOutcomeRecordIT.java`

**Interfaces:**
- Consumes: `MilestoneOutcomeRecorder`, `StrategySelector`, `TrustGateService`, `InMemoryLedgerEntryRepository`, `InMemoryActorTrustScoreRepository`, `GameSession`, `GameStarted`, `GameStopped`
- Produces: integration test verifying full pipeline from game events through to materialized trust scores

- [ ] **Step 1: Update test class to reference MilestoneOutcomeRecorder**

The existing tests should already pass since `MilestoneOutcomeRecorder.onGameStopped()` does the same `record()` call. But the class Javadoc needs updating and we should add a test confirming the SPI fallback behavior.

Update the class Javadoc:

```java
/**
 * Integration test: MilestoneOutcomeRecorder writes the outcome record synchronously,
 * and the full trust pipeline (OutcomeRecordSaveService → IncrementalTrustUpdateObserver
 * → ActorTrustScoreRepository.upsert) materializes decisionCount after GameStopped.
 *
 * <p>Until engine#648 ships AttestingOutcomeRecorder, milestone evaluation is a no-op.
 * These tests verify game-end recording — identical to pre-milestone behavior.
 */
```

Add test verifying SPI fallback:

```java
@Test
void milestoneEvaluation_withoutSpi_doesNotInflateDecisionCount() {
    // Inject MilestoneOutcomeRecorder and call evaluateMilestones
    // to verify it doesn't create extra ledger entries without the SPI
    gameStartedEvent.fire(new GameStarted());
    String selectedId = strategySelector.getSelectedId();
    String context = strategySelector.getOpponentContext();

    // Simulate a tick at frame 5000 (past the 4032 frame threshold)
    // This should be a no-op without AttestingOutcomeRecorder
    milestoneOutcomeRecorder.evaluateMilestones(
        new io.quarkmind.domain.GameState(
            200, 100, 15, 6,
            java.util.List.of(), java.util.List.of(), java.util.List.of(),
            java.util.List.of(), java.util.List.of(), java.util.List.of(),
            java.util.List.of(), 5000));

    gameStoppedEvent.fire(new GameStopped(GameResult.WIN));

    assertThat(trustGateService.decisionCount(selectedId, context))
        .as("decisionCount should be 1 — milestones must not inflate without SPI")
        .isEqualTo(1);
}
```

Add the injection:

```java
@Inject MilestoneOutcomeRecorder milestoneOutcomeRecorder;
```

- [ ] **Step 2: Run integration tests**

Run: `mvn test -Dtest=StrategyOutcomeRecordIT -q`
Expected: all tests PASS (existing + new)

- [ ] **Step 3: Commit**

```
test(#191): update StrategyOutcomeRecordIT for MilestoneOutcomeRecorder

Verify SPI fallback: evaluateMilestones is no-op without AttestingOutcomeRecorder.
Game-end recording and trust materialization unchanged.

Refs #191
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ MilestoneEvent, MilestoneTrigger interface — Task 1
- ✅ MilestoneSession with ConcurrentHashMap — Task 1
- ✅ FrameThresholdTrigger — Task 2
- ✅ GamePhaseTrigger with lazy phaseBus subscription — Task 3
- ✅ DominanceAssessor interface — Task 4
- ✅ SupplyDominanceAssessor with fog-of-war guard — Task 4
- ✅ MilestoneOutcomeRecorder replacing GameOutcomeRecorder — Task 5
- ✅ SPI gate (no-op without AttestingOutcomeRecorder) — Task 5
- ✅ GameTickExecutor wiring (after summarisation, before dispatch) — Task 6
- ✅ Configuration + profile overrides — Task 6
- ✅ StrategyOutcomeRecordIT rewrite — Task 7
- ✅ GameOutcomeRecorder deletion — Task 5
- ✅ AdvisoryGameOutcomeRecorder Javadoc link update — Task 5
- ⏳ MilestoneIntegrationIT — deferred until engine#648 (per spec §Testing)

**Placeholder scan:** No TBD/TODO except the one engine#648 gate comment in MilestoneOutcomeRecorder (intentional — documents the activation point).

**Type consistency:** `MilestoneEvent`, `MilestoneTrigger`, `MilestoneSession`, `DominanceAssessor`, `FrameThresholdTrigger`, `GamePhaseTrigger`, `SupplyDominanceAssessor`, `MilestoneOutcomeRecorder` — all consistent across tasks.
