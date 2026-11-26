# HANDOFF — quarkmind

## Orientation

**Physics-first calibration (#763/#768) — SC2 acceleration scaling corrected, cycle timing matched, contention model recalibrated.**

Per-loop SC2 velocity profiler analysis revealed the EM's acceleration was 40% too low (3.5 u/s² vs SC2's measured 4.84 u/s²). Root cause: acceleration (u/s²) should scale by ×1.4² = ×1.96 at Faster speed, not ×1.4. Combined with fixing intermediate A* waypoint handling, arrival threshold, reroute threshold, and collision dodge, the EM's mining cycle now matches SC2 exactly: 110 loops.

---

## Changes made (this session)

1. **Acceleration 3.5→4.9 u/s²** — SC2 per-loop profiler validated: accel=4.84 (σ=0.06, n=2239), decel=4.96 (σ=0.06, n=1851). Scales ×1.4² for per-second² quantities.

2. **ARRIVAL_THRESHOLD 0.5→0.17** — SC2 workers arrive at ~1.3 u/s, v²/(2a) = 0.17u.

3. **Intermediate waypoint physics** — `moveOneLoop()`/`moveLoops()` skip deceleration at intermediate A* waypoints. Workers fly through turns (matching SC2 flow-field). `advanceWaypoint()` maintains speed.

4. **Reroute threshold 22→30** — the old threshold equalled docking overhead, causing unnecessary rerouting at 12w. Workers now wait for the patch to become available before considering rerouting.

5. **Collision dodge scale 0.25→0.10** — higher acceleration makes collision dodge relatively more disruptive. Recalibrated to maintain correct 12w income.

6. **Physics doc comprehensively updated** — §1 scaling rule, §2 acceleration/threshold, §3 reroute/collision, §6 speed mining, §16 calibration status, §19 config properties.

7. **Diagnostic test suite** — 9 new diagnostic tests in `ContentionCalibrationDiagnosticTest`: cycle timing, waypoint counts, acceleration sweep, stagger sweep, reroute sweep, dodge sweep, regression isolation, offset comparison.

---

## Current state

| Baseline | SC2 | EM | Delta | Main |
|----------|-----|-----|-------|------|
| 8w (real terrain) | 2335 | 2415 | **+3.4%** | +0.4% |
| 12w (real terrain) | 3415 | 3395 | **-0.6%** | -0.3% |
| Pinned 36w (real terrain) | 3710 | 2595 | **-30.1%** | -15.1% |

Cycle timing (8w): walk(29.4) + mine(49) + walk_back(31.2) = **110 loops** (SC2: 110)

**All 3442 tests pass** (quarkmind-sc2). 86 tests pass (quarkmind-core).

---

## Next steps

1. **Pinned test gap (-30.1%, #763)** — worsened from -15.1%. The gap accumulates during 12→36 worker injection. Each new worker's first walk routes through A* obstacles (2-4 intermediate waypoints). With the new physics (fly-through, 0.17u threshold), first walks take longer. Needs SC2 profiler data at high saturation AND first-walk timing measurement.

2. **Re-record SC2 baselines** — existing baselines were recorded against main's physics. Use `PlaybookSC2RecorderTest` to re-record 8w, 12w, and gas baselines with SC2. Stale baselines may distort the accuracy numbers.

3. **Initial stagger recalibration** — currently 6.1 loops (calibrated with a=3.5). Needs SC2 re-sweep with a=4.9. The 8w overshoot (+3.4%) may be from startup timing.

4. **Gas approach docking** — `GAS_APPROACH_DOCKING_LOOPS=11.5` calibrated with a=3.5. Needs re-sweep.

5. **High-saturation contention** — per-worker rate at 24w and 36w needs SC2 profiler validation.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic`
- **Profiler data** in `quarkmind-sc2/src/test/resources/profiler/`: mineral-8w-run1.json (8800 obs), mineral-16w-run1.json, mineral-36w-run1.json
- **Stashed changes:** `git stash list` has uncommitted baseline re-recordings from previous session
- **SC2 is NOT running** — re-recording baselines requires starting SC2
- **Key finding:** acceleration u/s² scales by ×1.4² for game speed, not ×1.4 — applies to ALL per-second² quantities in SC2 data editor
