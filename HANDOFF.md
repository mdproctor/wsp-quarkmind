# HANDOFF — quarkmind

## Orientation for Next Session

**Four economy investigation issues completed** this session (#745, #746, #676, #747), including a significant rate algorithm improvement. The reference document is `docs/sc2-physics-constants.md` — read it first, use it throughout, and revise it as you go.

**Key changes landed this session:**

1. **Mineral patch geometry analysis (#745):** `MineralPatchGeometryDiagnosticTest` runs all 118 oracle replays twice (replay vs seeded positions). Surprising result: **seeded positions produce 5-12pp BETTER mineral rate MAPE** because the EM's mining physics were calibrated against uniform distances and don't generalise to distance variance (σ=0.53 close, σ=0.45 far). Benchmark written to `docs/benchmarks/mineral-patch-geometry.md`.

2. **Worker startup timing (#746):** `WorkerStartupTimingDiagnosticTest` measures per-worker first delivery timing. Controlled scenario: EM stagger path delivers 1 tick early vs SC2 (loop 117 vs ~132). Standalone oracle: first delivery at loop 134-205 (avg 167). Key finding: EM mineral balance is significantly lower than GT from tick 0 onwards due to cost deduction for tracker-driven initial workers (TrackerIntentExtractor converts UnitBorn→TrainIntent, which deducts minerals for free starting workers).

3. **240-loop sliding window rate algorithm (#676):** Replaced EMA (α=0.25/0.35) with 240-loop sliding window in EconomyTracker, matching SC2's discovered rate quantisation (28 minerals/min per delivery). MAPE improvements: mineralsCollectionRate 1-min 35.6%→21.6% (-14pp), vespeneCollectionRate 3-min 29.5%→23.1% (-6.4pp). The physics improvements since the previous failed attempt (#656, #661, #707, #733) tightened delivery timing enough that the window now outperforms EMA.

4. **Balance drift validation (#747):** Rate algorithm change has zero effect on balance MAPE (mineralsCurrent/vespeneCurrent unchanged). Rate and balance are independent metrics — rate is a reporting metric computed from income history, balance is actual state from income minus spending. The 84.9% mineralsCurrent MAPE is driven by income accuracy and spending accuracy, not rate reporting.

**Economy accuracy state (standalone baseline, 2026-11-22, post-sliding-window):**

| Field | MAPE | Floor | Status |
|-------|------|-------|--------|
| mineralsCollectionRate | 38.2% (7-min) | ~10-12% | Improved from 38.6% |
| vespeneCollectionRate | 18.7% (7-min) | ~12-15% | Improved from 20.5% |
| mineralsCurrent | 84.9% (7-min) | ~40-50% | Unchanged |
| vespeneCurrent | 61.1% (7-min) | ~35-45% | Unchanged |

---

## Completed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #745 | Per-replay mineral patch geometry vs EM distances | Done — replay positions already wired; seeded positions are BETTER (5-12pp) |
| #746 | Worker startup/transfer timing standalone | Done — startup timing is minor; initial worker cost deduction is the early-tick issue |
| #676 | Reverse-engineer SC2 rate computation algorithm | Done — 240-loop sliding window replaces EMA; MAPE improved 1-14pp |
| #747 | Balance drift MAPE validation after rate fixes | Done — balance unchanged (rate and balance are independent metrics) |

---

## Queue (next items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#666** | L/High | Forward simulation economy mode |
| 2 | #748 | XS/Low | Remove deprecated buildingRadius() |
| 3 | #749 | XS/Low | ApproachDockingSweepTest stale ranges |
| 4 | #750 | XS/Low | Verify NYDUS_CANAL footprint |

`work continue` picks up #666 — a large feature for running the EM without replay inputs.
