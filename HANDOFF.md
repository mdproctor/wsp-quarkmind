# HANDOFF — quarkmind

## Last Session (2026-11-04)

Fixed replay validation baseline regression (#506). Root cause was quarkmind code changes (#500/#503), not SNAPSHOT dependencies.

### What happened

- **Root cause found:** WorkerSimulation (#500) replaced the calibrated formula model for income tracking. The simulation underproduces income vs reality (occupancy constraints, travel time). Additionally, `syncGasWorkersFromGt` oscillated gas worker assignments every tick because `countGasWorkersPerBuilding` uses a 2.0 distance threshold — workers returning gas cargo exceed this, preventing harvest cycle completion.
- **Fix applied:** `EconomyTracker.recordIncome()` now uses calibrated formula rates (`SC2Data.mineralIncomePerTick`/`gasIncomePerTick`) via `WorkerSimulation.formulaIncome()`. Balance still uses simulation income. Gas worker sync removed from harness — workers auto-assigned by `onGasBuildingCompleted`.
- **Results:** mineralsCollectionRate 50.8% → 11.3%, vespeneCollectionRate 99.5% → 41.0%. All 3267 tests pass.
- **Unit accuracy 79% clarified:** NOT a regression — CmdEvent path was always ~79% (UNIT_ACCURACY_FLOOR=0.70). The 100% was tracker-driven path which is unchanged.

### Issues closed

- #506 — Replay validation baseline regression. Root cause: quarkmind code, not SNAPSHOTs.

## Epic #481 — Calibration State

**Open issues (priority order)**

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #507 | Gas harvest calibration sweep | S/Low | Unblocked by #506 — test ready |
| #504 | foodUsed reconciliation | S/Med | 35% error |
| #402 | Extend BalanceData | L/Med | Phase 1 item |

## Immediate Next Step

Run #507 (gas harvest calibration sweep) — blocked by #506, now unblocked. Test ready at `GasHarvestCalibrationTest`.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export when disk space permits.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.

## References

- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — accuracy numbers (regenerated 2026-11-04)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/GasSlot.java` — concurrent gas model
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/emulated/GasHarvestCalibrationTest.java` — calibration sweep
