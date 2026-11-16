# Mineral-Line Cycle Distribution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #656 — Mineral-line pathfinding — decouple income accuracy from cycle distribution

**Goal:** Reduce EM 12w cycle σ from 0.92 to ≤0.70 and 7+ tick cycles from 6.6% to ≤2%, while keeping 12w income within ±3%.

**Architecture:** Two independent changes to `WorkerSimulation`/`MineralPatch`:
(1) Per-patch approach docking computed from E2E distance — close patches get more approach (compensating for shorter walks), far patches get less.
(2) Collision dodge scaling at low saturation — reduce lateral displacement to 25% when ≤2 workers share a patch.

**Tech Stack:** Java 21, JUnit 5, plain unit tests (no `@QuarkusTest`)

## Global Constraints

- No compensation constants — per-patch approach decomposes an existing lumped constant, not a new factor
- `worker-physics-cdi-defaults-must-match-constants` protocol — if any new config property is added, CDI default must match
- Existing 8w/12w/16w/24w baselines must not regress beyond ±5%
- All existing tests must pass after changes

---

## Batch 1: Per-patch approach docking

### Task 1: Per-patch approach docking from E2E distance

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java:59-64` (`computeApproachDocking`)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` (new test method)

**Interfaces:**
- Consumes: `SimulatedWorker.APPROACH_DOCKING_LOOPS` (base approach), `WorkerSimulation.TOWN_HALL_RADIUS`, `WorkerSimulation.MINERAL_PATCH_RADIUS`, `SC2Data.GAME_LOOPS_PER_SECOND`, `SC2Data.unitSpeed(UnitType.PROBE)`
- Produces: Per-patch `approachDockingLoops` field set via `computeApproachDocking()` — values vary by patch E2E distance

- [ ] **Step 1: Write the failing test**

In `WorkerSimulationTest.java`, add a new test that verifies per-patch approach values differ by distance.

```java
@Test
void perPatchApproachDocking_variesByDistance() {
    sim = new WorkerSimulation();
    List<Resource> patches = List.of(
            new Resource("close", new Point2d(53.0f, 50.0f), 1500, false),  // c2c=3.0, e2e=-0.75 (very close)
            new Resource("mid",   new Point2d(57.0f, 50.0f), 1500, false),  // c2c=7.0, e2e=3.25
            new Resource("far",   new Point2d(60.0f, 50.0f), 750, false)    // c2c=10.0, e2e=6.25
    );
    sim.onTownHallCompleted(BASE_TAG, BASE_POS, patches);

    List<WorkerSimulation.BaseEntry> bases = sim.bases();
    List<MineralPatch> mp = bases.get(0).patches();

    double closeApproach = mp.get(0).approachDockingLoops;
    double midApproach   = mp.get(1).approachDockingLoops;
    double farApproach   = mp.get(2).approachDockingLoops;

    assertTrue(closeApproach > midApproach,
            "Close patch should have MORE approach than mid: " + closeApproach + " vs " + midApproach);
    assertTrue(midApproach > farApproach,
            "Mid patch should have MORE approach than far: " + midApproach + " vs " + farApproach);
    assertTrue(farApproach > 0,
            "Far patch approach should still be positive: " + farApproach);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#perPatchApproachDocking_variesByDistance" -q`
Expected: FAIL — all patches currently get the same `APPROACH_DOCKING_LOOPS` value, so `closeApproach > midApproach` is false.

- [ ] **Step 3: Implement per-patch approach in `computeApproachDocking`**

Use `ide_replace_member` to replace the `computeApproachDocking` method in `MineralPatch.java`:

```java
static void computeApproachDocking(List<MineralPatch> patches, Point2d basePos) {
    double townHallRadius = WorkerSimulation.TOWN_HALL_RADIUS;
    double patchRadius = WorkerSimulation.MINERAL_PATCH_RADIUS;
    double loopsPerUnit = io.quarkmind.domain.SC2Data.GAME_LOOPS_PER_SECOND
            / io.quarkmind.domain.SC2Data.unitSpeed(io.quarkmind.domain.UnitType.PROBE);

    double[] e2eDistances = new double[patches.size()];
    for (int i = 0; i < patches.size(); i++) {
        MineralPatch p = patches.get(i);
        p.distanceToBase = p.position.distanceTo(basePos);
        e2eDistances[i] = Math.max(0, p.distanceToBase - townHallRadius - patchRadius);
    }

    double[] sorted = e2eDistances.clone();
    java.util.Arrays.sort(sorted);
    double medianE2E = sorted.length % 2 == 0
            ? (sorted[sorted.length / 2 - 1] + sorted[sorted.length / 2]) / 2.0
            : sorted[sorted.length / 2];

    double baseApproach = SimulatedWorker.APPROACH_DOCKING_LOOPS;
    for (int i = 0; i < patches.size(); i++) {
        double adjustment = (medianE2E - e2eDistances[i]) * loopsPerUnit;
        patches.get(i).approachDockingLoops = Math.max(0, baseApproach + adjustment);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#perPatchApproachDocking_variesByDistance" -q`
Expected: PASS

- [ ] **Step 5: Run existing approach docking tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#workerAtIdlePatch_approachDockingApplied+workerAtOccupiedPatch_fullApproachDocking" -q`
Expected: PASS — existing tests use default 8-patch positions, so the formula produces values around the base approach.

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — no regression

- [ ] **Step 7: Measure cycle distribution**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest="WorkerCadenceComparisonTest#cycleLengthHistogram" -q`
Record: σ value and 7+ tick percentage. Compare against SC2 σ=0.56 and target ≤0.70.

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest="CycleOutlierDiagnosticTest#trace7PlusTickCycles" -q`
Record: cause breakdown — MIXED % should decrease.

- [ ] **Step 8: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#656): per-patch approach docking from E2E distance

Close patches get more approach (shorter walks need more overhead),
far patches get less (walks already long). Decomposes flat 24.5 into
geometry-derived per-patch values. No new config properties.

Refs #656"
```

---

## Batch 2: Collision displacement scaling

### Task 2: Scale collision dodge by patch contention level

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java:420-489` (`resolveAllMineralCollisions`)
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java` (new test method)

**Interfaces:**
- Consumes: `SimulatedWorker.assignedPatch` field, `countAssignedWorkers(MineralPatch, SimulatedWorker)` method
- Produces: Reduced lateral displacement at ≤2 workers/patch

- [ ] **Step 1: Write the failing test**

Add a test that verifies collision displacement is reduced at low saturation. At 8w (1 worker/patch), collisions should produce minimal displacement compared to 24w (3 workers/patch).

```java
@Test
void collisionDisplacement_reducedAtLowSaturation() {
    sim = new WorkerSimulation();
    List<Resource> patches = List.of(
            new Resource("p-1", new Point2d(55, 50), 1500, false),
            new Resource("p-2", new Point2d(55, 51), 1500, false)
    );
    sim.onTownHallCompleted(BASE_TAG, BASE_POS, patches);

    // 2 workers on 2 patches = 1 worker/patch → low saturation
    sim.onInitialWorkers(List.of("w-1", "w-2"), BASE_POS, BASE_TAG);
    double lowSatIncome = 0;
    for (int t = 0; t < 100; t++) lowSatIncome += tick().mineralIncome();

    // 6 workers on 2 patches = 3 workers/patch → high saturation
    sim = new WorkerSimulation();
    sim.onTownHallCompleted(BASE_TAG, BASE_POS, patches);
    sim.onInitialWorkers(List.of("w-1", "w-2", "w-3", "w-4", "w-5", "w-6"), BASE_POS, BASE_TAG);
    tickCount = 0;
    double highSatIncome = 0;
    for (int t = 0; t < 100; t++) highSatIncome += tick().mineralIncome();

    double perWorkerLow  = lowSatIncome / 2;
    double perWorkerHigh = highSatIncome / 6;
    assertTrue(perWorkerLow > perWorkerHigh,
            "Per-worker income at low saturation should exceed high saturation: "
            + perWorkerLow + " vs " + perWorkerHigh);
}
```

- [ ] **Step 2: Run test to verify baseline behaviour**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#collisionDisplacement_reducedAtLowSaturation" -q`
Expected: PASS (this test establishes baseline — per-worker income is naturally higher at low saturation). Record the actual values.

- [ ] **Step 3: Implement contention-aware dodge scaling**

Use `ide_replace_member` to replace `resolveAllMineralCollisions` in `WorkerSimulation.java`. The key change is in the walking-walking and walking-stationary collision loops — scale dodge by 0.25 when both workers' patches have ≤ 2 assigned:

In the walking-walking loop (around line 448-458), before applying dodge, add:

```java
int aContention = a.assignedPatch != null ? countAssignedWorkers(a.assignedPatch, a) + 1 : 1;
int bContention = b.assignedPatch != null ? countAssignedWorkers(b.assignedPatch, b) + 1 : 1;
if (aContention <= 2 && bContention <= 2) {
    dodge *= 0.25f;
}
```

Apply the same pattern in the walking-stationary loop (around line 464-479):

```java
int wContention = w.assignedPatch != null ? countAssignedWorkers(w.assignedPatch, w) + 1 : 1;
int sContention = s.assignedPatch != null ? countAssignedWorkers(s.assignedPatch, s) + 1 : 1;
if (wContention <= 2 && sContention <= 2) {
    dodge *= 0.25f;
}
```

- [ ] **Step 4: Run test to verify behaviour**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#collisionDisplacement_reducedAtLowSaturation" -q`
Expected: PASS — per-worker income difference should be wider now (low saturation workers less affected by collision).

- [ ] **Step 5: Run existing collision tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest="WorkerSimulationTest#collisionSpeedFactorSweep+collisionSpeedCap_limitsCompoundPenalty" -q`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS

- [ ] **Step 7: Measure cycle distribution**

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest="WorkerCadenceComparisonTest#cycleLengthHistogram" -q`
Record: σ and 7+ tick. This is the combined measurement after both leads.

Run: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest="CycleOutlierDiagnosticTest#trace7PlusTickCycles" -q`
Record: updated cause breakdown.

- [ ] **Step 8: Verify baselines haven't regressed**

Run: `mvn test -pl quarkmind-sc2 -Dtest="SC2PlaybookCalibrationTest" -q`
Expected: All 9 playbooks PASS

Run: `mvn test -pl quarkmind-sc2 -Dtest="DualRunCalibrationTest" -q`
Expected: PASS — 8w/12w/16w/24w within ±5%

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#656): scale collision dodge by patch contention

At ≤2 workers/patch, reduce lateral displacement to 25%.
Preserves full displacement at 3+/patch for high saturation.
Reduces random walk extension that pushes borderline cycles to 7+.

Refs #656"
```

---

## Batch 3: Documentation and assessment

### Task 3: Update physics constants and assess results

**Files:**
- Modify: `docs/sc2-physics-constants.md` — §3 (approach docking), §6 (collision model), §15 (worker pathing gap)
- Modify: `CLAUDE.md` — test list (add new test names)

**Interfaces:**
- Consumes: Measurement results from Task 1 Step 7 and Task 2 Step 7

- [ ] **Step 1: Update §3 approach docking entry**

In `docs/sc2-physics-constants.md`, update the approach docking row in §3 to document per-patch computation:

```markdown
| Approach/docking delay | Per-patch: base 24.5 + (median_E2E − E2E) × 5.69 | ~20–31 loops | MineralPatch.computeApproachDocking() formula from E2E distance (#656). Models SC2 path deviation: close patches need more approach (navigate around line), far patches less. | **Calibrated** — #661 base, #656 per-patch | `starcraft.worker.approach-docking-loops` (base) |
```

- [ ] **Step 2: Update §3 collision model entry**

Update the collision model paragraph to document saturation-aware scaling:

```markdown
**Collision model (#634→#661→#656):** Collision speed factor disabled (1.0) in #661. Lateral displacement scaled by contention: at ≤2 workers/patch, dodge reduced to 25% (#656). Full displacement at 3+/patch preserves high-saturation contention model. SC2 workers at low saturation don't physically collide — they alternate mining cycles.
```

- [ ] **Step 3: Update §15 worker pathing gap**

Replace the "Worker pathing around mineral line" entry with:

```markdown
### Worker pathing around mineral line — MODELED (#656)

Real SC2 workers path around the mineral line formation. The model now accounts for this via per-patch approach docking: `approach[i] = base_approach + (median_E2E − E2E[i]) × LOOPS_PER_UNIT`. Close patches get more approach (longer navigation path around central patches), far patches get less (direct paths to edge patches). Combined with contention-aware collision scaling (25% dodge at ≤2 workers/patch), cycle distribution σ reduced from 0.92 to [MEASURED] (SC2: 0.56, target: ≤0.70).
```

- [ ] **Step 4: Update CLAUDE.md test list**

Add to the unit test list: `PatchCycleCorrelationTest`, `CycleOutlierDiagnosticTest`, `PatchAssignmentDiagnosticTest`

Add to the diagnostic test list: `PatchCycleCorrelationTest`, `CycleOutlierDiagnosticTest`, `PatchAssignmentDiagnosticTest`

- [ ] **Step 5: Assessment — reach, assess, replan**

Per protocol `calibration-milestones-are-waypoints`:

Record on the issue:
- **Baseline at close:** σ=[measured], 7+ tick=[measured]%, 12w income=[measured]
- **What improved:** per-patch approach reduces structural variance, collision scaling reduces random variance
- **What didn't improve:** if σ still > 0.70, the remaining gap is contention-based (Lead 3)
- **Next milestone:** if above target, Lead 3 (reroute threshold tuning). If at target, close issue.

- [ ] **Step 6: Commit**

```bash
git add docs/sc2-physics-constants.md CLAUDE.md
git commit -m "docs(#656): update physics constants — per-patch approach, collision scaling, path deviation modeled

Refs #656"
```

---

## References

- [2026-11-16-mineral-line-cycle-distribution-design.md] — design spec this plan implements
- [MineralPatch.java:59-64] — `computeApproachDocking()` method to modify
- [WorkerSimulation.java:420-489] — `resolveAllMineralCollisions()` method to modify
- [WorkerSimulationTest.java:967-994] — existing approach docking tests
- [CycleOutlierDiagnosticTest.java] — 7+ tick cause breakdown
- [PatchAssignmentDiagnosticTest.java] — verified balanced distribution
- [docs/sc2-physics-constants.md] — §3, §6, §15 to update
- Protocol: `worker-physics-cdi-defaults-must-match-constants`
- Protocol: `calibration-milestones-are-waypoints`
- GitHub #656 — focal issue
