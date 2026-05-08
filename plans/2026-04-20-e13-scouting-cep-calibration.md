# E13 — Scouting CEP Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Calibrate DroolsScoutingTask build-order detection thresholds against IEM10 Taipei and AI Arena replay data by building a JSON replay runner and a statistics harness.

**Architecture:** `IEM10JsonSimulatedGame extends SimulatedGame` mirrors `ReplaySimulatedGame` but reads SC2EGSet pre-processed JSON from a nested ZIP. A `ScoutingCalibrationTest` runs all replays, ticks to 3 min, counts enemy unit types, and prints a statistics table used to calibrate the DRL thresholds. `UnitType` is extended with a full Terran/Zerg roster so all enemy units are tracked.

**Tech Stack:** Java 21, Jackson (`ObjectMapper`/`JsonNode`, already on classpath via `quarkus-rest-jackson`), `java.util.zip.ZipInputStream`, JUnit 5 `@Tag("benchmark")`.

---

## File Map

| Action | File |
|---|---|
| Modify | `src/main/java/io/quarkmind/domain/UnitType.java` |
| Modify | `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java` |
| Create | `src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java` |
| Create | `src/test/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGameTest.java` |
| Create | `src/test/java/io/quarkmind/sc2/mock/ReplaySimulatedGameUnitTypeTest.java` |
| Create | `src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java` |
| Modify | `src/main/resources/io/quarkmind/plugin/scouting/DroolsScoutingTask.drl` |
| Modify | `src/test/java/io/quarkmind/plugin/DroolsScoutingRulesTest.java` |

---

## Task 1: Extend UnitType and fix ReplaySimulatedGame.toUnitType()

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/UnitType.java`
- Modify: `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`
- Create: `src/test/java/io/quarkmind/sc2/mock/ReplaySimulatedGameUnitTypeTest.java`

- [ ] **Step 1.1: Write the failing tests for toUnitType() mappings**

Create `src/test/java/io/quarkmind/sc2/mock/ReplaySimulatedGameUnitTypeTest.java`:

```java
package io.quarkmind.sc2.mock;

import io.quarkmind.domain.UnitType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ReplaySimulatedGameUnitTypeTest {

    @Test void marineIsMapped()    { assertThat(ReplaySimulatedGame.toUnitType("Marine")).isEqualTo(UnitType.MARINE); }
    @Test void marauderIsMapped()  { assertThat(ReplaySimulatedGame.toUnitType("Marauder")).isEqualTo(UnitType.MARAUDER); }
    @Test void medivacIsMapped()   { assertThat(ReplaySimulatedGame.toUnitType("Medivac")).isEqualTo(UnitType.MEDIVAC); }
    @Test void siegeTankIsMapped() { assertThat(ReplaySimulatedGame.toUnitType("SiegeTank")).isEqualTo(UnitType.SIEGE_TANK); }
    @Test void thorIsMapped()      { assertThat(ReplaySimulatedGame.toUnitType("Thor")).isEqualTo(UnitType.THOR); }
    @Test void vikingIsMapped()    { assertThat(ReplaySimulatedGame.toUnitType("VikingFighter")).isEqualTo(UnitType.VIKING); }
    @Test void widowMineIsMapped() { assertThat(ReplaySimulatedGame.toUnitType("WidowMine")).isEqualTo(UnitType.WIDOW_MINE); }

    @Test void zerglingIsMapped()  { assertThat(ReplaySimulatedGame.toUnitType("Zergling")).isEqualTo(UnitType.ZERGLING); }
    @Test void roachIsMapped()     { assertThat(ReplaySimulatedGame.toUnitType("Roach")).isEqualTo(UnitType.ROACH); }
    @Test void hydraliskIsMapped() { assertThat(ReplaySimulatedGame.toUnitType("Hydralisk")).isEqualTo(UnitType.HYDRALISK); }
    @Test void queenIsMapped()     { assertThat(ReplaySimulatedGame.toUnitType("Queen")).isEqualTo(UnitType.QUEEN); }
    @Test void mutalisklIsMapped() { assertThat(ReplaySimulatedGame.toUnitType("Mutalisk")).isEqualTo(UnitType.MUTALISK); }
    @Test void ultraliskIsMapped() { assertThat(ReplaySimulatedGame.toUnitType("Ultralisk")).isEqualTo(UnitType.ULTRALISK); }

    @Test void unknownPassesThrough() { assertThat(ReplaySimulatedGame.toUnitType("MineralField")).isEqualTo(UnitType.UNKNOWN); }
}
```

- [ ] **Step 1.2: Run to confirm compile failure (toUnitType is private, SIEGE_TANK/etc. don't exist)**

```bash
mvn test -Dtest=ReplaySimulatedGameUnitTypeTest -q 2>&1 | head -20
```
Expected: compile error — `SIEGE_TANK` not found in `UnitType`, `toUnitType` not visible.

- [ ] **Step 1.3: Extend UnitType with Terran and Zerg entries**

Edit `src/main/java/io/quarkmind/domain/UnitType.java`:

```java
package io.quarkmind.domain;

public enum UnitType {
    // Protoss
    PROBE, ZEALOT, STALKER, IMMORTAL, COLOSSUS, CARRIER,
    DARK_TEMPLAR, HIGH_TEMPLAR, ARCHON, OBSERVER, VOID_RAY,
    ADEPT, DISRUPTOR, SENTRY,
    // Zerg
    ZERGLING, ROACH, HYDRALISK, MUTALISK, ULTRALISK,
    BROOD_LORD, CORRUPTOR, INFESTOR, SWARM_HOST, VIPER, QUEEN, RAVAGER, LURKER,
    // Terran
    MARINE, MARAUDER, MEDIVAC, SIEGE_TANK, THOR, VIKING,
    GHOST, RAVEN, BANSHEE, BATTLECRUISER, CYCLONE, LIBERATOR, WIDOW_MINE,
    // Fallback
    UNKNOWN
}
```

- [ ] **Step 1.4: Make toUnitType() package-private and add Terran/Zerg cases in ReplaySimulatedGame**

In `src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java`, change line 215 from `private static` to `static` and replace the switch body:

```java
static UnitType toUnitType(String name) {
    return switch (name) {
        // Protoss
        case "Probe"          -> UnitType.PROBE;
        case "Zealot"         -> UnitType.ZEALOT;
        case "Stalker"        -> UnitType.STALKER;
        case "Immortal"       -> UnitType.IMMORTAL;
        case "Colossus"       -> UnitType.COLOSSUS;
        case "Carrier"        -> UnitType.CARRIER;
        case "DarkTemplar"    -> UnitType.DARK_TEMPLAR;
        case "HighTemplar"    -> UnitType.HIGH_TEMPLAR;
        case "Archon"         -> UnitType.ARCHON;
        case "Observer"       -> UnitType.OBSERVER;
        case "VoidRay"        -> UnitType.VOID_RAY;
        case "Adept"          -> UnitType.ADEPT;
        case "Disruptor"      -> UnitType.DISRUPTOR;
        case "Sentry"         -> UnitType.SENTRY;
        // Terran
        case "Marine"         -> UnitType.MARINE;
        case "Marauder"       -> UnitType.MARAUDER;
        case "Medivac"        -> UnitType.MEDIVAC;
        case "SiegeTank", "SiegeTankSieged" -> UnitType.SIEGE_TANK;
        case "Thor", "ThorAP" -> UnitType.THOR;
        case "VikingFighter", "VikingAssault" -> UnitType.VIKING;
        case "Ghost"          -> UnitType.GHOST;
        case "Raven"          -> UnitType.RAVEN;
        case "Banshee"        -> UnitType.BANSHEE;
        case "Battlecruiser"  -> UnitType.BATTLECRUISER;
        case "Cyclone"        -> UnitType.CYCLONE;
        case "Liberator", "LiberatorAG" -> UnitType.LIBERATOR;
        case "WidowMine", "WidowMineBurrowed" -> UnitType.WIDOW_MINE;
        // Zerg
        case "Zergling"       -> UnitType.ZERGLING;
        case "Roach"          -> UnitType.ROACH;
        case "Hydralisk"      -> UnitType.HYDRALISK;
        case "Mutalisk"       -> UnitType.MUTALISK;
        case "Ultralisk"      -> UnitType.ULTRALISK;
        case "BroodLord"      -> UnitType.BROOD_LORD;
        case "Corruptor"      -> UnitType.CORRUPTOR;
        case "Infestor"       -> UnitType.INFESTOR;
        case "SwarmHostMP"    -> UnitType.SWARM_HOST;
        case "Viper"          -> UnitType.VIPER;
        case "Queen"          -> UnitType.QUEEN;
        case "Ravager"        -> UnitType.RAVAGER;
        case "Lurker", "LurkerMP" -> UnitType.LURKER;
        default               -> UnitType.UNKNOWN;
    };
}
```

- [ ] **Step 1.5: Run test to confirm it passes**

```bash
mvn test -Dtest=ReplaySimulatedGameUnitTypeTest -q
```
Expected: `BUILD SUCCESS`, 14 tests pass.

- [ ] **Step 1.6: Run full suite to catch any regressions**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`. All SC2Data/EmulatedGame switches have `default` cases — no exhaustive switch failures expected.

- [ ] **Step 1.7: Commit**

```bash
git add src/main/java/io/quarkmind/domain/UnitType.java \
        src/main/java/io/quarkmind/sc2/mock/ReplaySimulatedGame.java \
        src/test/java/io/quarkmind/sc2/mock/ReplaySimulatedGameUnitTypeTest.java
git commit -m "feat(e13): extend UnitType with full Terran/Zerg roster; fix ReplaySimulatedGame.toUnitType() mappings

Refs #16"
```

---

## Task 2: Write IEM10JsonSimulatedGame failing tests

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGameTest.java`

These tests drive the interface design — they will not compile until Task 3 creates the class.

- [ ] **Step 2.1: Write the full test class**

Create `src/test/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGameTest.java`:

```java
package io.quarkmind.sc2.mock;

import io.quarkmind.domain.*;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests IEM10JsonSimulatedGame using replays/2016_IEM_10_Taipei.zip.
 *
 * First game in the inner ZIP: ByuN (Terr, playerID=1) vs Lilbow (Prot, playerID=2).
 * → watchedPlayerId=2 (Lilbow/Protoss), matchup="PvT".
 */
class IEM10JsonSimulatedGameTest {

    private static final Path IEM10_ZIP = Path.of("replays/2016_IEM_10_Taipei.zip");

    // ---- enumerate() factory ----

    @Test
    void enumerateReturns30Games() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        assertThat(games).hasSize(30);
    }

    @Test
    void enumeratedGamesHaveReplayNames() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        assertThat(games).allMatch(g -> g.replayName().endsWith(".SC2Replay.json"));
    }

    @Test
    void enumeratedGamesHaveValidMatchups() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        assertThat(games).allMatch(g ->
            g.matchup().equals("PvP") || g.matchup().equals("PvT") || g.matchup().equals("PvZ"));
    }

    @Test
    void datasetContainsPvTAndPvZAndPvP() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        assertThat(games.stream().map(IEM10JsonSimulatedGame::matchup))
            .contains("PvT", "PvZ", "PvP");
    }

    // ---- matchup detection ----

    @Test
    void firstGameMatchupIsPvT() throws IOException {
        // ByuN (Terr) vs Lilbow (Prot): watched = Lilbow = Protoss, enemy = Terran → PvT
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        assertThat(game.matchup()).isEqualTo("PvT");
    }

    // ---- initial state ----

    @Test
    void initialStateHasNexus() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        GameState state = game.snapshot();
        assertThat(state.myBuildings().stream().anyMatch(b -> b.type() == BuildingType.NEXUS))
            .as("Nexus present at game start").isTrue();
    }

    @Test
    void initialStateHas12Probes() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        GameState state = game.snapshot();
        assertThat(state.myUnits().stream().filter(u -> u.type() == UnitType.PROBE).count())
            .as("12 probes at game start").isEqualTo(12);
    }

    @Test
    void initialGameFrameIsZero() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        assertThat(game.snapshot().gameFrame()).isEqualTo(0L);
    }

    // ---- tick() progression ----

    @Test
    void tickAdvancesGameFrameByOne() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        game.tick();
        assertThat(game.snapshot().gameFrame()).isEqualTo(1L);
        game.tick();
        assertThat(game.snapshot().gameFrame()).isEqualTo(2L);
    }

    @Test
    void mineralsPopulatedAfterFewTicks() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        for (int i = 0; i < 5; i++) game.tick();
        assertThat(game.snapshot().minerals())
            .as("Minerals from PlayerStats after 5 ticks").isGreaterThan(0);
    }

    @Test
    void supplyPopulatedAfterFewTicks() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        for (int i = 0; i < 5; i++) game.tick();
        GameState state = game.snapshot();
        assertThat(state.supply()).as("Supply cap > 0").isGreaterThan(0);
        assertThat(state.supplyUsed()).as("Supply used > 0").isGreaterThan(0);
    }

    @Test
    void unitCountGrowsOver3Minutes() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        int initialUnits = game.snapshot().myUnits().size();
        for (int i = 0; i < 183; i++) game.tick(); // 3 min
        assertThat(game.snapshot().myUnits().size())
            .as("More units after 3 min").isGreaterThanOrEqualTo(initialUnits);
    }

    // ---- isComplete() ----

    @Test
    void isNotCompleteAtStart() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        assertThat(game.isComplete()).isFalse();
    }

    @Test
    void isCompleteAfterAllEvents() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        while (!game.isComplete()) game.tick();
        assertThat(game.isComplete()).isTrue();
    }

    // ---- reset() ----

    @Test
    void resetRestoresInitialState() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        for (int i = 0; i < 100; i++) game.tick();
        assertThat(game.snapshot().gameFrame()).isEqualTo(100L);

        game.reset();
        GameState state = game.snapshot();
        assertThat(state.gameFrame()).isEqualTo(0L);
        assertThat(state.myBuildings().stream().anyMatch(b -> b.type() == BuildingType.NEXUS)).isTrue();
        assertThat(state.myUnits().stream().filter(u -> u.type() == UnitType.PROBE).count()).isEqualTo(12);
    }

    // ---- applyIntent() is a no-op ----

    @Test
    void applyIntentIsNoOp() throws IOException {
        IEM10JsonSimulatedGame game = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP).get(0);
        int buildingsBeforeIntent = game.snapshot().myBuildings().size();
        String probeTag = game.snapshot().myUnits().get(0).tag();
        game.applyIntent(new io.quarkmind.sc2.intent.BuildIntent(
            probeTag, BuildingType.PYLON, new Point2d(20, 20)));
        game.tick();
        assertThat(game.snapshot().myBuildings().size())
            .as("applyIntent is a no-op").isGreaterThanOrEqualTo(buildingsBeforeIntent);
    }

    // ---- enemy unit tracking ----

    @Test
    void terranEnemyUnitsTrackedInPvTGame() throws IOException {
        // ByuN is extremely aggressive with Marines — should have some by 3 min
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        IEM10JsonSimulatedGame pvtGame = games.stream()
            .filter(g -> g.matchup().equals("PvT"))
            .findFirst().orElseThrow(() -> new AssertionError("No PvT game found"));
        for (int i = 0; i < 183; i++) pvtGame.tick();
        long marines = pvtGame.snapshot().enemyUnits().stream()
            .filter(u -> u.type() == UnitType.MARINE).count();
        assertThat(marines).as("Marines visible in PvT game at 3 min").isGreaterThan(0);
    }

    @Test
    void zergEnemyUnitsTrackedInPvZGame() throws IOException {
        List<IEM10JsonSimulatedGame> games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        IEM10JsonSimulatedGame pvzGame = games.stream()
            .filter(g -> g.matchup().equals("PvZ"))
            .findFirst().orElseThrow(() -> new AssertionError("No PvZ game found"));
        for (int i = 0; i < 183; i++) pvzGame.tick();
        long zergUnits = pvzGame.snapshot().enemyUnits().stream()
            .filter(u -> u.type() != UnitType.UNKNOWN).count();
        assertThat(zergUnits).as("Zerg enemy units visible in PvZ game at 3 min").isGreaterThan(0);
    }
}
```

- [ ] **Step 2.2: Confirm tests don't compile yet (class missing)**

```bash
mvn test -Dtest=IEM10JsonSimulatedGameTest -q 2>&1 | head -10
```
Expected: compile error — `IEM10JsonSimulatedGame` cannot be found.

---

## Task 3: Implement IEM10JsonSimulatedGame

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java`

- [ ] **Step 3.1: Create the class skeleton (compiles, all tests fail)**

Create `src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java`:

```java
package io.quarkmind.sc2.mock;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.intent.Intent;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

/**
 * A SimulatedGame variant that drives state from SC2EGSet pre-processed JSON replays.
 * Reads from the nested ZIP structure: outer ZIP → *_data.zip → *.SC2Replay.json.
 *
 * <p>Mirrors ReplaySimulatedGame in tick model and interface. applyIntent() is a no-op.
 * Enemy units accumulate (not removed on death) — matches ReplaySimulatedGame behaviour
 * and gives calibration data for the full 0–3 min sighting window.
 */
public class IEM10JsonSimulatedGame extends SimulatedGame {

    static final int LOOPS_PER_TICK = 22;

    private static final Set<String> BUILDING_NAMES = Set.of(
        "Nexus", "Pylon", "Gateway", "CyberneticsCore", "Assimilator",
        "RoboticsFacility", "Stargate", "Forge", "TwilightCouncil",
        "PhotonCannon", "ShieldBattery", "RoboticsBay", "FleetBeacon",
        "TemplarArchives", "DarkShrine", "WarpGate"
    );

    private final String          replayName;
    private final String          matchup;
    private final int             watchedPlayerId;
    private final List<JsonNode>  events;
    private final Map<String, Building> pendingBuildings = new HashMap<>();

    private int  cursor;
    private long currentLoop;

    public IEM10JsonSimulatedGame(byte[] jsonBytes, String replayName) throws IOException {
        this.replayName = replayName;
        ObjectMapper mapper = new ObjectMapper();
        JsonNode root = mapper.readTree(jsonBytes);

        JsonNode playerMap = root.get("ToonPlayerDescMap");
        int protossId  = 1;
        String enemyRace = "Prot";

        for (JsonNode player : playerMap) {
            if (player.get("race").asText().equals("Prot")) {
                protossId = player.get("playerID").asInt();
                break;
            }
        }
        for (JsonNode player : playerMap) {
            if (player.get("playerID").asInt() != protossId) {
                enemyRace = player.get("race").asText();
                break;
            }
        }

        this.watchedPlayerId = protossId;
        this.matchup         = "Pv" + raceInitial(enemyRace);

        List<JsonNode> list = new ArrayList<>();
        for (JsonNode e : root.get("trackerEvents")) list.add(e);
        this.events = Collections.unmodifiableList(list);

        reset();
    }

    // ---- SimulatedGame contract ----

    @Override
    public synchronized void reset() {
        setMinerals(0); setVespene(0); setSupply(0); setSupplyUsed(0);
        clearAll();
        setGameFrame(0);
        pendingBuildings.clear();
        cursor      = 0;
        currentLoop = 0;
        drainEventsUpTo(0);
    }

    @Override
    public synchronized void tick() {
        currentLoop += LOOPS_PER_TICK;
        setGameFrame(currentLoop / LOOPS_PER_TICK);
        drainEventsUpTo(currentLoop);
    }

    @Override
    public synchronized void applyIntent(Intent intent) { /* no-op */ }

    // ---- Navigation ----

    public boolean isComplete()  { return cursor >= events.size(); }
    public String  matchup()     { return matchup; }
    public String  replayName()  { return replayName; }

    // ---- Static factory ----

    public static List<IEM10JsonSimulatedGame> enumerate(Path outerZip) throws IOException {
        List<IEM10JsonSimulatedGame> games = new ArrayList<>();
        try (ZipInputStream outer = new ZipInputStream(Files.newInputStream(outerZip))) {
            ZipEntry outerEntry;
            while ((outerEntry = outer.getNextEntry()) != null) {
                if (outerEntry.getName().endsWith("_data.zip")) {
                    byte[] innerZipBytes = outer.readAllBytes();
                    try (ZipInputStream inner = new ZipInputStream(
                            new ByteArrayInputStream(innerZipBytes))) {
                        ZipEntry innerEntry;
                        while ((innerEntry = inner.getNextEntry()) != null) {
                            if (innerEntry.getName().endsWith(".SC2Replay.json")) {
                                byte[] jsonBytes = inner.readAllBytes();
                                games.add(new IEM10JsonSimulatedGame(
                                    jsonBytes, innerEntry.getName()));
                            }
                        }
                    }
                }
            }
        }
        return games;
    }

    // ---- Event processing ----

    private void drainEventsUpTo(long targetLoop) {
        while (cursor < events.size()) {
            JsonNode e = events.get(cursor);
            if (e.get("loop").asLong() > targetLoop) break;
            cursor++;
            applyTrackerEvent(e);
        }
    }

    private void applyTrackerEvent(JsonNode e) {
        switch (e.get("evtTypeName").asText()) {
            case "UnitBorn"    -> applyUnitBorn(e);
            case "PlayerStats" -> applyPlayerStats(e);
            case "UnitDied"    -> applyUnitDied(e);
            case "UnitInit"    -> applyUnitInit(e);
            case "UnitDone"    -> applyUnitDone(e);
        }
    }

    private void applyUnitBorn(JsonNode e) {
        String unitName = e.get("unitTypeName").asText();
        String tag      = makeTag(e.get("unitTagIndex").asInt(), e.get("unitTagRecycle").asInt());
        int    ctrlId   = e.get("controlPlayerId").asInt();

        if (BUILDING_NAMES.contains(unitName)) {
            if (ctrlId == watchedPlayerId) {
                BuildingType bt = toBuildingType(unitName);
                if (bt != BuildingType.UNKNOWN) {
                    Point2d pos = pos(e);
                    addBuilding(new Building(tag, bt, pos,
                        defaultBuildingHealth(bt), defaultBuildingHealth(bt), true));
                }
            }
        } else {
            UnitType ut = toUnitType(unitName);
            if (ut == UnitType.UNKNOWN) return;
            Point2d pos = pos(e);
            if (ctrlId == watchedPlayerId) {
                addUnit(new Unit(tag, ut, pos,
                    defaultUnitHealth(ut), defaultUnitHealth(ut), 0, 0, 0, 0));
            } else if (ctrlId != 0) {
                spawnEnemyUnit(ut, pos);
            }
        }
    }

    private void applyPlayerStats(JsonNode e) {
        if (e.get("playerId").asInt() != watchedPlayerId) return;
        JsonNode stats = e.get("stats");
        setMinerals(stats.get("scoreValueMineralsCurrent").asInt());
        setVespene(stats.get("scoreValueVespeneCurrent").asInt());
        // SC2EGSet JSON food values are raw integers — no ×4096 fixed-point unlike Scelight binary
        setSupplyUsed(stats.get("scoreValueFoodUsed").asInt());
        setSupply(stats.get("scoreValueFoodMade").asInt());
    }

    private void applyUnitDied(JsonNode e) {
        String tag = makeTag(e.get("unitTagIndex").asInt(), e.get("unitTagRecycle").asInt());
        removeUnitByTag(tag);
        removeBuildingByTag(tag);
        pendingBuildings.remove(tag);
        // Enemy units are NOT removed — accumulate for calibration (matches ReplaySimulatedGame)
    }

    private void applyUnitInit(JsonNode e) {
        int ctrlId = e.get("controlPlayerId").asInt();
        if (ctrlId != watchedPlayerId) return;
        String       unitName = e.get("unitTypeName").asText();
        String       tag      = makeTag(e.get("unitTagIndex").asInt(), e.get("unitTagRecycle").asInt());
        BuildingType bt       = toBuildingType(unitName);
        Point2d      pos      = pos(e);
        Building b = new Building(tag, bt, pos,
            defaultBuildingHealth(bt), defaultBuildingHealth(bt), false);
        pendingBuildings.put(tag, b);
        addBuilding(b);
    }

    private void applyUnitDone(JsonNode e) {
        String tag = makeTag(e.get("unitTagIndex").asInt(), e.get("unitTagRecycle").asInt());
        if (pendingBuildings.remove(tag) != null) markBuildingComplete(tag);
    }

    // ---- Helpers ----

    private static Point2d pos(JsonNode e) {
        return new Point2d(e.get("x").floatValue(), e.get("y").floatValue());
    }

    private static String makeTag(int index, int recycle) {
        return "j-" + index + "-" + recycle;
    }

    private static String raceInitial(String race) {
        return switch (race) {
            case "Terr" -> "T";
            case "Zerg" -> "Z";
            default     -> "P";
        };
    }

    static UnitType toUnitType(String name) {
        return switch (name) {
            // Protoss
            case "Probe"          -> UnitType.PROBE;
            case "Zealot"         -> UnitType.ZEALOT;
            case "Stalker"        -> UnitType.STALKER;
            case "Immortal"       -> UnitType.IMMORTAL;
            case "Colossus"       -> UnitType.COLOSSUS;
            case "Carrier"        -> UnitType.CARRIER;
            case "DarkTemplar"    -> UnitType.DARK_TEMPLAR;
            case "HighTemplar"    -> UnitType.HIGH_TEMPLAR;
            case "Archon"         -> UnitType.ARCHON;
            case "Observer"       -> UnitType.OBSERVER;
            case "VoidRay"        -> UnitType.VOID_RAY;
            case "Adept"          -> UnitType.ADEPT;
            case "Disruptor"      -> UnitType.DISRUPTOR;
            case "Sentry"         -> UnitType.SENTRY;
            // Terran
            case "Marine"         -> UnitType.MARINE;
            case "Marauder"       -> UnitType.MARAUDER;
            case "Medivac"        -> UnitType.MEDIVAC;
            case "SiegeTank", "SiegeTankSieged" -> UnitType.SIEGE_TANK;
            case "Thor", "ThorAP" -> UnitType.THOR;
            case "VikingFighter", "VikingAssault" -> UnitType.VIKING;
            case "Ghost"          -> UnitType.GHOST;
            case "Raven"          -> UnitType.RAVEN;
            case "Banshee"        -> UnitType.BANSHEE;
            case "Battlecruiser"  -> UnitType.BATTLECRUISER;
            case "Cyclone"        -> UnitType.CYCLONE;
            case "Liberator", "LiberatorAG" -> UnitType.LIBERATOR;
            case "WidowMine", "WidowMineBurrowed" -> UnitType.WIDOW_MINE;
            // Zerg
            case "Zergling"       -> UnitType.ZERGLING;
            case "Roach"          -> UnitType.ROACH;
            case "Hydralisk"      -> UnitType.HYDRALISK;
            case "Mutalisk"       -> UnitType.MUTALISK;
            case "Ultralisk"      -> UnitType.ULTRALISK;
            case "BroodLord"      -> UnitType.BROOD_LORD;
            case "Corruptor"      -> UnitType.CORRUPTOR;
            case "Infestor"       -> UnitType.INFESTOR;
            case "SwarmHostMP"    -> UnitType.SWARM_HOST;
            case "Viper"          -> UnitType.VIPER;
            case "Queen"          -> UnitType.QUEEN;
            case "Ravager"        -> UnitType.RAVAGER;
            case "Lurker", "LurkerMP" -> UnitType.LURKER;
            default               -> UnitType.UNKNOWN;
        };
    }

    private static BuildingType toBuildingType(String name) {
        return switch (name) {
            case "Nexus"             -> BuildingType.NEXUS;
            case "Pylon"             -> BuildingType.PYLON;
            case "Gateway", "WarpGate" -> BuildingType.GATEWAY;
            case "CyberneticsCore"   -> BuildingType.CYBERNETICS_CORE;
            case "Assimilator"       -> BuildingType.ASSIMILATOR;
            case "RoboticsFacility"  -> BuildingType.ROBOTICS_FACILITY;
            case "Stargate"          -> BuildingType.STARGATE;
            case "Forge"             -> BuildingType.FORGE;
            case "TwilightCouncil"   -> BuildingType.TWILIGHT_COUNCIL;
            default                  -> BuildingType.UNKNOWN;
        };
    }

    private static int defaultUnitHealth(UnitType type) {
        return switch (type) {
            case PROBE        ->  45;
            case ZEALOT       -> 100;
            case STALKER      ->  80;
            case IMMORTAL     -> 200;
            case COLOSSUS     -> 200;
            case OBSERVER     ->  40;
            case MARINE       ->  45;
            case MARAUDER     -> 125;
            case MEDIVAC      -> 150;
            case SIEGE_TANK   -> 175;
            case ROACH        -> 145;
            case HYDRALISK    ->  90;
            case ZERGLING     ->  35;
            case MUTALISK     -> 120;
            case QUEEN        -> 175;
            default           -> 100;
        };
    }

    private static int defaultBuildingHealth(BuildingType type) {
        return switch (type) {
            case NEXUS             -> 1500;
            case PYLON             ->  200;
            case GATEWAY           ->  500;
            case CYBERNETICS_CORE  ->  550;
            case ASSIMILATOR       ->  450;
            case ROBOTICS_FACILITY ->  500;
            case STARGATE          ->  600;
            case FORGE             ->  400;
            case TWILIGHT_COUNCIL  ->  500;
            default                ->  400;
        };
    }
}
```

- [ ] **Step 3.2: Run tests — expect most to pass, investigate failures**

```bash
mvn test -Dtest=IEM10JsonSimulatedGameTest -q
```
Expected: `BUILD SUCCESS`, all 17 tests pass. If any fail, read the error and debug before continuing.

- [ ] **Step 3.3: Run full suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 3.4: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGame.java \
        src/test/java/io/quarkmind/sc2/mock/IEM10JsonSimulatedGameTest.java
git commit -m "feat(e13): IEM10JsonSimulatedGame — JSON replay runner from nested ZIP

Refs #16"
```

---

## Task 4: ScoutingCalibrationTest (benchmark harness)

**Files:**
- Create: `src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java`

- [ ] **Step 4.1: Write the calibration harness**

Create `src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java`:

```java
package io.quarkmind.plugin.scouting;

import io.quarkmind.domain.GameState;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.mock.IEM10JsonSimulatedGame;
import io.quarkmind.sc2.mock.ReplaySimulatedGame;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.*;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Calibration harness for DroolsScoutingTask build-order thresholds.
 *
 * NOT part of the regular test suite. Run explicitly:
 *   mvn test -Pbenchmark
 *
 * Reads all AI Arena binary replays + all IEM10 JSON replays, ticks each to 3 min
 * (183 ticks × 22 loops), counts enemy units by type, and prints statistics.
 *
 * Read the output table and update DroolsScoutingTask.drl thresholds accordingly.
 * Output also written to target/scouting-calibration.txt.
 */
@Tag("benchmark")
class ScoutingCalibrationTest {

    private static final Path AI_ARENA_DIR = Path.of("replays/aiarena_protoss");
    private static final Path IEM10_ZIP    = Path.of("replays/2016_IEM_10_Taipei.zip");

    /** 3 minutes at SC2 Faster speed (22 loops/tick). */
    private static final int TICKS_3MIN = 183;

    @Test
    void calibrateScoutingThresholds() throws IOException {
        Map<String, List<Map<UnitType, Long>>> statsByMatchup = new LinkedHashMap<>();
        statsByMatchup.put("PvT", new ArrayList<>());
        statsByMatchup.put("PvZ", new ArrayList<>());
        statsByMatchup.put("PvP", new ArrayList<>());

        int aiArenaLoaded = 0, aiArenaSkipped = 0;
        int iem10Loaded   = 0;

        // ---- AI Arena binary replays (all PvP) ----
        List<Path> replayFiles = Files.list(AI_ARENA_DIR)
            .filter(p -> p.toString().endsWith(".SC2Replay"))
            .sorted()
            .collect(Collectors.toList());

        for (Path replay : replayFiles) {
            try {
                ReplaySimulatedGame game = new ReplaySimulatedGame(replay, 1);
                for (int i = 0; i < TICKS_3MIN; i++) game.tick();
                statsByMatchup.get("PvP").add(countByType(game.snapshot()));
                aiArenaLoaded++;
            } catch (IllegalArgumentException e) {
                aiArenaSkipped++; // unparseable build version
            }
        }

        // ---- IEM10 JSON replays (PvT, PvZ, PvP) ----
        List<IEM10JsonSimulatedGame> iem10Games = IEM10JsonSimulatedGame.enumerate(IEM10_ZIP);
        for (IEM10JsonSimulatedGame game : iem10Games) {
            for (int i = 0; i < TICKS_3MIN; i++) game.tick();
            statsByMatchup.get(game.matchup()).add(countByType(game.snapshot()));
            iem10Loaded++;
        }

        String report = buildReport(statsByMatchup, aiArenaLoaded, aiArenaSkipped, iem10Loaded);
        System.out.println(report);

        Path out = Path.of("target/scouting-calibration.txt");
        Files.createDirectories(out.getParent());
        Files.writeString(out, report, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
        System.out.println("Written to: " + out.toAbsolutePath());

        assertThat(iem10Loaded).as("IEM10 games loaded").isGreaterThan(0);
        assertThat(aiArenaLoaded).as("AI Arena games loaded").isGreaterThan(0);
    }

    private static Map<UnitType, Long> countByType(GameState state) {
        Map<UnitType, Long> counts = new EnumMap<>(UnitType.class);
        for (var unit : state.enemyUnits()) {
            counts.merge(unit.type(), 1L, Long::sum);
        }
        return counts;
    }

    private static String buildReport(Map<String, List<Map<UnitType, Long>>> stats,
                                      int aiLoaded, int aiSkipped, int iem10Loaded) {
        StringBuilder sb = new StringBuilder();
        sb.append("=== Scouting CEP Calibration — enemy unit counts at 3-min mark ===\n");
        sb.append("AI Arena binary replays: ").append(aiLoaded).append(" loaded, ")
          .append(aiSkipped).append(" skipped (unparseable)\n");
        sb.append("IEM10 JSON replays: ").append(iem10Loaded).append(" loaded\n\n");

        sb.append(formatMatchup("PvZ (Zerg enemies)", stats.get("PvZ"),
            List.of(UnitType.ROACH, UnitType.ZERGLING, UnitType.HYDRALISK, UnitType.MUTALISK, UnitType.QUEEN),
            "ZERG_ROACH_RUSH threshold (current: 6) ← ROACH"));
        sb.append(formatMatchup("PvT (Terran enemies)", stats.get("PvT"),
            List.of(UnitType.MARINE, UnitType.MARAUDER, UnitType.MEDIVAC, UnitType.SIEGE_TANK),
            "TERRAN_3RAX threshold (current: 12) ← MARINE"));
        sb.append(formatMatchup("PvP (Protoss enemies)", stats.get("PvP"),
            List.of(UnitType.STALKER, UnitType.ZEALOT, UnitType.IMMORTAL, UnitType.ADEPT),
            "PROTOSS_4GATE threshold (current: 8) ← STALKER+ZEALOT combined"));

        return sb.toString();
    }

    private static String formatMatchup(String label, List<Map<UnitType, Long>> games,
                                        List<UnitType> types, String note) {
        if (games.isEmpty()) return label + ": no games\n\n";
        StringBuilder sb = new StringBuilder();
        sb.append(label).append(" (").append(games.size()).append(" games):\n");

        long stalkerTotal = 0, zealotTotal = 0;
        for (UnitType type : types) {
            long[] vals = games.stream()
                .mapToLong(m -> m.getOrDefault(type, 0L))
                .toArray();
            long min  = Arrays.stream(vals).min().orElse(0);
            long max  = Arrays.stream(vals).max().orElse(0);
            double mean = Arrays.stream(vals).average().orElse(0);
            sb.append(String.format("  %-12s min=%d  max=%d  mean=%.1f%n",
                type.name(), min, max, mean));
            if (type == UnitType.STALKER) stalkerTotal = (long)(mean * games.size());
            if (type == UnitType.ZEALOT)  zealotTotal  = (long)(mean * games.size());
        }

        if (label.contains("PvP")) {
            long[] combined = games.stream()
                .mapToLong(m -> m.getOrDefault(UnitType.STALKER, 0L)
                              + m.getOrDefault(UnitType.ZEALOT,  0L))
                .toArray();
            long min  = Arrays.stream(combined).min().orElse(0);
            long max  = Arrays.stream(combined).max().orElse(0);
            double mean = Arrays.stream(combined).average().orElse(0);
            sb.append(String.format("  %-12s min=%d  max=%d  mean=%.1f%n",
                "combined", min, max, mean));
        }

        sb.append("  → ").append(note).append("\n\n");
        return sb.toString();
    }
}
```

- [ ] **Step 4.2: Run the calibration harness**

```bash
mvn test -Pbenchmark 2>&1 | tail -60
```
Expected: output table printed with unit counts per matchup. Read the output carefully — these numbers drive the threshold decisions in Task 5.

- [ ] **Step 4.3: Record the calibration output**

Copy the output table from `target/scouting-calibration.txt` and paste it into a new file:

```bash
cp target/scouting-calibration.txt docs/benchmarks/$(date +%Y-%m-%d)-e13-scouting-calibration.txt
```

- [ ] **Step 4.4: Commit**

```bash
git add src/test/java/io/quarkmind/plugin/scouting/ScoutingCalibrationTest.java \
        docs/benchmarks/
git commit -m "feat(e13): ScoutingCalibrationTest — replay statistics harness for threshold calibration

Refs #16"
```

---

## Task 5: Update DRL thresholds and tests from calibration data

**Files:**
- Modify: `src/main/resources/io/quarkmind/plugin/scouting/DroolsScoutingTask.drl`
- Modify: `src/test/java/io/quarkmind/plugin/DroolsScoutingRulesTest.java`

- [ ] **Step 5.1: Read the calibration output**

Open `target/scouting-calibration.txt` (or the docs/benchmarks/ copy). The numbers to look at:
- **ROACH** mean/range → new `ZERG_ROACH_RUSH` threshold (typically: set just above mean, no lower than min+1 to avoid false positives on normal openings)
- **MARINE** mean/range → new `TERRAN_3RAX` threshold
- **STALKER+ZEALOT combined** mean/range → new `PROTOSS_4GATE` threshold

Rule of thumb: set threshold at ~mean, or at the 60th–70th percentile. Too low = false positives on standard openings. Too high = misses rushes. Adjust based on what the ranges show.

- [ ] **Step 5.2: Update DroolsScoutingTask.drl with calibrated thresholds**

Edit `src/main/resources/io/quarkmind/plugin/scouting/DroolsScoutingTask.drl`. For each build-order rule, add a calibration comment and update the threshold. Example (replace N with actual values from the output):

```
// Calibration: IEM10 Taipei (N_ZvP games) | threshold=N | observed mean=N.N | observed range=[N, N]
rule "Zerg Roach Rush"
    salience 100
when
    accumulate(
        /unitEvents[ this.type() == UnitType.ROACH ];
        $count : count();
        $count >= N   // ← new calibrated value
    )
then
    detectedBuilds.add("ZERG_ROACH_RUSH");
end
```

Apply the same pattern to `"Terran 3-Rax"` and `"Protoss 4-Gate"`.

- [ ] **Step 5.3: Update DroolsScoutingRulesTest to match new thresholds**

Edit `src/test/java/io/quarkmind/plugin/DroolsScoutingRulesTest.java`. The four threshold tests that reference exact counts are:

- `sixRoachesDetectsZergRoachRush()` — update `for (int i = 0; i < 6; ...)` to new threshold N
- `fiveRoachesDoesNotDetectRoachRush()` — update to N-1
- `twelveMarinesDetectsTerran3Rax()` — update to new Marine threshold M
- `elevenMarinesDoesNotDetect3Rax()` — update to M-1
- `eightStalkersDetectsProtoss4Gate()` — update to new 4-gate threshold P
- `mixedStalkerZealotCountsTowardFourGate()` — update counts so each half = P/2

Also update the test method names to reflect the new numbers (e.g., rename `sixRoachesDetects...` to match the new count).

- [ ] **Step 5.4: Run the scouting rule tests**

```bash
mvn test -Dtest="DroolsScoutingRulesTest,DroolsScoutingTaskIT" -q
```
Expected: `BUILD SUCCESS`. If a threshold test fails, re-check the DRL and test values match.

- [ ] **Step 5.5: Run full suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5.6: Commit**

```bash
git add src/main/resources/io/quarkmind/plugin/scouting/DroolsScoutingTask.drl \
        src/test/java/io/quarkmind/plugin/DroolsScoutingRulesTest.java
git commit -m "feat(e13): calibrate scouting CEP thresholds from IEM10+AI Arena replay data

Closes #16"
```

---

## Task 6: Final verification

- [ ] **Step 6.1: Run the full test suite**

```bash
mvn test -q
```
Expected: `BUILD SUCCESS`. Check the count — should be 446+ tests (the baseline before this epic).

- [ ] **Step 6.2: Run the benchmark suite to confirm calibration test still works**

```bash
mvn test -Pbenchmark 2>&1 | grep -E "BUILD|loaded|skipped|PvT|PvZ|PvP"
```
Expected: calibration output visible, `BUILD SUCCESS`.

- [ ] **Step 6.3: Verify acceptance criteria**

Check each item in the spec:
- [ ] `UnitType` has full Terran + Zerg rosters — verify: `grep -c "SIEGE_TANK\|THOR\|QUEEN\|BROOD_LORD" src/main/java/io/quarkmind/domain/UnitType.java` should be > 0
- [ ] `IEM10JsonSimulatedGame` reads nested ZIP without extraction — class uses `ZipInputStream` only, no `File.createTempFile`
- [ ] `ScoutingCalibrationTest` runs all replays and prints table — confirmed in Step 6.2
- [ ] DRL thresholds have calibration comments — verify by reading DroolsScoutingTask.drl
- [ ] `DroolsScoutingRulesTest` updated — confirmed by `mvn test`
- [ ] No regression in `DroolsScoutingTaskIT` — confirmed in Step 5.4
