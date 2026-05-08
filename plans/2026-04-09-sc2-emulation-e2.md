# SC2 Emulation Engine — Phase E2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add unit movement, a scripted enemy wave, full intent handling (train/build/move/attack), live config via `EmulatedConfig`, and a visualizer config panel — making the emulated game loop visually compelling.

**Architecture:** `SC2Data` gains cost methods used by `EmulatedGame`. `EmulatedGame` grows movement fields, enemy wave spawning, and a full `applyIntent()` switch. `EmulatedConfig` (`@ApplicationScoped`) holds `@ConfigProperty` defaults with volatile runtime-mutable fields; `EmulatedConfigResource` (GET/PUT `/qa/emulated/config`) exposes them. `EmulatedEngine` injects `EmulatedConfig`, configures the wave on `joinGame()`, and syncs speed each tick. The visualizer gains an HTML config sidebar that calls the API; the panel is hidden in non-`%emulated` profiles.

**Tech Stack:** Java 21 sealed records/switch, Quarkus 3.34.2 `@ConfigProperty`, MicroProfile Config, RESTEasy Jackson, PixiJS 8 (existing), vanilla JS fetch API.

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Modify | `src/main/java/io/quarkmind/domain/SC2Data.java` | Add `mineralCost(UnitType)`, `mineralCost(BuildingType)`, `gasCost(UnitType)` |
| Create | `src/main/java/io/quarkmind/sc2/emulated/EnemyWave.java` | Package-private wave definition record |
| Create | `src/main/java/io/quarkmind/qa/EmulatedConfig.java` | CDI config bean — properties + volatile runtime overrides |
| Create | `src/main/java/io/quarkmind/qa/EmulatedConfigResource.java` | GET/PUT `/qa/emulated/config` |
| Create | `src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java` | @QuarkusTest for config endpoint |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Movement, enemy waves, pending completions, full applyIntent() |
| Modify | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add 12 new E2 tests; update setUp() |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java` | Inject EmulatedConfig; configure wave on joinGame(); sync speed on tick() |
| Modify | `src/main/resources/META-INF/resources/visualizer.html` | Add config panel sidebar |
| Modify | `src/main/resources/META-INF/resources/visualizer.js` | Add `initConfigPanel()` function; call from `init()` |

---

## Task 1: Add cost methods to `SC2Data`

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java`

No new tests — cost values are tested implicitly through EmulatedGameTest in Task 4.

- [ ] **Step 1: Add three static methods to `SC2Data`**

Open `src/main/java/io/quarkmind/domain/SC2Data.java`. Append these three methods after the existing `maxBuildingHealth()` method:

```java
public static int mineralCost(UnitType type) {
    return switch (type) {
        case PROBE    -> 50;
        case ZEALOT   -> 100;
        case STALKER  -> 125;
        case IMMORTAL -> 250;
        case OBSERVER -> 25;
        default       -> 100;
    };
}

public static int mineralCost(BuildingType type) {
    return switch (type) {
        case NEXUS             -> 400;
        case PYLON             -> 100;
        case GATEWAY           -> 150;
        case CYBERNETICS_CORE  -> 150;
        case ASSIMILATOR       -> 75;
        case ROBOTICS_FACILITY -> 200;
        case STARGATE          -> 150;
        case FORGE             -> 150;
        case TWILIGHT_COUNCIL  -> 150;
        default                -> 100;
    };
}

public static int gasCost(UnitType type) {
    return switch (type) {
        case STALKER  -> 50;
        case IMMORTAL -> 100;
        case OBSERVER -> 75;
        default       -> 0;
    };
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd /path/to/worktree && mvn compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/domain/SC2Data.java
git commit -m "feat(domain): add mineral/gas costs to SC2Data for E2 intent handling"
```

---

## Task 2: `EnemyWave` record

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/EnemyWave.java`

- [ ] **Step 1: Create `EnemyWave`**

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import java.util.List;

/**
 * A scheduled enemy spawn event. Added to EmulatedGame via configureWave();
 * consumed by EmulatedGame.tick() when spawnFrame is reached.
 */
record EnemyWave(long spawnFrame, List<UnitType> unitTypes,
                 Point2d spawnPosition, Point2d targetPosition) {}
```

- [ ] **Step 2: Verify compilation**

```bash
mvn compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EnemyWave.java
git commit -m "feat(emulated): add EnemyWave record for scripted enemy spawning"
```

---

## Task 3: `EmulatedConfig` + `EmulatedConfigResource` (TDD)

**Files:**
- Create: `src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java`
- Create: `src/main/java/io/quarkmind/qa/EmulatedConfig.java`
- Create: `src/main/java/io/quarkmind/qa/EmulatedConfigResource.java`

- [ ] **Step 1: Write the failing test**

Create `src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java`:

```java
package io.quarkmind.qa;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class EmulatedConfigResourceTest {

    @Inject EmulatedConfig config;

    @AfterEach
    void resetConfig() {
        // Restore defaults so tests don't leak state
        config.setWaveSpawnFrame(200);
        config.setWaveUnitCount(4);
        config.setWaveUnitType("ZEALOT");
        config.setUnitSpeed(0.5);
    }

    @Test
    void getReturnsDefaultConfig() {
        given()
            .when().get("/qa/emulated/config")
            .then()
            .statusCode(200)
            .body("waveSpawnFrame", equalTo(200))
            .body("waveUnitCount",  equalTo(4))
            .body("waveUnitType",   equalTo("ZEALOT"))
            .body("unitSpeed",      closeTo(0.5, 0.001));
    }

    @Test
    void putUpdatesUnitSpeed() {
        given()
            .contentType("application/json")
            .body("{\"unitSpeed\": 0.8}")
            .when().put("/qa/emulated/config")
            .then()
            .statusCode(200)
            .body("unitSpeed", closeTo(0.8, 0.001));
    }

    @Test
    void putPartialUpdatePreservesOtherFields() {
        given()
            .contentType("application/json")
            .body("{\"waveUnitCount\": 6}")
            .when().put("/qa/emulated/config")
            .then()
            .statusCode(200)
            .body("waveUnitCount",  equalTo(6))
            .body("waveSpawnFrame", equalTo(200));  // unchanged
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -Dtest=EmulatedConfigResourceTest -q 2>&1 | tail -5
```

Expected: `BUILD FAILURE` — `EmulatedConfig` does not exist.

- [ ] **Step 3: Create `EmulatedConfig`**

Create `src/main/java/io/quarkmind/qa/EmulatedConfig.java`:

```java
package io.quarkmind.qa;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

/**
 * Live configuration for EmulatedGame.
 * Layer 1: read from application.properties with hardcoded defaults.
 * Layer 2: runtime-mutable via EmulatedConfigResource (PUT /qa/emulated/config).
 * Layer 3: visualizer config panel calls the REST endpoint.
 */
@ApplicationScoped
public class EmulatedConfig {

    // Layer 1: application.properties with hardcoded fallbacks
    @ConfigProperty(name = "emulated.wave.spawn-frame", defaultValue = "200")
    int defaultWaveSpawnFrame;

    @ConfigProperty(name = "emulated.wave.unit-count", defaultValue = "4")
    int defaultWaveUnitCount;

    @ConfigProperty(name = "emulated.wave.unit-type", defaultValue = "ZEALOT")
    String defaultWaveUnitType;

    @ConfigProperty(name = "emulated.unit.speed", defaultValue = "0.5")
    double defaultUnitSpeed;

    // Layer 2: volatile for thread safety (REST thread writes, scheduler thread reads)
    private volatile int    waveSpawnFrame;
    private volatile int    waveUnitCount;
    private volatile String waveUnitType;
    private volatile double unitSpeed;

    @PostConstruct
    void init() {
        waveSpawnFrame = defaultWaveSpawnFrame;
        waveUnitCount  = defaultWaveUnitCount;
        waveUnitType   = defaultWaveUnitType;
        unitSpeed      = defaultUnitSpeed;
    }

    public int    getWaveSpawnFrame() { return waveSpawnFrame; }
    public int    getWaveUnitCount()  { return waveUnitCount;  }
    public String getWaveUnitType()   { return waveUnitType;   }
    public double getUnitSpeed()      { return unitSpeed;      }

    public void setWaveSpawnFrame(int v)  { this.waveSpawnFrame = v; }
    public void setWaveUnitCount(int v)   { this.waveUnitCount  = v; }
    public void setWaveUnitType(String v) { this.waveUnitType   = v; }
    public void setUnitSpeed(double v)    { this.unitSpeed      = v; }

    /** Serialisable snapshot for the REST response body. */
    public record Snapshot(int waveSpawnFrame, int waveUnitCount,
                           String waveUnitType, double unitSpeed) {}

    public Snapshot snapshot() {
        return new Snapshot(waveSpawnFrame, waveUnitCount, waveUnitType, unitSpeed);
    }
}
```

- [ ] **Step 4: Create `EmulatedConfigResource`**

Create `src/main/java/io/quarkmind/qa/EmulatedConfigResource.java`:

```java
package io.quarkmind.qa;

import io.quarkus.arc.profile.UnlessBuildProfile;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.Map;

@UnlessBuildProfile("prod")
@Path("/qa/emulated/config")
@Produces(MediaType.APPLICATION_JSON)
public class EmulatedConfigResource {

    @Inject EmulatedConfig config;

    @GET
    public EmulatedConfig.Snapshot getConfig() {
        return config.snapshot();
    }

    @PUT
    @Consumes(MediaType.APPLICATION_JSON)
    public Response updateConfig(Map<String, Object> updates) {
        if (updates.containsKey("waveSpawnFrame"))
            config.setWaveSpawnFrame(((Number) updates.get("waveSpawnFrame")).intValue());
        if (updates.containsKey("waveUnitCount"))
            config.setWaveUnitCount(((Number) updates.get("waveUnitCount")).intValue());
        if (updates.containsKey("waveUnitType"))
            config.setWaveUnitType((String) updates.get("waveUnitType"));
        if (updates.containsKey("unitSpeed"))
            config.setUnitSpeed(((Number) updates.get("unitSpeed")).doubleValue());
        return Response.ok(config.snapshot()).build();
    }
}
```

- [ ] **Step 5: Run `EmulatedConfigResourceTest`**

```bash
mvn test -Dtest=EmulatedConfigResourceTest -q
```

Expected: `BUILD SUCCESS`, 3 tests pass.

- [ ] **Step 6: Run full suite**

```bash
mvn test 2>&1 | grep "Tests run: [0-9].*Failures: 0" | tail -1
```

Expected: `Tests run: 213, Failures: 0` (210 existing + 3 new).

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/quarkmind/qa/EmulatedConfig.java \
        src/main/java/io/quarkmind/qa/EmulatedConfigResource.java \
        src/test/java/io/quarkmind/qa/EmulatedConfigResourceTest.java
git commit -m "feat(emulated): add EmulatedConfig with @ConfigProperty defaults and live REST override"
```

---

## Task 4: `EmulatedGame` E2 core (TDD)

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Update `setUp()` and add new imports to `EmulatedGameTest`**

Replace the existing `setUp()` and add imports. The new setUp calls `configureWave()` with a far-out frame (9999) so waves don't interfere with existing E1 tests:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class EmulatedGameTest {

    EmulatedGame game;

    @BeforeEach
    void setUp() {
        game = new EmulatedGame();
        game.configureWave(9999, 4, UnitType.ZEALOT); // defer wave — doesn't fire in E1 tests
        game.reset();
    }

    // ... (all existing tests unchanged) ...
```

Also rename the existing `applyIntentIsNoOp` test to `moveIntentDoesNotChangeUnitCountOrMinerals` (the assertion is still valid but the name was misleading after E2):

```java
    @Test
    void moveIntentDoesNotChangeUnitCountOrMinerals() {
        game.applyIntent(new MoveIntent("probe-0", new Point2d(10, 10)));
        assertThat(game.snapshot().minerals()).isEqualTo(50);
        assertThat(game.snapshot().myUnits()).hasSize(12);
    }
```

- [ ] **Step 2: Add 12 new E2 test methods to `EmulatedGameTest`**

Append these tests after the existing ones (inside the class, before the closing `}`):

```java
    // ---- E2: movement ----

    @Test
    void unitMovesEachTickWhenTargetSet() {
        game.applyIntent(new MoveIntent("probe-0", new Point2d(15, 9)));
        Point2d before = game.snapshot().myUnits().get(0).position();
        game.tick();
        Point2d after = game.snapshot().myUnits().get(0).position();
        assertThat(after.x()).isGreaterThan(before.x()); // moved toward x=15
    }

    @Test
    void unitArrivesAtTarget() {
        // probe-0 starts at (9, 9), target at (9.1, 9) — one step at speed 0.5 overshoots, snaps to target
        game.applyIntent(new MoveIntent("probe-0", new Point2d(9.1f, 9)));
        game.tick();
        Point2d pos = game.snapshot().myUnits().get(0).position();
        assertThat(pos.x()).isCloseTo(9.1f, within(0.01f));
    }

    @Test
    void attackIntentSetsMovementTarget() {
        game.applyIntent(new AttackIntent("probe-0", new Point2d(20, 20)));
        Point2d before = game.snapshot().myUnits().get(0).position();
        game.tick();
        Point2d after = game.snapshot().myUnits().get(0).position();
        // moved toward (20, 20) — both x and y increase
        assertThat(after.x()).isGreaterThan(before.x());
        assertThat(after.y()).isGreaterThan(before.y());
    }

    @Test
    void stepTowardHelperMovesCorrectDistance() {
        Point2d from   = new Point2d(0, 0);
        Point2d to     = new Point2d(10, 0);
        Point2d result = EmulatedGame.stepToward(from, to, 0.5);
        assertThat(result.x()).isCloseTo(0.5f, within(0.001f));
        assertThat(result.y()).isCloseTo(0f,   within(0.001f));
    }

    @Test
    void stepTowardHelperSnapsToTargetWhenCloseEnough() {
        Point2d from   = new Point2d(0, 0);
        Point2d to     = new Point2d(0.3f, 0);
        Point2d result = EmulatedGame.stepToward(from, to, 0.5); // speed > distance
        assertThat(result.x()).isEqualTo(to.x());
        assertThat(result.y()).isEqualTo(to.y());
    }

    // ---- E2: enemy wave ----

    @Test
    void enemySpawnsAtConfiguredFrame() {
        game.configureWave(5, 2, UnitType.ZEALOT);
        game.reset();
        assertThat(game.snapshot().enemyUnits()).isEmpty();
        for (int i = 0; i < 5; i++) game.tick();
        assertThat(game.snapshot().enemyUnits()).hasSize(2);
        assertThat(game.snapshot().enemyUnits().get(0).type()).isEqualTo(UnitType.ZEALOT);
    }

    @Test
    void enemyMovesEachTickTowardNexus() {
        game.configureWave(1, 1, UnitType.ZEALOT);
        game.reset();
        game.tick(); // frame 1 — wave spawns at (26, 26)
        Point2d spawnPos = game.snapshot().enemyUnits().get(0).position();
        game.tick(); // frame 2 — enemy moves toward (8, 8)
        Point2d afterPos = game.snapshot().enemyUnits().get(0).position();
        assertThat(afterPos.x()).isLessThan(spawnPos.x()); // moving left toward nexus
        assertThat(afterPos.y()).isLessThan(spawnPos.y()); // moving down toward nexus
    }

    // ---- E2: train intent ----

    @Test
    void trainIntentDeductsMinerals() {
        game.setMineralsForTesting(200);
        game.applyIntent(new TrainIntent("nexus-0", UnitType.ZEALOT));
        assertThat(game.snapshot().minerals()).isEqualTo(100); // 200 - 100
    }

    @Test
    void trainedUnitAppearsAfterBuildTime() {
        game.setMineralsForTesting(500);
        game.applyIntent(new TrainIntent("nexus-0", UnitType.ZEALOT));
        int before = game.snapshot().myUnits().size();
        // Zealot train time = 28 ticks
        for (int i = 0; i < 28; i++) game.tick();
        assertThat(game.snapshot().myUnits()).hasSize(before + 1);
    }

    @Test
    void trainBlockedIfInsufficientMinerals() {
        // Start with 50 minerals, Zealot costs 100 — should be blocked
        game.applyIntent(new TrainIntent("nexus-0", UnitType.ZEALOT));
        assertThat(game.snapshot().minerals()).isEqualTo(50); // unchanged
    }

    // ---- E2: build intent ----

    @Test
    void buildIntentDeductsMinerals() {
        game.setMineralsForTesting(500);
        game.applyIntent(new BuildIntent("probe-0", BuildingType.PYLON, new Point2d(15, 15)));
        assertThat(game.snapshot().minerals()).isEqualTo(400); // 500 - 100
    }

    @Test
    void buildingCompletesAfterBuildTime() {
        game.setMineralsForTesting(500);
        game.applyIntent(new BuildIntent("probe-0", BuildingType.PYLON, new Point2d(15, 15)));
        int supplyBefore = game.snapshot().supply();
        // Pylon build time = 18 ticks
        for (int i = 0; i < 18; i++) game.tick();
        assertThat(game.snapshot().supply()).isEqualTo(supplyBefore + 8); // +8 from completed Pylon
    }
```

- [ ] **Step 3: Run tests to confirm they fail**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | tail -8
```

Expected: `BUILD FAILURE` — many methods do not exist yet (`configureWave`, `setMineralsForTesting`, `stepToward`, etc.).

- [ ] **Step 4: Implement the full E2 `EmulatedGame`**

Replace the entire content of `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.*;
import org.jboss.logging.Logger;

import java.util.*;

/**
 * Phase E2 physics engine.
 * Probe-driven mineral harvesting, unit movement, scripted enemy waves,
 * and full intent handling (train/build/move/attack).
 * Not a CDI bean — owned and instantiated by {@link EmulatedEngine}.
 */
public class EmulatedGame {

    private static final Logger log = Logger.getLogger(EmulatedGame.class);
    private static final Point2d NEXUS_POS = new Point2d(8, 8);

    // E1 fields
    private double mineralAccumulator;
    private int    miningProbes;
    private int    vespene;
    private int    supply;
    private int    supplyUsed;
    private long   gameFrame;
    private final List<Unit>     myUnits     = new ArrayList<>();
    private final List<Building> myBuildings = new ArrayList<>();
    private final List<Unit>     enemyUnits  = new ArrayList<>();
    private final List<Resource> geysers     = new ArrayList<>();

    // E2 fields
    private double unitSpeed = 0.5;
    private final Map<String, Point2d> unitTargets  = new HashMap<>();
    private final Map<String, Point2d> enemyTargets = new HashMap<>();
    private final List<EnemyWave>         pendingWaves       = new ArrayList<>();
    private final List<PendingCompletion> pendingCompletions = new ArrayList<>();
    private int nextTag = 200;

    private record PendingCompletion(long completesAtTick, Runnable action) {}

    public void reset() {
        mineralAccumulator = SC2Data.INITIAL_MINERALS;
        miningProbes       = SC2Data.INITIAL_PROBES;
        vespene            = SC2Data.INITIAL_VESPENE;
        supply             = SC2Data.INITIAL_SUPPLY;
        supplyUsed         = SC2Data.INITIAL_SUPPLY_USED;
        gameFrame          = 0;
        myUnits.clear();
        myBuildings.clear();
        enemyUnits.clear();
        geysers.clear();
        unitTargets.clear();
        enemyTargets.clear();
        pendingCompletions.clear();
        nextTag = 200;
        // pendingWaves NOT cleared — configured before reset() via configureWave()

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
        moveFriendlyUnits();
        moveEnemyUnits();
        fireCompletions();
        spawnEnemyWaves();
    }

    private void moveFriendlyUnits() {
        myUnits.replaceAll(u -> {
            Point2d target = unitTargets.get(u.tag());
            if (target == null) return u;
            Point2d newPos = stepToward(u.position(), target, unitSpeed);
            if (distance(newPos, target) < 0.2) unitTargets.remove(u.tag());
            return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth());
        });
    }

    private void moveEnemyUnits() {
        enemyUnits.replaceAll(u -> {
            Point2d target = enemyTargets.getOrDefault(u.tag(), NEXUS_POS);
            Point2d newPos = stepToward(u.position(), target, unitSpeed);
            return new Unit(u.tag(), u.type(), newPos, u.health(), u.maxHealth());
        });
    }

    private void fireCompletions() {
        pendingCompletions.removeIf(item -> {
            if (item.completesAtTick() > gameFrame) return false;
            item.action().run();
            return true;
        });
    }

    private void spawnEnemyWaves() {
        pendingWaves.removeIf(wave -> {
            if (wave.spawnFrame() > gameFrame) return false;
            for (int i = 0; i < wave.unitTypes().size(); i++) {
                UnitType type = wave.unitTypes().get(i);
                Point2d pos = new Point2d(wave.spawnPosition().x() + i * 0.5f,
                                          wave.spawnPosition().y());
                String tag = "enemy-" + nextTag++;
                int hp = SC2Data.maxHealth(type);
                enemyUnits.add(new Unit(tag, type, pos, hp, hp));
                enemyTargets.put(tag, wave.targetPosition());
            }
            log.infof("[EMULATED] Enemy wave spawned: %dx%s at frame %d",
                wave.unitTypes().size(), wave.unitTypes().get(0), gameFrame);
            return true;
        });
    }

    public void applyIntent(Intent intent) {
        switch (intent) {
            case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation());
            case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation());
            case TrainIntent  t -> handleTrain(t);
            case BuildIntent  b -> handleBuild(b);
        }
    }

    private void setTarget(String tag, Point2d target) {
        if (myUnits.stream().anyMatch(u -> u.tag().equals(tag))) {
            unitTargets.put(tag, target);
            log.debugf("[EMULATED] %s → (%.1f,%.1f)", tag, target.x(), target.y());
        }
    }

    private void handleTrain(TrainIntent t) {
        int mCost = SC2Data.mineralCost(t.unitType());
        int gCost = SC2Data.gasCost(t.unitType());
        int sCost = SC2Data.supplyCost(t.unitType());
        if ((int) mineralAccumulator < mCost || vespene < gCost || supplyUsed + sCost > supply) {
            log.debugf("[EMULATED] Cannot train %s — insufficient resources", t.unitType());
            return;
        }
        mineralAccumulator -= mCost;
        vespene -= gCost;
        long completesAt = gameFrame + SC2Data.trainTimeInTicks(t.unitType());
        pendingCompletions.add(new PendingCompletion(completesAt, () -> {
            supplyUsed += sCost;
            String tag = "unit-" + nextTag++;
            int hp = SC2Data.maxHealth(t.unitType());
            myUnits.add(new Unit(tag, t.unitType(), new Point2d(9, 9), hp, hp));
            log.debugf("[EMULATED] Trained %s (tag=%s)", t.unitType(), tag);
        }));
    }

    private void handleBuild(BuildIntent b) {
        int mCost = SC2Data.mineralCost(b.buildingType());
        if ((int) mineralAccumulator < mCost) {
            log.debugf("[EMULATED] Cannot build %s — insufficient minerals", b.buildingType());
            return;
        }
        mineralAccumulator -= mCost;
        String tag = "bldg-" + nextTag++;
        BuildingType bt = b.buildingType();
        myBuildings.add(new Building(tag, bt, b.location(),
            SC2Data.maxBuildingHealth(bt), SC2Data.maxBuildingHealth(bt), false));
        long completesAt = gameFrame + SC2Data.buildTimeInTicks(bt);
        pendingCompletions.add(new PendingCompletion(completesAt, () -> {
            markBuildingComplete(tag);
            supply += SC2Data.supplyBonus(bt);
            log.debugf("[EMULATED] Completed %s (tag=%s)", bt, tag);
        }));
    }

    private void markBuildingComplete(String tag) {
        myBuildings.replaceAll(b -> b.tag().equals(tag)
            ? new Building(b.tag(), b.type(), b.position(), b.health(), b.maxHealth(), true)
            : b);
    }

    /** Package-private for testing — linear interpolation toward target. */
    static Point2d stepToward(Point2d from, Point2d to, double speed) {
        double dx = to.x() - from.x();
        double dy = to.y() - from.y();
        double dist = Math.sqrt(dx * dx + dy * dy);
        if (dist <= speed) return to;
        return new Point2d(
            (float)(from.x() + dx * speed / dist),
            (float)(from.y() + dy * speed / dist));
    }

    static double distance(Point2d a, Point2d b) {
        double dx = a.x() - b.x(), dy = a.y() - b.y();
        return Math.sqrt(dx * dx + dy * dy);
    }

    public GameState snapshot() {
        return new GameState(
            (int) mineralAccumulator,  // floor: fractional minerals accumulate silently
            vespene, supply, supplyUsed,
            List.copyOf(myUnits), List.copyOf(myBuildings),
            List.copyOf(enemyUnits), List.copyOf(geysers),
            gameFrame);
    }

    // --- Package-private: called by EmulatedEngine ---

    /** Set unit movement speed in tiles/tick. Called by EmulatedEngine each tick for live config. */
    void setUnitSpeed(double speed) { this.unitSpeed = speed; }

    /** Configure the enemy wave. Call before reset() — pendingWaves survives reset(). */
    void configureWave(long spawnFrame, int unitCount, UnitType unitType) {
        pendingWaves.clear();
        List<UnitType> types = Collections.nCopies(unitCount, unitType);
        pendingWaves.add(new EnemyWave(
            spawnFrame,
            new ArrayList<>(types),
            new Point2d(26, 26),   // enemy spawn — far corner from our nexus
            new Point2d(8, 8)      // target — our nexus
        ));
    }

    // --- Package-private: used by EmulatedGameTest ---

    void setMiningProbes(int count) { this.miningProbes = count; }

    /** Direct mineral override for tests — avoids tick-based accumulation. */
    void setMineralsForTesting(int amount) { this.mineralAccumulator = amount; }
}
```

- [ ] **Step 5: Run `EmulatedGameTest`**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: `BUILD SUCCESS`. All 18 tests pass (6 existing E1 + 12 new E2).

- [ ] **Step 6: Run full suite**

```bash
mvn test 2>&1 | grep "Tests run: [0-9].*Failures: 0" | tail -1
```

Expected: `Tests run: 225, Failures: 0` (213 + 12 new).

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(emulated): E2 — movement, enemy waves, full intent handling with cost deduction"
```

---

## Task 5: `EmulatedEngine` config wiring

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java`

- [ ] **Step 1: Update `EmulatedEngine` to inject config and wire it**

Replace the entire file content:

```java
package io.quarkmind.sc2.emulated;

import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkmind.domain.GameState;
import io.quarkmind.domain.UnitType;
import io.quarkmind.qa.EmulatedConfig;
import io.quarkmind.sc2.IntentQueue;
import io.quarkmind.sc2.SC2Engine;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
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
    private final EmulatedConfig config;
    private final List<Consumer<GameState>> frameListeners = new CopyOnWriteArrayList<>();
    private boolean connected = false;

    @Inject
    public EmulatedEngine(IntentQueue intentQueue, EmulatedConfig config) {
        this.intentQueue = intentQueue;
        this.config      = config;
    }

    @Override
    public void connect() {
        connected = true;
        log.info("[EMULATED] Engine connected");
    }

    @Override
    public void joinGame() {
        // Apply wave config from EmulatedConfig before reset() so pendingWaves is populated
        game.configureWave(
            config.getWaveSpawnFrame(),
            config.getWaveUnitCount(),
            UnitType.valueOf(config.getWaveUnitType()));
        game.reset();
        log.infof("[EMULATED] Joined game — wave at frame %d (%dx%s), speed=%.2f",
            config.getWaveSpawnFrame(), config.getWaveUnitCount(),
            config.getWaveUnitType(), config.getUnitSpeed());
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
        game.setUnitSpeed(config.getUnitSpeed()); // live speed — reads config each tick
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
        intentQueue.drainAll().forEach(intent -> {
            log.debugf("[EMULATED] Dispatching: %s", intent);
            game.applyIntent(intent);
        });
    }

    @Override
    public void addFrameListener(Consumer<GameState> listener) {
        frameListeners.add(listener);
    }
}
```

- [ ] **Step 2: Run full suite**

```bash
mvn test 2>&1 | grep "Tests run: [0-9].*Failures: 0" | tail -1
```

Expected: `Tests run: 225, Failures: 0`.

- [ ] **Step 3: Smoke test the `%emulated` profile**

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

Watch for:
```
[EMULATED] Engine connected
[EMULATED] Joined game — wave at frame 200 (4xZEALOT), speed=0.50
```

Stop with Ctrl+C after confirming.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedEngine.java
git commit -m "feat(emulated): wire EmulatedConfig into EmulatedEngine — live speed, wave config on join"
```

---

## Task 6: Visualizer config panel

**Files:**
- Modify: `src/main/resources/META-INF/resources/visualizer.html`
- Modify: `src/main/resources/META-INF/resources/visualizer.js`

No automated tests — verified by running the app and observing the panel. The panel hides itself in non-`%emulated` profiles (fetch 404 → `display:none`).

- [ ] **Step 1: Update `visualizer.html`** — wrap canvas + add config sidebar

Replace the `<body>` content:

```html
<body>
    <h1>QUARKMIND</h1>
    <div style="display:flex; align-items:flex-start; gap:8px;">
        <div id="game"></div>
        <div id="config-panel" style="display:none; width:180px; background:#0d0d20;
             color:#ccccff; padding:12px; font-family:monospace; font-size:12px;
             border:1px solid #2a2a4e; align-self:stretch;">

            <div style="color:#8888ff; font-size:13px; margin-bottom:10px;">WAVE CONFIG</div>

            <label style="display:block; margin-bottom:2px;">Spawn frame</label>
            <input id="cfg-wave-frame" type="number" value="200" min="1" max="2000"
                   style="width:100%; margin-bottom:8px; background:#1a1a3e; color:#ccc; border:1px solid #333; padding:2px;">

            <label style="display:block; margin-bottom:2px;">Enemy count</label>
            <input id="cfg-unit-count" type="number" value="4" min="1" max="12"
                   style="width:100%; margin-bottom:8px; background:#1a1a3e; color:#ccc; border:1px solid #333; padding:2px;">

            <label style="display:block; margin-bottom:2px;">Unit type</label>
            <select id="cfg-unit-type"
                    style="width:100%; margin-bottom:12px; background:#1a1a3e; color:#ccc; border:1px solid #333; padding:2px;">
                <option value="ZEALOT">Zealot</option>
                <option value="STALKER">Stalker</option>
                <option value="ROACH">Roach</option>
                <option value="MARINE">Marine</option>
            </select>

            <div style="color:#8888ff; font-size:13px; margin-bottom:8px;">PHYSICS</div>
            <label style="display:block; margin-bottom:2px;">Speed: <span id="cfg-speed-val">0.5</span> t/tick</label>
            <input id="cfg-speed" type="range" min="0.05" max="1.5" step="0.05" value="0.5"
                   style="width:100%; margin-bottom:12px;">

            <button id="cfg-apply"
                    style="width:100%; margin-bottom:4px; background:#1a2a4e; color:#ccccff;
                    border:1px solid #444; padding:4px; cursor:pointer; font-family:monospace;">
                Apply Changes
            </button>
            <button id="cfg-restart"
                    style="width:100%; background:#2a1a1e; color:#ffaaaa; border:1px solid #444;
                    padding:4px; cursor:pointer; font-family:monospace;">
                Restart Game
            </button>
            <div id="cfg-status" style="margin-top:8px; font-size:11px; color:#88ff88; min-height:14px;"></div>
        </div>
    </div>
    <script src="/pixi.min.js"></script>
    <script src="/visualizer.js"></script>
</body>
```

- [ ] **Step 2: Add `initConfigPanel()` to `visualizer.js`**

Append this function before the `init()` function (before line `async function init() {`):

```javascript
/**
 * Initialise the config panel sidebar.
 * Probes GET /qa/emulated/config — shows panel only in %emulated profile,
 * hides silently in %mock, %sc2, etc. (endpoint returns 404 in those profiles).
 */
function initConfigPanel() {
    const panel       = document.getElementById('config-panel');
    const speedSlider = document.getElementById('cfg-speed');
    const speedVal    = document.getElementById('cfg-speed-val');
    const status      = document.getElementById('cfg-status');

    // Probe the endpoint — show panel only if it exists (%emulated profile)
    fetch('/qa/emulated/config')
        .then(r => { if (!r.ok) return null; panel.style.display = 'block'; return r.json(); })
        .then(cfg => {
            if (!cfg) return;
            document.getElementById('cfg-wave-frame').value = cfg.waveSpawnFrame;
            document.getElementById('cfg-unit-count').value = cfg.waveUnitCount;
            document.getElementById('cfg-unit-type').value  = cfg.waveUnitType;
            speedSlider.value    = cfg.unitSpeed;
            speedVal.textContent = cfg.unitSpeed;
        })
        .catch(() => {}); // not in %emulated — panel stays hidden

    // Speed is live — sends immediately on slider move (no restart needed)
    speedSlider.addEventListener('input', () => {
        speedVal.textContent = speedSlider.value;
        sendConfig({ unitSpeed: parseFloat(speedSlider.value) });
    });

    // Apply button — sends wave + speed config (wave takes effect on next restart)
    document.getElementById('cfg-apply').addEventListener('click', () => {
        sendConfig(currentConfig()).then(() => showStatus('Applied — restart to activate wave'));
    });

    // Restart — apply config then call /sc2/start
    document.getElementById('cfg-restart').addEventListener('click', () => {
        sendConfig(currentConfig())
            .then(() => fetch('/sc2/start', { method: 'POST' }))
            .then(() => showStatus('Game restarted'))
            .catch(() => showStatus('Restart failed', true));
    });

    function currentConfig() {
        return {
            waveSpawnFrame: parseInt(document.getElementById('cfg-wave-frame').value),
            waveUnitCount:  parseInt(document.getElementById('cfg-unit-count').value),
            waveUnitType:   document.getElementById('cfg-unit-type').value,
            unitSpeed:      parseFloat(speedSlider.value),
        };
    }

    function sendConfig(partial) {
        return fetch('/qa/emulated/config', {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(partial),
        }).then(r => r.json()).catch(() => showStatus('Update failed', true));
    }

    function showStatus(msg, isError = false) {
        status.textContent = msg;
        status.style.color = isError ? '#ff4444' : '#88ff88';
        setTimeout(() => { status.textContent = ''; }, 2500);
    }
}
```

- [ ] **Step 3: Call `initConfigPanel()` at the end of `init()`**

In `init()`, add one line after `connect()`:

```javascript
    await loadAssets();
    connect();
    initConfigPanel();   // ← add this line
}
```

- [ ] **Step 4: Run full suite**

```bash
mvn test 2>&1 | grep "Tests run: [0-9].*Failures: 0" | tail -1
```

Expected: `Tests run: 225, Failures: 0` (no regressions from JS changes).

- [ ] **Step 5: Smoke test the config panel**

```bash
mvn quarkus:dev -Dquarkus.profile=emulated
```

Open `http://localhost:8080/visualizer.html`. Verify:
- Config panel appears to the right of the canvas (black sidebar with WAVE CONFIG, PHYSICS sections)
- Spawn frame / enemy count / unit type / speed slider populated from defaults
- Move speed slider → HUD mineral counter continues; speed change is immediate (no restart)
- Click "Restart Game" → game restarts, wave configured with current values

Also verify panel is hidden in `%mock`:
```bash
mvn quarkus:dev   # default %mock profile
```
Open `http://localhost:8080/visualizer.html` — config panel must not appear.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/resources/visualizer.html \
        src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat(visualizer): add live config panel — wave timing, unit type, speed slider"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `SC2Data.mineralCost(UnitType/BuildingType)`, `gasCost(UnitType)` | Task 1 |
| `EnemyWave` package-private record | Task 2 |
| `EmulatedConfig` — `@ConfigProperty` defaults + volatile runtime fields | Task 3 |
| `EmulatedConfig.Snapshot` serialisable record | Task 3 |
| `EmulatedConfigResource` GET/PUT `/qa/emulated/config` | Task 3 |
| Partial PUT — only updates provided fields | Task 3 |
| `EmulatedGame` — `unitTargets`, `enemyTargets`, `pendingWaves`, `pendingCompletions`, `nextTag` | Task 4 |
| `EmulatedGame.reset()` clears new fields; does NOT clear `pendingWaves` | Task 4 |
| `EmulatedGame.configureWave()` — clears and adds one wave | Task 4 |
| `EmulatedGame.tick()` — movement + completions + wave spawning | Task 4 |
| `moveFriendlyUnits()` + `moveEnemyUnits()` — `replaceAll` pattern | Task 4 |
| `stepToward()` + `distance()` — package-private static helpers | Task 4 |
| `applyIntent()` — full sealed switch (Move/Attack/Train/Build) | Task 4 |
| `handleTrain()` — deducts minerals/gas, defers unit spawn | Task 4 |
| `handleBuild()` — deducts minerals, marks incomplete, defers completion | Task 4 |
| `setMineralsForTesting()` package-private | Task 4 |
| `EmulatedEngine` injects `EmulatedConfig` | Task 5 |
| `EmulatedEngine.joinGame()` calls `configureWave()` before `reset()` | Task 5 |
| `EmulatedEngine.tick()` calls `game.setUnitSpeed(config.getUnitSpeed())` | Task 5 |
| `EmulatedGameTest` — 12 new E2 tests | Task 4 |
| `EmulatedConfigResourceTest` — 3 tests | Task 3 |
| Visualizer HTML config panel (hidden by default, shown via JS fetch) | Task 6 |
| Speed slider — live (no restart) | Task 6 |
| Apply + Restart Game buttons | Task 6 |
| Panel hidden in non-`%emulated` profiles (404 → display:none) | Task 6 |

**Placeholder scan:** None found.

**Type consistency:**
- `configureWave(long, int, UnitType)` used consistently in Task 4 (EmulatedGame) and Task 5 (EmulatedEngine)
- `setUnitSpeed(double)` called from Task 5 (EmulatedEngine.tick()) — defined in Task 4 (EmulatedGame)
- `UnitType.valueOf(config.getWaveUnitType())` in Task 5 — `getWaveUnitType()` returns `String` from Task 3
- `EmulatedConfig.Snapshot` returned by Task 3 REST resource — serialised by Jackson as flat JSON object matching test expectations
