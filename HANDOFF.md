# HANDOFF — quarkmind

## Orientation

**Physics-first calibration methodology (#763) — Phase 1-2 complete. Three SC2-measured constants replace one swept constant. Collision model exposed as the real gap.**

Per-loop SC2 profiler (8w: 77 cycles, 16w: 183 cycles, 36w: 37 cycles) on TorchesAIE measured the actual mineral mining physics at game-loop resolution. Three findings replaced `DEPOSIT_DURATION_LOOPS=12` (swept) with two measured constants and confirmed a third:

| Constant | Old (swept) | New (measured) | SC2 σ | Source |
|----------|------------|---------------|-------|--------|
| Deposit dwell at TH | 12 loops | **0 loops** | 0.0 | Per-loop profiler: instantaneous deposit |
| Post-pickup departure | not modeled | **7 loops** | 0.0 | Per-loop profiler: departure animation at patch |
| Approach + mining | 20+22=42 | **42.1 loops** | 11.1 | Per-loop profiler: validates existing constants |

The 12-loop deposit was compensating for the missing 7-loop post-pickup animation. Both σ=0 — deterministic SC2 behavior.

**Wait-time absorption is wrong physics.** SC2 16w profiler data (183 cycles) shows waited workers get the full 20-loop approach animation on top of their wait. No absorption occurs — approach=20 always.

**Remaining gap: -16.4% at 36w with measured physics.** This is the collision/dodge model — perpendicular displacement extends walks at high saturation. The absorption formula was compensating for this, not modeling real SC2 behavior.

---

## Changes made (this session, 2026-11-25)

1. **WorkerPhysicsProfilerTest** (`quarkmind-sc2/src/test/java/io/quarkmind/sc2/real/`): Per-loop SC2 mineral worker profiler. Records position + cargo at every game loop. Three scenarios: 8w, 16w, 36w.

2. **DEPOSIT_DURATION_LOOPS = 0** (`WorkerPhysicsConfig.java`): SC2 deposits cargo in the same loop the worker arrives at TH. Zero dwell time. Deceleration is in movement physics.

3. **POST_PICKUP_LOOPS = 7** (`SimulatedWorker.java`): After cargo flip, worker stays at patch 7 loops before departing. Deterministic (σ=0). Added two-stage check in `stepOneMineralLoop`: cargo flips first, then departure after POST_PICKUP_LOOPS.

4. **Approach absorption removed** (`WorkerSimulation.tryStartMining`): Approach is now fixed at 20 loops regardless of wait state. Matches SC2 16w measurement showing no absorption.

5. **Physics doc updated** (`docs/sc2-physics-constants.md`): §3 constants updated with Measured confidence level. §16 pinned results updated.

6. **Design spec** (`specs/issue-761-intent-rejection-diagnostic/2026-11-25-physics-first-calibration-design.md`): Reusable physics-first calibration methodology with four-category classification framework.

7. **Diagnostic sweeps** (`ContentionCalibrationDiagnosticTest.java`): Five sweep tests (absorption threshold, dodge scale, approach animation, combined, min-approach). These explored the parameter space before the per-loop profiler provided direct measurements.

---

## Current calibration state

| Scenario | Pre-session | Post-measured-physics | Note |
|----------|------------|----------------------|------|
| Pinned single-base 36w | -6.2% | **-16.4%** | Collision model gap exposed |
| Pinned two-base | -8.5% | **-14.2%** | Same root cause |
| 8w dual-run | +0.4% | PASS (structural) | Foundation constants validated |
| 12w dual-run | -0.3% | PASS (structural) | Foundation constants validated |
| All 238 tests | PASS | PASS | No regressions |

---

## Next steps

1. **Collision/dodge model investigation** — the -16.4% gap at 36w is from the perpendicular dodge model. At 4.5 w/p, workers get displaced sideways on every collision, extending walks. SC2's flow-field pathfinder handles this more efficiently. Record SC2 worker walk distances at different saturation levels and compare to EM walks. The profiler data already has position data for this analysis.

2. **LOW_SATURATION_DODGE_SCALE validation** — with measured foundation constants, re-evaluate whether the 0.25 dodge scale at ≤2 w/p is still needed. Phase 4 of the methodology.

3. **INITIAL_WORKER_STAGGER validation** — record SC2 first-10-tick per-worker data. Phase 3 of the methodology.

4. **Gas approach docking measurement** — the 11.5-loop value is swept. `GasCycleProfilerTest` data already exists for this analysis.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic`
- **All 238 tests pass**
- **SC2 is running** on port 8168
- **Profiler data recorded** in `quarkmind-sc2/src/test/resources/profiler/`: mineral-8w-run1.json, mineral-16w-run1.json, mineral-36w-run1.json
- **IntelliJ MCP connected** at `/Users/mdproctor/claude/casehub/quarkmind`
