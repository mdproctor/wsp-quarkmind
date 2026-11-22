# HANDOFF — quarkmind

## Orientation for Next Session

**Five cleanup issues landed** this session (#739, #740, #738, #742, #743), plus a comprehensive physics doc revision, economy accuracy gap analysis, and six new economy issues filed and scheduled.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**. Major revision this session: §3 (approach values), §13 (Footprint record, gas/EvoChamber corrections), §14 (cycle breakdown, drift numbers), §15 (enforceWall optimization), §16 (all economy baseline numbers refreshed, accuracy plan added).

**Key changes landed this session:**

1. **Dead config removal (#739):** `APPROACH_DOCKING_LOOPS` (24.5) removed from `SimulatedWorker`, `WorkerPhysicsConfig`, and `WorkerPhysicsConfigProducer`. Was immediately overwritten by `computeApproachDocking()` — dead since #707.

2. **Immutable Footprint record (#740):** `SC2Data.buildingFootprint()` now returns `Footprint(width, height)` with `halfWidth()`/`halfHeight()` helpers instead of shared mutable `int[]`. All 9 callers updated.

3. **Building footprint corrections (#738):** EVOLUTION_CHAMBER 2×2→3×3 (matches SC2 data editor and all Zerg tech buildings). Gas buildings (Assimilator/Refinery/Extractor/Assimilator_Rich) 2×2→3×3 (matches 3×3 geysers). NYDUS_CANAL left at 2×2 pending verification (#750).

4. **Geyser footprint verified (#742):** `GEYSER_OBSTACLE_HALF_EXTENT = 1.5` (3×3) confirmed correct — matches gas building footprint from #738.

5. **enforceWall optimization (#743):** Building iteration now skips buildings registered as terrain obstacles (covered by `isWalkable` check). Reduces from O(units × all_buildings) to O(units × initial_buildings). Key finding: initial town hall CANNOT be registered as an obstacle — worker A* paths route through the base area, and registering it causes 4.6% MULE economy regression.

**Economy accuracy state (standalone baseline, 2026-11-22):**

| Field | MAPE | Floor | Status |
|-------|------|-------|--------|
| mineralsCollectionRate | 38.6% | ~10-12% | Gap: rate algorithm, map geometry, worker timing |
| vespeneCollectionRate | 20.5% | ~12-15% | Approaching floor |
| mineralsCurrent | 84.9% | ~40-50% | Cumulative drift from rate error |
| vespeneCurrent | 61.1% | ~35-45% | Cumulative drift |

---

## Completed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #739 | Remove dead APPROACH_DOCKING_LOOPS config | Done — constant, field, config property removed |
| #740 | buildingFootprint() shared mutable int[] | Done — replaced with immutable Footprint record |
| #738 | EVOLUTION_CHAMBER footprint verification | Done — corrected to 3×3, gas buildings to 3×3 |
| #742 | Verify geyser obstacle footprint | Done — confirmed 3×3 correct |
| #743 | Optimise enforceWall building iteration | Done — skip registered buildings |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #744 | Standalone mineral rate MAPE gap decomposition | M/Med | Identify what drives 38.6% mineral rate MAPE |
| #745 | Per-replay mineral patch geometry vs EM distances | S/Med | Map distance contribution to rate gap |
| #746 | Worker startup/transfer timing in standalone | S/Med | Allocation delay contribution |
| #747 | Balance drift MAPE validation after rate fixes | S/Low | Verify balance drops proportionally |
| #748 | Remove deprecated buildingRadius() | XS/Low | Zero callers remain |
| #749 | ApproachDockingSweepTest stale sweep ranges | XS/Low | Ranges calibrated for removed constant |
| #750 | Verify NYDUS_CANAL footprint empirically | XS/Low | Left at 2×2, may be 3×3 |

**Blog entries:** `2026-11-22-mdp02-the-obstacle-with-two-faces.md` — shared terrain grid has two consumers (army, workers) with different obstruction rules

---

## Queue (next items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#744** | M/Med | Gap decomposition — where does the 38.6% mineral rate MAPE come from? |
| 2 | #745 | S/Med | Per-replay mineral patch geometry vs EM distances |
| 3 | #746 | S/Med | Worker startup/transfer timing |
| 4 | #676 | M/High | Reverse-engineer SC2 rate computation algorithm |
| 5 | #747 | S/Low | Balance validation after rate fixes |
| 6 | #666 | L/High | Forward simulation economy mode |
| 7 | #748 | XS/Low | Remove deprecated buildingRadius() |
| 8 | #749 | XS/Low | Sweep test stale ranges |
| 9 | #750 | XS/Low | Verify Nydus Canal footprint |

`work continue` picks up #744 — the economy gap decomposition investigation.
