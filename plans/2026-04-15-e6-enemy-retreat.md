# E6: Enemy Retreat & Regroup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add configurable per-unit and army-wide retreat triggers to the enemy AI so wounded or depleted attack waves pull back to the staging area, preserve their HP, and rejoin the next wave.

**Architecture:** `EnemyAttackConfig` gains two new int fields (`retreatHealthPercent`, `retreatArmyPercent`; 0 = disabled). `EmulatedGame` gains a `Set<String> retreatingUnits`, an `initialAttackSize` counter, and a new `tickEnemyRetreat()` method inserted into the tick loop after `resolveCombat()`. Retreating units move visibly back to `STAGING_POS` and are transferred to `enemyStagingArea` on arrival with their current (damaged) HP.

**Tech Stack:** Pure Java records, JUnit 5, AssertJ, Maven Surefire. No new dependencies.

---

## File Map

| Action | Path | Purpose |
|---|---|---|
| Modify | `src/main/java/io/quarkmind/domain/EnemyAttackConfig.java` | Add `retreatHealthPercent`, `retreatArmyPercent` |
| Modify | `src/main/java/io/quarkmind/domain/EnemyStrategy.java` | Update `defaultProtoss()` with retreat defaults |
| Modify | `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | New fields, `STAGING_POS`, helpers, `tickEnemyRetreat()`, `resolveCombat()` cleanup |
| Modify | `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java` | Update 2-arg `EnemyAttackConfig` calls + add 2 retreat-config tests |
| Modify | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Update 10 existing 2-arg calls + add 9 retreat tests |

---

## Task 1: Create GitHub Epic and Issues

**Files:** none (GitHub only)

- [ ] **Step 1: Create the E6 milestone**

```bash
cd /Users/mdproctor/claude/quarkmind

MILESTONE=$(gh api repos/mdproctor/quarkmind/milestones \
  -f title="E6: Enemy Retreat & Regroup" \
  -f state=open \
  --jq '.number')
echo "Milestone: $MILESTONE"
```

- [ ] **Step 2: Create the four child issues**

```bash
gh issue create --repo mdproctor/quarkmind \
  --title "E6-1: EnemyAttackConfig — retreatHealthPercent + retreatArmyPercent fields" \
  --body "Add two new int fields to EnemyAttackConfig record. 0 = disabled. Update defaultProtoss() with retreatHealthPercent=30, retreatArmyPercent=50." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E6-2: EmulatedGame — retreatingUnits, initialAttackSize, STAGING_POS, test helpers" \
  --body "Add Set<String> retreatingUnits, int initialAttackSize fields. Extract STAGING_POS constant. Add retreatingUnitTags(), setInitialAttackSizeForTesting(), setEnemyHealthForTesting() helpers. Wire initialAttackSize on wave launch. Clean retreatingUnits on unit death in resolveCombat." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E6-3: EmulatedGame.tickEnemyRetreat() — per-unit + army-wide thresholds + staging transfer" \
  --body "New private method in tick loop after resolveCombat(). Per-unit health threshold, army-wide depletion threshold, arrival transfer to enemyStagingArea with damaged HP preserved." \
  --milestone "$MILESTONE"

gh issue create --repo mdproctor/quarkmind \
  --title "E6-4: Tests — 9 new EmulatedGameTest retreat tests + EnemyStrategyTest updates" \
  --body "9 new unit tests covering both retreat triggers, movement, staging transfer, HP preservation, disabled thresholds, pre-wave guard, dead-unit cleanup. Update all 2-arg EnemyAttackConfig calls." \
  --milestone "$MILESTONE"
```

- [ ] **Step 3: Record issue numbers**

The issues are sequential. Note all four numbers — you'll use them in commit messages as `Refs #N`.

---

## Task 2: EnemyAttackConfig — Two New Fields (TDD)

**Files:**
- Modify: `src/main/java/io/quarkmind/domain/EnemyAttackConfig.java`
- Modify: `src/main/java/io/quarkmind/domain/EnemyStrategy.java`
- Modify: `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`

This is a **breaking change** — adding two fields to a record means every existing constructor call needs updating. The tests enforce the correct defaults.

- [ ] **Step 1: Write two failing tests in EnemyStrategyTest**

In `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`, add at the end (before the closing `}`):

```java
@Test
void attackConfigHoldsRetreatThresholds() {
    EnemyAttackConfig cfg = new EnemyAttackConfig(3, 200, 30, 50);
    assertThat(cfg.retreatHealthPercent()).isEqualTo(30);
    assertThat(cfg.retreatArmyPercent()).isEqualTo(50);
}

@Test
void defaultProtossHasRetreatDefaults() {
    EnemyAttackConfig cfg = EnemyStrategy.defaultProtoss().attackConfig();
    assertThat(cfg.retreatHealthPercent()).isEqualTo(30);
    assertThat(cfg.retreatArmyPercent()).isEqualTo(50);
}
```

- [ ] **Step 2: Run — confirm tests fail**

```bash
mvn test -Dtest=EnemyStrategyTest -q 2>&1 | tail -5
```

Expected: compilation failure — `EnemyAttackConfig` has no 4-arg constructor.

- [ ] **Step 3: Update EnemyAttackConfig.java**

Replace the entire file:

```java
package io.quarkmind.domain;

public record EnemyAttackConfig(
    int armyThreshold,
    int attackIntervalFrames,
    int retreatHealthPercent,   // 0 = disabled; retreat when HP+shields < X% of max
    int retreatArmyPercent      // 0 = disabled; retreat when < X% of launched units alive
) {}
```

- [ ] **Step 4: Update defaultProtoss() in EnemyStrategy.java**

Change the `EnemyAttackConfig` constructor call in `defaultProtoss()`:

```java
// Before:
new EnemyAttackConfig(3, 200)

// After:
new EnemyAttackConfig(3, 200, 30, 50)
```

- [ ] **Step 5: Fix all existing 2-arg EnemyAttackConfig calls**

These files have 2-arg calls that must gain `, 0, 0` (no retreat — preserves existing behaviour):

**In `src/test/java/io/quarkmind/domain/EnemyStrategyTest.java`**, update two locations:

```java
// attackConfigHoldsBothTriggers (line 18):
// Before: new EnemyAttackConfig(3, 200)
// After:
EnemyAttackConfig cfg = new EnemyAttackConfig(3, 200, 0, 0);

// customStrategyCanBeConstructedProgrammatically (line 72):
// Before: new EnemyAttackConfig(2, 100)
// After:
new EnemyAttackConfig(2, 100, 0, 0)
```

**In `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`**, update all 10 occurrences — every `new EnemyAttackConfig(X, Y)` becomes `new EnemyAttackConfig(X, Y, 0, 0)`:

```java
// enemyAccumulatesMineralsEachTick:
new EnemyAttackConfig(10, 9999, 0, 0)

// enemyTrainsUnitWhenMineralsAfford:
new EnemyAttackConfig(10, 9999, 0, 0)

// enemyDoesNotTrainWhenInsufficientMinerals:
new EnemyAttackConfig(10, 9999, 0, 0)

// enemySendsAttackWhenArmyThresholdMet:
new EnemyAttackConfig(1, 9999, 0, 0)

// enemySendsAttackWhenTimerFires:
new EnemyAttackConfig(100, 5, 0, 0)

// timerFiresBeforeArmyThreshold:
new EnemyAttackConfig(3, 5, 0, 0)

// enemyStagingClearedAfterAttack:
new EnemyAttackConfig(1, 9999, 0, 0)

// enemyBuildOrderLoops:
new EnemyAttackConfig(10, 9999, 0, 0)

// enemyBuildOrderStopsWhenExhausted:
new EnemyAttackConfig(10, 9999, 0, 0)

// enemyUnitsStayAtSpawnUntilAttack:
new EnemyAttackConfig(5, 9999, 0, 0)
```

- [ ] **Step 6: Run EnemyStrategyTest — confirm all pass**

```bash
mvn test -Dtest=EnemyStrategyTest -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Run full suite — confirm no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS. All existing E4 economy tests still pass with `, 0, 0`.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/quarkmind/domain/EnemyAttackConfig.java \
        src/main/java/io/quarkmind/domain/EnemyStrategy.java \
        src/test/java/io/quarkmind/domain/EnemyStrategyTest.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e6): EnemyAttackConfig — retreatHealthPercent + retreatArmyPercent fields

Refs #<E6-1 issue number>"
```

---

## Task 3: EmulatedGame — State, Helpers, and Infrastructure

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Write one structural failing test**

At the end of `EmulatedGameTest` (before closing `}`), add:

```java
// ---- E6: retreat infrastructure ----

@Test
void retreatingUnitTagsIsInitiallyEmpty() {
    assertThat(game.retreatingUnitTags()).isEmpty();
}
```

- [ ] **Step 2: Run — confirm it fails**

```bash
mvn test -Dtest="EmulatedGameTest#retreatingUnitTagsIsInitiallyEmpty" -q 2>&1 | tail -5
```

Expected: compilation failure — `retreatingUnitTags()` does not exist.

- [ ] **Step 3: Add E6 fields and STAGING_POS constant to EmulatedGame**

In `EmulatedGame.java`, add after the existing `private static final Point2d NEXUS_POS` line:

```java
private static final Point2d STAGING_POS = new Point2d(26, 26);
```

After the E4 fields block (after `private final List<Unit> enemyStagingArea`), add:

```java
// E6: retreat tracking
private final Set<String> retreatingUnits  = new HashSet<>();
private int               initialAttackSize = 0;
```

- [ ] **Step 4: Update reset() to clear E6 state**

In `reset()`, after `enemyStagingArea.clear();`, add:

```java
retreatingUnits.clear();
initialAttackSize = 0;
```

- [ ] **Step 5: Extract STAGING_POS in tickEnemyStrategy() and configureWave()**

In `tickEnemyStrategy()`, find:
```java
enemyStagingArea.add(new Unit(tag, step.unitType(),
    new Point2d(26, 26), hp, hp,
```
Replace `new Point2d(26, 26)` with `STAGING_POS`.

In `configureWave()`, find:
```java
pendingWaves.add(new EnemyWave(
    spawnFrame,
    new ArrayList<>(types),
    new Point2d(26, 26),
    new Point2d(8, 8)
));
```
Replace `new Point2d(26, 26)` (spawn position) with `STAGING_POS`. Leave `new Point2d(8, 8)` (target = nexus) unchanged.

- [ ] **Step 6: Record initialAttackSize on wave launch in tickEnemyStrategy()**

In `tickEnemyStrategy()`, find the wave-launch block:
```java
if ((thresholdMet || timerFired) && !enemyStagingArea.isEmpty()) {
    for (Unit u : enemyStagingArea) {
```
Add one line immediately before the for-loop:
```java
if ((thresholdMet || timerFired) && !enemyStagingArea.isEmpty()) {
    initialAttackSize = enemyStagingArea.size();   // E6: denominator for army retreat check
    for (Unit u : enemyStagingArea) {
```

- [ ] **Step 7: Clean retreatingUnits in resolveCombat() dead-unit removal**

In `resolveCombat()`, find the enemy dead-unit block:
```java
enemyUnits.removeIf(u -> {
    if (u.health() <= 0) {
        enemyTargets.remove(u.tag());
        enemyCooldowns.remove(u.tag());
        return true;
    }
    return false;
});
```
Add `retreatingUnits.remove(u.tag());` alongside the other cleanups:
```java
enemyUnits.removeIf(u -> {
    if (u.health() <= 0) {
        enemyTargets.remove(u.tag());
        enemyCooldowns.remove(u.tag());
        retreatingUnits.remove(u.tag());  // E6: clean up if killed while retreating
        return true;
    }
    return false;
});
```

- [ ] **Step 8: Add three package-private test helpers at the bottom of EmulatedGame**

Add after the existing `setEnemyShieldsForTesting` helper:

```java
/** Returns a copy of retreating unit tags — for E6 retreat assertions. */
Set<String> retreatingUnitTags() { return Set.copyOf(retreatingUnits); }

/** Sets initialAttackSize directly — simulates a wave having been launched. */
void setInitialAttackSizeForTesting(int n) { this.initialAttackSize = n; }

/** Sets an enemy unit's health directly — for retreat health threshold tests. */
void setEnemyHealthForTesting(String tag, int health) {
    enemyUnits.replaceAll(u -> u.tag().equals(tag)
        ? new Unit(u.tag(), u.type(), u.position(), health, u.maxHealth(),
                   u.shields(), u.maxShields())
        : u);
}
```

- [ ] **Step 9: Run the structural test — confirm it passes**

```bash
mvn test -Dtest="EmulatedGameTest#retreatingUnitTagsIsInitiallyEmpty" -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 10: Run full suite — confirm no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 11: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e6): EmulatedGame — retreatingUnits, initialAttackSize, STAGING_POS, helpers

Refs #<E6-2 issue number>"
```

---

## Task 4: tickEnemyRetreat() — TDD

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Write all 9 failing tests**

Add this section to `EmulatedGameTest` after `retreatingUnitTagsIsInitiallyEmpty`:

```java
// ---- E6: retreat logic ----

@Test
void lowHealthUnitRetreats() {
    // Zealot HP+shields = 1/150 = 0.7% — below retreatHealthPercent=30
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 1);
    game.setEnemyShieldsForTesting(tag, 0);
    game.setInitialAttackSizeForTesting(1);

    game.tick();

    assertThat(game.retreatingUnitTags()).contains(tag);
}

@Test
void healthyUnitDoesNotRetreat() {
    // Zealot at 80 HP + 50 shields = 130/150 = 87% — well above retreatHealthPercent=30
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    game.setInitialAttackSizeForTesting(1);

    game.tick();

    assertThat(game.retreatingUnitTags()).isEmpty();
}

@Test
void armyDepletionTriggersGroupRetreat() {
    // 1 unit alive of 4 launched = 25% — below retreatArmyPercent=50
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 0, 50);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setInitialAttackSizeForTesting(4); // simulate: 4 launched, 3 died, 1 remains

    game.tick();

    assertThat(game.retreatingUnitTags()).contains(tag);
}

@Test
void retreatingUnitMovesTowardStaging() {
    // Tick 1: retreat fires → target becomes STAGING_POS (26,26)
    // Tick 2: unit moves toward (26,26) — measurably closer
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 1);
    game.setEnemyShieldsForTesting(tag, 0);
    game.setInitialAttackSizeForTesting(1);

    game.tick(); // retreat fires; unit first moved toward nexus, now has staging as target
    Point2d afterTick1 = game.snapshot().enemyUnits().stream()
        .filter(u -> u.tag().equals(tag)).findFirst().orElseThrow().position();

    game.tick(); // unit moves toward staging
    Point2d afterTick2 = game.snapshot().enemyUnits().stream()
        .filter(u -> u.tag().equals(tag)).findFirst().orElseThrow().position();

    double distBefore = EmulatedGame.distance(afterTick1, new Point2d(26, 26));
    double distAfter  = EmulatedGame.distance(afterTick2, new Point2d(26, 26));
    assertThat(distAfter).isLessThan(distBefore);
}

@Test
void retreatingUnitTransfersToStagingOnArrival() {
    // Unit placed exactly at STAGING_POS — tick 1: retreat fires, unit moves toward nexus
    // (ends up ~0.5 tiles from staging). Tick 2: unit snaps back to staging, transfer fires.
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(26, 26)); // at staging
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 1);
    game.setEnemyShieldsForTesting(tag, 0);
    game.setInitialAttackSizeForTesting(1);

    game.tick(); // retreat fires; unit ends up ~0.5 tiles from staging — not yet transferred
    assertThat(game.retreatingUnitTags()).contains(tag);

    game.tick(); // unit snaps to staging (dist <= speed); transfer fires
    assertThat(game.snapshot().enemyUnits().stream()
        .anyMatch(u -> u.tag().equals(tag))).isFalse();
    assertThat(game.snapshot().enemyStagingArea()).hasSize(1);
    assertThat(game.snapshot().enemyStagingArea().get(0).tag()).isEqualTo(tag);
}

@Test
void retreatedUnitKeepsDamagedHp() {
    // Unit retreats and arrives at staging with 40 HP — no healing
    // Zealot: 40+0 / 150 = 26.7% < 30% → retreats
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(26, 26)); // at staging
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 40);
    game.setEnemyShieldsForTesting(tag, 0);
    game.setInitialAttackSizeForTesting(1);

    game.tick(); // retreat fires; moved slightly toward nexus; not yet at staging
    game.tick(); // snaps back to staging; transfer fires

    assertThat(game.snapshot().enemyStagingArea()).hasSize(1);
    assertThat(game.snapshot().enemyStagingArea().get(0).health()).isEqualTo(40);
}

@Test
void disabledThresholdsNeverRetreat() {
    // retreatHealthPercent=0 AND retreatArmyPercent=0 — no retreat regardless of HP
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 0, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 1);
    game.setEnemyShieldsForTesting(tag, 0);
    game.setInitialAttackSizeForTesting(1);

    game.tick();

    assertThat(game.retreatingUnitTags()).isEmpty();
}

@Test
void retreatDoesNotFireBeforeFirstAttack() {
    // initialAttackSize = 0 (no wave launched yet) — guard prevents any retreat logic
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 50);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(14, 14));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 1);
    game.setEnemyShieldsForTesting(tag, 0);
    // DO NOT call setInitialAttackSizeForTesting — leave at 0

    game.tick(); // must not crash; must not retreat

    assertThat(game.retreatingUnitTags()).isEmpty();
}

@Test
void deadUnitRemovedFromRetreatingSet() {
    // Tick 1: unit retreats (HP=1 → retreatHealthPercent=30), probe fires, HP=1-4=−3... wait
    // Probe vs Zealot: 5−1(armour)=4. HP=5-4=1, survives. Retreat fires → in retreatingUnits.
    // Tick 2: probe cooldown=1, no fire. Zealot survives.
    // Tick 3: probe cooldown=0, fires. HP=1-4<0, dies. retreatingUnits cleaned.
    EnemyAttackConfig atk = new EnemyAttackConfig(10, 9999, 30, 0);
    game.setEnemyStrategy(new EnemyStrategy(List.of(), false, 0, atk));
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.3f, 9));
    String tag = game.snapshot().enemyUnits().get(0).tag();
    game.setEnemyHealthForTesting(tag, 5);
    game.setEnemyShieldsForTesting(tag, 0); // 5/150=3.3% < 30% — retreats
    game.setInitialAttackSizeForTesting(1);
    game.applyIntent(new AttackIntent("probe-0", new Point2d(9.3f, 9)));

    game.tick(); // probe fires (4 dmg), Zealot survives at HP=1; retreat fires
    assertThat(game.retreatingUnitTags()).contains(tag);

    game.tick(); // probe cooldown=1, no fire; Zealot survives
    assertThat(game.retreatingUnitTags()).contains(tag);

    game.tick(); // probe fires (4 dmg), Zealot HP<0, dies; retreatingUnits cleaned
    assertThat(game.retreatingUnitTags()).doesNotContain(tag);
    assertThat(game.snapshot().enemyUnits().stream()
        .anyMatch(u -> u.tag().equals(tag))).isFalse();
}
```

- [ ] **Step 2: Run — confirm all 9 new tests fail**

```bash
mvn test -Dtest=EmulatedGameTest -q 2>&1 | grep -E "FAIL|ERROR|Tests run" | tail -5
```

Expected: 9 failures (existing tests still pass; `tickEnemyRetreat` not yet implemented).

- [ ] **Step 3: Add tickEnemyRetreat() to EmulatedGame**

Add this private method to `EmulatedGame` (before `resolveCombat()` or after `tickEnemyStrategy()` — order of declaration doesn't matter):

```java
private void tickEnemyRetreat() {
    if (enemyStrategy == null || initialAttackSize == 0) return;
    EnemyAttackConfig atk = enemyStrategy.attackConfig();

    // 1. Per-unit health threshold
    if (atk.retreatHealthPercent() > 0) {
        for (Unit u : enemyUnits) {
            if (retreatingUnits.contains(u.tag())) continue;
            double totalHp    = u.health() + u.shields();
            double maxTotalHp = (double) SC2Data.maxHealth(u.type()) + SC2Data.maxShields(u.type());
            if (totalHp / maxTotalHp * 100 < atk.retreatHealthPercent()) {
                retreatingUnits.add(u.tag());
                enemyTargets.put(u.tag(), STAGING_POS);
                log.debugf("[EMULATED] Unit %s retreating (%.1f%% hp)", u.tag(),
                    totalHp / maxTotalHp * 100);
            }
        }
    }

    // 2. Army-wide depletion threshold
    if (atk.retreatArmyPercent() > 0) {
        double survivingPct = (double) enemyUnits.size() / initialAttackSize * 100;
        if (survivingPct < atk.retreatArmyPercent()) {
            for (Unit u : enemyUnits) {
                if (retreatingUnits.contains(u.tag())) continue;
                retreatingUnits.add(u.tag());
                enemyTargets.put(u.tag(), STAGING_POS);
            }
            log.infof("[EMULATED] Army retreat: %.0f%% surviving (%d/%d)",
                survivingPct, enemyUnits.size(), initialAttackSize);
        }
    }

    // 3. Transfer arrived retreating units back to enemyStagingArea.
    // Threshold < 0.1 (not 0.5): stepToward snaps units to exactly STAGING_POS when they
    // are within unitSpeed (0.5) of it. After snapping, distance = 0.0. Using < 0.1 avoids
    // the floating-point ambiguity of comparing 0.5 == 0.5 at the pre-snap position.
    enemyUnits.removeIf(u -> {
        if (!retreatingUnits.contains(u.tag())) return false;
        if (distance(u.position(), STAGING_POS) >= 0.1) return false;
        retreatingUnits.remove(u.tag());
        enemyTargets.remove(u.tag());
        enemyStagingArea.add(u);  // damaged HP preserved — no healing
        log.debugf("[EMULATED] Unit %s arrived at staging (hp=%d shields=%d)",
            u.tag(), u.health(), u.shields());
        return true;
    });
}
```

- [ ] **Step 4: Wire tickEnemyRetreat() into tick()**

In `tick()`, add the call after `resolveCombat()`:

```java
public void tick() {
    gameFrame++;
    mineralAccumulator += miningProbes * SC2Data.MINERALS_PER_PROBE_PER_TICK;
    moveFriendlyUnits();
    moveEnemyUnits();
    resolveCombat();
    tickEnemyRetreat();    // ← new
    fireCompletions();
    spawnEnemyWaves();
    tickEnemyStrategy();
}
```

- [ ] **Step 5: Run EmulatedGameTest — all tests must pass**

```bash
mvn test -Dtest=EmulatedGameTest -q
```

Expected: BUILD SUCCESS. All existing tests + 9 new retreat tests + 1 structural test pass.

- [ ] **Step 6: Run full suite — no regressions**

```bash
mvn test -q
```

Expected: BUILD SUCCESS, 321+ tests, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java
git commit -m "feat(e6): tickEnemyRetreat() — per-unit + army-wide thresholds + staging transfer

Closes #<E6-3 issue number>
Closes #<E6-4 issue number>"
```

---

## Self-Check Before Calling Done

```bash
mvn test
```

Verify:
- `EnemyStrategyTest`: all tests pass including the 2 new retreat-config tests
- `EmulatedGameTest`: all tests pass including the 9 new retreat tests
- All other test classes: unchanged, still passing

Confirm all four GitHub issues are linked in commits and close the E6 milestone.
