# HANDOFF — quarkmind

## Last Session

Closed 12 issues. 12w dual-run mineral accuracy improved from 0.7% to 0.1% (5 minerals over 305 ticks). Spending alignment, supply fixes, gas worker recalibration, patch affinity, and initial walk delay.

### What changed

1. **#579 (closed)** — Snapshot-aligned spending. `EconomyTracker` buffers spending, flushes at 160-loop boundaries. Technology minerals: 16.7% → 15.4%. Army minerals: 6.9% → 4.3%.

2. **#580 (closed)** — Zergling twin-spawn supply bug. `trainCount` not factored at three checkpoints. Also found `drainBuildingQueues` per-unit gate (#640).

3. **#615 (closed)** — Gas worker allocation. `GAS_PER_WORKER_PER_MIN` 54→38. `gasEfficiencyOverride=1.20` removed. Gas sync skips when GT rate is 0.

4. **#620 (routing)** — Distance-first routing rejected (saturation regression). Patch affinity implemented — `returnToPatchOrReassign()`. Routing is NOT the cadence bottleneck.

5. **#619 (closed)** — Initial walk delay. `INITIAL_WALK_DELAY_LOOPS=40`. 12w accuracy: 0.7% → **0.1%**.

6. **#614, #637, #638, #640, #581, #582** — Expansion transfer analysis, patch pairing diagnostic, mineralWorkersPerBase() API, supply audit, spending floors.

### Accuracy

| Metric | Value |
|--------|-------|
| 12w mineral total | +0.1% (3465 vs 3460) |
| Per-tick mismatch | 246/304 (80.9%) |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active, 43 commits | #578...#640 |

---

## Queue

```
Active: #639 — vespeneCollectionRate structural floor
[ ] #639 — vespeneCollectionRate structural floor (S/Med)
[ ] #616 — SC2 morph via ability API (needs SC2)
[ ] #634 — Worker cycle variance gap (S/Low)
[ ] #636 — Variable approach docking (S/Med)
[ ] #631 — Edge-to-edge targets not wired (S/Low)
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High)
```

### Key findings

1. **Routing is NOT the cadence bottleneck** — all routing models produce the same mismatch
2. **Gas rate MAPE insensitive to worker count** — 35% floor from concurrent geyser access model
3. **SC2 workers deliver in groups** — pattern (2, 3, 4, 3) at ticks 6-9
