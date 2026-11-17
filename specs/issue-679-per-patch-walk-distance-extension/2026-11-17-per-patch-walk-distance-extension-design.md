# Per-Patch Walk Distance Extension — Design Spec

**Issue:** casehubio/quarkmind#679
**Parent:** #607 (Epic: Standalone economy accuracy)
**Branch:** `issue-679-per-patch-walk-distance-extension`

## Problem

The EM's approach docking constant (24.5 loops) conflates two distinct phenomena:
1. **Approach animation** — the fixed docking/positioning overhead at the mineral patch
2. **Path deviation** — the extra time workers spend navigating around the mineral line going outbound

SC2 cadence data (8w, CycleDecompositionTest) shows clear outbound/return asymmetry:
- WalkTo ranges 2.69–4.29u across patches (longer — workers navigate around mineral line)
- WalkBack ranges 1.47–2.54u (shorter — workers take direct return paths)
- WalkTo is 1.4–1.9x WalkBack for every worker

The current equalization formula (`approach[i] = 24.5 + (medianE2E - E2E[i]) × 5.69 × 0.4`) adjusts approach per patch but absorbs both phenomena into one constant. The asymmetry is not modeled — the EM uses symmetric straight-line walks.

## Design

### Decomposition

Split `approachDockingLoops` into two components:

1. **`approachAnimationLoops`** — constant for all patches at a base. Models only the approach/docking animation overhead. Initial calibration: ~20 loops (derived from residual after extracting outbound delay).

2. **`outboundPathDelay`** — per-patch field on `MineralPatch`. Models the extra time the outbound walk takes due to navigating around the mineral line geometry. Computed from E2E distance via a parametric formula. Range: 0–5 loops.

Constraint: `approachAnimationLoops + outboundPathDelay[i] ≈ currentApproachDocking[i]` for each patch. Total cycle time per patch is preserved.

The return walk stays unchanged. Return walk being shorter is primarily an effective radius effect (workers stop at 3.755u from base, not 2.75u), deferred to a follow-up issue.

### MineralPatch changes

```java
class MineralPatch {
    // existing
    double approachDockingLoops;  // renamed semantically but kept for now
    
    // new
    double outboundPathDelay;     // per-patch extra loops for outbound path deviation
}
```

### computeApproachDocking update

The existing `MineralPatch.computeApproachDocking()` method is updated to compute both components:

```java
static void computeApproachDocking(List<MineralPatch> patches, Point2d basePos) {
    // ... existing E2E computation ...
    
    double approachAnimation = SimulatedWorker.APPROACH_ANIMATION_LOOPS;  // new constant
    
    for (int i = 0; i < patches.size(); i++) {
        // Outbound path delay: parametric from E2E distance
        // Close patches (smaller E2E) get more delay (longer navigation around mineral line)
        double pathDelay = Math.max(0, 
            (medianE2E - e2eDistances[i]) * loopsPerUnit * PATH_DEVIATION_SCALE 
            + PATH_DEVIATION_BASE);
        
        patches.get(i).outboundPathDelay = pathDelay;
        patches.get(i).approachDockingLoops = approachAnimation + pathDelay;
    }
}
```

The `approachDockingLoops` field is maintained as `approachAnimation + outboundPathDelay` so that `tryStartMining` doesn't need to change — it still reads `patch.approachDockingLoops` as the total outbound overhead. The decomposition is internal to `computeApproachDocking`.

### New constants

| Constant | Type | Location | Config property | Initial value |
|----------|------|----------|-----------------|---------------|
| `APPROACH_ANIMATION_LOOPS` | static | `SimulatedWorker` | `starcraft.worker.approach-animation-loops` | ~20.0 |
| `PATH_DEVIATION_SCALE` | static | `MineralPatch` | — | TBD (calibration sweep) |
| `PATH_DEVIATION_BASE` | static | `MineralPatch` | — | TBD (calibration sweep) |

`APPROACH_ANIMATION_LOOPS` replaces `APPROACH_DOCKING_LOOPS` as the base value. The `APPROACH_DOCKING_LOOPS` constant is retained for backward compatibility in `WorkerPhysicsConfig` but becomes the sum of animation + mean path delay.

`PATH_DEVIATION_SCALE` and `PATH_DEVIATION_BASE` are compile-time constants (not CDI config) since they're per-patch geometry parameters, not tuneable physics constants.

### Calibration

Calibration from 8w cadence data (CycleDecompositionTest):
- Approach animation (residual): ~23.0 loops mean (range 20.2–25.6)
- Outbound path delay: 0–4.6 loops (mean 1.6 when non-zero)
- Linear fit R² = 0.104 — E2E is a weak predictor; path deviation depends more on angular position in the mineral line than distance

The calibration sweep should:
1. Sweep `APPROACH_ANIMATION_LOOPS` in [18, 24] range
2. Sweep `PATH_DEVIATION_SCALE` in [0.1, 0.5] range  
3. Sweep `PATH_DEVIATION_BASE` in [0, 3] range
4. Validate against σ (target 0.50–0.60), 7+ tick % (target ≤ 2%), and dual-run mineral accuracy

The low R² is acceptable as a first approximation. The parametric model provides per-patch variation that the current equalization already achieves with similar R². Future refinement can add angular position as a second predictor.

### Integration points

**`tryStartMining` (WorkerSimulation:512-519):** No change needed. Still reads `w.assignedPatch.approachDockingLoops`, which now equals `approachAnimation + outboundPathDelay`.

**`computeApproachDocking`:** Updated to compute decomposed values. Called from `onTownHallCompleted`.

**`WorkerPhysicsConfig`:** Add `approachAnimationLoops` field. Protocol PP-20261116-5f16ed requires CDI default matches the Java constant.

**`WorkerPhysicsConfigProducer`:** Add `starcraft.worker.approach-animation-loops` config property.

**`SimulatedWorker.configure`:** Read and apply `approachAnimationLoops` from config.

### What doesn't change

- Walk targets (edge-to-edge targeting unchanged)
- Movement physics (trapezoidal velocity, acceleration/deceleration)
- Return walk (symmetric, no delay)
- Deposit duration (12 loops)
- Mining duration (22 loops)
- Collision model
- Total cycle time per patch (approach docking decomposition preserves total)

## Acceptance criteria

1. `outboundPathDelay` field on `MineralPatch`, computed per-patch from E2E distance
2. `APPROACH_ANIMATION_LOOPS` constant (animation-only, not path deviation)
3. `approachDockingLoops = approachAnimation + outboundPathDelay` (total preserved)
4. σ maintained within 0.50–0.60 (SC2: 0.56)
5. 7+ tick cycles remain ≤ 2%
6. Dual-run mineral accuracy within existing tolerances (8w +5%, 12w -2%, 16w -2%)
7. All existing tests pass (the total per-patch overhead is unchanged)

## Follow-up

File a separate issue for effective radii correction:
- Mining radius: EM 1.0 → observed 1.215 (workers stop further from patch center)
- Deposit distance: EM 2.75 → observed 3.755 (workers stop further from base center)
- This changes edge-to-edge walk targets, which affects all walk calculations

## References

- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java` — computeApproachDocking formula
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java:512-519` — tryStartMining
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/SimulatedWorker.java` — APPROACH_DOCKING_LOOPS
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` — physics constants
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/CycleDecompositionTest.java` — cadence data source
- `quarkmind-sc2/src/test/resources/calibration/cal-worker-cadence-8w.json` — 8w SC2 cadence baseline
- `docs/sc2-physics-constants.md` §3 (Mineral Mining), §14 (Mining cycle breakdown)
- `docs/protocols/worker-physics-cdi-defaults-must-match-constants.md` — CDI/constant sync protocol
- casehubio/quarkmind#656 — per-patch approach docking and equalization scale (predecessor)
- casehubio/quarkmind#661 — approach docking recalibration from 8w cadence
