# HANDOFF — quarkmind

## Last Session

Two economy accuracy investigations on Epic #534, both closing without code changes. The key finding: **the mineral mining model is at its accuracy ceiling (10–14% MAPE), and the supply timing model was already correct**. Both issues revealed that the existing model was more accurate than the hypothesis.

### Completed

- **#557 — Mineral collection rate per-base calibration (closed, investigation only).** Five approaches tried to reduce mineralsCollectionRate MAPE from 10–14%:
  1. Formula-based rates (like gas) — 15–25% MAPE, worse at every checkpoint
  2. Fractional trip timing (distance-based) — 15–46% MAPE, raw physics rate too fast
  3. Fractional + calibration factor — no single factor works across checkpoints (direction reversal: over at 1-min, under at 3-7 min)
  4. Convoy fix + contention model — 12–37% MAPE, contention penalty too aggressive
  5. Convoy fix without contention — 7.4% at 1-min (best) but 25.8% at 3-min (worst)

- **#558 — Supply timing foodUsed (closed as invalid).** Hypothesis: supply counted at completion, not training start. Tested: deferred `addSupplyUsed` to completion callback. Result: MAPE worsened (2.5% → 8.3% at 3-min). SC2 counts supply at training start (reservation). The original emulator semantics were correct.

### Key findings — load-bearing bug and accuracy ceiling

**The convoy bug.** After depositing minerals, `assignToNearestPatch` checks `miners.size()` to find available patches. But workers are removed from `miners` on harvest and only re-added on arrival. Between harvest and arrival, all patches show zero miners. All 16 workers converge on Patch 0 (first in the list). Only 2 can mine at a time; 14 wait idle.

**Why it's load-bearing.** The convoy suppresses income at 3–7 min checkpoints by keeping workers idle. This accidentally compensates for the mining rate being slightly faster than real SC2 (due to missing worker acceleration, pathfinding overhead, and mining animation details). Fixing the convoy exposes the over-production: 1-min improves (10.2% → 7.4%) but 3-min degrades (14.1% → 25.8%). No single change can fix both directions.

**Root causes of the remaining 10–14% MAPE** (outside the mining model):
1. Gas transition timing — EM instantly transfers workers to gas on building completion; real SC2 has a walk delay (workers earn neither mineral nor gas income during the transition)
2. Worker count drift — EM has 1.6–3.8% more workers than GT at all checkpoints
3. SC2 rate reporting cadence — PlayerStatsEvent `mineralsCollectionRate` uses internal smoothing that differs from our 7-tick rolling window

### What NOT to try

- **Global MINING_DURATION_LOOPS increase** — tested previously, over-corrects. Dwell=1 is the right average.
- **Formula-based mineral income** — tested this session. The calibrated rates (77.7/38.8/7.8 per tier) are tuned for the formula model, not the EmulatedGame context. Applying them to the EM produces 15–25% MAPE.
- **Fractional trip timing** — tested this session. Raw physics trip (~3.8 ticks at d=6.0) is shorter than the real SC2 trip (~5 ticks effective, including pathing/acceleration/animation overhead). No good calibration point exists because the error is distance-dependent.
- **Supply accounting at completion** — tested this session. SC2 reserves supply at training start. The emulator was already correct.
- **Any uniform rate multiplier on mineral income** — tested via calibration factors. Direction reverses between checkpoints (over at 1-min, under at 3-7 min), so no single scalar works.

### Blog entry

- `docs/blog/2026-11-07-mdp01-the-bug-that-held-the-roof-up.md` — covers both investigations, the convoy bug discovery, and the broader lesson about load-bearing accuracy in simulation systems. Two SVG diagrams: MAPE comparison and convoy mechanism.

## Current Accuracy (standalone, 118 oracle replays, all GT syncs disabled)

| Category | 1 min | 3 min | 5 min | 7 min |
|----------|-------|-------|-------|-------|
| Units | 100% | 100% | 100% | 100% |
| Buildings | 100% | 100% | 100% | 100% |
| Upgrades | 100% | 100% | 100% | 100% |
| mineralsCurrent MAPE | 51% | 64% | 78% | 76% |
| vespeneCurrent MAPE | 270% | 38% | 54% | 69% |
| mineralsCollectionRate MAPE | 10% | 14% | 14% | 14% |
| vespeneCollectionRate MAPE | 154% | 25% | 22% | 25% |
| foodMade / foodUsed MAPE | 1-4% | 2-3% | 3-6% | 3-4% |
| workersActiveCount MAPE | 4% | 2% | 3% | 3% |

## Remaining Queue

### Highest impact (from diagnostic findings)

| # | Title | Scale | Complexity | Status |
|---|-------|-------|-----------|--------|
| #563 | Gas income formula recalibration | S | Med | Open — remaining 25% vespene MAPE at 5-min |
| #562 | Gas income bidirectional variance | S | Med | Open — 78% gas over-collection |
| #561 | Spending under-deduction | M | Med | Open — secondary balance driver (209 mean error) |
| #560 | Protoss economy accuracy | M | High | Open — race-specific 2-3x worse |
| #559 | Spending category death-timing | S | Med | Open — army/tech vespene gaps |

### Closed this session

| # | Title | Outcome |
|---|-------|---------|
| #557 | Mineral collection rate per-base | Closed — at accuracy ceiling, investigation complete |
| #558 | Supply timing foodUsed | Closed as invalid — hypothesis incorrect |

### Suggested approach order (next session)

1. **#563 (gas income recalibration)** — S/Med, directly in the economy pipeline. The gas formula rates were calibrated from the same 118 replays; the 25% MAPE at 5-min may have a similar investigation pattern to #557 but gas has fewer confounding factors (formula-driven, no walk simulation).
2. **#559 (spending death-timing)** — S/Med, addresses 17–42% tech spending MAPE. Army and tech vespene gaps from death-timing variance.
3. **#561 (spending under-deduction)** — M/Med, systematic. 97/168 replays under-spend.

### Branches to clean up

- `issue-557-mineral-collection-rate-calibration` — empty branch (no code changes landed), can be deleted
- `issue-558-supply-timing-foodused` — contains revert commit only, can be deleted

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy numbers
- `EconomyDivergenceDiagnosticTest` — run with `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=EconomyDivergenceDiagnosticTest`
- `StandaloneEmulatedGameBaselineTest` — run with `mvn test -pl quarkmind-sc2 -Preport -Dtest=StandaloneEmulatedGameBaselineTest`
