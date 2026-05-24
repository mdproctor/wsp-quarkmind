# Design: BuildIntent Sub-Tick Timing (#145)

**Date:** 2026-05-24
**Issue:** mdproctor/quarkmind#145

---

## Context

`handleBuild` uses `gameFrame + SC2Data.buildTimeInTicks(bt)` — integer-tick resolution with no sub-tick offset. This is the same class of bug that was fixed for unit training in #142. The analogous fix requires:

1. Calibrated `buildTimeInLoops` values per building type (source of truth, from replay tracker events)
2. Sub-tick formula in `handleBuild`: `gameFrame + (loopOffset + buildTimeInLoops(bt)) / LOOPS_PER_TICK`
3. `applyIntent(TimedIntent)` must pass `ti.loop()` to `handleBuild` (currently it does not)

The offset=17 boundary test for **unit training** (Probe) requested in the issue was already delivered in #142 as `probeCompletesOneLaterWithBoundaryLoopOffset`.

Building commands in the replay harness are synced from tracker events (UnitInit/UnitDone), not from `BuildIntent` GAME_EVENTS. The sub-tick fix to `handleBuild` improves physics accuracy for **agent-issued builds** in the emulated game; the replay harness is unaffected.

---

## Protocol Constraint

`sc2data-train-times-require-calibration.md` requires calibration from replay ground truth before using any timing constant. The same constraint applies to building times: calibrate from UnitInit/UnitDone tracker events before applying the fix.

---

## Changes

### 1. `SC2BuildTimeCalibrationTest` (new)

Reads all AI Arena Protoss replays (`replays/aiarena_protoss/`). For each replay:
- `ID_UNIT_INIT` for the watched player: record `(tag → BuildingType, initLoop)`
- `ID_UNIT_DONE`: match by tag, compute `T_real = doneLoop - initLoop`

Groups all T_real values by `BuildingType`. Takes the mode. Unlike training calibration, no GAME_EVENTS command matching is needed — both endpoints come from tracker events. Asserts `SC2Data.buildTimeInLoops(bt)` matches calibrated modal value for any type with ≥2 observations.

Building types that do not appear in the Protoss replay corpus (Terran, Zerg structures) are excluded from assertions; their values remain `buildTimeInTicks(bt) × LOOPS_PER_TICK` estimates until cross-race replay data is available.

**Implementation note:** `ID_UNIT_DONE` events expose tag fields via `event.get("unitTagIndex")` / `event.get("unitTagRecycle")` (same as `ReplaySimulatedGame.applyUnitDone`) and loop via `event.getLoop()`.

### 2. `SC2Data.buildTimeInLoops(BuildingType)` (new method)

Integer game loops per building type, calibrated from tracker events (source of truth). Initial values are the current `buildTimeInTicks × LOOPS_PER_TICK` estimates; updated after calibration run.

```java
public static int buildTimeInLoops(BuildingType type) {
    return switch (type) {
        case PYLON  -> /* calibrated */;
        case GATEWAY -> /* calibrated */;
        // ... all types
        default -> 40 * LOOPS_PER_TICK; // uncalibrated fallback
    };
}
```

### 3. `SC2Data.buildTimeInTicks(BuildingType)` — derived

Replace the hardcoded switch with:
```java
public static int buildTimeInTicks(BuildingType type) {
    return buildTimeInLoops(type) / LOOPS_PER_TICK;
}
```

Callers (`SimulatedGame`, `EmulatedGameTest`, etc.) continue to work unchanged.

### 4. `EmulatedGame.handleBuild` — sub-tick formula

Add `absLoop` parameter; apply the same formula as `startTraining`:

```java
private void handleBuild(BuildIntent b, PlayerState state, long absLoop) {
    // resource check unchanged
    int loopOffset = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
    long completesAt = gameFrame
        + (loopOffset + SC2Data.buildTimeInLoops(b.buildingType())) / SC2Data.LOOPS_PER_TICK;
    // pendingCompletion unchanged
}
```

### 5. `applyIntent(TimedIntent)` — pass `ti.loop()`

```java
case BuildIntent b -> () -> handleBuild(b, friendly, ti.loop());
```

### 6. `applyIntent(Intent, PlayerState)` — default absLoop

```java
case BuildIntent b -> () -> handleBuild(b, state, gameFrame * SC2Data.LOOPS_PER_TICK);
```

`loopOffset = 0` → formula reduces to `gameFrame + buildTimeInLoops(bt) / LOOPS_PER_TICK = gameFrame + buildTimeInTicks(bt)`. No behavioral change for enemy AI builds.

### 7. Building sub-tick boundary test (new, `EmulatedGameTest`)

Analogous to `probeCompletesOneLaterWithBoundaryLoopOffset`. Pick a `BuildingType` where `buildTimeInLoops(bt) % LOOPS_PER_TICK != 0` (calibration determines which). Test:

- `applyIntent(new TimedIntent(0L, new BuildIntent(...)))` → completes in `buildTimeInTicks(bt)` ticks
- `applyIntent(new TimedIntent(boundaryLoop, new BuildIntent(...)))` → completes in `buildTimeInTicks(bt) + 1` ticks

If all building times happen to be exact multiples of `LOOPS_PER_TICK`, the offset=0 test still validates the formula produces the correct integer tick count.

---

## Implementation Order

1. Write `SC2BuildTimeCalibrationTest` (test body + expected range map, initial assertions use `buildTimeInTicks × LOOPS_PER_TICK` as expected)
2. Run calibration (`mvn test -Dtest=SC2BuildTimeCalibrationTest -q`) — prints modal T_real per building type
3. Add `buildTimeInLoops` to SC2Data with calibrated values; change `buildTimeInTicks` to derive
4. Re-run calibration — all assertions must pass
5. Fix `handleBuild` + update both `applyIntent` callers
6. Add boundary test in `EmulatedGameTest` (offset derived from calibrated values)
7. Run full test suite (`mvn test`)

---

## Acceptance Criteria

- `SC2BuildTimeCalibrationTest` passes: calibrated T_real per building type matches `SC2Data.buildTimeInLoops`
- `SC2Data.buildTimeInTicks` derives from `buildTimeInLoops / LOOPS_PER_TICK`
- All existing `buildTimeInTicks` callers pass without modification
- Building sub-tick boundary test passes in `EmulatedGameTest`
- `applyIntent(TimedIntent)` passes `ti.loop()` to `handleBuild`
- `applyIntent(Intent, PlayerState)` preserves existing integer-tick behavior (loopOffset=0)
- Full test suite passes (`mvn test`)
- `ReplayValidationTest` tolerance unchanged (building sync path in harness is unaffected)

---

## Out of Scope

- Extracting `BuildIntent` from GAME_EVENTS in replay data (buildings synced from tracker events; this remains correct)
- Terran/Zerg building calibration (no cross-race replay data in current corpus)
