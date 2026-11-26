# HANDOFF — quarkmind

## Orientation

**Physics-first calibration (#763/#768) — SC2 acceleration scaling corrected, cycle timing matched, reroute threshold fixed.**

Per-loop SC2 velocity profiler analysis revealed the EM's acceleration was 40% too low (3.5 u/s² vs SC2's measured 4.84 u/s²). Root cause: acceleration (u/s²) should scale by ×1.4² = ×1.96 at Faster speed, not ×1.4. Combined with fixing intermediate A* waypoint handling, arrival threshold, and reroute threshold, the EM's mining cycle now matches SC2 exactly: 110 loops.

---

## Changes made (this session)

1. **Acceleration 3.5→4.9 u/s²** — `WorkerPhysicsConfig.DEFAULT_ACCELERATION/DECELERATION`. Per-loop SC2 profiler validated: accel=4.84 (σ=0.06, n=2239), decel=4.96 (σ=0.06, n=1851).

2. **ARRIVAL_THRESHOLD 0.5→0.17** — SC2 workers arrive at ~1.3 u/s, corresponding to 0.17u effective threshold.

3. **Intermediate waypoint physics** — `moveOneLoop()` and `moveLoops()` skip deceleration at intermediate A* waypoints. Workers fly through turns (matching SC2 flow-field). `advanceWaypoint()` maintains speed instead of jumping to MAX_SPEED.

4. **Reroute threshold 22→30** — the old threshold equalled the docking overhead, causing workers to reroute at the exact moment patches became available. 12w: 67→0 reroute events, improved from -8.9% to -3.2%.

5. **Physics doc comprehensively updated** — §1 scaling rule corrected, §2 constants, §3 reroute threshold, §6 speed mining, §16 calibration status, §19 config properties.

6. **Diagnostic test suite** — `measureEmCycleTiming8w`, `measureEmCycleTiming12w`, `isolate768Regression`, `sweepAccelerationAndThreshold`, `sweepWaypointSpeedFactor`, `sweepRerouteThreshold`, `sweepInitialStagger`, `measureWaypointCounts`, `isolate768WithOffsetCoords`.

---

## Current state

| Baseline | SC2 | EM | Delta | Main |
|----------|-----|-----|-------|------|
| 8w (real terrain) | 2335 | 2425 | **+3.9%** | +0.4% |
| 12w (real terrain) | 3415 | 3300 | **-3.4%** | -0.3% |
| 8w (offset coords) | 2335 | 2470 | +5.8% | +0.4% |
| 12w (offset coords) | 3415 | 3360 | -1.6% | -0.3% |

Cycle timing (8w): walk(29.4) + mine(49) + walk_back(31.2) = **110 loops** (SC2: 110)

**All 3442 tests pass.**

---

## Next steps

1. **Re-record SC2 baselines** — existing baselines were recorded against main's physics. Use `PlaybookSC2RecorderTest` to re-record 8w and 12w with fresh SC2 data. The current deltas (+3.9%/-3.4%) are measured against stale baselines. Fresh baselines may shift the numbers.

2. **Recalibrate initial stagger** — currently 6.1 loops (calibrated with a=3.5). Needs re-sweep against SC2 with a=4.9. The 8w overshoot may partially be from startup timing.

3. **Gas approach docking recalibration** — `GAS_APPROACH_DOCKING_LOOPS=11.5` was swept with a=3.5. May need re-sweeping with a=4.9.

4. **Pinned test gap (#763)** — with the cycle fix, re-evaluate the first-walk timing and rally exit point fixes that were blocked on #768.

5. **12w contention model** — walks at 12w are 4-5 loops longer than 8w from collision displacement. May need collision model tuning.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic`
- **All 3442 tests pass** (quarkmind-sc2), 86 tests pass (quarkmind-core)
- **Profiler data** in `quarkmind-sc2/src/test/resources/profiler/`: mineral-8w-run1.json (8800 obs), mineral-16w-run1.json, mineral-36w-run1.json
- **Stashed changes:** `git stash list` has uncommitted baseline re-recordings from previous session
- **SC2 is NOT running** — re-recording baselines and stagger re-sweep require SC2
- **Key finding:** acceleration u/s² scales by ×1.4² for game speed, not ×1.4 — this applies to ALL per-second² quantities in the SC2 data editor
