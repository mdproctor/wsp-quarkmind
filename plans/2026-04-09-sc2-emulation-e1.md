# SC2 Emulation Engine — Phase E1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `EmulatedGame` + `EmulatedEngine` on a new `%emulated` Quarkus profile, with probe-driven mineral harvesting as the first real physics mechanic.

**Architecture:** `SC2Data` is extracted from `SimulatedGame` into `domain/` as the shared constant source for both engines. `EmulatedGame` (no CDI) owns the physics; `EmulatedEngine` (CDI, `@IfBuildProfile("emulated")`) wraps it behind the `SC2Engine` interface. `MockEngine`'s `@UnlessBuildProfile` is widened to exclude `"emulated"`, preventing CDI ambiguity. `MockStartupBean` fires in `%emulated` by design and drives the same `AgentOrchestrator` game loop.

**Tech Stack:** Java 21 records, Quarkus 3.34.2 CDI profile annotations (`@IfBuildProfile`, `@UnlessBuildProfile`), JUnit 5 + AssertJ (plain, no `@QuarkusTest`).

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Create | `src/main/java/io/quarkmind/domain/SC2Data.java` | Shared game constants — extracted from `SimulatedGame` |
| Modify | `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java` | Delegate data lookups to `SC2Data` |
| Modify | `src/main/java/io/quarkmind/sc2/mock/MockEngine.java` | Add `"emulated"` to `@UnlessBuildProfile` |
| Create | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Phase E1 physics engine |
| Create | `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java` | `SC2Engine` CDI impl for `%emulated` |
| Modify | `src/main/resources/application.properties` | Add `%emulated` profile comment block |
| Create | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Plain JUnit tests for harvest model |

---

## Task 1: Create `SC2Data` and wire `SimulatedGame` to it

**Files:**
- Create: `src/main/java/io/quarkmind/domain/SC2Data.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`

- [ ] **Step 1: Create `SC2Data`**

Create `src/main/java/io/quarkmind/domain/SC2Data.java`:

```java
package io.quarkmind.domain;

public final class SC2Data {

    private SC2Data() {}

    /** Minerals generated per mining probe per game tick at Faster speed (22.4 loops/sec). */
    public static final double MINERALS_PER_PROBE_PER_TICK = 50.0 / 60.0 / 22.4; // ≈ 0.0372

    public static final int INITIAL_MINERALS  = 50;
    public static final int INITIAL_VESPENE   = 0;
    public static final int INITIAL_SUPPLY    = 15;
    public static final int INITIAL_SUPPLY_USED = 12;
    public static final int INITIAL_PROBES    = 12;

    public static int trainTimeInTicks(UnitType type) {
        return switch (type) {
            case PROBE    -> 12;
            case ZEALOT   -> 28;
            case STALKER  -> 31;
            case IMMORTAL -> 40;
            case OBSERVER -> 22;
            default       -> 30;
        };
    }

    public static int buildTimeInTicks(BuildingType type) {
        return switch (type) {
            case PYLON             -> 18;
            case GATEWAY           -> 47;
            case CYBERNETICS_CORE  -> 37;
            case ASSIMILATOR       -> 21;
            case ROBOTICS_FACILITY -> 47;
            case STARGATE          -> 44;
            case FORGE             -> 30;
            case TWILIGHT_COUNCIL  -> 37;
            default                -> 40;
        };
    }

    public static int supplyCost(UnitType type) {
        return switch (type) {
            case PROBE    -> 1;
            case ZEALOT   -> 2;
            case STALKER  -> 2;
            case IMMORTAL -> 4;
            default       -> 2;
        };
    }

    public static int supplyBonus(BuildingType type) {
        return type == BuildingType.PYLON ? 8 : 0;
    }

    public static int maxHealth(UnitType type) {
        return switch (type) {
            case PROBE   -> 45;
            case ZEALOT  -> 100;
            case STALKER -> 80;
            default      -> 100;
        };
    }

    public static int maxBuildingHealth(BuildingType type) {
        return switch (type) {
            case NEXUS    -> 1500;
            case PYLON    -> 200;
            case GATEWAY  -> 500;
            default       -> 500;
        };
    }
}
```

- [ ] **Step 2: Update `SimulatedGame` to delegate to `SC2Data`**

In `src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java`, replace the four private static methods at the bottom and the hardcoded reset values:

Replace the `reset()` method's hardcoded `minerals = 50` line and the `nextTag = 200` block — the initial state literals stay (they are positional, not data), but mineral/supply/vespene initial values reference `SC2Data`:

```java
public synchronized void reset() {
    minerals = SC2Data.INITIAL_MINERALS;
    vespene = SC2Data.INITIAL_VESPENE;
    supply = SC2Data.INITIAL_SUPPLY;
    supplyUsed = SC2Data.INITIAL_SUPPLY_USED;
    gameFrame.set(0);
    myUnits.clear();
    myBuildings.clear();
    enemyUnits.clear();
    geysers.clear();
    pendingCompletions.clear();
    nextTag = 200;

    for (int i = 0; i < SC2Data.INITIAL_PROBES; i++) {
        myUnits.add(new Unit("probe-" + i, UnitType.PROBE, new Point2d(9 + i * 0.5f, 9), 45, 45));
    }
    myBuildings.add(new Building("nexus-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, true));
    geysers.add(new Resource("geyser-0", new Point2d(5, 11), 2250));
    geysers.add(new Resource("geyser-1", new Point2d(11, 5), 2250));
}
```

Replace the four private static methods at the bottom of `SimulatedGame` with delegation calls:

```java
private static int trainTimeInTicks(UnitType type)    { return SC2Data.trainTimeInTicks(type); }
private static int buildTimeInTicks(BuildingType type) { return SC2Data.buildTimeInTicks(type); }
private int supplyCost(UnitType type)                  { return SC2Data.supplyCost(type); }
private int supplyBonus(BuildingType type)             { return SC2Data.supplyBonus(type); }
private int maxHealth(UnitType type)                   { return SC2Data.maxHealth(type); }
private int maxBuildingHealth(BuildingType type)       { return SC2Data.maxBuildingHealth(type); }
```

- [ ] **Step 3: Run `SimulatedGameTest` to confirm no regression**

```bash
mvn test -Dtest=SimulatedGameTest -q
```

Expected: `BUILD SUCCESS`, all 7 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java \
        src/main/java/io/quarkmind/sc2/mock/SimulatedGame.java
git commit -m "refactor(domain): extract SC2Data — shared game constants for SimulatedGame and EmulatedGame"
```

---

## Task 2: Write `EmulatedGameTest` (failing)

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Create the test file**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.SC2Data;
import io.quarkmind.sc2.intent.MoveIntent;
import io.quarkmind.domain.Point2d;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class EmulatedGameTest {

    EmulatedGame game;

    @BeforeEach
    void setUp() {
        game = new EmulatedGame();
        game.reset();
    }

    @Test
    void initialMineralsAreFifty() {
        assertThat(game.snapshot().minerals()).isEqualTo(50);
    }

    @Test
    void mineralAccumulatesWithMiningProbes() {
        int ticks = 100;
        for (int i = 0; i < ticks; i++) game.tick();
        double expected = SC2Data.INITIAL_MINERALS + (SC2Data.INITIAL_PROBES * SC2Data.MINERALS_PER_PROBE_PER_TICK * ticks);
        assertThat(game.snapshot().minerals()).isCloseTo((int) expected, within(1));
    }

    @Test
    void zeroProbesYieldsNoMineralGain() {
        game.setMiningProbes(0);
        for (int i = 0; i < 100; i++) game.tick();
        assertThat(game.snapshot().minerals()).isEqualTo(50);
    }

    @Test
    void snapshotFrameDoesNotChangeAfterTick() {
        GameState before = game.snapshot();
        game.tick();
        assertThat(before.gameFrame()).isEqualTo(0L);
        assertThat(game.snapshot().gameFrame()).isEqualTo(1L);
    }

    @Test
    void resetRestoresInitialState() {
        for (int i = 0; i < 100; i++) game.tick();
        game.reset();
        assertThat(game.snapshot().minerals()).isEqualTo(50);
        assertThat(game.snapshot().gameFrame()).isEqualTo(0L);
    }

    @Test
    void applyIntentIsNoOp() {
        game.applyIntent(new MoveIntent("probe-0", new Point2d(10, 10)));
        assertThat(game.snapshot().minerals()).isEqualTo(50);
        assertThat(game.snapshot().myUnits()).hasSize(12);
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD FAILURE` — `EmulatedGame cannot be resolved to a type`.

---

## Task 3: Implement `EmulatedGame`

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Create `EmulatedGame`**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.Intent;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;

/**
 * Phase E1 physics engine.
 * Probe-driven mineral harvesting. All other mechanics are stubs.
 * Not a CDI bean — owned and instantiated by {@link EmulatedEngine}.
 */
public class EmulatedGame {

    private static final Logger log = Logger.getLogger(EmulatedGame.class);

    private double mineralAccumulator;
    private int miningProbes;
    private long gameFrame;
    private final List<Unit>     myUnits    = new ArrayList<>();
    private final List<Building> myBuildings = new ArrayList<>();
    private final List<Unit>     enemyUnits = new ArrayList<>();
    private final List<Resource> geysers    = new ArrayList<>();

    public void reset() {
        mineralAccumulator = SC2Data.INITIAL_MINERALS;
        miningProbes       = SC2Data.INITIAL_PROBES;
        gameFrame          = 0;
        myUnits.clear();
        myBuildings.clear();
        enemyUnits.clear();
        geysers.clear();

        for (int i = 0; i < SC2Data.INITIAL_PROBES; i++) {
            myUnits.add(new Unit("probe-" + i, UnitType.PROBE,
                new Point2d(9 + i * 0.5f, 9),
                SC2Data.maxHealth(UnitType.PROBE),
                SC2Data.maxHealth(UnitType.PROBE)));
        }
        myBuildings.add(new Building("nexus-0", BuildingType.NEXUS,
            new Point2d(8, 8),
            SC2Data.maxBuildingHealth(BuildingType.NEXUS),
            SC2Data.maxBuildingHealth(BuildingType.NEXUS),
            true));
        geysers.add(new Resource("geyser-0", new Point2d(5, 11), 2250));
        geysers.add(new Resource("geyser-1", new Point2d(11, 5), 2250));
    }

    public void tick() {
        gameFrame++;
        mineralAccumulator += miningProbes * SC2Data.MINERALS_PER_PROBE_PER_TICK;
    }

    /** E1 stub — intents are logged but not applied. */
    public void applyIntent(Intent intent) {
        log.debugf("[EMULATED] Intent (E1 no-op): %s", intent);
    }

    public GameState snapshot() {
        return new GameState(
            (int) mineralAccumulator,
            SC2Data.INITIAL_VESPENE,
            SC2Data.INITIAL_SUPPLY,
            SC2Data.INITIAL_SUPPLY_USED,
            List.copyOf(myUnits),
            List.copyOf(myBuildings),
            List.copyOf(enemyUnits),
            List.copyOf(geysers),
            gameFrame
        );
    }

    /** Package-private — used by tests to control mining probe count. */
    void setMiningProbes(int count) {
        this.miningProbes = count;
    }
}
```

- [ ] **Step 2: Run `EmulatedGameTest`**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`, all 6 tests pass.

- [ ] **Step 3: Run full test suite to confirm no regression**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`. All 187 existing tests plus 6 new = 193 total.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(emulated): add EmulatedGame with probe-driven mineral harvesting"
```

---

## Task 4: Create `EmulatedEngine` and wire the `%emulated` profile

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/MockEngine.java`
- Modify: `src/main/resources/application.properties`

- [ ] **Step 1: Create `EmulatedEngine`**

```java
package io.quarkmind.sc2.emulated;

import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkmind.domain.GameState;
import io.quarkmind.sc2.IntentQueue;
import io.quarkmind.sc2.SC2Engine;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

/**
 * SC2Engine implementation backed by {@link EmulatedGame}.
 * Active only in the {@code %emulated} profile.
 * Mirrors {@link io.quarkmind.sc2.mock.MockEngine} in structure.
 */
@IfBuildProfile("emulated")
@ApplicationScoped
public class EmulatedEngine implements SC2Engine {

    private static final Logger log = Logger.getLogger(EmulatedEngine.class);

    private final EmulatedGame game = new EmulatedGame();
    private final IntentQueue intentQueue;
    private final List<Consumer<GameState>> frameListeners = new ArrayList<>();
    private boolean connected = false;

    @Inject
    EmulatedEngine(IntentQueue intentQueue) {
        this.intentQueue = intentQueue;
    }

    @Override
    public void connect() {
        connected = true;
        log.info("[EMULATED] Engine connected");
    }

    @Override
    public void joinGame() {
        game.reset();
        log.info("[EMULATED] Joined game — 12 probes harvesting, emulated physics active");
    }

    @Override
    public void leaveGame() {
        connected = false;
        log.info("[EMULATED] Left game");
    }

    @Override
    public boolean isConnected() {
        return connected;
    }

    @Override
    public void tick() {
        game.tick();
    }

    @Override
    public GameState observe() {
        GameState state = game.snapshot();
        frameListeners.forEach(l -> l.accept(state));
        return state;
    }

    @Override
    public void dispatch() {
        intentQueue.drainAll().forEach(game::applyIntent);
    }

    @Override
    public void addFrameListener(Consumer<GameState> listener) {
        frameListeners.add(listener);
    }
}
```

- [ ] **Step 2: Exclude `MockEngine` from `%emulated`**

In `src/main/java/io/quarkmind/sc2/mock/MockEngine.java`, change line 1 of the class annotation:

```java
// Before
@UnlessBuildProfile(anyOf = {"sc2", "replay"})

// After
@UnlessBuildProfile(anyOf = {"sc2", "replay", "emulated"})
```

- [ ] **Step 3: Add `%emulated` profile to `application.properties`**

Append to `src/main/resources/application.properties`:

```properties
# Emulated SC2 profile — physics simulation, no real SC2 binary needed
# Run with: mvn quarkus:dev -Dquarkus.profile=emulated
# Phase E1: probe-driven mineral harvesting
```

- [ ] **Step 4: Run full test suite**

```bash
mvn test -q
```

Expected: `BUILD SUCCESS`, 193 tests pass. (Tests run in `%test` profile — `EmulatedEngine` is inactive, no CDI conflict.)

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java \
        src/main/java/io/quarkmind/sc2/mock/MockEngine.java \
        src/main/resources/application.properties
git commit -m "feat(emulated): add EmulatedEngine and %emulated Quarkus profile"
```

---

## Task 5: Smoke-test the `%emulated` profile

**Files:** none

- [ ] **Step 1: Boot the application on `%emulated`**

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

- [ ] **Step 2: Verify log output**

Within the first few seconds you should see (order may vary):

```
[EMULATED] Engine connected
[EMULATED] Joined game — 12 probes harvesting, emulated physics active
```

You should NOT see:
```
[MOCK] Engine connected    ← MockEngine must be silent
No beans found for SC2Engine  ← CDI must resolve cleanly
```

- [ ] **Step 3: Let it run for 5 seconds, observe mineral log (optional)**

The scheduler fires every 500ms. If debug logging is on, you'll see `[EMULATED] Intent (E1 no-op)` for each intent. Minerals in the game state increment each tick.

- [ ] **Step 4: Stop with CTRL+C**

- [ ] **Step 5: Final commit (update HANDOFF.md or design snapshot if session ends here)**

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Extract `SC2Data` from `SimulatedGame` | Task 1 |
| `SimulatedGame` delegates to `SC2Data` | Task 1 |
| `SC2Data.MINERALS_PER_PROBE_PER_TICK = 50/60/22.4` | Task 1 |
| `EmulatedGame` — probe harvest model | Task 3 |
| `EmulatedGame.setMiningProbes()` package-private | Task 3 |
| `applyIntent` is a no-op stub | Task 3 |
| Initial state mirrors `SimulatedGame.reset()` | Task 3 |
| `EmulatedEngine` mirrors `MockEngine` structure | Task 4 |
| `@IfBuildProfile("emulated")` on `EmulatedEngine` | Task 4 |
| `MockEngine` adds `"emulated"` to `@UnlessBuildProfile` | Task 4 |
| `SimulatedGame` CDI guard note | *(SimulatedGame stays active — no change needed; MockScenarioRunner is harmless in %emulated)* |
| `%emulated` profile in `application.properties` | Task 4 |
| `EmulatedGameTest` — 6 tests | Task 2 + 3 |
| `SimulatedGameTest` unchanged | Task 1 (verified in step 3) |
| Boot smoke test | Task 5 |

**Placeholder scan:** None found.

**Type consistency:** `SC2Data` method names used consistently across Task 1 (definition), Task 3 (`EmulatedGame` calls), and Task 2 (test references `SC2Data.INITIAL_PROBES`, `SC2Data.INITIAL_MINERALS`, `SC2Data.MINERALS_PER_PROBE_PER_TICK`). All match.
