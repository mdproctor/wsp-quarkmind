# HANDOFF — quarkmind

## Last Session

Gas harvest tick quantisation fix on Epic #534. Evolved from carry-forward hack to absolute loop timing — the right abstraction.

### Branch: issue-571-sub-tick-gas-harvest (7 commits, not yet landed)

**Ready to land.** Squash to one commit, rebase-merge to main, close #571 and #573.

### What was done

- **#571 — Sub-tick gas harvest.** Gas workers harvest for 31.696 loops but the 22-loop tick granularity rounded to 44 loops (38% overhead). Three approaches tried:
  1. Carry-forward at entry/exit/base transitions — worked but was a hack (patching a countdown model that shouldn't need patching)
  2. Carry-forward with full-tick exit bonus — calibration showed the bonus was compensating for missing movement physics (acceleration, priority pathing)
  3. **Absolute loop timing (final)** — replaced `cycleLoopsRemaining -= LOOPS_PER_TICK` with `phaseEndLoop = arrivalLoop + 31.696`. Same pattern as TrainIntent/BuildIntent/MorphIntent. No carry-forward, no (int) casts, no tick-boundary awareness.
  - vespeneCollectionRate MAPE: **22.8% → 19.6%** (7-min)

- **#573 — Fractional gas harvest duration (closed).** Changed `cycleLoopsRemaining` (now `phaseEndLoop`) from `int` to `double`. Harvest constant set to 31.696 (1.415s × 22.4 loops/sec, confirmed by Liquipedia). Eliminated 0.9% per-cycle rounding drift.

- **Blog:** "The Tick Boundary Tax" — revised to tell the carry-forward → absolute timing story. SVG diagram updated. In workspace blog dir.

### Key finding

Calibration at 31.696 loops gives ratio 0.90 (EM/GT). The 10% gap is the movement model — constant-speed linear movement vs SC2's acceleration/priority pathing near resource structures. Future work target.

### Next session

1. **Land the branch** — `work end` on issue-571-sub-tick-gas-harvest
2. **Continue epic #534 queue** — 6 open issues remaining (#565–#570 plus #573 done)
3. **Movement model** — the 10% gas ratio gap points here; may warrant a new issue

### Queue state

```
- [ ] #565 — DivergenceRegressionTest building accuracy regression
- [ ] #566 — Mineral mining efficiency factor
- [ ] #567 — Mineral patch distance calibration
- [ ] #568 — Gas income per-worker rate calibration
- [ ] #569 — 1-minute gas checkpoint delayed assignment
- [ ] #570 — Economy structural floor analysis
- [x] #573 — Fractional gas harvest duration (done, on branch)
```
