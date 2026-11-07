# HANDOFF — quarkmind

## Last Session

Diagnostic investigation on Epic #534 (EmulatedGame 100% accuracy). Three issues closed, three filed. The key finding reverses the previous understanding: **income error is the primary driver of balance variance (2x spending error), not spending**.

### Completed

- **#553 — Per-replay economy divergence diagnostic.** `EconomyDivergenceDiagnosticTest` runs StandaloneEmulatedGame against 118 oracle replays. Top-20 worst replays per field, per-race breakdown, error direction. Found Protoss is 2-3x worse than other races.
- **#538 — Mineral balance map geometry variance.** Added income vs spending decomposition to diagnostic. Mineral income error (mean 396) is 2x spending error (mean 209). Tested `MINING_DURATION_LOOPS=23` (dwell=2 ticks) — over-corrected, direction swung from 91/76 over to 3/163 under, Zerg MAPE doubled. Reverted. Conclusion: dwell=1 is correct on average; per-replay variance is from worker timing, not map geometry.
- **#539 — Vespene balance income deficit.** Fixes already landed (previous session). Remaining variance tracked by #556 and #562.

### Key finding — income vs spending decomposition

Previous understanding: "EM over-estimates balances because spending deductions are incomplete."
Updated understanding: "Income divergence is the primary driver. The mining model is correct on average but per-replay variance comes from worker production timing and task transitions."

| Component (5-min) | Mean Error | Direction |
|---|---|---|
| Mineral gross income | 396 | Balanced (91 over / 76 under) |
| Mineral spending | 209 | Under-spends (63 over / 97 under) |
| Vespene gross income | 175 | Over-collects (131 over / 37 under) |
| Vespene spending | 74 | Over-spends (103 over / 12 under) |

Protoss MAPE degrades from 100% to 141% between 5-min and 7-min while Terran/Zerg stay flat.

### New issues filed

- **#560 — Protoss economy accuracy (M/High)** — 2-3x worse than other races, degrades over time. Hypotheses: Chronoboost, warp-in timing, Protoss tech tree depth.
- **#561 — Spending under-deduction (M/Med)** — Secondary driver (209 mean error). 97/168 replays under-spend. Broader than #555's timing focus.
- **#562 — Gas income bidirectional variance (S/Med)** — 78% of replays over-collect gas. EM auto-assigns 3 workers on gas building completion; GT players may delay.

### What NOT to try

- Global `MINING_DURATION_LOOPS` increase — tested, over-corrects. Dwell=1 is the right average; variance needs per-base or sub-tick approaches.
- Formula gas income for balance — tried in previous session, caused Hatchery regression.

## Current Accuracy (standalone, 118 oracle replays, all GT syncs disabled)

| Category | 1 min | 3 min | 5 min | 7 min |
|----------|-------|-------|-------|-------|
| Units | 100% | 100% | 100% | 100% |
| Buildings | 100% | 100% | 100% | 100% |
| Upgrades | 100% | 100% | 100% | 100% |
| mineralsCurrent MAPE | 51% | 62% | 73% | 75% |
| vespeneCurrent MAPE | 195% | 39% | 55% | 66% |
| Collection rates MAPE | 10-114% | 13-30% | 16-31% | 17-38% |
| Spending MAPE | 0-2% | 4-19% | 5-38% | 6-42% |
| foodMade / foodUsed | 1-4% | 2-3% | 3-6% | 3-4% |
| workersActiveCount | 4% | 2% | 3% | 3% |

## Remaining Queue (10 items)

### Highest impact (from diagnostic findings)

| # | Title | Scale | What it addresses |
|---|-------|-------|-------------------|
| #562 | Gas income bidirectional variance | S/Med | 78% gas over-collection — clearest signal |
| #561 | Spending under-deduction | M/Med | Secondary balance driver (209 mean error) |
| #560 | Protoss economy accuracy | M/High | Race-specific 2-3x worse — Chronoboost/warp-in |

### Remaining original queue

| # | Title | Scale | What it addresses |
|---|-------|-------|-------------------|
| #556 | Gas geyser distance calibration | S/Med | Per-map gas walk time |
| #554 | Worker transfer to expansions | M/High | Income timing at expansion bases |
| #555 | InProgress spending tracker | M/High | Spending commitment vs completion timing |
| #557 | Mineral collection rate per-base | S/Med | Per-base mining calibration |
| #558 | Supply timing foodUsed | XS/Low | foodUsed 4-6% MAPE |
| #559 | Spending category death-timing | S/Med | 17-42% tech spending MAPE |

### Suggested approach order

1. #562 (gas over-collection) — quickest win, clearest signal
2. #555 (InProgress spending) — addresses the spending under-deduction structurally
3. #560 (Protoss) — needs investigation to identify which Protoss mechanics diverge
4. #554 (worker transfer) — could reduce mineral income variance across multi-base replays

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy numbers
- `EconomyDivergenceDiagnosticTest` — run with `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=EconomyDivergenceDiagnosticTest`
- `StandaloneEmulatedGameBaselineTest` — run with `mvn test -pl quarkmind-sc2 -Preport -Dtest=StandaloneEmulatedGameBaselineTest`
