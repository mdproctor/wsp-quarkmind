# Auto-Engage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the `attackingUnits` combat gate from `resolveCombat` and the stop-to-fight block from `moveEnemyUnits` so all units fire at enemies in weapon range each tick without requiring an explicit `AttackIntent`.

**Architecture:** Two one-line deletions in `EmulatedGame`: remove the `attackingUnits.contains()` guard from both the friendly and enemy loops in `resolveCombat`, and remove the range-stop block from `moveEnemyUnits`. No new state. `attackingUnits` is retained as dead state (cleanup tracked in #134). TDD: write each failing test before the implementation that makes it pass.

**Tech Stack:** Plain Java, JUnit 5, AssertJ, `EmulatedGame` package-private test access.

---

## File Map

| File | Change |
|------|--------|
| `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` | Remove `attackingUnits` gate (line 360 friendly, line 372 enemy); remove stop-to-fight block (lines 163–168) |
| `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` | Add `friendlyAutoEngagesWithoutAttackIntent`; add `enemyContinuesMovingWhileEngaging`; fix any regressions from stop-to-fight removal |

---

## Task 1: Write failing test — friendly auto-engage

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

Context: probe-0 starts at `(9, 9)`. `spawnEnemyForTesting` places an enemy at a given position and adds it to `enemy.attackingUnits`. The test needs a friendly unit (probe-0) to fire at the enemy *without* an `AttackIntent` — currently impossible because `resolveCombat` gates on `friendly.attackingUnits`.

- [ ] **Step 1: Add the test to the `// ---- #128: parallel training queues ----` section's neighbour — add a new section header and test**

Add after the parallel-training-queues section and before the enemy test section in `EmulatedGameTest.java`:

```java
// ---- #129: auto-engage ----

@Test
void friendlyAutoEngagesWithoutAttackIntent() {
    // probe-0 at (9,9). Zealot spawned within probe attack range (3.0 tiles).
    // No AttackIntent issued — probe must auto-engage.
    game.spawnEnemyForTesting(UnitType.ZEALOT, new Point2d(9.0f, 9.3f));
    int zealotMaxShields = SC2Data.maxShields(UnitType.ZEALOT); // 50

    game.tick();

    Unit zealot = game.enemy.units.stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    // Probe auto-engaged — Zealot shields must have dropped
    assertThat(zealot.shields()).isLessThan(zealotMaxShields);
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#friendlyAutoEngagesWithoutAttackIntent" -q 2>&1 | tail -10
```

Expected: FAIL — `zealot.shields()` is 50 (probe did not fire because it has no `AttackIntent`).

---

## Task 2: Remove `attackingUnits` gate from `resolveCombat` — friendly and enemy loops

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` (two one-line deletions)

The gate is the sole reason auto-engage doesn't work. Delete both instances.

- [ ] **Step 1: Remove the friendly gate (line 360)**

In `EmulatedGame.java`, find:
```java
        for (Unit attacker : friendly.units) {
            if (!friendly.attackingUnits.contains(attacker.tag())) continue;
            if (friendly.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
```

Replace with:
```java
        for (Unit attacker : friendly.units) {
            if (friendly.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
```

- [ ] **Step 2: Remove the enemy gate (line 372)**

In `EmulatedGame.java`, find:
```java
        for (Unit attacker : enemy.units) {
            if (!enemy.attackingUnits.contains(attacker.tag())) continue;
            if (enemy.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
```

Replace with:
```java
        for (Unit attacker : enemy.units) {
            if (enemy.unitCooldowns.getOrDefault(attacker.tag(), 0) > 0) continue;
```

- [ ] **Step 3: Build**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run the Task 1 test — must now pass**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#friendlyAutoEngagesWithoutAttackIntent" -q 2>&1 | tail -5
```
Expected: no output (PASS).

- [ ] **Step 5: Run the existing training and combat tests to catch early regressions**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#trainIntentDeductsMinerals+trainedUnitAppearsAfterBuildTime+shieldsAbsorbDamageBeforeHp+parallelTrainingTwoGateways+supplyReservedAtQueueTime" -q 2>&1 | tail -10
```
Expected: all pass.

---

## Task 3: Write failing test — enemy continues moving while engaging

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

Context: `moveEnemyUnits` currently stops a non-retreating enemy unit if any friendly is within its attack range (`nearestInRange` returns present). After stop-to-fight removal, the enemy advances toward its target while `resolveCombat` handles firing independently.

The test places a Zealot (melee range 0.5 tiles) at `(9.0, 9.4)` — probe-0 at `(9, 9)` is 0.4 tiles away, inside melee range. With stop-to-fight active, the Zealot's position is unchanged after one tick. After removal, it moves toward `NEXUS_POS (8, 8)`.

- [ ] **Step 1: Add the test inside the `// ---- #129: auto-engage ----` section**

```java
@Test
void enemyContinuesMovingWhileEngaging() {
    // Zealot at (9.0, 9.4) — probe-0 at (9,9) is 0.4 tiles away, inside melee range (0.5).
    // Currently: stop-to-fight holds Zealot in place. After fix: Zealot advances toward (8,8).
    Point2d initialPos = new Point2d(9.0f, 9.4f);
    game.spawnEnemyForTesting(UnitType.ZEALOT, initialPos);

    game.tick();

    Unit zealot = game.enemy.units.stream()
        .filter(u -> u.type() == UnitType.ZEALOT)
        .findFirst().orElseThrow();
    assertThat(zealot.position()).isNotEqualTo(initialPos);
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#enemyContinuesMovingWhileEngaging" -q 2>&1 | tail -10
```
Expected: FAIL — `zealot.position()` equals `initialPos` because stop-to-fight is still active.

---

## Task 4: Remove stop-to-fight block from `moveEnemyUnits`

**Files:**
- Modify: `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`

- [ ] **Step 1: Delete the stop-to-fight block (lines 163–168)**

In `EmulatedGame.java`, inside `moveEnemyUnits()`, find:
```java
            // Non-retreating units stop to fight when a friendly is within attack range.
            // Retreating units always move — they are disengaging, not attacking.
            if (!retreating.contains(u.tag()) &&
                    nearestInRange(u.position(), friendly.units, SC2Data.attackRange(u.type())).isPresent()) {
                return u; // stay and fight — resolveCombat() handles the attack this tick
            }
            Point2d target = enemy.unitTargets.getOrDefault(u.tag(), EnemyBehavior.NEXUS_POS);
```

Replace with:
```java
            Point2d target = enemy.unitTargets.getOrDefault(u.tag(), EnemyBehavior.NEXUS_POS);
```

- [ ] **Step 2: Build**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn compile -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run the Task 3 test — must now pass**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#enemyContinuesMovingWhileEngaging" -q 2>&1 | tail -5
```
Expected: PASS.

- [ ] **Step 4: Run both new tests together**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -Dtest="EmulatedGameTest#friendlyAutoEngagesWithoutAttackIntent+enemyContinuesMovingWhileEngaging" -q 2>&1 | tail -5
```
Expected: both PASS.

---

## Task 5: Run full suite and fix any regressions

**Files:**
- Modify: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java` (if needed)

Stop-to-fight removal changes enemy unit positions during multi-tick combat tests. If any test asserts a specific unit position after N ticks, or relied on enemies staying in place within melee range, it may fail. The fix in each case is to verify the enemy is still in range (it will be after one movement step, since the Zealot moves 0.5 tiles per tick toward a target that is often nearby) or adjust the assertion to be position-agnostic (check HP/shields rather than position).

- [ ] **Step 1: Run the complete test suite**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -q 2>&1 | tail -20
```

- [ ] **Step 2: If there are failures, investigate each one**

For each failing test:
1. Read the failure message — does it mention an unexpected position, unexpected damage, or unexpected unit count?
2. If the test placed an enemy unit and expected it to stay stationary: the enemy now moves each tick. Check whether it is still within attack range of the target after movement. If yes, the test should still pass — re-read the assertion carefully.
3. If the test asserts a specific enemy position: replace the position assertion with a distance-or-HP-based assertion that is independent of stop-to-fight.
4. Run the fixed test in isolation to confirm.

- [ ] **Step 3: Re-run the full suite to confirm 0 failures**

```bash
cd /Users/mdproctor/claude/quarkmind && mvn test -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`, exit code 0.

---

## Task 6: Commit

**Files:**
- `src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java`
- `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java`

- [ ] **Step 1: Stage and commit — all from `/Users/mdproctor/claude/quarkmind`**

```bash
cd /Users/mdproctor/claude/quarkmind && \
git add src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java \
        src/test/java/io/quarkmind/sc2/emulated/EmulatedGameTest.java && \
git commit -m "feat: auto-engage — all units fire at enemies in range without AttackIntent

Remove attackingUnits gate from resolveCombat (both friendly and enemy
loops). Remove stop-to-fight range check from moveEnemyUnits. Any unit
with an enemy in weapon range now fires every cooldown cycle. Units
advance toward their target while also engaging — fire-while-moving.

Closes #129"
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|----------------|------|
| Remove `attackingUnits` gate from `resolveCombat` (both loops) | Task 2 |
| Remove stop-to-fight from `moveEnemyUnits` | Task 4 |
| Symmetric for friendly and enemy | Task 2 (both loops in one step) |
| New test: `friendlyAutoEngagesWithoutAttackIntent` | Task 1 |
| Full suite passes | Task 5 |
| `attackingUnits` retained as dead state (cleanup in #134) | No code change needed |

All spec requirements covered. ✅

**Placeholder scan:** No TBDs, no vague steps, all code blocks complete. ✅

**Type consistency:** `Unit`, `Point2d`, `UnitType`, `SC2Data` — consistent throughout. ✅

**Tick-order note for implementer:** In each tick, `moveEnemyUnits()` runs before `resolveCombat()`. So after stop-to-fight removal, an enemy unit at position P moves to P' *first*, and then `resolveCombat` checks whether any enemy in P' (new position) has a friendly in range. This is the correct sequence and matches how the existing combat tests work.
