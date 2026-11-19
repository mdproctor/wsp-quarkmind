# HANDOFF — quarkmind

## Orientation for Next Session

**#718–#722 closed** in one batch. #727 filed with Liquipedia/internet research on the speed mining mechanism.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Key finding this session:** `LOW_SATURATION_APPROACH_BONUS` is a compensation constant that adds overhead at low saturation — the **opposite** of SC2's behaviour. At low saturation, workers mine at peak efficiency (no contention). The EM's straight-line walks are shorter than SC2's curved paths. The approach docking compensates for this, but a flat bonus doesn't scale with worker density. The proper fix is #707 (obstacle-aware pathfinding) so workers navigate around the TH and each other naturally. At low saturation, paths are straighter → less path deviation → faster cycles. At high saturation, paths curve → more deviation → slower cycles. The mechanism replaces the constant.

**Speed mining:** SC2 workers decelerate before targets. Speed mining (two-point commands) bypasses this for 10-12% income boost. The EM's acceleration/deceleration values (3.5 u/s²) match SC2 data editor. A future QuarkMind agent could use speed mining as a legitimate optimisation via the same two-point commands human players use.

**Current accuracy state:**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 4w minerals (tick 304) | 1245 | 1125 | **-9.6%** | bonus over-corrects (#727) |
| 6w minerals (tick 304) | 1785 | 1695 | **-5.0%** | bonus over-corrects (#727) |
| 8w minerals | 2330 | 2340 | **+0.4%** | |
| 10w minerals (tick 304) | 2890 | 2840 | **-1.7%** | bonus inactive, correct |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| MULE isolation (tick 305) | 3565 | 3615 | **+1.4%** | trip-based |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #717 | MULE trip-based mining model | Fixed — trip-based via WorkerSimulation; +1.4% delta |
| #718 | Morph name valueOf warning | Fixed — warning on fallback path in EmulatedGame + SC2DeliveryHandler |
| #719 | WorkerSim per-phase instrumentation | Added — WorkerPhaseListener callback, 5 phases, diagnostic test |
| #720 | Wire physics config constants | Fixed — 3 constants wired through WorkerPhysicsConfig |
| #721 | Validate low-saturation approach bonus | Done — SC2 baselines 4w/6w/10w, found bonus is backwards |
| #722 | Diagnostic tests debugCreateBuilding | Fixed — routed through BuildIntent.of() |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #724 | MULE worker snapshots not visible | XS/Low | workerSnapshots() doesn't include MULEs |
| #725 | Remove dead muleIncomePerTick | XS/Low | Flat rate method unused in production |
| #726 | MULE cargo calibration verify 25 vs 30 | S/Low | Empirical verification needed |
| #727 | Replace LOW_SATURATION_APPROACH_BONUS with density-dependent path deviation | M/Med | Bonus is backwards — proper fix is #707 pathfinding |

**Blog entries:** `2026-11-18-mdp05-mules-dont-get-a-flat-rate.md`, `2026-11-19-mdp01-the-bonus-that-worked-at-eight.md`

---

## Queue (7 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#724** | XS/Low | **NEXT** — MULE worker snapshots not visible |
| 2 | #725 | XS/Low | Remove dead muleIncomePerTick |
| 3 | #726 | S/Low | MULE cargo calibration verify 25 vs 30 |
| 4 | #727 | M/Med | Replace approach bonus with density-dependent path deviation |
| 5 | #723 | S/Med | Expansion-transfer SC2 divergence tick 62 |

**On main** — `work continue` to start #724.

**Recommended next:** #725 (XS dead code) or #724 (XS snapshot gap) for quick fixes. #727 (M/Med) is the most impactful — eliminates a compensation constant and replaces with mechanism. #707 (pathfinding) is the ultimate fix but L/High scale.

---

## Unpublished

Both project (8 commits) and workspace (9 commits) repos have unpublished commits on main.
