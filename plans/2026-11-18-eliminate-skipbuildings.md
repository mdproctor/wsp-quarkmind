# Eliminate skipBuildings Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #716 — EM building placement — eliminate skipBuildings
**Issue group:** #716, #713 (superseded)

**Goal:** Route all calibration builds through `EmulatedGame.handleBuild()` via `applyIntent(BuildIntent)`, eliminating the `skipBuildings` bypass and its `debugCreateBuilding()` instant-creation hack.

**Architecture:** The `calibrate()` method in `DualRunCalibrationTest` reimplements its own playbook execution loop with a `skipBuildings` branch that calls `debugCreateBuilding()` for instant building creation. The fix replaces that branch with `game.applyIntent(BuildIntent.of(...))`, which routes through the existing `handleBuild()` — resource deduction, construction time via `PendingCompletion`, supply on completion.

**Tech Stack:** Java, JUnit 5, EmulatedGame physics

## Global Constraints

- No compensation constants — if the EM doesn't match SC2, fix the root cause
- Always `double` arithmetic in the income path
- Building positions don't need SC2 map accuracy — economy calibration depends on construction TIME, not position
- `BuildIntent` import: `io.quarkmind.sc2.intent.BuildIntent`

---

## Batch 1: Eliminate skipBuildings from calibrate() and compare()

### Task 1: Refactor calibrate() — replace skipBuildings branch with applyIntent

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/DualRunCalibrationTest.java`

**Interfaces:**
- Consumes: `EmulatedGame.applyIntent(Intent)`, `BuildIntent.of(String, BuildingType, Point2d)`
- Produces: Simplified `calibrate()` with two overloads: `calibrate(playbook, baseline)` and `calibrate(playbook, baseline, unitTolerance)`

- [ ] **Step 1: Add BuildIntent import**

Add to the imports section (after line 6):
```java
import io.quarkmind.sc2.intent.BuildIntent;
```

- [ ] **Step 2: Replace the 5-overload calibrate chain with 3 overloads**

The current chain is:
```
calibrate(playbook, baseline)                                           → delegates
calibrate(playbook, baseline, skipBuildings)                            → delegates
calibrate(playbook, baseline, skipBuildings, unitTolerance)             → delegates
calibrate(playbook, baseline, skipBuildings, unitTolerance, deductCosts) → implementation
```

Replace with:
```
calibrate(playbook, baseline)                    → delegates with unitTolerance=0
calibrate(playbook, baseline, unitTolerance)     → implementation
```

The new `calibrate(String, CalibrationBaseline, int)` replaces the 5-param implementation. Remove `skipBuildings` and `deductBuildingCosts` parameters entirely.

- [ ] **Step 3: Replace the skipBuildings branch in the main loop**

Replace lines 836-855 (the `if (skipBuildings && "build".equals(step.action()))` branch) with unified handling. The entire `if (canFire)` block becomes:

```java
if (canFire) {
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
    } else {
        Map<String, Object> data = new HashMap<>(step.params());
        data.put("action", step.action());
        var outcome = handler.execute(step.action() + "-" + tick, data, null);
        if (step.supplyTrigger() > 0) {
            if (outcome.success()) {fired[i] = true;}
        } else {
            fired[i] = true;
        }
    }
}
```

Key differences from the old skipBuildings path:
- Uses `applyIntent(BuildIntent.of(...))` instead of `debugCreateBuilding()`
- No `deductBuildingCosts` check — `handleBuild` deducts resources automatically
- No manual `debugDeductResources()` call
- Gas building position resolution is kept (identical logic)
- Other buildings get position `(14, 14)` — a reasonable near-base location

- [ ] **Step 4: Simplify compare() — remove skipBuildings parameter**

Replace the two `compare()` overloads (lines 871-925) with a single method:

```java
private static CalibrationResult compare(CalibrationSnapshot expected, CalibrationSnapshot actual,
                                         int unitTolerance, int buildingTolerance) {
    for (Map.Entry<String, Integer> entry : expected.units().entrySet()) {
        int actualCount = actual.units().getOrDefault(entry.getKey(), 0);
        if (Math.abs(actualCount - entry.getValue()) > unitTolerance) {
            return CalibrationResult.diverged(expected.tick(),
                "units." + entry.getKey(),
                String.valueOf(entry.getValue()),
                String.valueOf(actualCount));
        }
    }
    for (Map.Entry<String, Integer> entry : actual.units().entrySet()) {
        if (!expected.units().containsKey(entry.getKey()) && entry.getValue() > unitTolerance) {
            return CalibrationResult.diverged(expected.tick(),
                "units." + entry.getKey(),
                "0",
                String.valueOf(entry.getValue()));
        }
    }

    for (Map.Entry<String, Integer> entry : expected.buildings().entrySet()) {
        int actualCount = actual.buildings().getOrDefault(entry.getKey(), 0);
        if (Math.abs(actualCount - entry.getValue()) > buildingTolerance) {
            return CalibrationResult.diverged(expected.tick(),
                "buildings." + entry.getKey(),
                String.valueOf(entry.getValue()),
                String.valueOf(actualCount));
        }
    }
    for (Map.Entry<String, Integer> entry : actual.buildings().entrySet()) {
        if (!expected.buildings().containsKey(entry.getKey()) && entry.getValue() > buildingTolerance) {
            return CalibrationResult.diverged(expected.tick(),
                "buildings." + entry.getKey(),
                "0",
                String.valueOf(entry.getValue()));
        }
    }

    if (!expected.upgrades().equals(actual.upgrades())) {
        return CalibrationResult.diverged(expected.tick(),
            "upgrades",
            expected.upgrades().toString(),
            actual.upgrades().toString());
    }

    return null;
}
```

Buildings are ALWAYS compared now. The `buildingTolerance` parameter allows tolerance for timing differences (buildings completing 1-2 ticks later due to construction time).

- [ ] **Step 5: Update the compare() call in calibrate()**

The call site (line 795) changes from:
```java
CalibrationResult divergence = compare(expected, actual, skipBuildings, unitTolerance);
```
to:
```java
CalibrationResult divergence = compare(expected, actual, unitTolerance, unitTolerance);
```

Using `unitTolerance` for both unit and building tolerance. This is conservative — if building timing causes divergences, the tolerance absorbs them.

- [ ] **Step 6: Update all callers**

Replace every `calibrate(...)` call that passes `skipBuildings`:

| Line | Old call | New call |
|------|----------|----------|
| 218 | `calibrate(..., sc2Baseline, true, 1)` | `calibrate(..., sc2Baseline, 1)` |
| 230 | `calibrate(..., sc2Baseline, true, 1)` | `calibrate(..., sc2Baseline, 1)` |
| 242 | `calibrate(..., sc2Baseline, true, 3)` | `calibrate(..., sc2Baseline, 3)` |
| 267 | `calibrate(..., sc2, true, 3)` | `calibrate(..., sc2, 3)` |
| 323 | `calibrate(..., sc2_6w, true, 3)` | `calibrate(..., sc2_6w, 3)` |
| 366 | `calibrate(..., sc2_3w, true, 1)` | `calibrate(..., sc2_3w, 1)` |
| 371 | `calibrate(..., sc2_6w, true, 3)` | `calibrate(..., sc2_6w, 3)` |
| 412 | `calibrate(..., sc2_3w, true, 1)` | `calibrate(..., sc2_3w, 1)` |
| 416 | `calibrate(..., sc2_6w, true, 3)` | `calibrate(..., sc2_6w, 3)` |
| 454 | `calibrate(..., sc2Baseline, true, 1)` | `calibrate(..., sc2Baseline, 1)` |
| 473 | `calibrate(..., sc2Baseline, true, 1)` | `calibrate(..., sc2Baseline, 1)` |
| 487 | `calibrate(..., sc2Baseline, true, 50, true)` | `calibrate(..., sc2Baseline, 5)` |

The expansion-transfer tolerance drops from 50 to 5 — construction delay eliminates the probe over-production. If tests diverge, adjust the tolerance based on actual results.

- [ ] **Step 7: Run all dual-run calibration tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: All tests pass. If any fail on building divergence, adjust `buildingTolerance` in the `compare()` call or the `unitTolerance` passed by the failing test.

- [ ] **Step 8: Run the full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/DualRunCalibrationTest.java
git commit -m "feat(#716): eliminate skipBuildings — route calibration builds through applyIntent

Buildings now go through EmulatedGame.handleBuild() with real construction
time via PendingCompletion. Supply is granted on completion, not instantly.
Removes skipBuildings and deductBuildingCosts parameters.

Refs #716
Closes #713

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Update ExpansionTransferDiagnosticTest

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/ExpansionTransferDiagnosticTest.java`

**Interfaces:**
- Consumes: `BuildIntent.of(String, BuildingType, Point2d)`, `EmulatedGame.applyIntent(Intent)`

- [ ] **Step 1: Read the diagnostic test to identify all debugCreateBuilding calls**

Read `ExpansionTransferDiagnosticTest.java` lines 55-65 and 185-200 to confirm the two call sites.

- [ ] **Step 2: Replace debugCreateBuilding with applyIntent in the isolation test**

Line 60: `game.debugCreateBuilding(BuildingType.NEXUS, EXP_POS);`
→ `game.applyIntent(BuildIntent.of("diag-expansion", BuildingType.NEXUS, EXP_POS));`

Add import: `import io.quarkmind.sc2.intent.BuildIntent;`

- [ ] **Step 3: Replace debugCreateBuilding in the playbook loop**

Line 196: `game.debugCreateBuilding(bt, pos);`
→ `game.applyIntent(BuildIntent.of("diag-playbook", bt, pos));`

- [ ] **Step 4: Run the diagnostic test**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ExpansionTransferDiagnosticTest -Pdiagnostic -q`
Expected: Test passes (this is a diagnostic — it prints but doesn't assert). Check the output: the expansion Nexus should now appear after construction time (NEXUS = 1600 loops = ~73 ticks), not instantly.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/ExpansionTransferDiagnosticTest.java
git commit -m "feat(#716): replace debugCreateBuilding with applyIntent in expansion diagnostic

Refs #716

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-18-eliminate-skipbuildings-design.md] — design spec
- EmulatedGame.java:763 — handleBuild with PendingCompletion
- EmulatedGame.java:635 — debugCreateBuilding (bypass being eliminated)
- SC2DeliveryHandler.java:82-89 — doBuild routing through applyIntent (reference implementation)
- DualRunCalibrationTest.java:836-855 — skipBuildings branch being removed
- DualRunCalibrationTest.java:896 — skipBuildings guard on building comparison
- SC2Data.java:276 — buildTimeInLoops for all building types
- CalibrationBaseline.java — geyserPositions and basePosition fields
- GitHub #716, #713 (superseded), #710
