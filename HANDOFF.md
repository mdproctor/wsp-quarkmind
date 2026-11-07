# HANDOFF — quarkmind

## Last Session

Closed #550 (add-on cost over-deduction) on Epic #534. Added `AddonKind` enum to `BuildIntent` so TrackerIntentExtractor correctly charges Reactor (50m/50g) and TechLab (50m/25g) instead of parent building cost. Technology spending MAPE roughly halved at all checkpoints.

**Units, Buildings, Upgrades all 100%.** Remaining gaps are economy research only.

## Remaining Queue (3 items)

| # | Title | Scale | Complexity | Impact |
|---|-------|-------|------------|--------|
| #551 | Tech spending structural gap | M | High | Research: tech spending 9-17% MAPE (post-#550) |
| #538 | Mineral balance — map geometry variance | M | High | Research: mineralsCurrent 54-87% MAPE |
| #539 | Vespene balance — income deficit + spending | M | Med | Research: vespeneCurrent 46-101% MAPE |

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
