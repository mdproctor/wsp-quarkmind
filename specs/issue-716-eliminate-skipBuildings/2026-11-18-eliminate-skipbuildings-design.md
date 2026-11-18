# Eliminate skipBuildings from calibration tests

**Issue:** #716
**Date:** 2026-11-18

## Problem

`DualRunCalibrationTest.calibrate()` has a `skipBuildings` flag that bypasses the EmulatedGame building lifecycle. When `skipBuildings=true`, buildings are created instantly via `debugCreateBuilding()` — no resource deduction, no construction time, immediate supply. This causes:

- Supply arrives immediately, letting the EM train workers earlier than SC2
- Economy deltas skewed 2-20% depending on playbook (#710)
- The expansion-transfer test requires `unitTolerance=50` to absorb the 30+ extra Probes
- #713 proposed layering a fake delay — a workaround on a workaround

## Solution

Replace the `skipBuildings` branch with `game.applyIntent(BuildIntent.of(...))`, routing through the existing `EmulatedGame.handleBuild()` which already implements the full building lifecycle:

1. Resource deduction (minerals + vespene)
2. Building created as incomplete
3. Construction time via `PendingCompletion` with `SC2Data.buildTimeInLoops()`
4. Supply granted only on completion
5. Town hall registration and gas building worker assignment on completion

## Changes

### DualRunCalibrationTest.calibrate()

**Remove parameters:** `skipBuildings`, `deductBuildingCosts` — the real physics handles both.

**Replace the skipBuildings branch** (lines 836-855) with:
```java
if ("build".equals(step.action())) {
    BuildingType bt = BuildingType.valueOf((String) step.params().get("build"));
    Point2d pos;
    if (SC2Data.isGasBuilding(bt)) {
        var geysers = game.snapshot().geysers();
        pos = (geysers != null && gasIdx < geysers.size())
              ? geysers.get(gasIdx++).position()
              : new Point2d(16, 12);
    } else {
        pos = new Point2d(14, 14);
    }
    game.applyIntent(BuildIntent.of("cal-playbook", bt, pos));
    fired[i] = true;
}
```

This merges with the existing `else` branch — both now go through `applyIntent` or `handler.execute()`.

### DualRunCalibrationTest.compare()

**Remove the `skipBuildings` guard** on building comparison (line 896). Buildings should always be compared now that the EM creates them with real timing.

### Callers

Update all `calibrate(...)` calls that pass `skipBuildings=true`:
- `calibrate(..., true, 1)` → `calibrate(..., 1)` (tolerance only)
- `calibrate(..., true, 3)` → `calibrate(..., 3)`
- `calibrate(..., true, 50, true)` → `calibrate(..., N)` with reduced tolerance

The expansion-transfer `unitTolerance=50` should drop substantially — the probe over-production was caused by instant supply from instant building completion.

### ExpansionTransferDiagnosticTest

Also uses `debugCreateBuilding()` for the same reason. Update to use `applyIntent(BuildIntent.of(...))`.

## What doesn't change

- `EmulatedGame.handleBuild()` — already correct
- `SC2DeliveryHandler` — already routes through `applyIntent`
- `SC2PlaybookCalibrationTest` — already uses the proper path
- `CalibrationBaseline` — no schema changes
- `debugCreateBuilding()` — kept for other test infrastructure uses

## Acceptance criteria

- [ ] `skipBuildings` and `deductBuildingCosts` parameters removed from `calibrate()`
- [ ] All build steps route through `game.applyIntent(BuildIntent.of(...))`
- [ ] Building comparison always runs in `compare()` (no skip guard)
- [ ] All dual-run calibration tests pass
- [ ] Expansion-transfer tolerance reduced from 50

## References

- EmulatedGame.java:763 — handleBuild with PendingCompletion
- SC2DeliveryHandler.java:82-89 — doBuild routing through applyIntent
- DualRunCalibrationTest.java:836-855 — skipBuildings branch to remove
- CalibrationBaseline.java — geyserPositions and basePosition fields
- #710 — skipBuildings cost deduction investigation
- #713 — superseded construction delay workaround
