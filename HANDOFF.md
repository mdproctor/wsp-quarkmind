# HANDOFF — quarkmind

## Last Session (2026-11-04)

Fixed replay validation baseline regression (#506) and gas rate calibration (#507). Two issues closed, all economy metrics now within tight regression ceilings.

### What happened

- **#506 — Baseline regression fixed:** WorkerSimulation (#500) replaced the calibrated formula model for income tracking. The simulation underproduces vs reality. Additionally, `syncGasWorkersFromGt` oscillated gas worker assignments due to a 2.0 distance threshold. Fix: EconomyTracker uses calibrated formula rates for rate computation; simulation income still drives balance.
- **#507 — Gas rate calibration:** The calibration sweep was ineffective (formula rates are independent of `GAS_HARVEST_DURATION_LOOPS`). Found WorkerSimulation had duplicate GasSlots (intent-created + GT-injected buildings). Fix: harness syncs gas rate from GT gas building count × 3 workers via calibrated formula. Position-based dedup added to `onGasBuildingCompleted`.
- **Results at 5-min:** mineralsCollectionRate 10.4% error (was 50.8%), vespeneCollectionRate 1.5% error (was 99.5%). Regression ceilings tightened: minerals 16%, vespene 7%.
- **Blog written:** "The Model You Trust Is the One You Calibrate" — on why calibration matters more than model complexity.

### Issues closed

- #506 — Replay validation baseline regression
- #507 — Gas rate calibration (approach changed from harvest duration to GT-synced formula)

## Epic #481 — Calibration State

**Open issues (priority order)**

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #504 | foodUsed reconciliation | S/Med | 20.7% error |
| #402 | Extend BalanceData | L/Med | Phase 1 item |

## Immediate Next Step

#504 (foodUsed reconciliation) — 20.7% error, S/Med. Or pivot to other epic work.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export when disk space permits.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.

## References

- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — accuracy numbers (regenerated 2026-11-04, vespene 1.5%)
- `blog/2026-11-04-mdp01-the-model-you-trust.md` — blog on calibration vs complexity
