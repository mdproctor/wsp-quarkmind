# HANDOFF — quarkmind

## Last Session

Improved vespene balance (#539) on Epic #534. Two fixes to gas income model:

- **Gas worker death replacement:** Activated dead code `rebalanceGasWorkers()` — when a gas worker dies, a mineral worker replaces it (respects MIN_MINERAL_WORKERS_BEFORE_GAS=8 threshold). `WorkerSimulation.onWorkerDied()`.
- **Gas harvest-to-walk transition:** After harvest completes inside geyser, worker immediately starts walking instead of wasting a full tick. Reduces gas trip from 6 to 5 ticks. `WorkerSimulation.tickGasWorker()`.

**Results:** vespeneCurrent 7-min MAPE: 101.3% → 77.3%. vespeneCollectionRate 7-min: 48.8% → 38.2%. EM/GT gas aggregate at 7-min: 0.22x → 1.01x (matched).

**Units, Buildings, Upgrades all 100%.** Remaining gaps: mineral/vespene current balance (per-replay variance).

### Approaches tried (didn't help)

- Worker-to-gas on produce: zero measurable impact (on-death rebalance already fills vacancies)
- Spending timing fix (record at intent vs completion): made spending categories worse — death-subtraction accounting mismatch when spending is recorded before building completes

### Key finding for #538

mineralsCurrent gap is spending-driven, not income-driven. EM income is within 10% of GT, but spending is systematically lower because spending categories are recorded at completion time while GT records at commitment. However, fixing this timing directly disrupts the death-subtraction accounting and makes things worse. A different approach is needed — possibly separate InProgress tracking that doesn't interact with the Current/death accounting.

## Remaining Queue (2 items)

| # | Title | Scale | Complexity | Impact |
|---|-------|-------|------------|--------|
| #538 | Mineral balance — map geometry variance | M | High | Research: mineralsCurrent 54-84% MAPE |
| #539 | Vespene balance — income deficit + spending | M | Med | Improved: vespeneCurrent 39-77% MAPE (was 46-101%) |

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
