# DroolsScoutingTask Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `DroolsScoutingTask` — a Drools rule-unit–based scouting plugin that accumulates enemy unit observation events across game ticks and detects build-order patterns (Zerg roach rush, Terran 3-rax, Protoss 4-gate), timing attacks, and expansion posture.

**Architecture:** Java-side event buffers maintain temporal windows (3-min build-order, 10-sec army alert); each tick a fresh `RuleUnitInstance<ScoutingRuleUnit>` is fired against the current buffer contents. `ScoutingSessionManager` owns the buffers and resets on game restart. `DroolsScoutingTask` writes three new CaseFile keys plus the existing passive-intel keys.

**Tech Stack:** Drools 10 rule units (`drools-quarkus`, `drools-ruleunits-api`), CaseHub `TaskDefinition`, Quarkus CDI, JUnit 5 + AssertJ, `@QuarkusTest`.

---

## Implementation Notes

- `DataSource.createStore()` requires Quarkus build-time initialisation (GE-0053) — rule unit tests must be `@QuarkusTest`.
- UnitType currently has Protoss units only. Add enemy unit types (Task 2) before the DRL is compiled.
- `BasicScoutingTask` is NOT removed — it stays as the reference implementation. `DroolsScoutingTask` replaces it as the active CDI bean via the same `@ApplicationScoped @CaseType("starcraft-game")` annotations. Quarkus CDI will throw an ambiguous-bean error at startup — resolve by adding `@Priority` or `@Alternative`+`@Alternative` on `BasicScoutingTask`. See Task 8.
- DRL path convention: matches Java package of `RuleUnitData` class — `src/main/resources/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.drl`.
- Game time in ms: `gameTimeMs = (long)(frame * (1000.0 / 22.4))` — SC2 Faster speed = 22.4 fps.

---

## File Map

| Action | Path | Responsibility |
|---|---|---|
| Modify | `domain/UnitType.java` | Add ROACH, MARINE, ZERGLING, MARAUDER |
| Modify | `agent/StarCraftCaseFile.java` | Add ENEMY_BUILD_ORDER, TIMING_ATTACK_INCOMING, ENEMY_POSTURE |
| Create | `plugin/scouting/events/EnemyUnitFirstSeen.java` | Event record: enemy unit first observed |
| Create | `plugin/scouting/events/EnemyExpansionSeen.java` | Event record: enemy expansion-area unit sighted |
| Create | `plugin/scouting/events/EnemyArmyNearBase.java` | Event record: enemy army near our Nexus |
| Create | `plugin/scouting/ScoutingRuleUnit.java` | RuleUnitData: event DataStores + output lists |
| Create | `resources/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.drl` | 6 Drools rules |
| Create | `plugin/scouting/ScoutingSessionManager.java` | Event buffer lifecycle manager |
| Create | `plugin/scouting/DroolsScoutingTask.java` | @ApplicationScoped @CaseType task bean |
| Create | `test/.../DroolsScoutingRulesTest.java` | @QuarkusTest — DRL rule correctness |
| Create | `test/.../DroolsScoutingTaskIT.java` | @QuarkusTest — full pipeline |
| Modify | `NATIVE.md` | Mark Drools rule units + CEP pattern ✅ |

---

## Task 1: Create GitHub epic and child issues

**Files:** none (GitHub only)

- [ ] **Step 1: Create the epic**

```bash
gh issue create --repo mdproctor/starcraft \
  --title "ScoutingTask — R&D plugin integration (Phase 4)" \
  --label "enhancement,epic" \
  --body "Fourth and final plugin seam. R&D angle: Drools rule units with temporal event accumulation for build-order detection, timing attack alerts, and expansion posture inference. Replaces BasicScoutingTask as the active CDI bean."
```

- [ ] **Step 2: Create child issues**

```bash
gh issue create --repo mdproctor/starcraft \
  --title "Extend UnitType + add CaseFile keys for multi-race scouting" \
  --label "enhancement" \
  --body "Add ROACH, MARINE, ZERGLING, MARAUDER to UnitType. Add ENEMY_BUILD_ORDER, TIMING_ATTACK_INCOMING, ENEMY_POSTURE to StarCraftCaseFile. Refs #<epic>"

gh issue create --repo mdproctor/starcraft \
  --title "ScoutingRuleUnit, event records, and DRL rules" \
  --label "enhancement" \
  --body "Create EnemyUnitFirstSeen, EnemyExpansionSeen, EnemyArmyNearBase records. ScoutingRuleUnit (RuleUnitData). DroolsScoutingTask.drl with 6 rules. Refs #<epic>"

gh issue create --repo mdproctor/starcraft \
  --title "ScoutingSessionManager and DroolsScoutingTask implementation" \
  --label "enhancement" \
  --body "ScoutingSessionManager (Java event buffer lifecycle). DroolsScoutingTask (@ApplicationScoped @CaseType). Refs #<epic>"

gh issue create --repo mdproctor/starcraft \
  --title "Tests: DroolsScoutingRulesTest and DroolsScoutingTaskIT" \
  --label "enhancement" \
  --body "@QuarkusTest rule tests and integration tests for DroolsScoutingTask. Refs #<epic>"
```

Note the issue numbers assigned — use them for `Refs #N` in all subsequent commits.

---

## Task 2: Extend UnitType with multi-race types

**Files:**
- Modify: `src/main/java/org/acme/starcraft/domain/UnitType.java`

The DRL rules reference `UnitType.ROACH`, `UnitType.MARINE`, etc. These must exist before the DRL is compiled at Quarkus build time.

- [ ] **Step 1: Add enemy unit types to the enum**

Open `src/main/java/org/acme/starcraft/domain/UnitType.java`. Replace the current content:

```java
package org.acme.starcraft.domain;

public enum UnitType {
    // Protoss
    PROBE, ZEALOT, STALKER, IMMORTAL, COLOSSUS, CARRIER,
    DARK_TEMPLAR, HIGH_TEMPLAR, ARCHON, OBSERVER, VOID_RAY,
    // Zerg
    ZERGLING, ROACH, HYDRALISK,
    // Terran
    MARINE, MARAUDER, MEDIVAC,
    // Fallback
    UNKNOWN
}
```

- [ ] **Step 2: Verify no existing tests break**

```bash
mvn test -Dtest=BasicScoutingTaskTest,DroolsTacticsRuleUnitTest,BasicStrategyTaskTest -q
```

Expected: BUILD SUCCESS — UnitType extension is additive.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/domain/UnitType.java
git commit -m "feat(domain): add multi-race UnitType values for scouting CEP rules

Refs #<domain-issue>"
```

---

## Task 3: Add CaseFile keys

**Files:**
- Modify: `src/main/java/org/acme/starcraft/agent/StarCraftCaseFile.java`

- [ ] **Step 1: Add three new constants**

Open `src/main/java/org/acme/starcraft/agent/StarCraftCaseFile.java`. After the existing `NEAREST_THREAT` line, add:

```java
    public static final String ENEMY_BUILD_ORDER       = "agent.intel.enemy.build";
    public static final String TIMING_ATTACK_INCOMING  = "agent.intel.enemy.timing";
    public static final String ENEMY_POSTURE           = "agent.intel.enemy.posture";
```

- [ ] **Step 2: Verify build still passes**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/agent/StarCraftCaseFile.java
git commit -m "feat(agent): add scouting intel CaseFile keys

Refs #<domain-issue>"
```

---

## Task 4: Create event records

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/events/EnemyUnitFirstSeen.java`
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/events/EnemyExpansionSeen.java`
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/events/EnemyArmyNearBase.java`

These are plain Java records — no framework imports. The DRL declares the event role and timestamp via `declare ... @role(event) @timestamp(...) end`.

- [ ] **Step 1: Create EnemyUnitFirstSeen**

```java
package org.acme.starcraft.plugin.scouting.events;

import org.acme.starcraft.domain.UnitType;

/**
 * Fired once per enemy unit tag — on first observation each game.
 * Proxy for "unit produced" given SC2 only provides observation snapshots.
 * Inserted into ScoutingRuleUnit.unitEvents DataStore.
 */
public record EnemyUnitFirstSeen(UnitType type, long gameTimeMs) {}
```

- [ ] **Step 2: Create EnemyExpansionSeen**

```java
package org.acme.starcraft.plugin.scouting.events;

import org.acme.starcraft.domain.Point2d;

/**
 * Fired when an enemy unit is observed far from the estimated enemy main base.
 * Proxy for expansion detection — inserted at most once per map grid cell.
 * Inserted into ScoutingRuleUnit.expansionEvents DataStore.
 */
public record EnemyExpansionSeen(Point2d position, long gameTimeMs) {}
```

- [ ] **Step 3: Create EnemyArmyNearBase**

```java
package org.acme.starcraft.plugin.scouting.events;

/**
 * Fired when ≥ MIN_ARMY_SIZE enemy units are within NEAR_BASE_DISTANCE tiles of our Nexus.
 * Inserted into ScoutingRuleUnit.armyNearBaseEvents DataStore.
 * Java evicts events older than 10 seconds — so a non-empty DataStore means "active threat".
 */
public record EnemyArmyNearBase(int armySize, long gameTimeMs) {}
```

- [ ] **Step 4: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/scouting/
git commit -m "feat(scouting): add CEP event records

Refs #<rules-issue>"
```

---

## Task 5: Create ScoutingRuleUnit

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/ScoutingRuleUnit.java`

Follows the same pattern as `TacticsRuleUnit` — `DataStore` inputs, `List<String>` outputs.

- [ ] **Step 1: Write a failing build to verify the DRL will be discovered**

The DRL (Task 6) must be in `src/main/resources/org/acme/starcraft/plugin/scouting/` because Drools Quarkus discovers rules by matching the DRL package declaration to the `RuleUnitData` class package. Create the class first so the package exists.

```java
package org.acme.starcraft.plugin.scouting;

import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;
import org.acme.starcraft.plugin.scouting.events.EnemyArmyNearBase;
import org.acme.starcraft.plugin.scouting.events.EnemyExpansionSeen;
import org.acme.starcraft.plugin.scouting.events.EnemyUnitFirstSeen;

import java.util.ArrayList;
import java.util.List;

/**
 * Rule Unit data context for Drools scouting evaluation.
 *
 * <p>Inputs (populated from Java-side event buffers each tick):
 * <ul>
 *   <li>{@link #unitEvents} — enemy units first seen within the 3-min build-order window</li>
 *   <li>{@link #expansionEvents} — permanent: enemy-area unit sightings (expansion proxy)</li>
 *   <li>{@link #armyNearBaseEvents} — enemy army near our Nexus within the 10-sec window</li>
 * </ul>
 *
 * <p>Outputs (written by rules, read by DroolsScoutingTask after fire()):
 * <ul>
 *   <li>{@link #detectedBuilds} — detected build-order strings, e.g. "ZERG_ROACH_RUSH"</li>
 *   <li>{@link #timingAlerts} — non-empty = timing attack detected this tick</li>
 *   <li>{@link #postureDecisions} — "ALL_IN", "MACRO", or empty (= "UNKNOWN")</li>
 * </ul>
 */
public class ScoutingRuleUnit implements RuleUnitData {

    private final DataStore<EnemyUnitFirstSeen>  unitEvents         = DataSource.createStore();
    private final DataStore<EnemyExpansionSeen>  expansionEvents    = DataSource.createStore();
    private final DataStore<EnemyArmyNearBase>   armyNearBaseEvents = DataSource.createStore();

    private final List<String>  detectedBuilds   = new ArrayList<>();
    private final List<Boolean> timingAlerts     = new ArrayList<>();
    private final List<String>  postureDecisions = new ArrayList<>();

    public DataStore<EnemyUnitFirstSeen>  getUnitEvents()         { return unitEvents; }
    public DataStore<EnemyExpansionSeen>  getExpansionEvents()    { return expansionEvents; }
    public DataStore<EnemyArmyNearBase>   getArmyNearBaseEvents() { return armyNearBaseEvents; }

    public List<String>  getDetectedBuilds()   { return detectedBuilds; }
    public List<Boolean> getTimingAlerts()     { return timingAlerts; }
    public List<String>  getPostureDecisions() { return postureDecisions; }
}
```

- [ ] **Step 2: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS (no DRL yet — that's fine at compile stage).

- [ ] **Step 3: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/scouting/ScoutingRuleUnit.java
git commit -m "feat(scouting): add ScoutingRuleUnit data class

Refs #<rules-issue>"
```

---

## Task 6: Write the DRL rules

**Files:**
- Create: `src/main/resources/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.drl`

Six rules. Java manages temporal windows, so rules simply accumulate whatever is in the DataStore. Thresholds are declared as constants at the top.

- [ ] **Step 1: Create the DRL**

```
src/main/resources/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.drl
```

```drl
package org.acme.starcraft.plugin.scouting;
unit ScoutingRuleUnit;

import org.acme.starcraft.domain.UnitType;

// ---------------------------------------------------------------------------
// Build-order fingerprinting
//
// Java evicts unit events older than UNIT_WINDOW_MS (3 min) before each tick,
// so /unitEvents contains only events within the active window.
//
// Thresholds are R&D estimates — calibrate against replay data.
// ---------------------------------------------------------------------------

rule "Zerg Roach Rush"
    salience 100
when
    accumulate(
        /unitEvents[ this.type() == UnitType.ROACH ];
        $count : count();
        $count >= 6
    )
then
    detectedBuilds.add("ZERG_ROACH_RUSH");
end

rule "Terran 3-Rax"
    salience 100
when
    accumulate(
        /unitEvents[ this.type() == UnitType.MARINE ];
        $count : count();
        $count >= 12
    )
then
    detectedBuilds.add("TERRAN_3RAX");
end

rule "Protoss 4-Gate"
    salience 100
when
    accumulate(
        /unitEvents[ this.type() == UnitType.STALKER || this.type() == UnitType.ZEALOT ];
        $count : count();
        $count >= 8
    )
then
    detectedBuilds.add("PROTOSS_4GATE");
end

// ---------------------------------------------------------------------------
// Timing attack warning
//
// Java evicts armyNearBase events older than ARMY_WINDOW_MS (10 sec).
// A non-empty DataStore means "enemy army was near our Nexus in the last 10s".
// ---------------------------------------------------------------------------

rule "Timing Attack Incoming"
    salience 200
when
    /armyNearBaseEvents
then
    timingAlerts.add(Boolean.TRUE);
end

// ---------------------------------------------------------------------------
// Expansion posture
//
// expansionEvents accumulate permanently. "ALL_IN" when we have some scouting
// data (at least one unit sighted) but no expansion detected yet.
// Rules are mutually exclusive: not /expansionEvents vs /expansionEvents.
// ---------------------------------------------------------------------------

rule "Expansion: All-In"
    salience 90
when
    /unitEvents
    not /expansionEvents
then
    postureDecisions.add("ALL_IN");
end

rule "Expansion: Macro"
    salience 90
when
    /expansionEvents
then
    postureDecisions.add("MACRO");
end
```

- [ ] **Step 2: Verify Quarkus compiles the DRL**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS. If there is a DRL compilation error, the output will contain `Error` or `DRL` in the message — fix the offending rule before continuing.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/org/acme/starcraft/plugin/scouting/
git commit -m "feat(scouting): add DroolsScoutingTask.drl — 6 scouting rules

Refs #<rules-issue>"
```

---

## Task 7: Implement ScoutingSessionManager

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/ScoutingSessionManager.java`

Owns the Java-side event buffers. `DroolsScoutingTask` calls it each tick to process enemies, evict stale events, and build a `ScoutingRuleUnit` snapshot for firing.

- [ ] **Step 1: Write a failing test for reset behaviour**

Create `src/test/java/org/acme/starcraft/plugin/ScoutingSessionManagerTest.java`:

```java
package org.acme.starcraft.plugin;

import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.scouting.ScoutingSessionManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ScoutingSessionManagerTest {

    ScoutingSessionManager manager;

    @BeforeEach void setUp() { manager = new ScoutingSessionManager(); }

    @Test
    void firstSeenEnemyAddsUnitEvent() {
        manager.processFrame(List.of(roach("r-0", 10, 10)), 1000L, nexus(), enemyBase());
        assertThat(manager.buildRuleUnit().getUnitEvents()).isNotNull();
        // Unit event inserted — we verify via the rule unit; DataStore size not directly accessible
        // Instead, fire and check detectedBuilds in DroolsScoutingRulesTest
        // Here just verify no exception is thrown
    }

    @Test
    void sameTagNotInsertedTwice() {
        manager.processFrame(List.of(roach("r-0", 10, 10)), 1000L, nexus(), enemyBase());
        manager.processFrame(List.of(roach("r-0", 10, 10)), 2000L, nexus(), enemyBase());
        // seenTags prevents duplicate insertion — verify via seenTagCount()
        assertThat(manager.seenTagCount()).isEqualTo(1);
    }

    @Test
    void evictRemovesExpiredUnitEvents() {
        manager.processFrame(List.of(roach("r-0", 10, 10)), 0L, nexus(), enemyBase());
        manager.evict(ScoutingSessionManager.UNIT_WINDOW_MS + 1);
        assertThat(manager.unitBufferSize()).isEqualTo(0);
    }

    @Test
    void evictKeepsRecentUnitEvents() {
        manager.processFrame(List.of(roach("r-0", 10, 10)), 0L, nexus(), enemyBase());
        manager.evict(ScoutingSessionManager.UNIT_WINDOW_MS - 1);
        assertThat(manager.unitBufferSize()).isEqualTo(1);
    }

    @Test
    void resetClearsAllBuffers() {
        manager.processFrame(List.of(roach("r-0", 10, 10)), 1000L, nexus(), enemyBase());
        manager.reset();
        assertThat(manager.seenTagCount()).isEqualTo(0);
        assertThat(manager.unitBufferSize()).isEqualTo(0);
    }

    @Test
    void armyNearBaseAddsEvent() {
        // 3 units near our nexus at (8,8) — within NEAR_BASE_DISTANCE
        List<Unit> enemies = List.of(
            roach("r-0", 10, 10), roach("r-1", 11, 10), roach("r-2", 12, 10));
        manager.processFrame(enemies, 1000L, nexus(), enemyBase());
        assertThat(manager.armyBufferSize()).isEqualTo(1);
    }

    @Test
    void armyEventsExpireAfterWindow() {
        List<Unit> enemies = List.of(
            roach("r-0", 10, 10), roach("r-1", 11, 10), roach("r-2", 12, 10));
        manager.processFrame(enemies, 0L, nexus(), enemyBase());
        manager.evict(ScoutingSessionManager.ARMY_WINDOW_MS + 1);
        assertThat(manager.armyBufferSize()).isEqualTo(0);
    }

    @Test
    void enemyFarFromEnemyBaseAddsExpansionEvent() {
        // Unit at (128,128) — far from estimated enemy base at (224,224) by ~135 tiles
        // but also far from our nexus at (8,8); within EXPANSION_DISTANCE_THRESHOLD from center
        // enemyBase() returns (224,224); unit at (64,64) is ~226 tiles away — triggers expansion
        Unit farUnit = new Unit("x-0", UnitType.ROACH, new Point2d(64, 64), 100, 100);
        manager.processFrame(List.of(farUnit), 1000L, nexus(), enemyBase());
        assertThat(manager.expansionBufferSize()).isEqualTo(1);
    }

    @Test
    void expansionEventNotInsertedTwiceForSameCell() {
        Unit farUnit = new Unit("x-0", UnitType.ROACH, new Point2d(64, 64), 100, 100);
        Unit farUnit2 = new Unit("x-1", UnitType.ROACH, new Point2d(64, 64), 100, 100);
        manager.processFrame(List.of(farUnit), 1000L, nexus(), enemyBase());
        manager.processFrame(List.of(farUnit2), 2000L, nexus(), enemyBase());
        assertThat(manager.expansionBufferSize()).isEqualTo(1); // same cell, deduped
    }

    // helpers
    private Unit roach(String tag, float x, float y) {
        return new Unit(tag, UnitType.ROACH, new Point2d(x, y), 100, 100);
    }
    private Point2d nexus()     { return new Point2d(8, 8); }
    private Point2d enemyBase() { return new Point2d(224, 224); }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
mvn test -Dtest=ScoutingSessionManagerTest -q 2>&1 | head -20
```

Expected: compilation error — `ScoutingSessionManager` does not exist yet.

- [ ] **Step 3: Implement ScoutingSessionManager**

```java
package org.acme.starcraft.plugin.scouting;

import jakarta.enterprise.context.ApplicationScoped;
import org.acme.starcraft.domain.Point2d;
import org.acme.starcraft.domain.Unit;
import org.acme.starcraft.plugin.scouting.events.EnemyArmyNearBase;
import org.acme.starcraft.plugin.scouting.events.EnemyExpansionSeen;
import org.acme.starcraft.plugin.scouting.events.EnemyUnitFirstSeen;

import java.util.*;

/**
 * Manages Java-side event buffers for {@link DroolsScoutingTask}.
 *
 * <p>Maintains three rolling windows:
 * <ul>
 *   <li>Unit first-seen events — 3-minute build-order window (evicted by timestamp)</li>
 *   <li>Army-near-base events — 10-second threat window (evicted by timestamp)</li>
 *   <li>Expansion events     — permanent for the life of the game</li>
 * </ul>
 *
 * <p>Call {@link #reset()} on game restart (detected by frame going backwards).
 */
@ApplicationScoped
public class ScoutingSessionManager {

    /** Build-order detection window: 3 minutes at SC2 Faster speed. */
    public static final long UNIT_WINDOW_MS = 3L * 60 * 1000;

    /** Timing-attack threat window: 10 seconds. */
    public static final long ARMY_WINDOW_MS = 10L * 1000;

    /**
     * Distance threshold (tiles) from estimated enemy main base beyond which
     * a sighted unit is treated as evidence of an expansion or forward base.
     */
    public static final float EXPANSION_DISTANCE_THRESHOLD = 50f;

    /** Minimum enemy units near our Nexus to trigger an army-near-base event. */
    public static final int MIN_ARMY_NEAR_BASE = 3;

    /** Distance (tiles) from our Nexus that counts as "near our base". */
    public static final float NEAR_BASE_DISTANCE = 30f;

    private final Set<String>                  seenUnitTags       = new HashSet<>();
    private final Set<String>                  seenExpansionCells = new HashSet<>();
    private final Deque<EnemyUnitFirstSeen>    unitBuffer         = new ArrayDeque<>();
    private final Deque<EnemyArmyNearBase>     armyBuffer         = new ArrayDeque<>();
    private final List<EnemyExpansionSeen>     expansionBuffer    = new ArrayList<>();

    /** Clears all buffers. Call when a new game starts. */
    public void reset() {
        seenUnitTags.clear();
        seenExpansionCells.clear();
        unitBuffer.clear();
        armyBuffer.clear();
        expansionBuffer.clear();
    }

    /**
     * Processes visible enemy units for this tick, inserting new events into buffers.
     *
     * @param enemies          currently visible enemy units
     * @param gameTimeMs       current game time in milliseconds (frame × 1000/22.4)
     * @param ourNexus         position of our first Nexus (home base reference)
     * @param estimatedEnemyBase estimated position of the enemy main base
     */
    public void processFrame(List<Unit> enemies, long gameTimeMs,
                             Point2d ourNexus, Point2d estimatedEnemyBase) {
        long nearCount = 0;
        for (Unit e : enemies) {
            // Unit first-seen: insert once per unique tag
            if (seenUnitTags.add(e.tag())) {
                unitBuffer.add(new EnemyUnitFirstSeen(e.type(), gameTimeMs));
            }

            // Expansion proxy: enemy unit far from their estimated main base
            float distToEnemyBase = e.position().distanceTo(estimatedEnemyBase);
            if (distToEnemyBase > EXPANSION_DISTANCE_THRESHOLD) {
                String cell = (int) e.position().x() + ":" + (int) e.position().y();
                if (seenExpansionCells.add(cell)) {
                    expansionBuffer.add(new EnemyExpansionSeen(e.position(), gameTimeMs));
                }
            }

            // Count units near our base for army-near-base event
            if (e.position().distanceTo(ourNexus) < NEAR_BASE_DISTANCE) {
                nearCount++;
            }
        }

        if (nearCount >= MIN_ARMY_NEAR_BASE) {
            armyBuffer.add(new EnemyArmyNearBase((int) nearCount, gameTimeMs));
        }
    }

    /**
     * Removes events that have fallen outside their temporal window.
     * Call once per tick AFTER {@link #processFrame}.
     */
    public void evict(long currentGameTimeMs) {
        unitBuffer.removeIf(e -> currentGameTimeMs - e.gameTimeMs() > UNIT_WINDOW_MS);
        armyBuffer.removeIf(e -> currentGameTimeMs - e.gameTimeMs() > ARMY_WINDOW_MS);
        // expansionBuffer never evicted — permanent for the game
    }

    /**
     * Builds a fresh {@link ScoutingRuleUnit} populated from the current buffer contents.
     * Call after {@link #evict} so the rule unit only sees events within their windows.
     */
    public ScoutingRuleUnit buildRuleUnit() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        unitBuffer.forEach(data.getUnitEvents()::add);
        expansionBuffer.forEach(data.getExpansionEvents()::add);
        armyBuffer.forEach(data.getArmyNearBaseEvents()::add);
        return data;
    }

    // ---- Testability accessors (package-private) ----
    int seenTagCount()       { return seenUnitTags.size(); }
    int unitBufferSize()     { return unitBuffer.size(); }
    int armyBufferSize()     { return armyBuffer.size(); }
    int expansionBufferSize(){ return expansionBuffer.size(); }
}
```

- [ ] **Step 4: Run tests**

```bash
mvn test -Dtest=ScoutingSessionManagerTest -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/scouting/ScoutingSessionManager.java \
        src/test/java/org/acme/starcraft/plugin/ScoutingSessionManagerTest.java
git commit -m "feat(scouting): implement ScoutingSessionManager — event buffer lifecycle

Refs #<impl-issue>"
```

---

## Task 8: Implement DroolsScoutingTask

**Files:**
- Create: `src/main/java/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.java`
- Modify: `src/main/java/org/acme/starcraft/plugin/BasicScoutingTask.java` — deactivate as CDI bean

`DroolsScoutingTask` replaces `BasicScoutingTask` as the active implementation. Both have `@ApplicationScoped @CaseType("starcraft-game")` — Quarkus CDI will throw ambiguous-bean error at startup. Fix: add `@jakarta.annotation.Priority(1)` to `DroolsScoutingTask` and `@jakarta.enterprise.inject.Alternative` to `BasicScoutingTask` (without `@Priority`, alternatives are inactive by default in Quarkus Arc).

- [ ] **Step 1: Deactivate BasicScoutingTask**

Open `src/main/java/org/acme/starcraft/plugin/BasicScoutingTask.java`. Add `@Alternative` import and annotation:

```java
import jakarta.enterprise.inject.Alternative;

@Alternative                       // add this — deactivates bean unless explicitly selected
@ApplicationScoped
@CaseType("starcraft-game")
public class BasicScoutingTask implements ScoutingTask {
```

- [ ] **Step 2: Create DroolsScoutingTask**

```java
package org.acme.starcraft.plugin.scouting;

import io.casehub.annotation.CaseType;
import io.casehub.core.CaseFile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.ScoutingTask;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.MoveIntent;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.jboss.logging.Logger;

import java.util.Comparator;
import java.util.List;
import java.util.Set;

/**
 * Drools-backed {@link ScoutingTask} — fourth R&D integration.
 *
 * <p>Each tick:
 * <ol>
 *   <li>Detects game restarts (frame going backwards) and resets buffers.</li>
 *   <li>Computes passive intel: {@code ENEMY_ARMY_SIZE} and {@code NEAREST_THREAT}.</li>
 *   <li>Updates Java event buffers via {@link ScoutingSessionManager}; evicts expired events.</li>
 *   <li>Fires a fresh {@link RuleUnitInstance} from the current buffer state.</li>
 *   <li>Writes {@code ENEMY_BUILD_ORDER}, {@code TIMING_ATTACK_INCOMING}, {@code ENEMY_POSTURE}.</li>
 *   <li>Dispatches active probe scout (same logic as BasicScoutingTask).</li>
 * </ol>
 *
 * <p>Replaces {@link org.acme.starcraft.plugin.BasicScoutingTask} as the active CDI bean
 * (BasicScoutingTask is marked {@code @Alternative} to avoid ambiguous-bean conflict).
 */
@ApplicationScoped
@CaseType("starcraft-game")
public class DroolsScoutingTask implements ScoutingTask {

    /** Game-speed constant: SC2 Faster = 22.4 frames per second. */
    static final double FRAMES_PER_SECOND = 22.4;

    /** Delay before sending a scout — let the economy stabilise first. */
    static final int SCOUT_DELAY_TICKS = 20;

    private static final Logger log = Logger.getLogger(DroolsScoutingTask.class);

    private final RuleUnit<ScoutingRuleUnit> ruleUnit;
    private final ScoutingSessionManager     sessionManager;
    private final IntentQueue                intentQueue;

    /** Tag of the probe currently assigned to scout. Null when no active scout. */
    private volatile String scoutProbeTag;
    /** Last seen game frame — used to detect game restart in mock loop. */
    private volatile long lastFrame = -1;

    @Inject
    public DroolsScoutingTask(RuleUnit<ScoutingRuleUnit> ruleUnit,
                               ScoutingSessionManager sessionManager,
                               IntentQueue intentQueue) {
        this.ruleUnit       = ruleUnit;
        this.sessionManager = sessionManager;
        this.intentQueue    = intentQueue;
    }

    @Override public String getId()   { return "scouting.drools-cep"; }
    @Override public String getName() { return "Drools CEP Scouting"; }
    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys()  {
        return Set.of(
            StarCraftCaseFile.ENEMY_ARMY_SIZE,
            StarCraftCaseFile.NEAREST_THREAT,
            StarCraftCaseFile.ENEMY_BUILD_ORDER,
            StarCraftCaseFile.TIMING_ATTACK_INCOMING,
            StarCraftCaseFile.ENEMY_POSTURE);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void execute(CaseFile caseFile) {
        List<Unit>     enemies   = (List<Unit>)     caseFile.get(StarCraftCaseFile.ENEMY_UNITS,  List.class).orElse(List.of());
        List<Building> buildings = (List<Building>) caseFile.get(StarCraftCaseFile.MY_BUILDINGS, List.class).orElse(List.of());
        List<Unit>     workers   = (List<Unit>)     caseFile.get(StarCraftCaseFile.WORKERS,      List.class).orElse(List.of());
        long frame = caseFile.get(StarCraftCaseFile.GAME_FRAME, Long.class).orElse(0L);

        // Detect game restart (mock loop resets frame to 0)
        if (frame < lastFrame) {
            sessionManager.reset();
            scoutProbeTag = null;
        }
        lastFrame = frame;

        long gameTimeMs = (long) (frame * (1000.0 / FRAMES_PER_SECOND));
        Point2d ourNexus      = nexusPosition(buildings);
        Point2d estimatedBase = estimatedEnemyBase(ourNexus);

        // --- Passive intel (plain Java, no rules needed) ---
        caseFile.put(StarCraftCaseFile.ENEMY_ARMY_SIZE, enemies.size());
        if (!enemies.isEmpty()) {
            enemies.stream()
                .min(Comparator.comparingDouble(e -> e.position().distanceTo(ourNexus)))
                .ifPresent(nearest -> caseFile.put(StarCraftCaseFile.NEAREST_THREAT, nearest.position()));
        }

        // --- CEP event accumulation ---
        sessionManager.processFrame(enemies, gameTimeMs, ourNexus, estimatedBase);
        sessionManager.evict(gameTimeMs);

        // --- Drools rules firing ---
        ScoutingRuleUnit data = sessionManager.buildRuleUnit();
        try (RuleUnitInstance<ScoutingRuleUnit> instance = ruleUnit.createInstance(data)) {
            instance.fire();
        }

        // --- Write CEP intel to CaseFile ---
        if (!data.getDetectedBuilds().isEmpty()) {
            caseFile.put(StarCraftCaseFile.ENEMY_BUILD_ORDER, data.getDetectedBuilds().get(0));
        }
        caseFile.put(StarCraftCaseFile.TIMING_ATTACK_INCOMING, !data.getTimingAlerts().isEmpty());
        String posture = data.getPostureDecisions().isEmpty()
            ? "UNKNOWN"
            : data.getPostureDecisions().get(0);
        caseFile.put(StarCraftCaseFile.ENEMY_POSTURE, posture);

        log.debugf("[SCOUTING] enemies=%d | build=%s | timing=%b | posture=%s",
            enemies.size(),
            data.getDetectedBuilds().isEmpty() ? "?" : data.getDetectedBuilds().get(0),
            !data.getTimingAlerts().isEmpty(),
            posture);

        // --- Active scouting (same as BasicScoutingTask) ---
        if (enemies.isEmpty()) {
            maybeSendScout(frame, buildings, workers);
        } else {
            scoutProbeTag = null; // enemies found — release scout
        }
    }

    private void maybeSendScout(long frame, List<Building> buildings, List<Unit> workers) {
        if (frame < SCOUT_DELAY_TICKS) return;
        if (workers.isEmpty()) return;

        if (scoutProbeTag != null) {
            boolean alive = workers.stream().anyMatch(w -> w.tag().equals(scoutProbeTag));
            if (alive) return;
            scoutProbeTag = null;
        }

        Unit scout = workers.get(workers.size() - 1);
        scoutProbeTag = scout.tag();
        Point2d target = estimatedEnemyBase(nexusPosition(buildings));
        intentQueue.add(new MoveIntent(scout.tag(), target));
        log.infof("[SCOUTING] Scout probe %s dispatched toward %s", scoutProbeTag, target);
    }

    static Point2d estimatedEnemyBase(Point2d ourBase) {
        float targetX = ourBase.x() < 64 ? 224 : 32;
        float targetY = ourBase.y() < 64 ? 224 : 32;
        return new Point2d(targetX, targetY);
    }

    private static Point2d nexusPosition(List<Building> buildings) {
        return buildings.stream()
            .filter(b -> b.type() == BuildingType.NEXUS)
            .findFirst()
            .map(Building::position)
            .orElse(new Point2d(0, 0));
    }
}
```

- [ ] **Step 3: Compile**

```bash
mvn compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Run all tests**

```bash
mvn test -q
```

Expected: BUILD SUCCESS. If `BasicScoutingTaskTest` fails because `BasicScoutingTask` is marked `@Alternative` and can no longer be instantiated directly — it can still be instantiated with `new` in plain JUnit (CDI annotations have no effect outside Quarkus). Tests should still pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/org/acme/starcraft/plugin/BasicScoutingTask.java \
        src/main/java/org/acme/starcraft/plugin/scouting/DroolsScoutingTask.java
git commit -m "feat(scouting): implement DroolsScoutingTask — Drools CEP scouting plugin

Refs #<impl-issue>"
```

---

## Task 9: DRL rules tests

**Files:**
- Create: `src/test/java/org/acme/starcraft/plugin/DroolsScoutingRulesTest.java`

Tests each of the six DRL rules in isolation. Requires `@QuarkusTest` because `DataSource.createStore()` requires Quarkus build-time initialisation (GE-0053).

- [ ] **Step 1: Write the test**

```java
package org.acme.starcraft.plugin;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.domain.UnitType;
import org.acme.starcraft.plugin.scouting.ScoutingRuleUnit;
import org.acme.starcraft.plugin.scouting.events.EnemyArmyNearBase;
import org.acme.starcraft.plugin.scouting.events.EnemyExpansionSeen;
import org.acme.starcraft.plugin.scouting.events.EnemyUnitFirstSeen;
import org.acme.starcraft.domain.Point2d;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests each DroolsScoutingTask.drl rule in isolation.
 *
 * <p>Requires {@code @QuarkusTest} — {@link org.drools.ruleunits.api.DataSource#createStore()}
 * is initialised at Quarkus build time (GE-0053).
 */
@QuarkusTest
class DroolsScoutingRulesTest {

    @Inject RuleUnit<ScoutingRuleUnit> ruleUnit;

    // ---- Build-order rules ----

    @Test
    void sixRoachesDetectsZergRoachRush() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 6; i++) data.getUnitEvents().add(unit(UnitType.ROACH, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).contains("ZERG_ROACH_RUSH");
    }

    @Test
    void fiveRoachesDoesNotDetectRoachRush() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 5; i++) data.getUnitEvents().add(unit(UnitType.ROACH, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).doesNotContain("ZERG_ROACH_RUSH");
    }

    @Test
    void twelveMariinesDetectsTerran3Rax() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 12; i++) data.getUnitEvents().add(unit(UnitType.MARINE, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).contains("TERRAN_3RAX");
    }

    @Test
    void elevenMarinesDoesNotDetect3Rax() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 11; i++) data.getUnitEvents().add(unit(UnitType.MARINE, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).doesNotContain("TERRAN_3RAX");
    }

    @Test
    void eightStalkersDetectsProtoss4Gate() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 8; i++) data.getUnitEvents().add(unit(UnitType.STALKER, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).contains("PROTOSS_4GATE");
    }

    @Test
    void mixedStalkerZealotCountsTowardFourGate() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        for (int i = 0; i < 4; i++) data.getUnitEvents().add(unit(UnitType.STALKER, i * 1000L));
        for (int i = 0; i < 4; i++) data.getUnitEvents().add(unit(UnitType.ZEALOT, i * 1000L));
        fire(data);
        assertThat(data.getDetectedBuilds()).contains("PROTOSS_4GATE");
    }

    // ---- Timing attack rule ----

    @Test
    void armyNearBaseEventDetectsTimingAttack() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        data.getArmyNearBaseEvents().add(new EnemyArmyNearBase(5, 1000L));
        fire(data);
        assertThat(data.getTimingAlerts()).isNotEmpty();
    }

    @Test
    void noArmyNearBaseProducesNoTimingAlert() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        fire(data);
        assertThat(data.getTimingAlerts()).isEmpty();
    }

    // ---- Expansion posture rules ----

    @Test
    void unitEventsWithNoExpansionDetectsAllIn() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        data.getUnitEvents().add(unit(UnitType.MARINE, 1000L));
        fire(data);
        assertThat(data.getPostureDecisions()).contains("ALL_IN");
    }

    @Test
    void expansionEventDetectsMacro() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        data.getExpansionEvents().add(new EnemyExpansionSeen(new Point2d(64, 64), 1000L));
        fire(data);
        assertThat(data.getPostureDecisions()).contains("MACRO");
    }

    @Test
    void noEventsPostureIsUnknown() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        fire(data);
        // No unit events → "All-In" rule doesn't fire; no expansion → "Macro" doesn't fire
        assertThat(data.getPostureDecisions()).isEmpty();
    }

    @Test
    void expansionPreventsAllInPosture() {
        ScoutingRuleUnit data = new ScoutingRuleUnit();
        data.getUnitEvents().add(unit(UnitType.MARINE, 1000L));
        data.getExpansionEvents().add(new EnemyExpansionSeen(new Point2d(64, 64), 1000L));
        fire(data);
        assertThat(data.getPostureDecisions()).doesNotContain("ALL_IN");
        assertThat(data.getPostureDecisions()).contains("MACRO");
    }

    // ---- Helpers ----

    private void fire(ScoutingRuleUnit data) {
        try (RuleUnitInstance<ScoutingRuleUnit> instance = ruleUnit.createInstance(data)) {
            instance.fire();
        }
    }

    private EnemyUnitFirstSeen unit(UnitType type, long gameTimeMs) {
        return new EnemyUnitFirstSeen(type, gameTimeMs);
    }
}
```

- [ ] **Step 2: Run tests**

```bash
mvn test -Dtest=DroolsScoutingRulesTest -q
```

Expected: BUILD SUCCESS, all 13 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/org/acme/starcraft/plugin/DroolsScoutingRulesTest.java
git commit -m "test(scouting): DroolsScoutingRulesTest — all 6 DRL rules verified

Refs #<test-issue>"
```

---

## Task 10: Integration test

**Files:**
- Create: `src/test/java/org/acme/starcraft/plugin/DroolsScoutingTaskIT.java`

Tests the full CDI pipeline: CaseFile state → `DroolsScoutingTask.execute()` → CaseFile keys written.

- [ ] **Step 1: Write the integration test**

```java
package org.acme.starcraft.plugin;

import io.casehub.annotation.CaseType;
import io.casehub.core.DefaultCaseFile;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.acme.starcraft.agent.StarCraftCaseFile;
import org.acme.starcraft.agent.plugin.ScoutingTask;
import org.acme.starcraft.domain.*;
import org.acme.starcraft.plugin.scouting.DroolsScoutingTask;
import org.acme.starcraft.plugin.scouting.ScoutingSessionManager;
import org.acme.starcraft.sc2.IntentQueue;
import org.acme.starcraft.sc2.intent.MoveIntent;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * End-to-end integration tests for DroolsScoutingTask.
 */
@QuarkusTest
class DroolsScoutingTaskIT {

    @Inject @CaseType("starcraft-game") ScoutingTask scoutingTask;
    @Inject IntentQueue intentQueue;
    @Inject ScoutingSessionManager sessionManager;

    @BeforeEach @AfterEach
    void reset() {
        intentQueue.drainAll();
        sessionManager.reset(); // clear persistent buffers between tests
    }

    // ---- Passive intel ----

    @Test
    void writesArmySizeEachTick() {
        var cf = caseFile(List.of(enemy(10, 10), enemy(20, 20)), List.of(), 100L);
        scoutingTask.execute(cf);
        assertThat(cf.get(StarCraftCaseFile.ENEMY_ARMY_SIZE, Integer.class)).contains(2);
    }

    @Test
    void writesNearestThreat() {
        var cf = caseFile(List.of(enemy(10, 10), enemy(100, 100)), List.of(), 100L);
        scoutingTask.execute(cf);
        assertThat(cf.get(StarCraftCaseFile.NEAREST_THREAT, Point2d.class))
            .contains(new Point2d(10, 10));
    }

    // ---- CEP keys written each tick ----

    @Test
    void timingAttackFalseWhenNoArmyNearBase() {
        var cf = caseFile(List.of(enemy(200, 200)), List.of(), 100L); // far from nexus at (8,8)
        scoutingTask.execute(cf);
        assertThat(cf.get(StarCraftCaseFile.TIMING_ATTACK_INCOMING, Boolean.class))
            .contains(Boolean.FALSE);
    }

    @Test
    void postureUnknownWhenNoEnemiesEverSeen() {
        var cf = caseFile(List.of(), List.of(), 100L);
        scoutingTask.execute(cf);
        assertThat(cf.get(StarCraftCaseFile.ENEMY_POSTURE, String.class))
            .contains("UNKNOWN");
    }

    @Test
    void buildOrderDetectedAfterEnoughSightings() {
        // Execute 6 ticks with unique ROACH tags — accumulates in buffer
        for (int i = 0; i < 6; i++) {
            var cf = caseFile(
                List.of(new Unit("r-" + i, UnitType.ROACH, new Point2d(200, 200), 100, 100)),
                List.of(),
                (long)(i + 1) * 500);  // 500ms apart — all within 3-min window
            scoutingTask.execute(cf);
        }
        var finalCf = caseFile(List.of(), List.of(), 6 * 500L);
        scoutingTask.execute(finalCf);
        assertThat(finalCf.get(StarCraftCaseFile.ENEMY_BUILD_ORDER, String.class))
            .contains("ZERG_ROACH_RUSH");
    }

    @Test
    void scoutProbeDispatchedAfterDelay() {
        var cf = caseFile(List.of(), List.of(probe("p-0")),
            (long) DroolsScoutingTask.SCOUT_DELAY_TICKS);
        scoutingTask.execute(cf);
        assertThat(intentQueue.pending())
            .hasSize(1)
            .first().isInstanceOf(MoveIntent.class);
    }

    // ---- Helpers ----

    private DefaultCaseFile caseFile(List<Unit> enemies, List<Unit> workers, long frame) {
        var cf = new DefaultCaseFile("test-" + System.nanoTime(), "starcraft-game", null, null);
        cf.put(StarCraftCaseFile.ENEMY_UNITS,  enemies);
        cf.put(StarCraftCaseFile.WORKERS,      workers);
        cf.put(StarCraftCaseFile.MY_BUILDINGS, List.of(nexus()));
        cf.put(StarCraftCaseFile.GAME_FRAME,   frame);
        cf.put(StarCraftCaseFile.READY,        Boolean.TRUE);
        return cf;
    }

    private Unit enemy(float x, float y) {
        return new Unit("e-" + System.nanoTime(), UnitType.ZEALOT, new Point2d(x, y), 100, 100);
    }

    private Unit probe(String tag) {
        return new Unit(tag, UnitType.PROBE, new Point2d(9, 9), 45, 45);
    }

    private Building nexus() {
        return new Building("n-0", BuildingType.NEXUS, new Point2d(8, 8), 1500, 1500, true);
    }
}
```

- [ ] **Step 2: Run tests**

```bash
mvn test -Dtest=DroolsScoutingTaskIT -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3: Run full test suite**

```bash
mvn test -q
```

Expected: BUILD SUCCESS, all tests pass (previous count was 152; expect ~185+ with new tests).

- [ ] **Step 4: Commit**

```bash
git add src/test/java/org/acme/starcraft/plugin/DroolsScoutingTaskIT.java
git commit -m "test(scouting): DroolsScoutingTaskIT — full pipeline integration tests

Refs #<test-issue>"
```

---

## Task 11: Update NATIVE.md and close issues

**Files:**
- Modify: `NATIVE.md`

- [ ] **Step 1: Add DroolsScoutingTask entry**

Open `NATIVE.md`. Add a row to the Dependencies table:

```markdown
| drools-cep-scouting | — | ✅ Native-safe | Rule unit model with DataStore accumulation; no runtime bytecode gen. Drools Executable Model handles AOT compilation. |
```

- [ ] **Step 2: Close GitHub issues**

```bash
gh issue close <rules-issue> --repo mdproctor/starcraft \
  --comment "ScoutingRuleUnit, event records, and DRL compiled and tested."

gh issue close <impl-issue> --repo mdproctor/starcraft \
  --comment "ScoutingSessionManager and DroolsScoutingTask implemented."

gh issue close <test-issue> --repo mdproctor/starcraft \
  --comment "DroolsScoutingRulesTest (13 tests) and DroolsScoutingTaskIT passing."

gh issue close <domain-issue> --repo mdproctor/starcraft \
  --comment "UnitType and CaseFile keys extended."

gh issue close <epic> --repo mdproctor/starcraft \
  --comment "Phase 4 complete — DroolsScoutingTask shipped. Full test suite passing."
```

- [ ] **Step 3: Commit**

```bash
git add NATIVE.md
git commit -m "docs: update NATIVE.md for DroolsScoutingTask CEP pattern

Closes #<epic>"
```
