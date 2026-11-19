# HANDOFF — quarkmind

## Orientation for Next Session

**#724–#726 closed** in one session. MULE cleanup sweep after the trip-based model change (#717).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**#727 is next** — replace `LOW_SATURATION_APPROACH_BONUS` with density-dependent path deviation. This is an M/Med task. The bonus is a compensation constant that adds overhead at low saturation — the **opposite** of SC2's behaviour. At low saturation, workers mine at peak efficiency (no contention). The EM's straight-line walks are shorter than SC2's curved paths. The approach docking compensates for this, but a flat bonus doesn't scale with worker density. The proper fix: make `PATH_DEVIATION_SCALE` density-dependent in `MineralPatch.computeApproachDocking()`. At low density, minimal path deviation (straighter paths). At high density, full path deviation.

**Current accuracy state:**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 4w minerals (tick 304) | 1245 | 1125 | **-9.6%** | bonus over-corrects (#727) |
| 6w minerals (tick 304) | 1785 | 1695 | **-5.0%** | bonus over-corrects (#727) |
| 8w minerals | 2330 | 2340 | **+0.4%** | |
| 10w minerals (tick 304) | 2890 | 2840 | **-1.7%** | bonus inactive, correct |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| MULE isolation (tick 305) | 3565 | 3615 | **+1.4%** | trip-based, cargo=25 verified (#726) |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #724 | MULE worker snapshots not visible | Fixed — `boolean mule` field in WorkerSnapshot, merged workers+mules in workerSnapshots() |
| #725 | Remove dead muleIncomePerTick | Fixed — method removed, test removed, diagnostic inlined constant |
| #726 | MULE cargo calibration verify 25 vs 30 | Verified — MuleCargoVerificationTest: 13 trips × 25 = 325 min/lifetime, +1.4% vs SC2; cargo=30 rejected at +12.3% |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #728 | MuleCargoVerificationTest lifetimeTicks misleads | XS/Low | WorkerSimulation doesn't handle MULE expiry |
| #729 | SC2 build version not captured in calibration baselines | S/Low | All baselines have sc2Build=unknown |

**Blog entries:** `2026-11-19-mdp02-the-other-half-of-a-model-change.md` — propagating a model change through API surfaces

**Published:** project (2 commits), workspace (3 commits), casehub blog (4 entries), personal blog (4 entries)

---

## Queue (6 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#727** | M/Med | **NEXT** — Replace approach bonus with density-dependent path deviation |
| 2 | #723 | S/Med | Expansion-transfer SC2 divergence tick 62 |
| 3 | #728 | XS/Low | MuleCargoVerificationTest lifetimeTicks misleads |
| 4 | #729 | S/Low | SC2 build version not captured in calibration baselines |

**On main** — `work continue` to start #727. Branch needs creation.
