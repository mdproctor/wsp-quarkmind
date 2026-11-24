# HANDOFF — quarkmind

## Orientation

**Two-base distribution fixed (#762). Real-terrain baselines recorded (#764). A* pathfinding discrepancy discovered (#763).**

The EM over-produces minerals by +6-19% on real TorchesAIE terrain vs ±3% on the old 64×64 flat grid. Same mineral distances (7.08u avg), same walkability (all mineral-line tiles walkable on both grids), same obstacle registration (radius 0.3 circles). The A* pathfinder produces shorter worker paths on the 160×208 grid. This is the blocking issue.

**Start with:** Instrument A* path lengths for a few workers on both grid sizes (64×64 flat vs 160×208 real) with identical relative mineral positions. Compare waypoint count and total path distance. The difference will reveal whether the pathfinder, the obstacle model, or `nearestWalkable()` snapping is causing shorter routes on the real grid.

---

## Key data

| Test | Flat grid | Real terrain | Note |
|------|----------|-------------|------|
| 8w dual-run | +0.4% | **+14.1%** | Largest gap — few workers, short paths |
| 12w dual-run | -0.3% | **+4.8%** | Physics constants calibrated here |
| Pinned single-base 36w | — | **-5.0%** | Contention overtakes path advantage at high saturation |
| Pinned two-base | +80% (old) | **-5.2%** | #762 fixed distribution; per-base SC2 data validates |

**SC2 per-base data confirmed:** SC2 w/base=[24, 0], EM w/base=[30, 0]. All workers at main Nexus. 6 SC2 workers unassigned/in-transit at any moment.

---

## What's ready

- **Worker distribution fix (#762):** `onWorkerProducedAtBase()`, `injectWorkerAtBase()`, `autoTransferEnabled` flag
- **Per-base tracking:** `CalibrationSnapshot.workersPerBase`, `PlaybookSC2Adapter.extractWorkersPerBase()`, per-base columns in `pinnedCalibrate()`
- **13 re-recorded SC2 baselines (#764):** All include per-base `workersPerBase` from SC2 `assignedHarvesters`
- **32w and 40w playbooks (#763):** Both supply-blocked at 36 probes (need longer duration or more pylons)
- **TorchesAIE pathing grid:** 160×208, all mineral-line tiles verified walkable
- **Wait-time absorption finding:** `tryStartMining()` floor of `APPROACH_ANIMATION_LOOPS` makes absorption a no-op. Fixing to `max(0,...)` overshoots by +7-18% at all saturation levels (brief shuffle-order waits trigger reduced approach on every cycle)

---

## Investigation findings (this session)

1. **Two-base +80% root cause:** `onWorkerProduced()` routed to `leastSaturatedBaseIndex()` → [32,32] distribution. SC2 playbook trains all probes at first Nexus → all at main. Fix: `onWorkerProducedAtBase()` + `autoTransferEnabled=false`.

2. **Real terrain reveals A* bias:** Re-recording all 13 baselines from SC2 and running with real TorchesAIE terrain shows +6-19% over-production. The physics constants (approach=20, deposit=12) were effectively calibrated for the flat grid's A* paths.

3. **Wait-time absorption is dead code:** `max(APPROACH_ANIMATION_LOOPS, APPROACH_ANIMATION_LOOPS - waited)` = 20 always. Fixing to `max(0, ...)` overshoots because brief shuffle-order waits (1-2 loops) are common at ALL saturation levels, not just high.

4. **Pinned vs dual-run paradox:** Pinned test shows -5% (under-production at 36w high saturation). Dual-run shows +6-19% (over-production at 8-12w). Both on real terrain. The physics constants are caught between two effects: shorter A* paths (boost income) and high-saturation contention (reduce income). The A* discrepancy must be resolved before the contention model can be evaluated.

---

## Next steps (priority order)

1. **Diagnose A* path length discrepancy** — add logging or a diagnostic test that computes A* paths from base to each mineral patch on BOTH grid sizes (64×64 flat, 160×208 real) with the same relative positions. Report waypoint count and total path distance. This tells us exactly where the paths diverge.

2. **Fix the pathfinder or obstacle model** — once the path length difference is understood, fix it so both grids produce consistent paths. Then re-run the saturation curve.

3. **Re-evaluate saturation curve** — with consistent A* paths, the dual-run results should return to ±3%. Then the pinned test's -5% at 36w isolates the genuine contention model gap.

4. **Wait-time absorption** — needs a different approach than `max(0, ...)`. Either: (a) threshold the "genuine wait" at >5 loops (not brief shuffle-order waits), or (b) accept approach animation as fixed physics.

---

## Notes

- **Branch:** `issue-761-intent-rejection-diagnostic` (stacked: 756→757→758→761→762→763→764)
- **All 50 DualRunCalibrationTest pass** — divergence tests are structural (probe count), not mineral accuracy assertions
- **7/7 CalibrationBaselineTest pass** — including workersPerBase round-trip
- **SC2 is NOT running** — killed after recordings
- **Worktree cleanup:** `/Users/mdproctor/claude/casehub/quarkmind/.claude/worktrees/agent-ace0ecd96715c77a7` can be removed — its baseline commit was cherry-picked into the main branch
