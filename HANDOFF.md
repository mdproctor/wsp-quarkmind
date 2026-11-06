# HANDOFF — quarkmind

## Last Session

Closed 4 issues on Epic #534: #545 (OC/Lair morph), #546 (ghost completions), #547 (add-on swaps), #548 (Creep Tumor burrow). Filed 2 new issues (#550, #551) from investigation findings.

**Buildings now 100% at ALL checkpoints.** Units 100% at 1-min and 3-min, 99.5% at 5-min, 98.8% at 7-min (remaining gap = mode-tracked units #549).

Key fixes: PendingCompletion entityTag for cancellation, ADDON_SWAP_TARGETS for instant add-on type changes, tracker-driven morphs as instant type swaps, per-tag UnitDone queue in pre-pass, CreepTumorBurrowed morph target.

## Remaining Queue (5 items)

| # | Title | Scale | Complexity | Impact |
|---|-------|-------|------------|--------|
| #549 | Unit mode tracking (SIEGE_TANK_SIEGED etc.) | S | Med | 0% for mode types, 1.2% total unit gap |
| #550 | Add-on cost over-deduction | S | Low | Economy accuracy |
| #538 | Mineral balance — map geometry variance | M | High | Research |
| #539 | Vespene balance — income deficit + spending | M | Med | Research |
| #551 | Tech spending structural gap | M | High | Research |

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
