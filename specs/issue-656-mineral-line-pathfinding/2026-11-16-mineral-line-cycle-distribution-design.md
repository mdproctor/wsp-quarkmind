# Mineral-Line Cycle Distribution — Design Spec

**Issue:** #656
**Branch:** issue-656-mineral-line-pathfinding
**Date:** 2026-11-16

## Problem

The EM's 12w mineral cycle distribution is wider than SC2's. Three variance sources compound to produce 6.6% outlier cycles that SC2 never generates.

| Metric | SC2 | EM (current) | Target |
|--------|-----|-------------|--------|
| Mean | 5.28 ticks | 5.39 ticks | — |
| σ | 0.56 | 0.92 | ≤ 0.70 |
| 7+ tick cycles | 0% | 6.6% (43/654) | ≤ 2% |
| 12w income | 3460 | 3395 (−1.9%) | ±3% |

### Root cause analysis

CycleOutlierDiagnosticTest traced all 43 outlier cycles:

| Cause | Count | % | Mechanism |
|-------|-------|---|-----------|
| WAITING_AT_PATCH | 16 | 37% | Worker arrives at occupied patch, waits 2-3 ticks |
| MIXED | 17 | 40% | Walk slightly extended + brief 1-tick wait |
| LONG_WALK | 10 | 23% | Far-patch walk takes 4+ ticks |

PatchAssignmentDiagnosticTest verified worker distribution is balanced (4 patches × 2 workers, 4 patches × 1 worker). The earlier PatchCycleCorrelationTest appeared to show 5 workers on one patch — this was a measurement artifact from position-based matching on spatially adjacent patches (P0/P4/P6 within 11° and 0.2u of each other).

### Structural cause

The flat `approach_docking_loops=24.5` treats all patches identically. Close patches (E2E 2.27u) and far patches (E2E 3.88u) get the same overhead, but far patches already have longer walks. Workers on far patches with 2-worker sharing get pushed to 7+ ticks because the longer walk + flat approach + contention wait exceeds the 6-tick threshold.

SC2 avoids this because per-patch path deviation from mineral-line navigation naturally desynchronises workers. The 8w cadence data (CycleDecompositionTest) shows per-worker WalkTo ranging 2.69–4.29u with deviation factors 0.72–1.10× of straight-line E2E.

## Design

Three independent leads, applied together, each addressing a distinct variance source.

### Lead 1: Per-patch approach docking

**Change:** `MineralPatch.computeApproachDocking()` computes per-patch values from E2E distance instead of copying the global constant.

**Formula:**

```
approach[i] = base_approach + (median_E2E - E2E[i]) × LOOPS_PER_UNIT
```

Where:
- `base_approach` = global `approach_docking_loops` (24.5 — unchanged)
- `median_E2E` = median edge-to-edge distance across all patches at this base
- `E2E[i]` = `distanceToBase - TOWN_HALL_RADIUS - MINERAL_PATCH_RADIUS` for patch i
- `LOOPS_PER_UNIT` = `GAME_LOOPS_PER_SECOND / WORKER_MAX_SPEED` = 22.4 / 3.94 ≈ 5.69

**Effect for TorchesAIE_v4 calibration patches** (median E2E ≈ 3.38):

| Patch | E2E | Adjustment | Approach |
|-------|-----|-----------|----------|
| P3 | 2.27 | +6.3 | 30.8 |
| P1 | 2.75 | +3.6 | 28.1 |
| P7 | 3.05 | +1.9 | 26.4 |
| P2 | 3.41 | −0.2 | 24.3 |
| P6 | 3.68 | −1.7 | 22.8 |
| P4 | 3.75 | −2.1 | 22.4 |
| P5 | 3.82 | −2.5 | 22.0 |
| P0 | 3.88 | −2.8 | 21.7 |

Close patches get more approach (compensating for shorter walks, modeling the longer navigation path around central patches). Far patches get less (walks are already long, direct paths to edge patches). Total approach time across all patches is preserved by construction (adjustments sum to zero around the median).

**Not a compensation constant:** This decomposes an existing lumped constant into per-patch components modeling a documented physical effect (SC2 workers navigate around mineral line formation — physics constants §15). The approach_docking already absorbs both approach animation and path deviation overhead; this separates them.

**Files changed:**
- `MineralPatch.computeApproachDocking()` — new formula using per-patch E2E distance and median
- `MineralPatch` — `distanceToBase` field already exists, set in `computeApproachDocking()`
- No new config properties — `approach_docking_loops` remains the base; per-patch scaling is derived

### Lead 2: Collision displacement scaling at low saturation

**Change:** `WorkerSimulation.resolveAllMineralCollisions()` scales perpendicular dodge distance by patch contention level.

**Mechanism:** Before applying the perpendicular push, check `w.assignedPatch` worker count. At ≤ 2 workers per patch (the common case at ≤ 16w), scale dodge to 25%: `dodge *= 0.25`. At 3+ workers per patch, keep full dodge.

**Rationale:** The 40% "MIXED" outlier cycles have 1-tick waits caused by collision deflection adding ~0.3–0.5u to walk distance, pushing borderline 6-tick cycles to 7. At ≤ 2 workers/patch, SC2 workers don't physically collide — they share patches by alternating cycles, not by physically pushing each other aside. The lateral push at low saturation adds noise without modeling a real physical effect.

At 3+ workers/patch (24w+), collision IS the physical mechanism — workers crowd around patches and physically jostle. Full displacement preserved there.

**Implementation:** For each walking worker in the collision resolution loop, look up `countAssignedWorkers(w.assignedPatch, w)`. This is cheap — `assignedPatch` is a field reference, and the count iterates only the worker map. The count is computed once per walking worker per tick, not per collision pair.

**Files changed:**
- `WorkerSimulation.resolveAllMineralCollisions()` — add contention-aware dodge scaling

### Lead 3: Contention (deferred — measure after Leads 1-2)

No code change. The 37% "WAITING_AT_PATCH" outliers are expected to reduce when per-patch approach docking creates better cycle alignment between workers sharing a patch. If σ is still above 0.70 after Leads 1-2, the next iteration addresses reroute threshold tuning.

## Validation

1. **σ and 7+ tick:** `WorkerCadenceComparisonTest#cycleLengthHistogram` — σ ≤ 0.70, 7+ tick ≤ 2%
2. **12w income:** `DualRunCalibrationTest` — 12w within ±3% (currently −1.9%)
3. **All playbooks:** `SC2PlaybookCalibrationTest` — all 9 pass
4. **Assignment stability:** `PatchAssignmentDiagnosticTest` — distribution stays balanced
5. **Saturation curve:** 8w, 16w, 24w baselines — no regression beyond ±5%
6. **Outlier trace:** Re-run `CycleOutlierDiagnosticTest` — cause breakdown should shift away from MIXED
7. **Physics doc:** Update `docs/sc2-physics-constants.md` §3 (per-patch approach), §6 (collision scaling), §15 (path deviation now modeled)

## Files changed summary

| File | Change |
|------|--------|
| `MineralPatch.java` | `computeApproachDocking()` — per-patch formula from E2E distance |
| `WorkerSimulation.java` | `resolveAllMineralCollisions()` — contention-aware dodge scaling |
| `docs/sc2-physics-constants.md` | §3, §6, §15 updates |
| New: `PatchCycleCorrelationTest.java` | Already committed — per-patch cycle analysis |
| New: `CycleOutlierDiagnosticTest.java` | Already committed — 7+ tick cause tracing |
| New: `PatchAssignmentDiagnosticTest.java` | Already committed — assignment verification |

## References

- `CycleOutlierDiagnosticTest.java` — 7+ tick outlier cause breakdown (37% waiting, 40% mixed, 23% long walk)
- `PatchAssignmentDiagnosticTest.java` — verified balanced distribution at tick 0/50/150/304
- `PatchCycleCorrelationTest.java` — per-patch cycle length data (position-matching artifact documented)
- `CycleDecompositionTest.java` — 8w SC2 cadence per-worker path deviation (0.72–1.10×)
- `WorkerCadenceComparisonTest.java` — SC2 σ=0.56, EM σ=0.92 histogram
- `docs/sc2-physics-constants.md` §3 (mining constants), §15 (worker pathing gap)
- Protocol: `worker-physics-cdi-defaults-must-match-constants` — if a new config property is added, CDI default must match
- Protocol: `sc2data-spatial-constants-require-calibration` — spatial constants from replay data
- Protocol: `calibration-milestones-are-waypoints` — reach, assess, replan after measuring
