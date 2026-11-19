# HANDOFF — quarkmind

## Orientation for Next Session

**#727, #728, #729 closed** this session. Density-dependent path deviation, MULE test rename, SC2 build version capture.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**#707 is next** — EM obstacle-aware pathfinding. This is Scale L / Complexity High. The EM's `WorkerSimulation.moveOneLoop()` moves workers in straight lines. SC2 workers navigate around buildings, mineral patches, and other units. Every approach docking constant, path deviation formula, and density scaling is compensating for this missing physics.

**What #707 replaces:** The entire approach docking compensation stack:
- `computeApproachDocking()` — per-patch static path delay
- `PATH_DEVIATION_SCALE` / `PATH_DEVIATION_BASE` — geometry-based deviation
- `effectiveApproachDocking()` — density-dependent scaling (from #727)
- The wait-time absorption mechanism
- The outbound/return walk asymmetry compensation

The approach animation (20 loops) stays — it models deceleration at the patch, which is real SC2 behaviour.

**Cascading recalibration warning:** Fixing pathfinding will shift OTHER timings. Constants calibrated on top of straight-line walks will overcompensate. Plan for a full recalibration pass: mineral docking, gas docking, deposit, collision model.

**Data correction applied this session:** The #721 finding had SC2/EM columns swapped. DualRunCalibrationTest `[econ]` output format is `SC2_expected vs EM_actual`. Previous handoff table was read as `EM vs SC2`, causing the wrong fix direction. Corrected in physics doc, blog, and forage memory. Issue #730 filed to audit remaining blog entries.

**Current accuracy state (corrected — SC2 first, EM second):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 4w minerals (tick 304) | 1125 | 1165 | **+3.6%** | density scaling (#727) |
| 6w minerals (tick 304) | 1695 | 1740 | **+2.7%** | density scaling (#727) |
| 8w minerals (tick 304) | 2330 | 2325 | **-0.2%** | anchor point |
| 10w minerals (tick 304) | 2840 | 2890 | **+1.8%** | density > 1.0, unchanged |
| 12w minerals (tick 304) | 3460 | 3450 | **-0.3%** | unchanged |
| MULE isolation (tick 305) | 3565 | 3615 | **+1.4%** | trip-based, cargo=25 (#726) |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #727 | LOW_SATURATION_APPROACH_BONUS → density scaling | Fixed — inverse-density formula: `outboundPathDelay / max(density, 0.5)`. 4w +3.6% (was +10.7%), no regression at 8w/12w |
| #728 | MuleCargoVerificationTest lifetimeTicks misleads | Fixed — renamed to `ticksToRun` with clarifying comment |
| #729 | SC2 build version not captured in baselines | Fixed — `QuarkusSC2Transport.baseBuild()` captures from executable path, all 10 recorder call sites updated |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #730 | Blog SC2/EM column swap audit | S/Low | Audit blog entries for propagated column-swap error |

**Blog entries:** `2026-11-19-mdp01-the-bonus-that-worked-at-eight.md` revised — corrected SC2/EM data, documented inverse-density fix, acknowledged it's still compensation

**Published:** project (4 commits), workspace (7 commits)

---

## Queue (2 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#707** | L/High | **NEXT** — EM obstacle-aware pathfinding (root cause of all approach docking compensation) |
| 2 | #730 | S/Low | Audit blog entries for SC2/EM column swap propagation |

**On main** — `work start #707` to begin. Fresh session recommended — L/High task in a different domain (pathfinding vs calibration).
