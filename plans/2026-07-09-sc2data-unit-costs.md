# SC2Data Unit Cost Tables Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #229 — chore: expand SC2Data mineral/gas cost tables for all unit types
**Issue group:** #229

**Goal:** Replace three independent switch-with-default methods (`mineralCost`, `gasCost`, `supplyCost`) in SC2Data with a single `EnumMap<UnitType, UnitCosts>` covering all 67 UnitType values with correct SC2 LotV final balance costs.

**Architecture:** New `UnitCosts` record in `io.quarkmind.domain` stores mineral/gas/supply per unit type. SC2Data populates an `EnumMap` in a static initializer with completeness validation. Existing public methods become thin delegates — no caller changes.

**Tech Stack:** Plain Java (domain package — no framework deps)

## Global Constraints

- Values target SC2 LotV final balance (patch 5.0.11+ / 5.0.14 / 5.0.15, November 2020 — no cost changes since)
- Source: Liquipedia SC2 wiki (https://liquipedia.net/starcraft2/Units_(Protoss), /Units_(Terran), /Units_(Zerg))
- Domain package (`io.quarkmind.domain`) must remain plain Java — no CDI, no Quarkus imports
- Morph units (Baneling, Ravager, Lurker, Brood Lord, Overseer) use morph cost (not total investment) — matches what SC2 deducts from bank
- Transformation forms (SIEGE_TANK_SIEGED, VIKING_ASSAULT, LIBERATOR_AG, WARP_PRISM_PHASING, HELLBAT) use same cost as base type
- Spawned tokens (INTERCEPTOR, LOCUST, BROODLING, etc.) are 0/0/0
- Per-individual costs: Zergling mineral/gas is per individual (25m = half of 50m batch). Supply is per training command (1 = 0.5 per individual × 2)

---

### Task 1: UnitCosts record, UNIT_COSTS EnumMap, delegate methods, exhaustive tests

**Files:**
- Create: `src/main/java/io/quarkmind/domain/UnitCosts.java`
- Modify: `src/main/java/io/quarkmind/domain/SC2Data.java` (replace 3 switch methods, add EnumMap + accessor)
- Modify: `src/test/java/io/quarkmind/domain/SC2DataTest.java` (add exhaustive cost tests + fleet scenario)

**Interfaces:**
- Produces: `UnitCosts` record (`int mineral, int gas, int supply`)
- Produces: `SC2Data.unitCosts(UnitType)` returning `UnitCosts`
- Preserves: `SC2Data.mineralCost(UnitType)`, `SC2Data.gasCost(UnitType)`, `SC2Data.supplyCost(UnitType)` — same signatures, now delegating to map

- [ ] **Step 1: Create UnitCosts record**

Create `src/main/java/io/quarkmind/domain/UnitCosts.java`:

```java
package io.quarkmind.domain;

/**
 * Per-individual-unit costs. For batch-trained units (Zergling, trainCount=2),
 * mineral and gas are per individual (Zergling: 25m = half of 50m batch cost).
 * Supply is per training command (Zergling: 1 = 0.5 per individual, rounded to int).
 * For all units with trainCount=1, the distinction is moot.
 *
 * Consumers that deduct costs per training command must multiply mineral/gas
 * by {@link SC2Data#trainCount} — see #234.
 */
public record UnitCosts(int mineral, int gas, int supply) {}
```

- [ ] **Step 2: Write failing tests — exhaustive costs + fleet scenario**

Add to `SC2DataTest.java`. Use a helper to keep assertions DRY:

```java
// --- UnitCosts exhaustive coverage (LotV final balance, Liquipedia-verified) ---

private static void assertCosts(UnitType type, int mineral, int gas, int supply) {
    assertThat(SC2Data.mineralCost(type)).as(type + " mineral").isEqualTo(mineral);
    assertThat(SC2Data.gasCost(type)).as(type + " gas").isEqualTo(gas);
    assertThat(SC2Data.supplyCost(type)).as(type + " supply").isEqualTo(supply);
}

@Test
void allProtossUnitCosts() {
    assertCosts(UnitType.PROBE,              50,   0, 1);
    assertCosts(UnitType.ZEALOT,            100,   0, 2);
    assertCosts(UnitType.STALKER,           125,  50, 2);
    assertCosts(UnitType.IMMORTAL,          250, 100, 4);
    assertCosts(UnitType.COLOSSUS,          300, 200, 6);
    assertCosts(UnitType.CARRIER,           350, 250, 6);
    assertCosts(UnitType.DARK_TEMPLAR,      125, 125, 2);
    assertCosts(UnitType.HIGH_TEMPLAR,       50, 150, 2);
    assertCosts(UnitType.ARCHON,              0,   0, 0);
    assertCosts(UnitType.OBSERVER,           25,  75, 1);
    assertCosts(UnitType.VOID_RAY,          250, 150, 4);
    assertCosts(UnitType.ADEPT,             100,  25, 2);
    assertCosts(UnitType.DISRUPTOR,         150, 150, 3);
    assertCosts(UnitType.SENTRY,             50, 100, 2);
    assertCosts(UnitType.PHOENIX,           150, 100, 2);
    assertCosts(UnitType.ORACLE,            150, 150, 3);
    assertCosts(UnitType.TEMPEST,           250, 175, 4);
    assertCosts(UnitType.MOTHERSHIP,        400, 400, 8);
    assertCosts(UnitType.WARP_PRISM,        200,   0, 2);
    assertCosts(UnitType.WARP_PRISM_PHASING,200,   0, 2);
    assertCosts(UnitType.INTERCEPTOR,        15,   0, 0);
    assertCosts(UnitType.ADEPT_PHASE_SHIFT,   0,   0, 0);
}

@Test
void allTerranUnitCosts() {
    assertCosts(UnitType.MARINE,             50,   0, 1);
    assertCosts(UnitType.MARAUDER,          100,  25, 2);
    assertCosts(UnitType.MEDIVAC,           100, 100, 2);
    assertCosts(UnitType.SIEGE_TANK,        150, 125, 3);
    assertCosts(UnitType.SIEGE_TANK_SIEGED, 150, 125, 3);
    assertCosts(UnitType.THOR,              300, 200, 6);
    assertCosts(UnitType.VIKING,            150,  75, 2);
    assertCosts(UnitType.GHOST,             150, 125, 2);
    assertCosts(UnitType.RAVEN,             100, 200, 2);
    assertCosts(UnitType.BANSHEE,           150, 100, 3);
    assertCosts(UnitType.BATTLECRUISER,     400, 300, 6);
    assertCosts(UnitType.CYCLONE,           150, 100, 3);
    assertCosts(UnitType.LIBERATOR,         150, 150, 3);
    assertCosts(UnitType.WIDOW_MINE,         75,  25, 2);
    assertCosts(UnitType.SCV,                50,   0, 1);
    assertCosts(UnitType.REAPER,             50,  50, 1);
    assertCosts(UnitType.HELLION,           100,   0, 2);
    assertCosts(UnitType.HELLBAT,           100,   0, 2);
    assertCosts(UnitType.MULE,                0,   0, 0);
    assertCosts(UnitType.VIKING_ASSAULT,    150,  75, 2);
    assertCosts(UnitType.LIBERATOR_AG,      150, 150, 3);
    assertCosts(UnitType.AUTO_TURRET,         0,   0, 0);
}

@Test
void allZergUnitCosts() {
    assertCosts(UnitType.ZERGLING,           25,   0, 1);
    assertCosts(UnitType.ROACH,              75,  25, 2);
    assertCosts(UnitType.HYDRALISK,         100,  50, 2);
    assertCosts(UnitType.MUTALISK,          100, 100, 2);
    assertCosts(UnitType.ULTRALISK,         300, 200, 6);
    assertCosts(UnitType.BROOD_LORD,        150, 150, 2);
    assertCosts(UnitType.CORRUPTOR,         150, 100, 2);
    assertCosts(UnitType.INFESTOR,          100, 150, 2);
    assertCosts(UnitType.SWARM_HOST,        100,  75, 3);
    assertCosts(UnitType.VIPER,             100, 200, 3);
    assertCosts(UnitType.QUEEN,             150,   0, 2);
    assertCosts(UnitType.RAVAGER,            25,  75, 1);
    assertCosts(UnitType.LURKER,             50, 100, 1);
    assertCosts(UnitType.DRONE,              50,   0, 1);
    assertCosts(UnitType.OVERLORD,          100,   0, 0);
    assertCosts(UnitType.OVERSEER,           50,  50, 0);
    assertCosts(UnitType.BANELING,           25,  25, 0);
    assertCosts(UnitType.LOCUST,              0,   0, 0);
    assertCosts(UnitType.BROODLING,           0,   0, 0);
    assertCosts(UnitType.INFESTED_TERRAN,     0,   0, 0);
    assertCosts(UnitType.CHANGELING,          0,   0, 0);
    assertCosts(UnitType.EGG,                 0,   0, 0);
}

@Test
void unknownSentinelCosts() {
    assertCosts(UnitType.UNKNOWN, 0, 0, 0);
}

@Test
void unitCostsAccessorMatchesDelegates() {
    for (UnitType type : UnitType.values()) {
        UnitCosts costs = SC2Data.unitCosts(type);
        assertThat(costs.mineral()).as(type + " mineral via unitCosts").isEqualTo(SC2Data.mineralCost(type));
        assertThat(costs.gas()).as(type + " gas via unitCosts").isEqualTo(SC2Data.gasCost(type));
        assertThat(costs.supply()).as(type + " supply via unitCosts").isEqualTo(SC2Data.supplyCost(type));
    }
}

@Test
void fleetScenarioCorrectValuation() {
    // Issue #229: 4 Carriers + 3 Colossi + 2 Void Rays
    // Real value = 4700, old default-based value was ~900 (81% undervaluation)
    int value = 4 * (SC2Data.mineralCost(UnitType.CARRIER) + SC2Data.gasCost(UnitType.CARRIER))
              + 3 * (SC2Data.mineralCost(UnitType.COLOSSUS) + SC2Data.gasCost(UnitType.COLOSSUS))
              + 2 * (SC2Data.mineralCost(UnitType.VOID_RAY) + SC2Data.gasCost(UnitType.VOID_RAY));
    assertThat(value).isEqualTo(4700);
}
```

Run: `mvn test -Dtest=SC2DataTest -q`
Expected: FAIL — `unitCosts` method does not exist; many cost values differ from old defaults.

- [ ] **Step 3: Add UNIT_COSTS EnumMap + unitCosts() accessor + replace delegate methods**

In `SC2Data.java`, add these imports at the top (after existing imports):

```java
import java.util.Collections;
import java.util.EnumMap;
```

Replace the `mineralCost(UnitType)` method (lines 403-426), `gasCost(UnitType)` method (lines 485-495), and `supplyCost(UnitType)` method (lines 247-264) with delegates. Add the `UNIT_COSTS` map and `unitCosts()` accessor.

Insert the map field and static initializer BEFORE `supplyCost` (before the existing method at line 247). The map and initializer:

```java
private static final Map<UnitType, UnitCosts> UNIT_COSTS;
static {
    var map = new EnumMap<UnitType, UnitCosts>(UnitType.class);
    // Protoss
    map.put(UnitType.PROBE,              new UnitCosts( 50,   0, 1));
    map.put(UnitType.ZEALOT,             new UnitCosts(100,   0, 2));
    map.put(UnitType.STALKER,            new UnitCosts(125,  50, 2));
    map.put(UnitType.IMMORTAL,           new UnitCosts(250, 100, 4));
    map.put(UnitType.COLOSSUS,           new UnitCosts(300, 200, 6));
    map.put(UnitType.CARRIER,            new UnitCosts(350, 250, 6));
    map.put(UnitType.DARK_TEMPLAR,       new UnitCosts(125, 125, 2));
    map.put(UnitType.HIGH_TEMPLAR,       new UnitCosts( 50, 150, 2));
    map.put(UnitType.ARCHON,             new UnitCosts(  0,   0, 0));
    map.put(UnitType.OBSERVER,           new UnitCosts( 25,  75, 1));
    map.put(UnitType.VOID_RAY,           new UnitCosts(250, 150, 4));
    map.put(UnitType.ADEPT,              new UnitCosts(100,  25, 2));
    map.put(UnitType.DISRUPTOR,          new UnitCosts(150, 150, 3));
    map.put(UnitType.SENTRY,             new UnitCosts( 50, 100, 2));
    map.put(UnitType.PHOENIX,            new UnitCosts(150, 100, 2));
    map.put(UnitType.ORACLE,             new UnitCosts(150, 150, 3));
    map.put(UnitType.TEMPEST,            new UnitCosts(250, 175, 4));
    map.put(UnitType.MOTHERSHIP,         new UnitCosts(400, 400, 8));
    map.put(UnitType.WARP_PRISM,         new UnitCosts(200,   0, 2));
    map.put(UnitType.WARP_PRISM_PHASING, new UnitCosts(200,   0, 2));
    map.put(UnitType.INTERCEPTOR,        new UnitCosts( 15,   0, 0));
    map.put(UnitType.ADEPT_PHASE_SHIFT,  new UnitCosts(  0,   0, 0));
    // Zerg
    map.put(UnitType.ZERGLING,           new UnitCosts( 25,   0, 1));
    map.put(UnitType.ROACH,              new UnitCosts( 75,  25, 2));
    map.put(UnitType.HYDRALISK,          new UnitCosts(100,  50, 2));
    map.put(UnitType.MUTALISK,           new UnitCosts(100, 100, 2));
    map.put(UnitType.ULTRALISK,          new UnitCosts(300, 200, 6));
    map.put(UnitType.BROOD_LORD,         new UnitCosts(150, 150, 2));
    map.put(UnitType.CORRUPTOR,          new UnitCosts(150, 100, 2));
    map.put(UnitType.INFESTOR,           new UnitCosts(100, 150, 2));
    map.put(UnitType.SWARM_HOST,         new UnitCosts(100,  75, 3));
    map.put(UnitType.VIPER,              new UnitCosts(100, 200, 3));
    map.put(UnitType.QUEEN,              new UnitCosts(150,   0, 2));
    map.put(UnitType.RAVAGER,            new UnitCosts( 25,  75, 1));
    map.put(UnitType.LURKER,             new UnitCosts( 50, 100, 1));
    map.put(UnitType.DRONE,              new UnitCosts( 50,   0, 1));
    map.put(UnitType.OVERLORD,           new UnitCosts(100,   0, 0));
    map.put(UnitType.OVERSEER,           new UnitCosts( 50,  50, 0));
    map.put(UnitType.BANELING,           new UnitCosts( 25,  25, 0));
    map.put(UnitType.LOCUST,             new UnitCosts(  0,   0, 0));
    map.put(UnitType.BROODLING,          new UnitCosts(  0,   0, 0));
    map.put(UnitType.INFESTED_TERRAN,    new UnitCosts(  0,   0, 0));
    map.put(UnitType.CHANGELING,         new UnitCosts(  0,   0, 0));
    map.put(UnitType.EGG,                new UnitCosts(  0,   0, 0));
    // Terran
    map.put(UnitType.MARINE,             new UnitCosts( 50,   0, 1));
    map.put(UnitType.MARAUDER,           new UnitCosts(100,  25, 2));
    map.put(UnitType.MEDIVAC,            new UnitCosts(100, 100, 2));
    map.put(UnitType.SIEGE_TANK,         new UnitCosts(150, 125, 3));
    map.put(UnitType.SIEGE_TANK_SIEGED,  new UnitCosts(150, 125, 3));
    map.put(UnitType.THOR,               new UnitCosts(300, 200, 6));
    map.put(UnitType.VIKING,             new UnitCosts(150,  75, 2));
    map.put(UnitType.GHOST,              new UnitCosts(150, 125, 2));
    map.put(UnitType.RAVEN,              new UnitCosts(100, 200, 2));
    map.put(UnitType.BANSHEE,            new UnitCosts(150, 100, 3));
    map.put(UnitType.BATTLECRUISER,      new UnitCosts(400, 300, 6));
    map.put(UnitType.CYCLONE,            new UnitCosts(150, 100, 3));
    map.put(UnitType.LIBERATOR,          new UnitCosts(150, 150, 3));
    map.put(UnitType.WIDOW_MINE,         new UnitCosts( 75,  25, 2));
    map.put(UnitType.SCV,                new UnitCosts( 50,   0, 1));
    map.put(UnitType.REAPER,             new UnitCosts( 50,  50, 1));
    map.put(UnitType.HELLION,            new UnitCosts(100,   0, 2));
    map.put(UnitType.HELLBAT,            new UnitCosts(100,   0, 2));
    map.put(UnitType.MULE,               new UnitCosts(  0,   0, 0));
    map.put(UnitType.VIKING_ASSAULT,     new UnitCosts(150,  75, 2));
    map.put(UnitType.LIBERATOR_AG,       new UnitCosts(150, 150, 3));
    map.put(UnitType.AUTO_TURRET,        new UnitCosts(  0,   0, 0));
    // Fallback
    map.put(UnitType.UNKNOWN,            new UnitCosts(  0,   0, 0));

    for (UnitType t : UnitType.values()) {
        if (!map.containsKey(t))
            throw new ExceptionInInitializerError("Missing UnitCosts for " + t);
    }
    UNIT_COSTS = Collections.unmodifiableMap(map);
}

public static UnitCosts unitCosts(UnitType type) { return UNIT_COSTS.get(type); }
```

Replace `supplyCost(UnitType)` (lines 247-264) with:

```java
public static int supplyCost(UnitType type) { return UNIT_COSTS.get(type).supply(); }
```

Replace `mineralCost(UnitType)` (lines 403-426) with:

```java
public static int mineralCost(UnitType type) { return UNIT_COSTS.get(type).mineral(); }
```

Replace `gasCost(UnitType)` (lines 485-495) with:

```java
public static int gasCost(UnitType type) { return UNIT_COSTS.get(type).gas(); }
```

- [ ] **Step 4: Run tests to verify everything passes**

Run: `mvn test -Dtest=SC2DataTest -q`
Expected: PASS — all new exhaustive tests + all old tests pass.

Then run the full suite:

Run: `mvn test -q`
Expected: PASS — no regressions across 1305+ tests. The delegate methods preserve the existing API; only values change (all correct now).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/quarkmind/domain/UnitCosts.java \
       src/main/java/io/quarkmind/domain/SC2Data.java \
       src/test/java/io/quarkmind/domain/SC2DataTest.java
git commit -m "feat(#229): UnitCosts record + exhaustive EnumMap — eliminate default-clause coverage gaps

Replace three switch-with-default methods (mineralCost, gasCost, supplyCost)
with an EnumMap<UnitType, UnitCosts> validated at class load time. All 67
UnitType values now have Liquipedia-verified LotV final balance costs.

Fixes 81% army undervaluation in MultiFactorDominanceAssessor.

Refs #229"
```
