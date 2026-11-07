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

mineralsCurrent gap is spending-driven, not income-driven. EM income is within 10% of GT, but spending is systematically lower because spending categories are recorded at completion time while GT records at commitment. However, fixing this timing directly disrupts the death-subtraction accounting and makes things worse. #555 (InProgress tracker) proposes a clean solution: separate InProgress accumulator alongside Current.

## Remaining Queue (10 items, priority ordered)

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| #552 | Floor mineral balance at zero | XS | Low | — |
| #558 | Supply timing — foodUsed | XS | Low | — |
| #553 | Per-replay economy divergence diagnostic | S | Med | — |
| #538 | Mineral balance — per-replay variance | M | High | #553 |
| #539 | Vespene balance — income deficit | M | Med | #553 |
| #556 | Gas geyser distance calibration | S | Med | #553 |
| #554 | Worker transfer to expansions | M | High | #553 |
| #557 | Mineral collection rate — per-base calibration | S | Med | #553, #554 |
| #555 | InProgress spending tracker | M | High | #553 |
| #559 | Spending category death-timing variance | S | Med | #553, #555 |

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
