# HANDOFF — quarkmind

## Orientation for Next Session

Cycle distribution fixed (#656). The EM's mineral cycle σ was 0.92 (SC2: 0.56) because a flat `approach_docking_loops=24.5` treated all patches identically. First-principles analysis revealed SC2's cycle variance is entirely structural — per-patch, deterministic, bounded — not stochastic (collision-induced random noise).

Fix: per-patch approach docking from E2E distance (40% equalization scale) + contention-aware collision scaling (25% dodge at ≤2 workers/patch). The distribution now matches SC2 within 1% at every tick bucket.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#656:**

| Metric | SC2 | Before #656 | After #656 |
|--------|-----|-------------|------------|
| σ | 0.56 | 0.92 | **0.57** |
| Mean cycle | 5.28 | 5.39 | **5.29** |
| 7+ tick | 0% | 6.6% | **0.6%** |
| 4-tick % | 5.7% | 8.9% | **5.3%** |
| 5-tick % | 60.7% | 54.4% | **61.1%** |
| 6-tick % | 33.6% | 30.1% | **33.1%** |
| 12w income | 3460 | 3395 (−1.9%) | within ±3% |
| Full test suite | — | 3365 pass | 3367 pass |

**Key constants (new):**
- `MineralPatch.APPROACH_EQUALIZATION_SCALE` = 0.4 — per-patch approach compensates 40% of walk distance differences
- `WorkerSimulation.LOW_SATURATION_DODGE_SCALE` = 0.25 — collision dodge reduction at ≤2 workers/patch
- Both in `SimulatedWorker.resetToDefaults()` but NOT yet in `WorkerPhysicsConfig` (#681)

**Design insight:** SC2's cycle variance is a mixture of point masses at tick buckets 4, 5, 6 — each worker sits in one bucket consistently. No tuning of collision noise can produce this bounded variance. Collision adds unbounded random tails via contention cascade (3-loop displacement → 46-loop contention wait). The right variance source is per-patch approach equalization, not collision displacement.

---

## Filed Issues

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #679 | Per-patch walk distance extension | M/Med | Model SC2 path geometry directly — WalkTo/WalkBack asymmetry |
| #680 | Eliminate residual 0.6% 7+ tick outliers | S/Med | Contention cascade from dodge=0.25 — 4 cycles per 665 |
| #681 | Add equalization/dodge to WorkerPhysicsConfig | XS/Low | Config properties for Quarkus override |

---

## Next Queue Items

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #677 | Gas approach docking calibration test | XS/Low | GAS_APPROACH_DOCKING still at 11.5 — needs 8w-style cadence measurement |
| #676 | Reverse-engineer SC2 rate computation | M/High | 240-loop sliding window; needs tighter physics first |
| #672 | Gas walk arrival-threshold compensation | S/Med | Remaining walk shortening (~9 loops/leg vs SC2) |
| #673 | Dual-run 6w gas surplus two-geyser asymmetry | S/Low | +6.7% surplus at 6w, two geysers |

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment — now closer after #661/#656.

---

## Branch State

Branch `issue-656-mineral-line-pathfinding` — **ready for work-end**.

Key commits:
1. `wip(#656): add CycleOutlierDiagnosticTest — traces 7+ tick cycle root causes`
2. `wip(#656): add PatchCycleCorrelationTest — maps workers to patch positions`
3. `wip(#656): verify patch assignment distribution is balanced`
4. `feat(#656): per-patch approach docking from E2E distance`
5. `feat(#656): scale collision dodge by patch contention`
6. `feat(#656): calibrate approach equalization scale to 0.4`
7. `feat(#656): extract LOW_SATURATION_DODGE_SCALE + sweep diagnostic`
8. `fix(#656): code review fixes — reset chain + unmodifiable bases accessor`
9. `docs(#656): update physics constants`

Issues to close: **#656** (done).

---

## Last Session

**Work completed:**
- #656: Cycle distribution matching — σ reduced from 0.92 to 0.57 (SC2: 0.56)
- Root cause investigation: 43 outlier cycles traced to contention cascade + flat approach
- False lead eliminated: worker distribution verified balanced (position-matching artifact)
- First-principles analysis: SC2 variance is structural (per-patch), not stochastic (collision)
- Per-patch approach docking with 40% equalization scale
- Contention-aware collision scaling (25% at ≤2 workers/patch)
- Equalization sweep found 0.4 as SC2-matching value
- Dodge sweep confirmed 0.25 is load-bearing for cycle time
- Blog: "Right Total, Wrong Shape"
- Filed #679, #680, #681 for follow-up work
