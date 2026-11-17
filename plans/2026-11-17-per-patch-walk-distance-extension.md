# Per-Patch Walk Distance Extension Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #679 — Per-patch walk distance extension — model SC2 path geometry directly
**Issue group:** #679

**Goal:** Decompose the approach docking constant into approach animation (fixed) and outbound path delay (per-patch), separating the docking animation from path deviation around the mineral line.

**Architecture:** Split `approachDockingLoops` computation in `MineralPatch.computeApproachDocking()` into two components: a constant `APPROACH_ANIMATION_LOOPS` and a per-patch `outboundPathDelay`. The total `approachDockingLoops` field is maintained as their sum so `tryStartMining` requires no change. Initial values preserve current behavior exactly (pure refactoring), then a calibration sweep explores optimised values.

**Tech Stack:** Java 23, Quarkus 3.x, JUnit 5, Maven

## Global Constraints

- No compensation constants — if the EM doesn't match SC2, fix the root cause
- Protocol PP-20261116-5f16ed: CDI `defaultValue` must match `DEFAULT_*` constant
- All double arithmetic in the income path
- Per-tick delivery deltas must be exact multiples of 5

---

## Batch 1: Decompose approach docking into animation + path delay

### Task 1: Add outboundPathDelay field and decomposed computeApproachDocking

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java:25-84`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java:50,57-72`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java:1-76`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java:50-78`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java`

**Interfaces:**
- Produces: `MineralPatch.outboundPathDelay` (double field, per-patch loops for outbound path deviation)
- Produces: `MineralPatch.PATH_DEVIATION_SCALE` (static double, 0.4)
- Produces: `MineralPatch.PATH_DEVIATION_BASE` (static double, 4.5)
- Produces: `SimulatedWorker.APPROACH_ANIMATION_LOOPS` (static double, 20.0)
- Produces: `WorkerPhysicsConfig.approachAnimationLoops()` (record field)
- Produces: `WorkerPhysicsConfig.DEFAULT_APPROACH_ANIMATION_LOOPS` (20.0)

- [ ] **Step 1: Write failing test — decomposition produces same total as before**

Add a test to `WorkerSimulationTest.java` that verifies:
1. `outboundPathDelay` is populated per-patch
2. `approachDockingLoops = APPROACH_ANIMATION_LOOPS + outboundPathDelay` for each patch
3. Close patches have higher `outboundPathDelay` than far patches
4. Total `approachDockingLoops` per-patch matches the current formula output

```java
@Test
void approachDockingDecomposition_preservesTotalAndSplitsComponents() {
    sim = new WorkerSimulation();
    List<Resource> patches = List.of(
            new Resource("close", new Point2d(53.0f, 50.0f), 1500, false),
            new Resource("mid", new Point2d(57.0f, 50.0f), 1500, false),
            new Resource("far", new Point2d(60.0f, 50.0f), 750, false)
    );
    sim.onTownHallCompleted(BASE_TAG, BASE_POS, patches);

    List<MineralPatch> mp = sim.bases().get(0).patches();

    // outboundPathDelay is populated
    for (MineralPatch p : mp) {
        assertTrue(p.outboundPathDelay >= 0,
                "outboundPathDelay must be non-negative: " + p.outboundPathDelay);
    }

    // decomposition: total = animation + pathDelay
    double animation = SimulatedWorker.APPROACH_ANIMATION_LOOPS;
    for (MineralPatch p : mp) {
        assertEquals(animation + p.outboundPathDelay, p.approachDockingLoops, 0.001,
                "approachDockingLoops should equal animation + pathDelay");
    }

    // close patches get more path delay (more mineral line navigation)
    assertTrue(mp.get(0).outboundPathDelay > mp.get(1).outboundPathDelay,
            "Close patch path delay should exceed mid");
    assertTrue(mp.get(1).outboundPathDelay > mp.get(2).outboundPathDelay,
            "Mid patch path delay should exceed far");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#approachDockingDecomposition_preservesTotalAndSplitsComponents -q`
Expected: FAIL — `outboundPathDelay` field does not exist

- [ ] **Step 3: Add outboundPathDelay field to MineralPatch**

In `MineralPatch.java`, after line 25 (`double approachDockingLoops`), add:

```java
double outboundPathDelay;
```

Add new static constants after `APPROACH_EQUALIZATION_SCALE` (line 26):

```java
static double PATH_DEVIATION_SCALE = 0.4;
static double PATH_DEVIATION_BASE = 4.5;
```

- [ ] **Step 4: Add APPROACH_ANIMATION_LOOPS constant to SimulatedWorker**

In `SimulatedWorker.java`, after line 50 (`APPROACH_DOCKING_LOOPS`), add:

```java
static double APPROACH_ANIMATION_LOOPS = WorkerPhysicsConfig.DEFAULT_APPROACH_ANIMATION_LOOPS;
```

- [ ] **Step 5: Add approachAnimationLoops to WorkerPhysicsConfig**

In `WorkerPhysicsConfig.java`, add new constant:

```java
public static final double DEFAULT_APPROACH_ANIMATION_LOOPS = 20.0;
```

Add `approachAnimationLoops` as a new record field (after `approachDockingLoops`).

Update `defaults()` to include `DEFAULT_APPROACH_ANIMATION_LOOPS`.

Update convenience constructors to pass `DEFAULT_APPROACH_ANIMATION_LOOPS`.

- [ ] **Step 6: Update SimulatedWorker.configure() to read approachAnimationLoops**

In `SimulatedWorker.configure()` (line 57-72), add after line 67:

```java
APPROACH_ANIMATION_LOOPS = config.approachAnimationLoops();
```

- [ ] **Step 7: Add CDI config property to WorkerPhysicsConfigProducer**

In `WorkerPhysicsConfigProducer.java`, add after `approachDockingLoops` field (line 52):

```java
@ConfigProperty(name = "starcraft.worker.approach-animation-loops",
                defaultValue = "20.0")
double approachAnimationLoops;
```

Update `produce()` method to pass `approachAnimationLoops` to the `WorkerPhysicsConfig` constructor.

- [ ] **Step 8: Update computeApproachDocking to compute decomposed values**

Replace the body of `MineralPatch.computeApproachDocking()` (lines 60-84):

```java
static void computeApproachDocking(List<MineralPatch> patches, Point2d basePos) {
    double townHallRadius = WorkerSimulation.TOWN_HALL_RADIUS;
    double patchRadius    = WorkerSimulation.MINERAL_PATCH_RADIUS;
    double loopsPerUnit = io.quarkmind.domain.SC2Data.GAME_LOOPS_PER_SECOND
                          / io.quarkmind.domain.SC2Data.unitSpeed(io.quarkmind.domain.UnitType.PROBE);

    double[] e2eDistances = new double[patches.size()];
    for (int i = 0; i < patches.size(); i++) {
        MineralPatch p = patches.get(i);
        p.distanceToBase = p.position.distanceTo(basePos);
        e2eDistances[i]  = Math.max(0, p.distanceToBase - townHallRadius - patchRadius);
    }

    double[] sorted = e2eDistances.clone();
    java.util.Arrays.sort(sorted);
    double medianE2E = sorted.length % 2 == 0
                       ? (sorted[sorted.length / 2 - 1] + sorted[sorted.length / 2]) / 2.0
                       : sorted[sorted.length / 2];

    double approachAnimation = SimulatedWorker.APPROACH_ANIMATION_LOOPS;
    for (int i = 0; i < patches.size(); i++) {
        double pathDelay = Math.max(0,
                (medianE2E - e2eDistances[i]) * loopsPerUnit * PATH_DEVIATION_SCALE
                + PATH_DEVIATION_BASE);
        patches.get(i).outboundPathDelay = pathDelay;
        patches.get(i).approachDockingLoops = approachAnimation + pathDelay;
    }
}
```

- [ ] **Step 9: Run new test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#approachDockingDecomposition_preservesTotalAndSplitsComponents -q`
Expected: PASS

- [ ] **Step 10: Run existing approach docking test to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest#perPatchApproachDocking_variesByDistance -q`
Expected: PASS — close > mid > far ordering preserved

- [ ] **Step 11: Run full WorkerSimulationTest suite**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkerSimulationTest -q`
Expected: All tests PASS

- [ ] **Step 12: Run DualRunCalibrationTest to verify mineral accuracy preserved**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DualRunCalibrationTest -q`
Expected: All self-baseline tests PASS (EM produces identical output since the formula is equivalent)

- [ ] **Step 13: Run full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests PASS

- [ ] **Step 14: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java \
       quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/WorkerSimulationTest.java
git commit -m "feat(#679): decompose approach docking into animation + outbound path delay

Split approach docking (24.5 loops) into APPROACH_ANIMATION_LOOPS (20.0)
and per-patch outboundPathDelay (computed from E2E distance via
PATH_DEVIATION_BASE=4.5 + median-relative scaling). The sum equals the
previous per-patch approachDockingLoops exactly — pure refactoring.

Refs #679"
```

---

## Batch 2: Calibration sweep and documentation

### Task 2: Calibration sweep test for decomposed constants

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ApproachDockingSweepTest.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ApproachDockingSweepTest.java`

**Interfaces:**
- Consumes: `MineralPatch.PATH_DEVIATION_SCALE` (static double)
- Consumes: `MineralPatch.PATH_DEVIATION_BASE` (static double)
- Consumes: `SimulatedWorker.APPROACH_ANIMATION_LOOPS` (static double)

- [ ] **Step 1: Write calibration sweep test**

Add a new test method to `ApproachDockingSweepTest.java` that sweeps the three decomposed constants against the 8w and 12w dual-run baselines:

```java
@Test
@Tag("diagnostic")
void sweepDecomposedApproachConstants() throws IOException {
    CalibrationBaseline baseline8w  = loadBaseline("calibration/cal-mineral-baseline-8w.json");
    CalibrationBaseline baseline12w = loadBaseline("calibration/cal-mineral-baseline-12w.json");

    double bestAnimation = 0, bestScale = 0, bestBase = 0;
    double bestCombinedError = Double.MAX_VALUE;

    System.out.printf("%-6s %-6s %-6s | %8s %8s | %8s%n",
            "Anim", "Scale", "Base", "8w Δ%", "12w Δ%", "Combined");
    System.out.println("-".repeat(60));

    for (double animation = 18.0; animation <= 24.0; animation += 1.0) {
        for (double scale = 0.1; scale <= 0.5; scale += 0.1) {
            for (double base = 0.0; base <= 6.0; base += 0.5) {
                SimulatedWorker.APPROACH_ANIMATION_LOOPS = animation;
                MineralPatch.PATH_DEVIATION_SCALE = scale;
                MineralPatch.PATH_DEVIATION_BASE = base;

                double delta8w  = runBaseline(baseline8w, 304);
                double delta12w = runBaseline(baseline12w, 304);
                double combined = Math.abs(delta8w) + Math.abs(delta12w);

                if (combined < bestCombinedError) {
                    bestCombinedError = combined;
                    bestAnimation = animation;
                    bestScale = scale;
                    bestBase = base;
                }
            }
        }
    }

    SimulatedWorker.resetToDefaults();

    System.out.printf("%nBest: animation=%.1f, scale=%.2f, base=%.1f (combined=%.2f%%)%n",
            bestAnimation, bestScale, bestBase, bestCombinedError);
}
```

The `runBaseline` helper already exists in `ApproachDockingSweepTest` — it creates a `WorkerSimulation`, seeds patches from the calibration baseline, runs ticks, and returns the mineral delta percentage vs SC2.

- [ ] **Step 2: Run sweep to find optimal values**

Run: `mvn test -pl quarkmind-sc2 -Dtest=ApproachDockingSweepTest#sweepDecomposedApproachConstants -Pdiagnostic -q`
Expected: Prints sweep results with best combined error. If the best differs from initial values (20.0, 0.4, 4.5), update the constants in the next step.

- [ ] **Step 3: Update constants if sweep finds better values**

If the sweep identifies values with lower combined error while maintaining:
- σ within 0.50–0.60
- 7+ tick cycles ≤ 2%
- 8w delta ≤ +5%, 12w delta ≤ −2%

Update `DEFAULT_APPROACH_ANIMATION_LOOPS` in `WorkerPhysicsConfig.java`, `PATH_DEVIATION_SCALE` and `PATH_DEVIATION_BASE` in `MineralPatch.java`, and the CDI `defaultValue` in `WorkerPhysicsConfigProducer.java`.

- [ ] **Step 4: Run full test suite with updated values**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/ApproachDockingSweepTest.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java \
       quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfigProducer.java
git commit -m "feat(#679): calibration sweep for decomposed approach constants

Sweep APPROACH_ANIMATION_LOOPS [18-24], PATH_DEVIATION_SCALE [0.1-0.5],
PATH_DEVIATION_BASE [0-6] against 8w and 12w dual-run baselines.

Refs #679"
```

### Task 3: Update physics documentation

**Files:**
- Modify: `docs/sc2-physics-constants.md`

**Interfaces:**
- Consumes: Final calibrated values from Task 2

- [ ] **Step 1: Update §3 Mineral Mining table**

In the approach/docking delay row of `docs/sc2-physics-constants.md` §3, update:

```
| Approach animation | 20.0 loops | ~20–22 loops | Base animation/docking overhead — constant for all patches at a base. Decomposed from approach docking (#679). | **Calibrated** — #679 | `starcraft.worker.approach-animation-loops` |
| Outbound path delay | Per-patch: max(0, (median_E2E − E2E[i]) × 5.69 × 0.4 + 4.5) | 0–5 loops | Extra outbound time for navigating around mineral line. Close patches get more delay. Decomposed from approach docking (#679). | **Calibrated** — #679 cadence data | — (compile-time) |
```

- [ ] **Step 2: Update §14 Mining cycle breakdown**

Update the approach/dock row to show the decomposed values and explain what each component models.

- [ ] **Step 3: Update §19 Quarkus Config Properties**

Add `starcraft.worker.approach-animation-loops=20.0` entry. Document `PATH_DEVIATION_SCALE` and `PATH_DEVIATION_BASE` in the static-field constants section.

- [ ] **Step 4: Commit**

```bash
git add docs/sc2-physics-constants.md
git commit -m "docs(#679): update physics constants for decomposed approach docking

Document APPROACH_ANIMATION_LOOPS, PATH_DEVIATION_SCALE, PATH_DEVIATION_BASE
in §3, §14, and §19.

Refs #679"
```

---

## References

- [2026-11-17-per-patch-walk-distance-extension-design.md] — design spec this plan implements
- [MineralPatch.java:60-84] — computeApproachDocking formula (modified)
- [SimulatedWorker.java:50,57-72] — APPROACH_DOCKING_LOOPS and configure() (modified)
- [WorkerPhysicsConfig.java] — physics constants record (modified)
- [WorkerPhysicsConfigProducer.java:50-78] — CDI config (modified)
- [WorkerSimulationTest.java:1037-1060] — existing perPatchApproachDocking test
- [ApproachDockingSweepTest.java] — existing sweep tests (modified)
- [docs/sc2-physics-constants.md] — physics documentation (modified)
- [PP-20261116-5f16ed] — CDI default must match DEFAULT_ constant
- [GitHub #679] — focal issue
- [GitHub #656] — per-patch approach docking predecessor
- [GitHub #661] — approach docking recalibration from 8w cadence
