# HANDOFF — quarkmind

## Last Session

Closed #545 (OC/Lair morph timing), #546 (tech spending ghost completion), and #547 (Factory/Starport add-on swap). Three distinct fixes:
1. Ghost completion: added entityTag to PendingCompletion, cancel on building death
2. Add-on swaps: added ADDON_SWAP_TARGETS for instant type changes on add-on morphs
3. Building morphs: tracker-driven morphs now instant type swaps (matching GT); pre-pass uses per-tag Deque for multiple UnitDone events

Key accuracy improvements: OC 84.9→100%, Lair 22.2→100%, Factory/Starport/Barracks 86-91→100%, units 98.5→100% at 3-min. Building totals 94.7→95.8% at 7-min.

## Remaining Queue (4 items)

| # | Title | Scale | Complexity | Impact |
|---|-------|-------|------------|--------|
| #548 | Creep Tumor undercount | XS | Low | 0-57% accuracy |
| #549 | Unit mode tracking (SIEGE_TANK_SIEGED etc.) | S | Med | 0% for mode types |
| #538 | Mineral balance — map geometry variance | M | High | Research |
| #539 | Vespene balance — income deficit + spending | M | Med | Research |

#548 and #549 are the remaining fixable building/unit accuracy gaps. #538 and #539 are research items.

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
