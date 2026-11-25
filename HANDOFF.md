# HANDOFF — quarkmind

## Orientation

**Physics-first calibration (#763) — per-loop SC2 profiler built, foundation constants measured, aggregate-fit coupling exposed and broken.**

Built a reusable per-loop SC2 worker profiler (`WorkerPhysicsProfilerTest`) and used it to directly measure mineral mining physics at game-loop resolution. The profiler revealed that two constants were compensating for each other — a treadmill that parameter sweeping could never fix.

---

## Key findings

### 1. Deposit is instantaneous — not 12 loops

`DEPOSIT_DURATION_LOOPS` was 12 (swept from 8w cadence fitting #661). SC2 per-loop profiler: deposit dwell at TH = **0 loops (σ=0)**. The worker drops cargo in the same loop it arrives. Deceleration is in movement physics, not a separate deposit phase.

### 2. Post-pickup departure = 7 loops (new)

After cargo flip, the worker stays at the patch exactly **7 loops (σ=0)** before departing. Deterministic — zero variance across 76 observations. Previously not modeled. The old `DEPOSIT_DURATION_LOOPS=12` was compensating for this missing 7-loop animation.

### 3. Approach = 20 loops always — no absorption

SC2 16w profiler (183 cycles): waited workers get the **full 20-loop approach** on top of their wait time. `pre_pickup = wait + approach(20) + mine(22)`. SC2 does not absorb approach during waiting. The approach animation is a fixed physical constant.

### 4. Steady-state rates are correct

| Workers | EM rate | SC2 reference | EM vs SC2 |
|---------|---------|--------------|-----------|
| 12w | 0.925/w/tick | 0.95/w/tick | -3% |
| 16w | 0.872/w/tick | 0.88/w/tick | -1% |
| 36w | 0.458/w/tick | ~0.37/w/tick | +24% |

EM matches or exceeds SC2 at all saturation levels at steady state.

### 5. Pinned test gap (-16.4%) is transient

The pinned test adds workers one-by-one over 304 ticks. The gap accumulates during the 24w→36w transition — each newly-born worker walks from Nexus to its patch, delaying first delivery. The EM tracks SC2 through tick 150 (24w, Δ=+10). The gap opens from tick 150-304 as probes are added. This is a worker injection/assignment path issue, not steady-state physics.

---

## Changes made (this session)

1. **WorkerPhysicsProfilerTest** — per-loop SC2 mineral worker profiler. Records position + cargo at every game loop. Three recordings: 8w (8800 obs), 16w (25300 obs), 36w (7416 obs).

2. **DEPOSIT_DURATION_LOOPS = 0** — measured: instantaneous deposit.

3. **POST_PICKUP_LOOPS = 7** — measured: departure animation at patch. Two-stage check in `stepOneMineralLoop`: cargo flips first, then departure after POST_PICKUP_LOOPS.

4. **Approach absorption removed** — SC2 measurement disproved it. Approach is fixed 20 loops regardless of wait state.

5. **Physics doc comprehensively revised** — §3 cycle decomposition rewritten with per-loop profiler data. §14 cycle breakdown updated. §16 pinned results updated with transient gap analysis. Walk distance by saturation table added.

6. **Design spec** — reusable physics-first calibration methodology with four-category classification framework.

7. **COLLISION_ENABLED flag** — diagnostic toggle for collision model analysis. Default true.

---

## Next steps

1. **Worker injection path investigation** — why does the EM's new-worker-to-first-delivery time create a -16.4% gap in the pinned test when steady-state rates are correct? Compare the EM's `onWorkerProducedAtBase → assignToNearestPatch → walk to patch` path against SC2's rally point behavior. The profiler data has per-worker position traces during the ramp-up period for this analysis.

2. **Gas approach docking measurement** — `GAS_APPROACH_DOCKING_LOOPS=11.5` is swept. `GasCycleProfilerTest` data already exists for per-loop analysis.

3. **Initial worker stagger measurement** — record SC2 first-10-tick per-worker data via the profiler.

4. **Phase 4: validate emergent constants** — with measured foundation constants, re-evaluate `LOW_SATURATION_DODGE_SCALE=0.25` and `INITIAL_WORKER_STAGGER=6.1`.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic`
- **All 238 tests pass**
- **SC2 is running** on port 8168
- **Profiler data** in `quarkmind-sc2/src/test/resources/profiler/`: mineral-8w-run1.json, mineral-16w-run1.json, mineral-36w-run1.json
- **Known code/doc mismatch:** `MINERAL_PATCH_OBSTACLE_RADIUS` — `WorkerPhysicsConfig` defaults to 0.5, `WorkerPhysicsConfigProducer` defaults to 0.3. Needs independent fix.
