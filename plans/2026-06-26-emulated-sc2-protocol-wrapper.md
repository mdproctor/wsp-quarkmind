# EmulatedGame SC2 Protocol Wrapper — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wrap EmulatedGame in a server-side SC2 WebSocket/protobuf layer so RealSC2Engine connects to it identically to real SC2. New `%emulated-sc2` profile — no SC2 installation required.

**Architecture:** Promote `FakeSC2Server` (test inner class) into a production CDI bean (`EmulatedSC2Server`) backed by `EmulatedGame`. Two pure-function translators bridge GameState ↔ protobuf. `RealSC2Engine`, `QuarkusSC2Transport`, and `SC2BotAgent` are reused unchanged — only their `@IfBuildProfile` gates widen. A shared `SC2WebSocketCodec` deduplicates WebSocket framing across client, test server, and emulated server.

**Tech Stack:** Java 21, Quarkus 3.34.2, SC2 protobuf (`SC2APIProtocol.*`), ocraft-s2client-protocol 0.4.21

**Spec:** `docs/superpowers/specs/2026-06-25-emulated-sc2-protocol-wrapper-design.md` (revision 4)

## Global Constraints

- Java 21 language features on Java 26 JVM
- `domain/` must remain plain Java — no CDI, no Quarkus imports
- Never `@QuarkusTest` for tests that can be plain JUnit
- All commits reference issue: `Refs #171`
- Run `mvn test -q` after each task to verify no regressions across all 1006+ existing tests

---

### Task 1: TerrainGrid.toPathingGrid()

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/TerrainGrid.java`
- Test: `src/test/java/io/quarkmind/domain/TerrainGridTest.java` (existing file — add test)

**Interfaces:**
- Consumes: `TerrainGrid.emulatedMap()`, `TerrainGrid.fromPathingGrid(byte[], int, int)`, `TerrainGrid.isWalkable(int, int)`, `TerrainGrid.width()`, `TerrainGrid.height()`
- Produces: `TerrainGrid.toPathingGrid() → byte[]` — encodes walkability into SC2 bit-packed bitmap. Used by Task 6 (EmulatedSC2Server) to populate StartRaw.pathingGrid in ResponseGameInfo.

- [ ] **Step 1: Write the failing round-trip test**

Add to `src/test/java/io/quarkmind/domain/TerrainGridTest.java`:

```java
@Test
void toPathingGrid_roundTripsWithFromPathingGrid() {
    TerrainGrid original = TerrainGrid.emulatedMap();
    byte[] encoded = original.toPathingGrid();

    // 64×64 at 1 bpp = 64*64/8 = 512 bytes
    assertThat(encoded).hasSize(512);

    TerrainGrid roundTripped = TerrainGrid.fromPathingGrid(encoded, 64, 64);
    for (int x = 0; x < 64; x++) {
        for (int y = 0; y < 64; y++) {
            assertThat(roundTripped.isWalkable(x, y))
                .as("walkability at (%d,%d)", x, y)
                .isEqualTo(original.isWalkable(x, y));
        }
    }
}

@Test
void toPathingGrid_wallTilesEncodeAsZero_walkableTilesEncodeAsOne() {
    TerrainGrid grid = TerrainGrid.emulatedMap();
    byte[] encoded = grid.toPathingGrid();

    // y=18, x=5 is WALL → bit should be 0
    int wallIndex = 5 + 18 * 64;
    int wallBit = (encoded[wallIndex / 8] >> (7 - wallIndex % 8)) & 1;
    assertThat(wallBit).as("wall tile (5,18)").isEqualTo(0);

    // y=18, x=12 is RAMP → bit should be 1
    int rampIndex = 12 + 18 * 64;
    int rampBit = (encoded[rampIndex / 8] >> (7 - rampIndex % 8)) & 1;
    assertThat(rampBit).as("ramp tile (12,18)").isEqualTo(1);

    // y=10, x=30 is LOW → bit should be 1
    int lowIndex = 30 + 10 * 64;
    int lowBit = (encoded[lowIndex / 8] >> (7 - lowIndex % 8)) & 1;
    assertThat(lowBit).as("low tile (30,10)").isEqualTo(1);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=TerrainGridTest#toPathingGrid_roundTripsWithFromPathingGrid -q`
Expected: FAIL — `toPathingGrid()` does not exist

- [ ] **Step 3: Implement toPathingGrid()**

Add to `src/main/java/io/quarkmind/domain/TerrainGrid.java` after the `fromPathingGrid` method:

```java
/**
 * Encodes this grid's walkability into SC2's bit-packed pathing grid format.
 * Inverse of {@link #fromPathingGrid(byte[], int, int)}.
 * Walkable tiles (LOW, RAMP, HIGH) encode as 1; WALL encodes as 0.
 * Bit encoding: index = x + y*width; bit = (data[index/8] >> (7 - index%8)) & 1
 */
public byte[] toPathingGrid() {
    int totalBits = width * height;
    byte[] data = new byte[(totalBits + 7) / 8];
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            if (isWalkable(x, y)) {
                int index = x + y * width;
                data[index / 8] |= (byte) (1 << (7 - index % 8));
            }
        }
    }
    return data;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=TerrainGridTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS (1006+ tests)

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/domain/TerrainGrid.java src/test/java/io/quarkmind/domain/TerrainGridTest.java
git commit -m "feat(#171): add TerrainGrid.toPathingGrid() — inverse of fromPathingGrid

Refs #171"
```

---

### Task 2: GameStateToProtobuf translator + observation round-trip test

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/server/GameStateToProtobuf.java`
- Create: `src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java`

**Interfaces:**
- Consumes: `GameState` record (minerals, vespene, supply, supplyUsed, myUnits, myBuildings, enemyUnits, enemyBuildings, enemyStagingArea, geysers, mineralPatches, gameFrame). `ObservationTranslator.translate(Observation)`. `ObservationTranslator.mapUnitType(Units)` / `mapBuildingType(Units)` for reverse map construction. `ResponseObservation.from(Sc2Api.Response)` for ocraft validation.
- Produces: `GameStateToProtobuf.translate(GameState) → Sc2Api.ResponseObservation` — builds a complete protobuf observation that survives ocraft parsing. Used by Task 6 (EmulatedSC2Server) for `RequestObservation` responses. Also produces `GameStateToProtobuf.UNIT_TYPE_TO_PROTO` and `BUILDING_TYPE_TO_PROTO` reverse lookup maps — `EnumMap<UnitType, Units>` and `EnumMap<BuildingType, Units>`.

- [ ] **Step 1: Write the failing observation round-trip test**

Create `src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java`:

```java
package io.quarkmind.sc2.emulated.server;

import SC2APIProtocol.Sc2Api;
import com.github.ocraft.s2client.protocol.observation.Observation;
import com.github.ocraft.s2client.protocol.response.ResponseObservation;
import io.quarkmind.domain.*;
import io.quarkmind.sc2.emulated.EmulatedGame;
import io.quarkmind.sc2.real.ObservationTranslator;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

class GameStateRoundTripTest {

    @Test
    void observationRoundTrip_survivesOcraftParsing() {
        EmulatedGame game = new EmulatedGame();
        game.reset();
        for (int i = 0; i < 10; i++) game.tick();
        GameState original = game.snapshot();

        Sc2Api.ResponseObservation protoObs = GameStateToProtobuf.translate(original);
        Sc2Api.Response response = Sc2Api.Response.newBuilder()
            .setObservation(protoObs)
            .setStatus(Sc2Api.Status.in_game)
            .build();

        // Through ocraft parsing — catches malformed protobuf
        ResponseObservation ro = ResponseObservation.from(response);
        Observation obs = ro.getObservation();
        GameState roundTripped = ObservationTranslator.translate(obs);

        // Preserved fields
        assertThat(roundTripped.minerals()).isEqualTo(original.minerals());
        assertThat(roundTripped.vespene()).isEqualTo(original.vespene());
        assertThat(roundTripped.supply()).isEqualTo(original.supply());
        assertThat(roundTripped.supplyUsed()).isEqualTo(original.supplyUsed());
        assertThat(roundTripped.gameFrame()).isEqualTo(original.gameFrame());
        assertThat(roundTripped.myUnits()).hasSize(original.myUnits().size());
        assertThat(roundTripped.myBuildings()).hasSize(original.myBuildings().size());

        // Lossy fields — enemies collapse into enemyUnits
        int totalEnemies = original.enemyUnits().size()
            + original.enemyBuildings().size()
            + original.enemyStagingArea().size();
        assertThat(roundTripped.enemyUnits()).hasSize(totalEnemies);
        assertThat(roundTripped.enemyBuildings()).isEmpty();
        assertThat(roundTripped.enemyStagingArea()).isEmpty();

        // Lossy fields — not in SC2 protobuf
        assertThat(roundTripped.geysers()).isEmpty();
        roundTripped.myUnits().forEach(u -> {
            assertThat(u.weaponCooldownTicks()).as("cooldown zeroed").isZero();
            assertThat(u.blinkCooldownTicks()).as("blink cooldown zeroed").isZero();
        });
    }

    @Test
    void observationRoundTrip_preservesUnitPositionAndHealth() {
        GameState original = new GameState(
            400, 200, 30, 22,
            List.of(new Unit("1", UnitType.STALKER, new Point2d(10f, 20f), 80, 80, 80, 80, 0, 0)),
            List.of(new Building("2", BuildingType.NEXUS, new Point2d(30f, 30f), 1000, 1000, true)),
            List.of(new Unit("3", UnitType.ZEALOT, new Point2d(40f, 40f), 100, 100, 50, 50, 0, 0)),
            List.of(),
            List.of(),
            List.of(),
            List.of(),
            150L
        );

        Sc2Api.ResponseObservation protoObs = GameStateToProtobuf.translate(original);
        Sc2Api.Response response = Sc2Api.Response.newBuilder()
            .setObservation(protoObs).setStatus(Sc2Api.Status.in_game).build();
        ResponseObservation ro = ResponseObservation.from(response);
        GameState rt = ObservationTranslator.translate(ro.getObservation());

        Unit rtStalker = rt.myUnits().get(0);
        assertThat(rtStalker.type()).isEqualTo(UnitType.STALKER);
        assertThat(rtStalker.position().x()).isCloseTo(10f, within(0.01f));
        assertThat(rtStalker.position().y()).isCloseTo(20f, within(0.01f));
        assertThat(rtStalker.health()).isEqualTo(80);
        assertThat(rtStalker.shields()).isEqualTo(80);

        Building rtNexus = rt.myBuildings().get(0);
        assertThat(rtNexus.type()).isEqualTo(BuildingType.NEXUS);
        assertThat(rtNexus.isComplete()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=GameStateRoundTripTest -q`
Expected: FAIL — `GameStateToProtobuf` does not exist

- [ ] **Step 3: Create the package directory**

Run: `mkdir -p src/main/java/io/quarkmind/sc2/emulated/server`

- [ ] **Step 4: Implement GameStateToProtobuf**

Create `src/main/java/io/quarkmind/sc2/emulated/server/GameStateToProtobuf.java`:

```java
package io.quarkmind.sc2.emulated.server;

import SC2APIProtocol.Common;
import SC2APIProtocol.Raw;
import SC2APIProtocol.Sc2Api;
import com.github.ocraft.s2client.protocol.data.Units;
import com.google.protobuf.ByteString;
import io.quarkmind.domain.*;

import java.util.EnumMap;
import java.util.Map;

public final class GameStateToProtobuf {

    private GameStateToProtobuf() {}

    static final Map<UnitType, Units> UNIT_TYPE_TO_PROTO = new EnumMap<>(UnitType.class);
    static final Map<BuildingType, Units> BUILDING_TYPE_TO_PROTO = new EnumMap<>(BuildingType.class);

    static {
        UNIT_TYPE_TO_PROTO.put(UnitType.PROBE, Units.PROTOSS_PROBE);
        UNIT_TYPE_TO_PROTO.put(UnitType.ZEALOT, Units.PROTOSS_ZEALOT);
        UNIT_TYPE_TO_PROTO.put(UnitType.STALKER, Units.PROTOSS_STALKER);
        UNIT_TYPE_TO_PROTO.put(UnitType.IMMORTAL, Units.PROTOSS_IMMORTAL);
        UNIT_TYPE_TO_PROTO.put(UnitType.COLOSSUS, Units.PROTOSS_COLOSSUS);
        UNIT_TYPE_TO_PROTO.put(UnitType.CARRIER, Units.PROTOSS_CARRIER);
        UNIT_TYPE_TO_PROTO.put(UnitType.DARK_TEMPLAR, Units.PROTOSS_DARK_TEMPLAR);
        UNIT_TYPE_TO_PROTO.put(UnitType.HIGH_TEMPLAR, Units.PROTOSS_HIGH_TEMPLAR);
        UNIT_TYPE_TO_PROTO.put(UnitType.ARCHON, Units.PROTOSS_ARCHON);
        UNIT_TYPE_TO_PROTO.put(UnitType.OBSERVER, Units.PROTOSS_OBSERVER);
        UNIT_TYPE_TO_PROTO.put(UnitType.VOID_RAY, Units.PROTOSS_VOIDRAY);

        BUILDING_TYPE_TO_PROTO.put(BuildingType.NEXUS, Units.PROTOSS_NEXUS);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.PYLON, Units.PROTOSS_PYLON);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.GATEWAY, Units.PROTOSS_GATEWAY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.CYBERNETICS_CORE, Units.PROTOSS_CYBERNETICS_CORE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ASSIMILATOR, Units.PROTOSS_ASSIMILATOR);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ROBOTICS_FACILITY, Units.PROTOSS_ROBOTICS_FACILITY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.STARGATE, Units.PROTOSS_STARGATE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.FORGE, Units.PROTOSS_FORGE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.TWILIGHT_COUNCIL, Units.PROTOSS_TWILIGHT_COUNCIL);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.PHOTON_CANNON, Units.PROTOSS_PHOTON_CANNON);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SHIELD_BATTERY, Units.PROTOSS_SHIELD_BATTERY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.DARK_SHRINE, Units.PROTOSS_DARK_SHRINE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.TEMPLAR_ARCHIVES, Units.PROTOSS_TEMPLAR_ARCHIVE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.FLEET_BEACON, Units.PROTOSS_FLEET_BEACON);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ROBOTICS_BAY, Units.PROTOSS_ROBOTICS_BAY);
        // Terran buildings
        BUILDING_TYPE_TO_PROTO.put(BuildingType.COMMAND_CENTER, Units.TERRAN_COMMAND_CENTER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ORBITAL_COMMAND, Units.TERRAN_ORBITAL_COMMAND);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.PLANETARY_FORTRESS, Units.TERRAN_PLANETARY_FORTRESS);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SUPPLY_DEPOT, Units.TERRAN_SUPPLY_DEPOT);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.BARRACKS, Units.TERRAN_BARRACKS);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ENGINEERING_BAY, Units.TERRAN_ENGINEERING_BAY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ARMORY, Units.TERRAN_ARMORY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.MISSILE_TURRET, Units.TERRAN_MISSILE_TURRET);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.BUNKER, Units.TERRAN_BUNKER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SENSOR_TOWER, Units.TERRAN_SENSOR_TOWER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.GHOST_ACADEMY, Units.TERRAN_GHOST_ACADEMY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.FACTORY, Units.TERRAN_FACTORY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.STARPORT, Units.TERRAN_STARPORT);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.FUSION_CORE, Units.TERRAN_FUSION_CORE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.REFINERY, Units.TERRAN_REFINERY);
        // Zerg buildings
        BUILDING_TYPE_TO_PROTO.put(BuildingType.HATCHERY, Units.ZERG_HATCHERY);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.LAIR, Units.ZERG_LAIR);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.HIVE, Units.ZERG_HIVE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SPAWNING_POOL, Units.ZERG_SPAWNING_POOL);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.EVOLUTION_CHAMBER, Units.ZERG_EVOLUTION_CHAMBER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ROACH_WARREN, Units.ZERG_ROACH_WARREN);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.BANELING_NEST, Units.ZERG_BANELING_NEST);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SPINE_CRAWLER, Units.ZERG_SPINE_CRAWLER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SPORE_CRAWLER, Units.ZERG_SPORE_CRAWLER);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.HYDRALISK_DEN, Units.ZERG_HYDRALISK_DEN);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.LURKER_DEN, Units.ZERG_LURKER_DEN_MP);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.INFESTATION_PIT, Units.ZERG_INFESTATION_PIT);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.SPIRE, Units.ZERG_SPIRE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.GREATER_SPIRE, Units.ZERG_GREATER_SPIRE);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.NYDUS_NETWORK, Units.ZERG_NYDUS_NETWORK);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.NYDUS_CANAL, Units.ZERG_NYDUS_CANAL);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.ULTRALISK_CAVERN, Units.ZERG_ULTRALISK_CAVERN);
        BUILDING_TYPE_TO_PROTO.put(BuildingType.EXTRACTOR, Units.ZERG_EXTRACTOR);
    }

    public static Sc2Api.ResponseObservation translate(GameState state) {
        Raw.ObservationRaw.Builder rawObs = Raw.ObservationRaw.newBuilder();

        // Friendly units (Alliance.Self, not buildings)
        for (Unit u : state.myUnits()) {
            rawObs.addUnits(toProtoUnit(u, Raw.Alliance.Self));
        }
        // Friendly buildings (Alliance.Self, building types)
        for (Building b : state.myBuildings()) {
            rawObs.addUnits(toProtoBuilding(b, Raw.Alliance.Self));
        }
        // Enemy units, buildings, staging — all Alliance.Enemy
        for (Unit u : state.enemyUnits()) {
            rawObs.addUnits(toProtoUnit(u, Raw.Alliance.Enemy));
        }
        for (Building b : state.enemyBuildings()) {
            rawObs.addUnits(toProtoBuilding(b, Raw.Alliance.Enemy));
        }
        for (Unit u : state.enemyStagingArea()) {
            rawObs.addUnits(toProtoUnit(u, Raw.Alliance.Enemy));
        }

        // PlayerRaw with camera — ocraft requires camera (orElseThrow)
        rawObs.setPlayer(Raw.PlayerRaw.newBuilder()
            .setCamera(Common.Point.newBuilder().setX(50).setY(50).build())
            .build());

        // MapState — ocraft requires visibility and creep ImageData
        Common.ImageData emptyImg = Common.ImageData.newBuilder()
            .setBitsPerPixel(8)
            .setSize(Common.Size2DI.newBuilder().setX(1).setY(1).build())
            .setData(ByteString.copyFrom(new byte[]{0}))
            .build();
        rawObs.setMapState(Raw.MapState.newBuilder()
            .setVisibility(emptyImg)
            .setCreep(emptyImg)
            .build());

        Sc2Api.Observation obs = Sc2Api.Observation.newBuilder()
            .setGameLoop((int) state.gameFrame())
            .setPlayerCommon(Sc2Api.PlayerCommon.newBuilder()
                .setPlayerId(1)
                .setMinerals(state.minerals())
                .setVespene(state.vespene())
                .setFoodCap(state.supply())
                .setFoodUsed(state.supplyUsed())
                .setFoodArmy(0)
                .setFoodWorkers(0)
                .setIdleWorkerCount(0)
                .setArmyCount(0)
                .build())
            .setRawData(rawObs.build())
            .build();

        return Sc2Api.ResponseObservation.newBuilder()
            .setObservation(obs)
            .build();
    }

    private static Raw.Unit toProtoUnit(Unit u, Raw.Alliance alliance) {
        Units protoType = UNIT_TYPE_TO_PROTO.getOrDefault(u.type(), Units.INVALID);
        return Raw.Unit.newBuilder()
            .setTag(Long.parseLong(u.tag()))
            .setUnitType(protoType.getUnitTypeId())
            .setDisplayType(Raw.DisplayType.Visible)
            .setAlliance(alliance)
            .setPos(Common.Point.newBuilder()
                .setX(u.position().x()).setY(u.position().y()).setZ(0).build())
            .setHealth(u.health())
            .setHealthMax(u.maxHealth())
            .setShield(u.shields())
            .setShieldMax(u.maxShields())
            .setBuildProgress(1.0f)
            .build();
    }

    private static Raw.Unit toProtoBuilding(Building b, Raw.Alliance alliance) {
        Units protoType = BUILDING_TYPE_TO_PROTO.getOrDefault(b.type(), Units.INVALID);
        return Raw.Unit.newBuilder()
            .setTag(Long.parseLong(b.tag()))
            .setUnitType(protoType.getUnitTypeId())
            .setDisplayType(Raw.DisplayType.Visible)
            .setAlliance(alliance)
            .setPos(Common.Point.newBuilder()
                .setX(b.position().x()).setY(b.position().y()).setZ(0).build())
            .setHealth(b.health())
            .setHealthMax(b.maxHealth())
            .setShield(0).setShieldMax(0)
            .setBuildProgress(b.isComplete() ? 1.0f : 0.5f)
            .build();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -Dtest=GameStateRoundTripTest -q`
Expected: ALL PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/server/GameStateToProtobuf.java src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java
git commit -m "feat(#171): add GameStateToProtobuf translator + observation round-trip test

Round-trip goes through ocraft ResponseObservation.from() to catch
malformed protobuf that ocraft would reject at runtime.

Refs #171"
```

---

### Task 3: ProtobufToIntent translator + intent round-trip tests

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/server/ProtobufToIntent.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java` (add intent tests)

**Interfaces:**
- Consumes: `ActionTranslator.translate(List<Intent>)`, `ActionTranslator.mapBuildAbility(BuildingType)`, `ActionTranslator.mapTrainAbility(UnitType)`. Sealed `Intent` interface with permits: `BuildIntent`, `TrainIntent`, `AttackIntent`, `MoveIntent`, `BlinkIntent`, `MuleCalldownIntent`. `ResolvedCommand(Tag, Abilities, Optional<Point2d>)`. `Abilities.ATTACK`, `Abilities.MOVE`, `Abilities.EFFECT_BLINK_STALKER`, `Abilities.CALLDOWN_MULE`.
- Produces: `ProtobufToIntent.translate(Raw.ActionRawUnitCommand) → Intent` — converts a protobuf unit command into the corresponding domain Intent. Returns `null` for unrecognized abilities. Used by Task 6 (EmulatedSC2Server) for `RequestAction` handling.

- [ ] **Step 1: Write the failing intent round-trip tests**

Add to `src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java`:

```java
import SC2APIProtocol.Raw;
import io.quarkmind.sc2.intent.*;
import io.quarkmind.sc2.real.ActionTranslator;
import io.quarkmind.sc2.real.ResolvedCommand;
import com.github.ocraft.s2client.protocol.data.Abilities;

@Test
void intentRoundTrip_attackIntent() {
    AttackIntent original = new AttackIntent("42", new Point2d(10f, 20f));
    List<ResolvedCommand> commands = ActionTranslator.translate(List.of(original));
    assertThat(commands).hasSize(1);

    ResolvedCommand cmd = commands.get(0);
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(cmd.tag().getValue())
        .setAbilityId(cmd.ability().getAbilityId())
        .setTargetWorldSpacePos(cmd.target().get().toSc2Api())
        .build();

    Intent roundTripped = ProtobufToIntent.translate(proto);
    assertThat(roundTripped).isInstanceOf(AttackIntent.class);
    AttackIntent rt = (AttackIntent) roundTripped;
    assertThat(rt.unitTag()).isEqualTo("42");
    assertThat(rt.targetLocation().x()).isCloseTo(10f, within(0.01f));
    assertThat(rt.targetLocation().y()).isCloseTo(20f, within(0.01f));
}

@Test
void intentRoundTrip_moveIntent() {
    MoveIntent original = new MoveIntent("99", new Point2d(5f, 15f));
    List<ResolvedCommand> commands = ActionTranslator.translate(List.of(original));
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(commands.get(0).tag().getValue())
        .setAbilityId(commands.get(0).ability().getAbilityId())
        .setTargetWorldSpacePos(commands.get(0).target().get().toSc2Api())
        .build();

    Intent rt = ProtobufToIntent.translate(proto);
    assertThat(rt).isEqualTo(new MoveIntent("99", new Point2d(5f, 15f)));
}

@Test
void intentRoundTrip_trainIntent() {
    TrainIntent original = new TrainIntent("7", UnitType.STALKER);
    List<ResolvedCommand> commands = ActionTranslator.translate(List.of(original));
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(commands.get(0).tag().getValue())
        .setAbilityId(commands.get(0).ability().getAbilityId())
        .build();

    Intent rt = ProtobufToIntent.translate(proto);
    assertThat(rt).isEqualTo(new TrainIntent("7", UnitType.STALKER));
}

@Test
void intentRoundTrip_buildIntent() {
    BuildIntent original = new BuildIntent("5", BuildingType.PYLON, new Point2d(25f, 30f));
    List<ResolvedCommand> commands = ActionTranslator.translate(List.of(original));
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(commands.get(0).tag().getValue())
        .setAbilityId(commands.get(0).ability().getAbilityId())
        .setTargetWorldSpacePos(commands.get(0).target().get().toSc2Api())
        .build();

    Intent rt = ProtobufToIntent.translate(proto);
    assertThat(rt).isEqualTo(new BuildIntent("5", BuildingType.PYLON, new Point2d(25f, 30f)));
}

@Test
void intentOneDirectional_blinkIntent_fromProtobuf() {
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(42L)
        .setAbilityId(Abilities.EFFECT_BLINK_STALKER.getAbilityId())
        .build();

    Intent result = ProtobufToIntent.translate(proto);
    assertThat(result).isEqualTo(new BlinkIntent("42"));
}

@Test
void intentOneDirectional_muleCalldownIntent_fromProtobuf() {
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(99L)
        .setAbilityId(Abilities.CALLDOWN_MULE.getAbilityId())
        .build();

    Intent result = ProtobufToIntent.translate(proto);
    assertThat(result).isEqualTo(new MuleCalldownIntent("99"));
}

@Test
void intentTranslate_unknownAbility_returnsNull() {
    Raw.ActionRawUnitCommand proto = Raw.ActionRawUnitCommand.newBuilder()
        .addUnitTags(1L)
        .setAbilityId(99999)
        .build();

    assertThat(ProtobufToIntent.translate(proto)).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=GameStateRoundTripTest#intentRoundTrip_attackIntent -q`
Expected: FAIL — `ProtobufToIntent` does not exist

- [ ] **Step 3: Implement ProtobufToIntent**

Create `src/main/java/io/quarkmind/sc2/emulated/server/ProtobufToIntent.java`:

```java
package io.quarkmind.sc2.emulated.server;

import SC2APIProtocol.Raw;
import com.github.ocraft.s2client.protocol.data.Abilities;
import io.quarkmind.domain.BuildingType;
import io.quarkmind.domain.Point2d;
import io.quarkmind.domain.UnitType;
import io.quarkmind.sc2.intent.*;
import io.quarkmind.sc2.real.ActionTranslator;

import java.util.HashMap;
import java.util.Map;

public final class ProtobufToIntent {

    private ProtobufToIntent() {}

    @FunctionalInterface
    interface IntentFactory {
        Intent create(String tag, Raw.ActionRawUnitCommand cmd);
    }

    private static final Map<Integer, IntentFactory> ABILITY_TO_INTENT = new HashMap<>();

    static {
        ABILITY_TO_INTENT.put(Abilities.ATTACK.getAbilityId(),
            (tag, cmd) -> new AttackIntent(tag, toPoint(cmd)));
        ABILITY_TO_INTENT.put(Abilities.MOVE.getAbilityId(),
            (tag, cmd) -> new MoveIntent(tag, toPoint(cmd)));
        ABILITY_TO_INTENT.put(Abilities.EFFECT_BLINK_STALKER.getAbilityId(),
            (tag, cmd) -> new BlinkIntent(tag));
        ABILITY_TO_INTENT.put(Abilities.CALLDOWN_MULE.getAbilityId(),
            (tag, cmd) -> new MuleCalldownIntent(tag));

        // Build abilities — reverse of ActionTranslator.mapBuildAbility()
        for (BuildingType bt : BuildingType.values()) {
            Abilities ability = ActionTranslator.mapBuildAbility(bt);
            if (ability != null) {
                ABILITY_TO_INTENT.put(ability.getAbilityId(),
                    (tag, cmd) -> new BuildIntent(tag, bt, toPoint(cmd)));
            }
        }

        // Train abilities — reverse of ActionTranslator.mapTrainAbility()
        for (UnitType ut : UnitType.values()) {
            Abilities ability = ActionTranslator.mapTrainAbility(ut);
            if (ability != null) {
                ABILITY_TO_INTENT.put(ability.getAbilityId(),
                    (tag, cmd) -> new TrainIntent(tag, ut));
            }
        }
    }

    public static Intent translate(Raw.ActionRawUnitCommand cmd) {
        String tag = String.valueOf(cmd.getUnitTags(0));
        IntentFactory factory = ABILITY_TO_INTENT.get(cmd.getAbilityId());
        return factory != null ? factory.create(tag, cmd) : null;
    }

    private static Point2d toPoint(Raw.ActionRawUnitCommand cmd) {
        if (!cmd.hasTargetWorldSpacePos()) return new Point2d(0, 0);
        var p = cmd.getTargetWorldSpacePos();
        return new Point2d(p.getX(), p.getY());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -Dtest=GameStateRoundTripTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/server/ProtobufToIntent.java src/test/java/io/quarkmind/sc2/emulated/server/GameStateRoundTripTest.java
git commit -m "feat(#171): add ProtobufToIntent translator + intent round-trip tests

Covers 4/6 intent types bidirectionally (Attack, Move, Build, Train).
Blink and MULE tested one-directionally — ActionTranslator drops them.

Refs #171"
```

---

### Task 4: SC2WebSocketCodec extraction

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/SC2WebSocketCodec.java`
- Modify: `src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java` — delegate to codec
- Modify: `src/test/java/io/quarkmind/sc2/real/QuarkusSC2TransportTest.java` — delegate FakeSC2Server to codec

**Interfaces:**
- Consumes: nothing — standalone utility
- Produces: `SC2WebSocketCodec.encodeClientFrame(byte[]) → byte[]` (masked, client→server), `SC2WebSocketCodec.encodeServerFrame(byte[]) → byte[]` (unmasked, server→client), `SC2WebSocketCodec.readFrame(InputStream) → byte[]` (reads one complete WebSocket frame, handles fragmentation), `SC2WebSocketCodec.performClientHandshake(OutputStream, InputStream, int port)`, `SC2WebSocketCodec.performServerHandshake(InputStream, OutputStream)`. Used by Task 6 (EmulatedSC2Server) for all WebSocket I/O.

- [ ] **Step 1: Create SC2WebSocketCodec with static methods extracted from QuarkusSC2Transport and FakeSC2Server**

Create `src/main/java/io/quarkmind/sc2/SC2WebSocketCodec.java`. Extract:
- `encodeClientFrame(byte[])` from `QuarkusSC2Transport.encodeClientFrame()` (lines 350-372)
- `encodeServerFrame(byte[])` — new, unmasked equivalent for server→client frames (from FakeSC2Server.serveFrames lines 125-127)
- `readFrame(InputStream)` — unified frame reader from `QuarkusSC2Transport.startFrameReader()` (lines 382-424) and FakeSC2Server.serveFrames (lines 111-118)
- `performClientHandshake(OutputStream, InputStream, int port)` from `QuarkusSC2Transport.performWebSocketHandshake()` (lines 437-466)
- `performServerHandshake(InputStream, OutputStream)` from `FakeSC2Server.doHandshake()` (lines 76-104)

```java
package io.quarkmind.sc2;

import java.io.*;
import java.security.MessageDigest;
import java.util.Base64;

public final class SC2WebSocketCodec {

    private static final java.security.SecureRandom SECURE_RANDOM = new java.security.SecureRandom();
    private static final String WS_GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

    private SC2WebSocketCodec() {}

    public static byte[] encodeClientFrame(byte[] payload) {
        byte[] mask = new byte[4];
        SECURE_RANDOM.nextBytes(mask);
        byte[] masked = payload.clone();
        for (int i = 0; i < masked.length; i++) masked[i] ^= mask[i % 4];
        ByteArrayOutputStream frame = new ByteArrayOutputStream(10 + payload.length);
        frame.write(0x82);
        writeLength(frame, payload.length, true);
        frame.write(mask, 0, 4);
        frame.write(masked, 0, masked.length);
        return frame.toByteArray();
    }

    public static byte[] encodeServerFrame(byte[] payload) {
        ByteArrayOutputStream frame = new ByteArrayOutputStream(10 + payload.length);
        frame.write(0x82);
        writeLength(frame, payload.length, false);
        frame.write(payload, 0, payload.length);
        return frame.toByteArray();
    }

    private static void writeLength(ByteArrayOutputStream frame, int length, boolean masked) {
        int maskBit = masked ? 0x80 : 0;
        if (length < 126) {
            frame.write(maskBit | length);
        } else if (length <= 65535) {
            frame.write(maskBit | 126);
            frame.write((length >> 8) & 0xFF);
            frame.write(length & 0xFF);
        } else {
            frame.write(maskBit | 127);
            long l = length;
            for (int i = 7; i >= 0; i--) frame.write((int) ((l >> (8 * i)) & 0xFF));
        }
    }

    public static byte[] readFrame(InputStream in) throws IOException {
        ByteArrayOutputStream msgBuf = new ByteArrayOutputStream();
        while (true) {
            int b0 = in.read(); if (b0 < 0) return null;
            int b1 = in.read(); if (b1 < 0) return null;
            boolean maskedFrame = (b1 & 0x80) != 0;
            int len = b1 & 0x7F;
            if (len == 126) {
                int h = in.read(), l = in.read();
                if (h < 0 || l < 0) return null;
                len = (h << 8) | l;
            } else if (len == 127) {
                long llen = 0;
                for (int i = 0; i < 8; i++) {
                    int b = in.read();
                    if (b < 0) return null;
                    llen = (llen << 8) | b;
                }
                len = (int) llen;
            }
            byte[] maskBytes = maskedFrame ? in.readNBytes(4) : new byte[0];
            byte[] chunk = in.readNBytes(len);
            if (maskedFrame) for (int i = 0; i < chunk.length; i++) chunk[i] ^= maskBytes[i % 4];
            msgBuf.write(chunk);
            boolean fin = (b0 & 0x80) != 0;
            if (fin) return msgBuf.toByteArray();
        }
    }

    public static void performClientHandshake(OutputStream out, InputStream in, int port)
            throws Exception {
        byte[] keyBytes = new byte[16];
        new java.util.Random().nextBytes(keyBytes);
        String wsKey = Base64.getEncoder().encodeToString(keyBytes);
        String request = "GET /sc2api HTTP/1.1\r\n"
            + "Host: 127.0.0.1:" + port + "\r\n"
            + "Upgrade: websocket\r\n"
            + "Connection: Upgrade\r\n"
            + "Sec-WebSocket-Key: " + wsKey + "\r\n"
            + "Sec-WebSocket-Version: 13\r\n\r\n";
        out.write(request.getBytes());
        out.flush();
        String response = readHttpHeaders(in);
        if (!response.contains("101"))
            throw new IOException("WebSocket upgrade failed: " + response.split("\r\n")[0]);
    }

    public static void performServerHandshake(InputStream in, OutputStream out) throws Exception {
        String headers = readHttpHeaders(in);
        String key = null;
        for (String line : headers.split("\r\n")) {
            if (line.startsWith("Sec-WebSocket-Key:"))
                key = line.substring("Sec-WebSocket-Key:".length()).trim();
        }
        String accept = Base64.getEncoder().encodeToString(
            MessageDigest.getInstance("SHA-1").digest((key + WS_GUID).getBytes()));
        out.write(("HTTP/1.1 101 Switching Protocols\r\n"
            + "Upgrade: websocket\r\n"
            + "Connection: Upgrade\r\n"
            + "Sec-WebSocket-Accept: " + accept + "\r\n\r\n").getBytes());
        out.flush();
    }

    private static String readHttpHeaders(InputStream in) throws IOException {
        StringBuilder sb = new StringBuilder();
        int b;
        while ((b = in.read()) >= 0) {
            sb.append((char) b);
            int len = sb.length();
            if (len >= 4 && sb.charAt(len-4)=='\r' && sb.charAt(len-3)=='\n'
                    && sb.charAt(len-2)=='\r' && sb.charAt(len-1)=='\n') break;
        }
        return sb.toString();
    }
}
```

- [ ] **Step 2: Update QuarkusSC2Transport to delegate to SC2WebSocketCodec**

In `src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java`:
- Replace `encodeClientFrame(payload)` call in `sendSync()` with `SC2WebSocketCodec.encodeClientFrame(payload)`
- Replace the inline frame reader in `startFrameReader()` with `SC2WebSocketCodec.readFrame(in)`
- Replace `performWebSocketHandshake()` body with `SC2WebSocketCodec.performClientHandshake(socketOut, socket.getInputStream(), sc2Port)`
- Remove the now-unused private methods: `encodeClientFrame()`, `performWebSocketHandshake()`, and the inline frame-reading logic
- Keep the `SECURE_RANDOM` field removal (it moves to SC2WebSocketCodec)

- [ ] **Step 3: Update FakeSC2Server in QuarkusSC2TransportTest to delegate to SC2WebSocketCodec**

In `src/test/java/io/quarkmind/sc2/real/QuarkusSC2TransportTest.java`:
- Replace `doHandshake()` body with `SC2WebSocketCodec.performServerHandshake(s.getInputStream(), s.getOutputStream())`
- Replace the frame reading in `serveFrames()` with `SC2WebSocketCodec.readFrame(raw)` to get the payload
- Replace the frame writing in `serveFrames()` with `out.write(SC2WebSocketCodec.encodeServerFrame(respBytes))`
- Also update `FragmentingPingServer.handle()` handshake to use `SC2WebSocketCodec.performServerHandshake()`

- [ ] **Step 4: Run existing transport tests to verify no regressions**

Run: `mvn test -Dtest=QuarkusSC2TransportTest -q`
Expected: ALL PASS — behavior unchanged, only delegation changed

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/SC2WebSocketCodec.java src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java src/test/java/io/quarkmind/sc2/real/QuarkusSC2TransportTest.java
git commit -m "refactor(#171): extract SC2WebSocketCodec from transport and test server

Deduplicates WebSocket handshake and frame encode/decode across
QuarkusSC2Transport (client), FakeSC2Server (test), and the upcoming
EmulatedSC2Server. Pure static methods, no state.

Refs #171"
```

---

### Task 5: Profile gates + configuration

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java:29`
- Modify: `src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java:40,80`
- Modify: `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java:33`
- Modify: `src/main/java/io/quarkmind/sc2/real/SC2StartupBean.java:15`
- Modify: `src/main/java/io/quarkmind/sc2/mock/MockStartupBean.java:18`
- Modify: `src/main/resources/application.properties`

**Interfaces:**
- Consumes: nothing new — modifying existing annotations and config
- Produces: `%emulated-sc2` profile that activates `RealSC2Engine`, `QuarkusSC2Transport`, `SC2BotAgent`, `SC2StartupBean`; excludes `MockStartupBean`; `QuarkusSC2Transport.skipProcessLaunch` is now `@ConfigProperty(name = "starcraft.sc2.skip-launch", defaultValue = "false")`

- [ ] **Step 1: Widen profile gate on RealSC2Engine**

In `src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java`, change line 29:
```java
// Before
@IfBuildProfile("sc2")
// After
@IfBuildProfile(anyOf = {"sc2", "emulated-sc2"})
```

- [ ] **Step 2: Widen profile gate on QuarkusSC2Transport + make skipProcessLaunch config-driven**

In `src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java`:
- Change line 40: `@IfBuildProfile("sc2")` → `@IfBuildProfile(anyOf = {"sc2", "emulated-sc2"})`
- Change line 79-80: replace `boolean skipProcessLaunch = false;` with:
```java
@ConfigProperty(name = "starcraft.sc2.skip-launch", defaultValue = "false")
boolean skipProcessLaunch;
```

- [ ] **Step 3: Widen profile gates on SC2BotAgent and SC2StartupBean**

In `src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java` line 33:
```java
@IfBuildProfile(anyOf = {"sc2", "emulated-sc2"})
```

In `src/main/java/io/quarkmind/sc2/real/SC2StartupBean.java` line 15:
```java
@IfBuildProfile(anyOf = {"sc2", "emulated-sc2"})
```

- [ ] **Step 4: Add emulated-sc2 to MockStartupBean exclusion**

In `src/main/java/io/quarkmind/sc2/mock/MockStartupBean.java` line 18:
```java
@UnlessBuildProfile(anyOf = {"sc2", "emulated-sc2", "replay", "test", "prod"})
```

- [ ] **Step 5: Add %emulated-sc2 configuration to application.properties**

Append to `src/main/resources/application.properties`:

```properties

# --- %emulated-sc2 profile: SC2 protocol over EmulatedGame physics ---
# Run with: mvn quarkus:dev -Dquarkus.profile=emulated-sc2
%emulated-sc2.starcraft.sc2.skip-launch=true
%emulated-sc2.starcraft.sc2.port=8168
%emulated-sc2.starcraft.sc2.connect.retry=5
%emulated-sc2.starcraft.sc2.connect.retry-interval-ms=200
%emulated-sc2.emulated.active=true
%emulated-sc2.scouting.map.width=64
%emulated-sc2.quarkmind.tactics.kite.strategy=terrain-aware
%emulated-sc2.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository,\
  io.casehub.ledger.memory.InMemoryAgentSigner,\
  io.casehub.ledger.memory.InMemoryActorIdentityBindingRepository,\
  io.casehub.ledger.memory.InMemoryReactiveLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryReactiveKeyRotationRepository
%emulated-sc2.quarkus.index-dependency.casehub-ledger-memory.group-id=io.casehub
%emulated-sc2.quarkus.index-dependency.casehub-ledger-memory.artifact-id=casehub-ledger-memory
%emulated-sc2.casehub.ledger.hash-chain.enabled=false
```

- [ ] **Step 6: Verify compilation**

Run: `mvn compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run full test suite (default profile unchanged)**

Run: `mvn test -q`
Expected: ALL PASS — default `%mock`/`%test` profiles unaffected by new profile gates

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/real/RealSC2Engine.java src/main/java/io/quarkmind/sc2/real/QuarkusSC2Transport.java src/main/java/io/quarkmind/sc2/real/SC2BotAgent.java src/main/java/io/quarkmind/sc2/real/SC2StartupBean.java src/main/java/io/quarkmind/sc2/mock/MockStartupBean.java src/main/resources/application.properties
git commit -m "feat(#171): widen profile gates for %emulated-sc2 + add config

RealSC2Engine, QuarkusSC2Transport, SC2BotAgent, SC2StartupBean now
active in both %sc2 and %emulated-sc2. MockStartupBean excluded.
skipProcessLaunch is config-driven. CDI, scouting, and tactics config
added for the new profile.

Refs #171"
```

---

### Task 6: EmulatedSC2Server + integration test

**Files:**
- Create: `src/main/java/io/quarkmind/sc2/emulated/server/EmulatedSC2Server.java`
- Create: `src/test/java/io/quarkmind/sc2/emulated/server/EmulatedSC2ServerTest.java`

**Interfaces:**
- Consumes: `SC2WebSocketCodec` (Task 4), `GameStateToProtobuf.translate(GameState)` (Task 2), `ProtobufToIntent.translate(ActionRawUnitCommand)` (Task 3), `TerrainGrid.toPathingGrid()` (Task 1), `EmulatedGame` (reset, tick, snapshot, applyIntent, setMovementStrategy, setTerrainGrid, setPlayerRaceModel, setEnemyBehavior), `EmulatedConfig` (getPlayerRace, getEnemyRace, getEnemyStrategyName, getUnitSpeed), `PathfindingMovement`, `RaceModelFactory`, `EnemyStrategyLibrary`, `EnemyBehavior`, `TechTree`, `QuarkusSC2Transport` (connect, createGame, joinGame, runGameLoop, quit, isRunning, sendActions), `SC2FrameCallback`, `ResolvedCommand`
- Produces: `EmulatedSC2Server` CDI bean — starts a ServerSocket on the configured port, speaks SC2 WebSocket/protobuf, delegates to EmulatedGame. Active only in `%emulated-sc2` profile.

- [ ] **Step 1: Write the failing integration test**

Create `src/test/java/io/quarkmind/sc2/emulated/server/EmulatedSC2ServerTest.java`:

```java
package io.quarkmind.sc2.emulated.server;

import SC2APIProtocol.Sc2Api;
import com.github.ocraft.s2client.protocol.observation.Observation;
import com.github.ocraft.s2client.protocol.response.ResponseGameInfo;
import io.quarkmind.sc2.GameResult;
import io.quarkmind.sc2.real.QuarkusSC2Transport;
import io.quarkmind.sc2.real.SC2FrameCallback;
import org.junit.jupiter.api.*;

import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

import static org.assertj.core.api.Assertions.*;

class EmulatedSC2ServerTest {

    private EmulatedSC2Server server;
    private QuarkusSC2Transport transport;

    @BeforeEach
    void setUp() throws Exception {
        server = new EmulatedSC2Server();
        server.startOnPort(0); // ephemeral port
    }

    @AfterEach
    void tearDown() {
        if (transport != null) transport.shutdown();
        if (server != null) server.stop();
    }

    private QuarkusSC2Transport connectedTransport() throws Exception {
        QuarkusSC2Transport t = new QuarkusSC2Transport();
        t.sc2Port = server.port();
        t.mapName = "emulated-map";
        t.difficultyStr = "VERY_EASY";
        t.aiRaceStr = "RANDOM";
        t.botRaceStr = "PROTOSS";
        t.connectRetryCount = 5;
        t.connectRetryIntervalMs = 200;
        t.skipProcessLaunch = true;
        t.connect();
        return t;
    }

    @Test @Timeout(10)
    void connect_pingSucceeds() throws Exception {
        transport = connectedTransport();
        // connect() sends a ping — if we get here, it succeeded
    }

    @Test @Timeout(10)
    void gameLoop_receivesObservationsAndEnds() throws Exception {
        transport = connectedTransport();
        transport.createGame();
        transport.joinGame();

        AtomicInteger stepCount = new AtomicInteger();
        CountDownLatch gameStarted = new CountDownLatch(1);
        CountDownLatch gameEnded = new CountDownLatch(1);

        transport.runGameLoop(new SC2FrameCallback() {
            @Override public void onGameStart(ResponseGameInfo info) {
                gameStarted.countDown();
            }
            @Override public void onStep(Observation obs) throws InterruptedException {
                assertThat(obs).isNotNull();
                assertThat(obs.getGameLoop()).isGreaterThan(0);
                if (stepCount.incrementAndGet() >= 5) {
                    transport.quit();
                }
            }
            @Override public void onGameEnd(GameResult result) {
                gameEnded.countDown();
            }
        });

        assertThat(gameStarted.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(gameEnded.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(stepCount.get()).isGreaterThanOrEqualTo(5);
    }

    @Test @Timeout(10)
    void gameInfo_containsStartRaw_withPathingGrid() throws Exception {
        transport = connectedTransport();
        transport.createGame();
        transport.joinGame();

        AtomicReference<ResponseGameInfo> capturedInfo = new AtomicReference<>();
        CountDownLatch gameStarted = new CountDownLatch(1);
        CountDownLatch gameEnded = new CountDownLatch(1);

        transport.runGameLoop(new SC2FrameCallback() {
            @Override public void onGameStart(ResponseGameInfo info) {
                capturedInfo.set(info);
                gameStarted.countDown();
            }
            @Override public void onStep(Observation obs) throws InterruptedException {
                transport.quit();
            }
            @Override public void onGameEnd(GameResult result) {
                gameEnded.countDown();
            }
        });

        assertThat(gameStarted.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(gameEnded.await(5, TimeUnit.SECONDS)).isTrue();

        ResponseGameInfo info = capturedInfo.get();
        assertThat(info).isNotNull();
        assertThat(info.getStartRaw()).isPresent();
        assertThat(info.getStartRaw().get().getPathingGrid()).isNotNull();
        assertThat(info.getStartRaw().get().getMapSize().getX()).isEqualTo(64);
        assertThat(info.getStartRaw().get().getMapSize().getY()).isEqualTo(64);
    }

    @Test @Timeout(10)
    void observations_containMineralsAndUnits() throws Exception {
        transport = connectedTransport();
        transport.createGame();
        transport.joinGame();

        AtomicReference<Observation> capturedObs = new AtomicReference<>();
        CountDownLatch stepped = new CountDownLatch(1);
        CountDownLatch ended = new CountDownLatch(1);

        transport.runGameLoop(new SC2FrameCallback() {
            @Override public void onGameStart(ResponseGameInfo info) {}
            @Override public void onStep(Observation obs) throws InterruptedException {
                capturedObs.set(obs);
                stepped.countDown();
                transport.quit();
            }
            @Override public void onGameEnd(GameResult result) { ended.countDown(); }
        });

        assertThat(stepped.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(ended.await(5, TimeUnit.SECONDS)).isTrue();

        Observation obs = capturedObs.get();
        assertThat(obs.getPlayerCommon().getMinerals()).isGreaterThanOrEqualTo(0);
        assertThat(obs.getRaw()).isPresent();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -Dtest=EmulatedSC2ServerTest#connect_pingSucceeds -q`
Expected: FAIL — `EmulatedSC2Server` does not exist

- [ ] **Step 3: Implement EmulatedSC2Server**

Create `src/main/java/io/quarkmind/sc2/emulated/server/EmulatedSC2Server.java`:

```java
package io.quarkmind.sc2.emulated.server;

import SC2APIProtocol.Common;
import SC2APIProtocol.Raw;
import SC2APIProtocol.Sc2Api;
import com.google.protobuf.ByteString;
import io.quarkmind.domain.EnemyStrategy;
import io.quarkmind.domain.Race;
import io.quarkmind.domain.TechTree;
import io.quarkmind.domain.TerrainGrid;
import io.quarkmind.qa.EmulatedConfig;
import io.quarkmind.sc2.SC2WebSocketCodec;
import io.quarkmind.sc2.emulated.*;
import io.quarkmind.sc2.intent.Intent;
import io.quarkus.arc.profile.IfBuildProfile;
import io.quarkus.runtime.Startup;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.io.*;
import java.net.*;

@IfBuildProfile("emulated-sc2")
@ApplicationScoped
@Startup
public class EmulatedSC2Server {

    private static final Logger log = Logger.getLogger(EmulatedSC2Server.class);

    @Inject EmulatedConfig config;

    @ConfigProperty(name = "starcraft.sc2.port", defaultValue = "8168")
    int configuredPort;

    private final EmulatedGame game = new EmulatedGame();
    private ServerSocket serverSocket;
    private volatile boolean running;
    private Thread acceptThread;

    @PostConstruct
    void start() {
        try {
            startOnPort(configuredPort);
        } catch (IOException e) {
            throw new RuntimeException("[EMULATED-SC2] Failed to start server", e);
        }
    }

    void startOnPort(int port) throws IOException {
        serverSocket = new ServerSocket(port);
        running = true;
        acceptThread = Thread.ofVirtual().name("emulated-sc2-acceptor").start(this::acceptLoop);
        log.infof("[EMULATED-SC2] Server listening on port %d", serverSocket.getLocalPort());
    }

    int port() { return serverSocket.getLocalPort(); }

    @PreDestroy
    void stop() {
        running = false;
        if (serverSocket != null) try { serverSocket.close(); } catch (IOException ignored) {}
    }

    private void acceptLoop() {
        try {
            while (running && !serverSocket.isClosed()) {
                Socket client = serverSocket.accept();
                Thread.ofVirtual().name("emulated-sc2-handler").start(() -> handleClient(client));
            }
        } catch (IOException e) {
            if (running) log.debugf("[EMULATED-SC2] Accept loop closed: %s", e.getMessage());
        }
    }

    private void handleClient(Socket client) {
        try {
            SC2WebSocketCodec.performServerHandshake(client.getInputStream(), client.getOutputStream());
            serveProtocol(client);
        } catch (Exception e) {
            log.debugf("[EMULATED-SC2] Client handler closed: %s", e.getMessage());
        } finally {
            try { client.close(); } catch (IOException ignored) {}
        }
    }

    private void serveProtocol(Socket client) throws Exception {
        InputStream in = client.getInputStream();
        OutputStream out = client.getOutputStream();

        while (!client.isClosed() && running) {
            byte[] payload = SC2WebSocketCodec.readFrame(in);
            if (payload == null) break;

            Sc2Api.Request req = Sc2Api.Request.parseFrom(payload);
            Sc2Api.Response resp = buildResponse(req);
            out.write(SC2WebSocketCodec.encodeServerFrame(resp.toByteArray()));
            out.flush();
        }
    }

    private Sc2Api.Response buildResponse(Sc2Api.Request req) {
        Sc2Api.Response.Builder b = Sc2Api.Response.newBuilder();

        if (req.hasPing()) {
            return b.setPing(Sc2Api.ResponsePing.getDefaultInstance())
                    .setStatus(Sc2Api.Status.launched).build();
        }
        if (req.hasCreateGame()) {
            return b.setCreateGame(Sc2Api.ResponseCreateGame.getDefaultInstance())
                    .setStatus(Sc2Api.Status.init_game).build();
        }
        if (req.hasJoinGame()) {
            wireEmulatedGame();
            return b.setJoinGame(Sc2Api.ResponseJoinGame.newBuilder().setPlayerId(1).build())
                    .setStatus(Sc2Api.Status.in_game).build();
        }
        if (req.hasGameInfo()) {
            return b.setGameInfo(buildGameInfo()).setStatus(Sc2Api.Status.in_game).build();
        }
        if (req.hasObservation()) {
            return b.setObservation(GameStateToProtobuf.translate(game.snapshot()))
                    .setStatus(Sc2Api.Status.in_game).build();
        }
        if (req.hasAction()) {
            handleActions(req.getAction());
            return b.setAction(Sc2Api.ResponseAction.getDefaultInstance())
                    .setStatus(Sc2Api.Status.in_game).build();
        }
        if (req.hasStep()) {
            game.tick();
            return b.setStep(Sc2Api.ResponseStep.getDefaultInstance())
                    .setStatus(Sc2Api.Status.in_game).build();
        }
        if (req.hasQuit()) {
            return b.setQuit(Sc2Api.ResponseQuit.getDefaultInstance())
                    .setStatus(Sc2Api.Status.quit).build();
        }
        return b.setStatus(Sc2Api.Status.in_game).build();
    }

    private void wireEmulatedGame() {
        TerrainGrid grid = TerrainGrid.emulatedMap();
        game.setMovementStrategy(new PathfindingMovement(grid));
        game.setTerrainGrid(grid);

        Race playerRace = config != null ? config.getPlayerRace() : Race.PROTOSS;
        game.setPlayerRaceModel(RaceModelFactory.forRace(playerRace));

        Race enemyRace = config != null ? config.getEnemyRace() : Race.PROTOSS;
        String strategyName = config != null ? config.getEnemyStrategyName() : null;
        EnemyStrategy strategy = (strategyName != null && !strategyName.isBlank())
            ? EnemyStrategyLibrary.forName(strategyName)
            : EnemyStrategyLibrary.randomForRace(enemyRace);
        game.setEnemyBehavior(new EnemyBehavior(strategy, game.enemy, new TechTree()));

        game.reset();
        log.infof("[EMULATED-SC2] Game wired — player=%s, enemy strategy=%s",
            playerRace, strategy.name());
    }

    private Sc2Api.ResponseGameInfo buildGameInfo() {
        TerrainGrid grid = TerrainGrid.emulatedMap();
        byte[] pathingData = grid.toPathingGrid();

        // 8bpp terrain height stub — ocraft requires this (orElseThrow)
        Common.ImageData heightStub = Common.ImageData.newBuilder()
            .setBitsPerPixel(8)
            .setSize(Common.Size2DI.newBuilder().setX(1).setY(1).build())
            .setData(ByteString.copyFrom(new byte[]{0}))
            .build();

        // 1bpp placement grid stub — 8×1 to avoid 1×1 1bpp integer division edge case
        Common.ImageData placementStub = Common.ImageData.newBuilder()
            .setBitsPerPixel(1)
            .setSize(Common.Size2DI.newBuilder().setX(8).setY(1).build())
            .setData(ByteString.copyFrom(new byte[]{0}))
            .build();

        Raw.StartRaw startRaw = Raw.StartRaw.newBuilder()
            .setMapSize(Common.Size2DI.newBuilder().setX(64).setY(64).build())
            .setPathingGrid(Common.ImageData.newBuilder()
                .setBitsPerPixel(1)
                .setSize(Common.Size2DI.newBuilder().setX(64).setY(64).build())
                .setData(ByteString.copyFrom(pathingData))
                .build())
            .setTerrainHeight(heightStub)
            .setPlacementGrid(placementStub)
            .setPlayableArea(SC2APIProtocol.Common.RectangleI.newBuilder()
                .setP0(Common.PointI.newBuilder().setX(0).setY(0).build())
                .setP1(Common.PointI.newBuilder().setX(64).setY(64).build())
                .build())
            .build();

        return Sc2Api.ResponseGameInfo.newBuilder()
            .setMapName("emulated-64x64")
            .addPlayerInfo(Sc2Api.PlayerInfo.newBuilder()
                .setPlayerId(1)
                .setType(Sc2Api.PlayerType.Participant)
                .setRaceRequested(Common.Race.Protoss)
                .build())
            .setStartRaw(startRaw)
            .setOptions(Sc2Api.InterfaceOptions.newBuilder().setRaw(true).build())
            .build();
    }

    private void handleActions(Sc2Api.RequestAction actionReq) {
        for (Sc2Api.Action action : actionReq.getActionsList()) {
            if (action.hasActionRaw() && action.getActionRaw().hasUnitCommand()) {
                Intent intent = ProtobufToIntent.translate(action.getActionRaw().getUnitCommand());
                if (intent != null) {
                    game.applyIntent(intent);
                }
            }
        }
    }
}
```

- [ ] **Step 4: Run integration tests to verify they pass**

Run: `mvn test -Dtest=EmulatedSC2ServerTest -q`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn test -q`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/server/EmulatedSC2Server.java src/test/java/io/quarkmind/sc2/emulated/server/EmulatedSC2ServerTest.java
git commit -m "feat(#171): add EmulatedSC2Server — SC2 protocol wrapper for EmulatedGame

CDI bean that starts a ServerSocket speaking SC2 WebSocket/protobuf.
RealSC2Engine connects to it identically to real SC2. Includes
StartRaw with full pathing grid, ocraft-validated observations,
and action→intent translation.

Refs #171"
```
