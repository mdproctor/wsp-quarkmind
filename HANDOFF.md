# HANDOFF — quarkmind

## Orientation

**Physics-first calibration (#763/#768) — SC2 acceleration corrected, cycle timing matched, fresh baselines recorded.**

Per-loop SC2 velocity profiler revealed the EM's acceleration was 40% too low (3.5→4.9 u/s²). Combined with intermediate waypoint fly-through, corrected arrival threshold (0.17u), reroute threshold (30 loops), collision dodge (0.10), and rally spawn position, the EM now matches SC2's mining cycle exactly (110 loops) with 0% error at 8w.

---

## Changes made (this session)

1. **Acceleration 3.5→4.9 u/s²** — SC2 profiler: accel=4.84 (n=2239), decel=4.96 (n=1851). Per-second² quantities scale by ×1.4² at Faster speed.
2. **ARRIVAL_THRESHOLD 0.5→0.17** — SC2 workers arrive at ~1.3 u/s.
3. **Intermediate waypoint fly-through** — no deceleration at A* waypoints. Speed maintained through turns.
4. **Reroute threshold 22→30** — prevented unnecessary rerouting that caused 329-loop walks at 12w.
5. **Collision dodge 0.25→0.10** — higher acceleration amplifies displacement.
6. **Rally spawn position** — workers spawn at Nexus edge toward minerals (SC2 measured: 2.75u from center).
7. **Fresh SC2 baselines** — all 11 baselines re-recorded from SC2 (8w, 12w, 10w, Terran, Zerg, MULE, 3w gas, 6w gas, pinned, terrain grid).
8. **First-delivery profiler** — SC2 mean 170 loops (7.7 ticks) for new probes.
9. **Physics doc comprehensively updated** — scaling rule, acceleration, threshold, reroute, collision, speed mining, calibration status.
10. **9 diagnostic tests** — cycle timing, waypoint counts, acceleration sweep, stagger sweep, reroute sweep, dodge sweep, regression isolation.

---

## Current state

| Baseline | SC2 | EM | Delta |
|----------|-----|-----|-------|
| 8w (fresh recording) | 2325 | 2325 | **0.0%** |
| 12w (fresh recording) | 3415 | 3395 | **-0.6%** |
| Pinned 36w | 3700 | 2595 | -29.9% |

Cycle timing (8w): walk(29.4) + mine(49) + walk_back(31.2) = **110 loops** (SC2: 110)

**All 3442 tests pass.**

---

## Remaining: pinned test gap (-29.9%)

The gap is NOT from first-walk starting position (rally fix: marginal). It's from **per-worker rate deficit at 2-3 w/p**:

| Workers | w/patch | EM rate | SC2 reference | EM vs SC2 |
|---------|---------|---------|--------------|-----------|
| 12 | 1.5 | 0.862 | 0.95 | -9.3% |
| 16 | 2.0 | 0.809 | 0.88 | -8.1% |
| 36 | 4.5 | 0.389 | ~0.37 | +5.1% |

With collision disabled: 12w rate = 0.958 (+0.8%) — perfect. The collision model is the dominant factor at moderate saturation.

**Root cause hypothesis**: collision dodge displaces workers from their A* path. With the smaller arrival threshold (0.17u), even small displacements require full correction cycles. At ≤2 w/p, dodge scale 0.10 fixes this. At 2-3 w/p (the pinned test's 20-28 worker range), full dodge (1.0) applies and over-penalises.

**Fix path**: investigate a contention-dependent dodge scale that reduces dodge at 2-3 w/p, or improve the collision resolution to avoid displacing workers from their target direction.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (82 commits ahead of main)
- **All 3442 tests pass**
- **SC2 is running** — can record more baselines if needed
- **Profiler data** in `profiler/`: mineral-8w, mineral-16w, mineral-36w, first-delivery-timing
- **Key finding**: acceleration u/s² scales by ×1.4² for game speed — applies to ALL per-second² quantities
