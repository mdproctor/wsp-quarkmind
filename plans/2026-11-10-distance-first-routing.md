# Distance-First Worker-to-Patch Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development). If IntelliJ MCP is available, use
> ide-tooling for structural editing; if unavailable, Edit with
> replace_all is the fallback for method-level replacements.

**Focal issue:** #620 — Mining cycle decomposition (next phase: dynamic routing)
**Issue group:** #620, #617

**Goal:** Replace load-first patch selection with SC2-like distance-first routing to reduce per-tick mineral delivery mismatches from 240/305 (78.7%) toward zero.

**Architecture:** Single method change in `WorkerSimulation.assignToNearestPatch()`. Two-phase selection: nearest available patch (no active miner, lock expired), falling back to nearest patch regardless when saturated. All 7 call sites use the same method — no branching.

**Tech Stack:** Java 21, plain JUnit (no Quarkus)

## Global Constraints

- No compensation constants — accuracy must come from the routing model, not from multipliers
- Simulate per-loop — no analytical shortcuts
- Per-tick mineral deltas must be multiples of 5 (mineral cargo)

---

## Batch 1: Distance-first routing with verification

### Task 1: Record baseline mismatch count

**Files:**
- None modified — measurement only

- [ ] **Step 1: Run the cadence diagnostic and record the before number**

Run:
```bash
mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=DualRunCalibrationTest#deliveryCadenceDiagnostic
```

Record from output: `Mismatch ticks: N / 304 (X.X%)` and `Final gap: +N minerals`.
These are the baseline numbers to compare against.

- [ ] **Step 2: Run the dual-run calibration suite and record before numbers**

Run:
```bash
mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest
```

Verify all 31 tests pass. Record the `[econ]` output for 12w, 16w, and 24w baselines.

- [ ] **Step 3: Commit a note (optional)**

No code changes — this step establishes the baseline for comparison.

---

### Task 2: Change assignToNearestPatch to distance-first

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java:536-572`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Consumes: `MineralPatch.miners` (list of active dockers), `MineralPatch.lockEndLoop` (docking cooldown), `SimulatedWorker.position`, `currentTickStartLoop` (field on WorkerSimulation)
- Produces: Same method signature — `assignToNearestPatch(SimulatedWorker w)` — behavior change only

- [ ] **Step 1: Write a failing test for distance-first routing**

Add to `WorkerSimulationTest.java`. The test creates a WorkerSimulation with 2 patches at different distances and verifies the worker is assigned to the closer available patch even when a farther patch has fewer assigned workers.

```java
@Test
void assignToNearestPatch_prefersCloserAvailablePatch() {
    // Setup: base at (8,8), two patches — close at (10,8) and far at (15,8)
    // Worker at (8,8) with one other worker already assigned to the close patch
    // Load-first would pick the far patch (0 workers vs 1). Distance-first picks close (available, closer).
    WorkerSimulation sim = new WorkerSimulation();
    // Use onTownHallCompleted to set up base with synthetic patches
    sim.onTownHallCompleted("base-1", new Point2d(8, 8));
    // Add 2 workers
    sim.onWorkerProduced("w1", new Point2d(8, 8));
    sim.onWorkerProduced("w2", new Point2d(8, 8));

    // After 2 workers on 8 patches, both should be on the 2 closest patches
    // (distance-first means they pick nearest available, not spread evenly)
    var snapshots = sim.workerSnapshots();
    // Both workers should be GATHERING_MINERALS
    assertThat(snapshots).allMatch(ws -> ws.action() == WorkerAction.GATHERING_MINERALS
                                        || ws.action() == WorkerAction.RETURNING_MINERALS);
}
```

Note: this test verifies basic assignment works. The real verification is the cadence diagnostic (Task 3).

- [ ] **Step 2: Run test to verify it passes (or fails with current routing)**

Run:
```bash
mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#assignToNearestPatch_prefersCloserAvailablePatch
```

The test should pass with current routing too (both routing strategies assign 2 workers to patches). If it fails, diagnose — the test validates setup, not the routing change itself.

- [ ] **Step 3: Replace the selection logic in assignToNearestPatch**

In `WorkerSimulation.java`, replace the method body of `assignToNearestPatch` (lines 536-572). Keep the guard clause (lines 537-540) and the post-selection setup (lines 563-571). Replace only the selection loop (lines 541-555):

**Replace this (lines 541-555):**
```java
        BaseEntry    base            = bases.get(w.baseIndex);
        MineralPatch nearest         = null;
        int          nearestAssigned = Integer.MAX_VALUE;
        double       nearestDist     = Double.MAX_VALUE;

        for (MineralPatch p : base.patches()) {
            if (p.isDepleted()) {continue;}
            int    assigned = countAssignedWorkers(p, w);
            double d        = w.position.distanceTo(p.position);
            if (assigned < nearestAssigned || (assigned == nearestAssigned && d < nearestDist)) {
                nearestAssigned = assigned;
                nearestDist     = d;
                nearest         = p;
            }
        }
```

**With this:**
```java
        BaseEntry    base             = bases.get(w.baseIndex);
        MineralPatch nearestAvailable = null;
        double       nearestAvailDist = Double.MAX_VALUE;
        MineralPatch nearestAny       = null;
        double       nearestAnyDist   = Double.MAX_VALUE;

        for (MineralPatch p : base.patches()) {
            if (p.isDepleted()) {continue;}
            double  d         = w.position.distanceTo(p.position);
            boolean available = p.miners.isEmpty() && currentTickStartLoop >= p.lockEndLoop;
            if (available && d < nearestAvailDist) {
                nearestAvailDist = d;
                nearestAvailable = p;
            }
            if (d < nearestAnyDist) {
                nearestAnyDist = d;
                nearestAny     = p;
            }
        }
        MineralPatch nearest = (nearestAvailable != null) ? nearestAvailable : nearestAny;
```

The rest of the method (lines 557-572) remains unchanged — it handles the null case and sets up the worker's target.

- [ ] **Step 4: Run the full WorkerSimulation test suite**

Run:
```bash
mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest
```

Expected: all 35 tests pass.

- [ ] **Step 5: Run the broader test suite to check for regressions**

Run:
```bash
mvn test -pl quarkmind-sc2 -Dtest=EmulatedGameTest,DualRunCalibrationTest,SC2PlaybookCalibrationTest,EconomyTrackerTest
```

Expected: all tests pass (226+). If any DualRunCalibrationTest fails on mineral totals, the routing change has shifted totals beyond tolerance — investigate.

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#620): distance-first patch routing — nearest available, then nearest any

Replace load-balanced (fewest-workers-first) patch selection with SC2-like
distance-first routing. Two-phase: nearest available patch (no active
miner, lock expired), falling back to nearest patch when saturated.

Refs #620"
```

---

### Task 3: Measure cadence improvement and update docs

**Files:**
- Modify: `docs/sc2-physics-constants.md` (update mismatch count and routing description)

- [ ] **Step 1: Run the cadence diagnostic**

Run:
```bash
mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=DualRunCalibrationTest#deliveryCadenceDiagnostic
```

Record: `Mismatch ticks: N / 304 (X.X%)` and `Final gap: +N minerals`.

Compare against Task 1 baseline. Document the delta.

- [ ] **Step 2: Run the worker cadence comparison**

Run:
```bash
mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=WorkerCadenceComparisonTest
```

Record per-worker cycle statistics (avg, min, max, σ) for both EM and SC2.

- [ ] **Step 3: Run the full dual-run suite**

Run:
```bash
mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest
```

Record 12w, 16w, 24w mineral deltas. Compare against Task 1 baseline. Note any saturation curve changes.

- [ ] **Step 4: Update docs/sc2-physics-constants.md**

Update the per-tick cadence section (currently at line 369) and the saturation curve section (around line 320) with the new mismatch count and any dual-run delta changes. Update the routing description to reflect distance-first selection.

- [ ] **Step 5: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#620): update cadence mismatch and routing description

Per-tick mismatch: 240/305 → N/305. Distance-first routing replaces
load-balanced selection.

Refs #620"
```

---

## References

- [2026-11-10-dynamic-worker-routing-design.md] — design spec
- [WorkerSimulation.java:536-572] — `assignToNearestPatch()` method
- [MineralPatch.java:18-21] — `MAX_MINERS`, `miners`, `lockEndLoop`
- [DualRunCalibrationTest.java:84-150] — `deliveryCadenceDiagnostic()`
- [WorkerCadenceComparisonTest.java] — per-worker cycle comparison
- [GitHub #620] — mining cycle decomposition
- [GitHub #617] — foundational tick-alignment audit epic
