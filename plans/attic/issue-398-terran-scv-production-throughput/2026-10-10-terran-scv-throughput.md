# Terran SCV Production Throughput Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #398 — Terran SCV count below real SC2 target — production throughput ceiling with 2 CCs

**Goal:** Raise Terran economy playbook SCV count from ~40 to 48-50 at tick 305 by fixing a morph-queue fidelity bug and rewriting the playbook build order.

**Architecture:** Two independent changes: (1) a one-line guard in `drainBuildingQueues()` to pause queued training on incomplete/morphing buildings (EmulatedGame fidelity fix), and (2) a playbook YAML rewrite with proper supply depot pre-building, earlier CC-2, and dual OC morph based on published Terran macro build orders.

**Tech Stack:** Java 21, Quarkus, EmulatedGame physics, SC2 playbook YAML

## Global Constraints

- Train times, build times, and morph times are calibrated constants in SC2Data — do not change them.
- EmulatedGame is the living spec for SC2 behavior — fidelity over convenience.
- Playbook build order should reference published Terran macro builds, not first-principles derivation.
- No military units in the economy-only playbook.
- Protocol `sc2data-train-times-require-calibration` applies if any timing constants are touched (they should not be).

## Known limitation: multi-OC MULE calldowns

`resolveBuilding("r-orbital_command")` in EmulatedGame always returns the first complete OC. With 2 OCs, MULE calldowns always target OC-1 — OC-2's energy builds up unused. The 2nd OC morph in the playbook provides an additional SCV production building but NOT double MULE income. Fixing multi-OC calldown resolution is out of scope — file a follow-up issue if MULE income becomes the bottleneck after the playbook rewrite.

---

## Batch 1: EmulatedGame fidelity — queue-pause during morph

### Task 1: Fix drainBuildingQueues + morph-during-train test coverage

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java:469-484`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java`

**Interfaces:**
- Consumes: `EmulatedGame.applyIntent(MorphIntent)`, `EmulatedGame.applyIntent(TrainIntent)`, `EmulatedGame.tick()`, `EmulatedGame.snapshot()`
- Produces: Corrected `drainBuildingQueues()` behavior — queued units pause on incomplete buildings

- [ ] **Step 1: Write failing test — queued SCV starts training during OC morph (should not)**

Add to `TerranEmulatedGameTest`:

```java
@Test
void morphToOC_queuedSCVs_pauseDuringMorph() {
    // Start training + queue a second SCV on the CC
    game.setMineralsForTesting(500);
    final Building cc = game.snapshot().myBuildings().stream()
        .filter(b -> b.type() == BuildingType.COMMAND_CENTER)
        .findFirst().orElseThrow();

    game.applyIntent(new TrainIntent(cc.tag(), UnitType.SCV));  // in-progress
    game.applyIntent(new TrainIntent(cc.tag(), UnitType.SCV));  // queued

    // Morph CC → OC mid-training
    game.applyIntent(new MorphIntent(cc.tag(), "COMMAND_CENTER", "OrbitalCommand"));

    // Advance past the first SCV's completion (12 ticks) but before morph completes (25 ticks)
    final int scvTicks = SC2Data.trainTimeInTicks(UnitType.SCV);
    for (int i = 0; i < scvTicks + 1; i++) game.tick();

    // First SCV should complete (in-progress training fires on tick count)
    final long scvCount = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.SCV).count();
    assertThat(scvCount).isEqualTo(13); // 12 initial + 1 completed

    // The QUEUED SCV should NOT have started training yet — building is mid-morph
    // If it started, we'd see it completing ~12 ticks later (tick ~25).
    // Advance to tick 25 (morph not yet complete at this point due to rounding):
    for (int i = 0; i < 12; i++) game.tick();
    final long scvCountAfter = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.SCV).count();
    // Should still be 13 — queued SCV should NOT have trained during morph
    assertThat(scvCountAfter).isEqualTo(13);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerranEmulatedGameTest#morphToOC_queuedSCVs_pauseDuringMorph -q`

Expected: FAIL — queued SCV starts training during morph because `drainBuildingQueues()` doesn't check `isComplete`.

- [ ] **Step 3: Fix drainBuildingQueues — add isComplete guard**

In `EmulatedGame.java`, modify `drainBuildingQueues()` (line 469-484). After the `buildingTrainingUntil` check and before popping from the queue, verify the building is complete:

```java
private void drainBuildingQueues(PlayerState state, PhysicsState physics) {
    for (String buildingTag : new ArrayList<>(physics.buildingQueues.keySet())) {
        if (physics.buildingTrainingUntil.containsKey(buildingTag)) continue;
        boolean buildingReady = state.buildings().stream()
            .anyMatch(b -> b.tag().equals(buildingTag) && b.isComplete());
        if (!buildingReady) continue;
        Deque<UnitType> queue = physics.buildingQueues.get(buildingTag);
        if (queue == null || queue.isEmpty()) {
            physics.buildingQueues.remove(buildingTag);
            physics.buildingCompletionAtLoop.remove(buildingTag);
            continue;
        }
        UnitType next = queue.poll();
        if (queue.isEmpty()) physics.buildingQueues.remove(buildingTag);
        long completionLoop = physics.buildingCompletionAtLoop.getOrDefault(buildingTag, 0L);
        physics.buildingCompletionAtLoop.remove(buildingTag);
        startTraining(buildingTag, next, state, physics, completionLoop);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerranEmulatedGameTest#morphToOC_queuedSCVs_pauseDuringMorph -q`

Expected: PASS

- [ ] **Step 5: Write test — in-progress SCV completes during OC morph**

This tests existing correct behavior (regression protection):

```java
@Test
void morphToOC_inProgressSCV_completesNormally() {
    game.setMineralsForTesting(200);
    final Building cc = game.snapshot().myBuildings().stream()
        .filter(b -> b.type() == BuildingType.COMMAND_CENTER)
        .findFirst().orElseThrow();

    game.applyIntent(new TrainIntent(cc.tag(), UnitType.SCV));
    game.applyIntent(new MorphIntent(cc.tag(), "COMMAND_CENTER", "OrbitalCommand"));

    // Advance past SCV completion
    final int scvTicks = SC2Data.trainTimeInTicks(UnitType.SCV);
    for (int i = 0; i < scvTicks + 1; i++) game.tick();

    final long scvCount = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.SCV).count();
    assertThat(scvCount).isEqualTo(13); // 12 initial + 1 trained during morph
}
```

- [ ] **Step 6: Write test — queued SCV resumes after morph completes**

```java
@Test
void morphToOC_queuedSCV_resumesAfterMorphComplete() {
    game.setMineralsForTesting(500);
    final Building cc = game.snapshot().myBuildings().stream()
        .filter(b -> b.type() == BuildingType.COMMAND_CENTER)
        .findFirst().orElseThrow();

    game.applyIntent(new TrainIntent(cc.tag(), UnitType.SCV));  // in-progress
    game.applyIntent(new TrainIntent(cc.tag(), UnitType.SCV));  // queued

    game.applyIntent(new MorphIntent(cc.tag(), "COMMAND_CENTER", "OrbitalCommand"));

    // Advance past morph completion (25 ticks) + SCV train time (12 ticks) + margin
    final int morphTicks = SC2Data.buildTimeInLoops(BuildingType.ORBITAL_COMMAND) / SC2Data.LOOPS_PER_TICK;
    final int scvTicks = SC2Data.trainTimeInTicks(UnitType.SCV);
    for (int i = 0; i < morphTicks + scvTicks + 2; i++) game.tick();

    // Both SCVs should have completed: first during morph, second after morph
    final long scvCount = game.snapshot().myUnits().stream()
        .filter(u -> u.type() == UnitType.SCV).count();
    assertThat(scvCount).isEqualTo(14); // 12 initial + 2 trained
}
```

- [ ] **Step 7: Run all morph tests + full test suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=TerranEmulatedGameTest -q`
Expected: PASS

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS (no regressions from the `drainBuildingQueues` change)

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/TerranEmulatedGameTest.java
git commit -m "fix: drainBuildingQueues pauses queue on incomplete buildings

In real SC2, production queues pause during building morphs (e.g.
CC→OC). drainBuildingQueues() was missing the isComplete check that
handleTrain() already had, allowing queued units to start training
on morphing buildings.

Added morph-during-train test coverage:
- In-progress SCV completes during morph (correct, regression test)
- Queued SCVs pause during morph (was broken, now fixed)
- Queued SCVs resume after morph completes

Refs #398"
```

---

## Batch 2: Playbook rewrite — Terran economy build order

### Task 2: Rewrite economy-only-terran.yaml with proper build order

**Files:**
- Modify: `quarkmind-sc2/src/test/resources/playbooks/economy-only-terran.yaml`

**Interfaces:**
- Consumes: EmulatedGame mechanics (train, build, morph, calldown), SC2DeliveryHandler actions
- Produces: Updated Terran economy playbook hitting 48+ SCVs at tick 305

**Approach:** Research published standard Terran macro build orders (e.g. Spawning Tool's "Standard Terran" openings, TerranCraft macro guides). The economy-only playbook strips military units but keeps the production building timing, depot cadence, and morph order from standard macro Terran play.

Key changes from current playbook:
- Pre-build supply depots ~2 supply before cap (e.g. first depot at 13 instead of 14)
- Earlier CC-2 (around 16-17 supply instead of 20) to reduce OC morph production blackout
- 2nd OC morph on CC-2 immediately after completion
- Adjusted depot spacing for higher SCV production rate with 2 buildings
- Updated assertions: tick 305 SCV >= 48

- [ ] **Step 1: Research standard Terran macro build order**

Search Spawning Tool and TerranCraft for "1 Rax FE" or "CC first" macro builds. Note the supply thresholds for: first depot, CC-2, OC morph, subsequent depots. Convert real-time seconds to ticks using `seconds × 22.4 / 22 ≈ seconds × 1.018` (loops per second / loops per tick).

The economy-only playbook uses supply thresholds, not tick thresholds, so the key data is the supply number at which each action fires.

- [ ] **Step 2: Write the updated playbook YAML**

Update `economy-only-terran.yaml` with the researched build order. The structure stays the same — only the supply triggers and step ordering change. Add a second `morph: OrbitalCommand` step for CC-2.

Example structure (exact values from research):

```yaml
playbook: economy-only-terran
schema: sc2
meta:
  race: TERRAN
  duration: 310 ticks

steps:
  - train: SCV
    loop: continuous

  - build: SUPPLY_DEPOT
    at: 13 supply

  - build: COMMAND_CENTER
    at: 16 supply

  - build: SUPPLY_DEPOT
    at: 18 supply

  - morph: OrbitalCommand
    source: COMMAND_CENTER
    at: 22 supply

  - calldown: MULE
    loop: continuous

  - build: SUPPLY_DEPOT
    at: 24 supply

  - morph: OrbitalCommand
    source: COMMAND_CENTER
    at: 28 supply

  - build: SUPPLY_DEPOT
    at: 30 supply

  - build: SUPPLY_DEPOT
    at: 36 supply

  - build: SUPPLY_DEPOT
    at: 42 supply

  - build: SUPPLY_DEPOT
    at: 48 supply

  - build: SUPPLY_DEPOT
    at: 54 supply

  - build: SUPPLY_DEPOT
    at: 60 supply

  - build: SUPPLY_DEPOT
    at: 66 supply

  - assert: tick-61
    at: 61 ticks
    expect:
      units:
        SCV: { min: 15 }

  - assert: tick-183
    at: 183 ticks
    expect:
      units:
        SCV: { min: 28 }

  - assert: tick-305
    at: 305 ticks
    expect:
      units:
        SCV: { min: 48 }
      minerals:
        min: 200
```

Note: exact supply values, depot spacing, and assertion thresholds will be tuned based on the research in Step 1 and the calibration run in Step 3. The above is a starting template.

- [ ] **Step 3: Run calibration test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=SC2PlaybookCalibrationTest#economyOnlyTerran -q`

If assertion fails: read the `[PLAYBOOK]` output lines showing actual SCV/mineral counts at each checkpoint. Adjust depot timing or CC-2/OC ordering as needed. Common issues:
- Supply blocks: depot triggers too late → move earlier
- Mineral shortages: CC-2 too early → delay by 1-2 supply
- SCV count below target at 305: check for production gaps (ticks with zero production buildings)

Iterate until assertions pass.

- [ ] **Step 4: Run full regression suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — other playbooks (Protoss, Zerg) unaffected

- [ ] **Step 5: Run replay divergence report**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=ReplayValidationReportTest -q`

Check that the `drainBuildingQueues` fidelity fix doesn't worsen divergence metrics. The fix should slightly improve Terran economy divergence since queued units now pause correctly during morphs.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/test/resources/playbooks/economy-only-terran.yaml
git commit -m "feat: rewrite Terran economy playbook — 48+ SCVs at tick 305

Optimized build order based on standard Terran macro play:
- Pre-build supply depots 2 supply before cap
- Earlier CC-2 at 16 supply (was 20)
- 2nd OC morph on CC-2 for additional production building
- Adjusted depot cadence for dual-OC production rate

SCV count at tick 305: 48+ (was ~40). Validates that production
throughput is no longer the bottleneck with proper build order timing.

Refs #398"
```

- [ ] **Step 7: File follow-up issue for multi-OC MULE calldown resolution**

`resolveBuilding("r-orbital_command")` always returns the first OC. With 2 OCs, MULE calldowns only use OC-1. File a follow-up issue to add energy-aware OC resolution for MULE calldowns.

---

## References

- [2026-10-10-terran-scv-throughput-design.md] — design spec this plan implements
- [EmulatedGame.java:469-484] — `drainBuildingQueues()` (fidelity fix target)
- [EmulatedGame.java:350-431] — `handleTrain()` (has isComplete check — reference)
- [EmulatedGame.java:802-833] — `handleMorph()` (sets isComplete=false)
- [EmulatedGame.java:721-729] — `resolveBuilding()` (first-match — multi-OC limitation)
- [PhysicsState.java:39-45] — `fireCompletions()` (tick-based, correct)
- [TerranEmulatedGameTest.java] — existing Terran tests (test file for new morph tests)
- [economy-only-terran.yaml] — current Terran playbook (rewrite target)
- [SC2PlaybookCalibrationTest.java] — calibration test runner
- [SC2DeliveryHandler.java:104-112] — MULE calldown handler (multi-OC limitation)
- [PP-20260522-572156] — sc2data-train-times-require-calibration protocol
- [GitHub #398] — Terran SCV count below real SC2 target
