# HANDOFF — quarkmind

## Current Work (2026-11-06)

Branch `issue-534-emulated-game-100-accuracy` — Epic #534: EmulatedGame 100% accuracy.

### Completed This Session

**#535 — Building accuracy: CC/Hatchery expansion undercount** (CLOSED)
- Root cause: `ReplaySimulatedGame` (GT) didn't handle `ID_UNIT_TYPE_CHANGE` — morphed buildings kept original type
- Fix: Added `applyUnitTypeChange()` to `ReplaySimulatedGame`
- CC 7-min: 15.7% → 100%, Hatchery 7-min: 64.0% → 100%

**#536 — Technology spending: defer recording to completion** (CLOSED)
- Root cause: `recordBuildSpending` called at order time, but SC2 moves costs to "Current" only at completion
- Fix: Moved `recordBuildSpending` into PendingCompletion callback
- mineralsUsedCurrentTechnology 1-min: EM=25,100 → EM=0 (anomaly eliminated)
- mineralsUsedCurrentTechnology 3-min: 46% → 16.8% MAPE

### Current Accuracy (post-#535, #536)

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| Units | 100% | 98.5% | 98.5% | 98.0% |
| Buildings | 100% | 98.8% | ~97% | 94.7% |
| Upgrades | 100% | 100% | 100% | 100% |
| foodMade | 1.0% | 1.5% | 2.5% | 3.1% |
| foodUsed | 4.1% | 3.4% | 6.0% | 4.0% |
| workersActiveCount | 3.8% | 3.3% | 3.8% | 3.2% |
| mineralsCollectionRate | 21.5% | 11.9% | 22.8% | 29.3% |
| vespeneCollectionRate | 237% | 26.1% | 26.1% | 38.6% |
| mineralsCurrent | 55% | 58% | 86% | 101% |
| vespeneCurrent | 245% | 47% | 68% | 105% |
| mineralsUsedCurrentArmy | 0% | 5.2% | 9.9% | 7.9% |
| mineralsUsedCurrentEconomy | 27.0% | 22.2% | 16.9% | 14.4% |
| mineralsUsedCurrentTechnology | 0% | 16.8% | 24.8% | 30.7% |
| vespeneUsedCurrentArmy | 0% | 4.5% | 19.7% | 10.9% |
| vespeneUsedCurrentEconomy | 0% | 0% | 100% | 100% |
| vespeneUsedCurrentTechnology | 0% | 46.1% | 47.8% | 46.6% |

### Queue — Epic #534

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #535 | Building accuracy — CC/Hatchery expansion undercount | M | DONE |
| #536 | Technology spending — defer recording to completion | M | DONE |
| #537 | Rate reporting — simulated income instead of formula | S | Next |
| #540 | Economy spending undercount (13-20% MAPE) | S | — |
| #544 | Vespene economy spending — always zero in EM | XS | — |
| #541 | Army vespene spending — death timing variance | S | — |
| #538 | Mineral balance — per-replay map geometry variance | M | Blocked by #537 |
| #539 | Vespene balance — income deficit compounds | M | Blocked by #537, #536 (now unblocked) |
| #542 | SCV unit count undercount (4%) | S | Blocked by #535 (now unblocked) |
| #543 | foodUsed overcount (4-6%) | XS | Blocked by #542 |
| #545 | Building morph completion timing — OC/Lair undercount | S | — |

### Key Files

| File | What it does |
|------|-------------|
| `quarkmind-sc2/.../mock/ReplaySimulatedGame.java` | GT — now handles UnitTypeChange |
| `quarkmind-sc2/.../mock/SimulatedGame.java` | Base class — new replaceBuildingType/replaceUnitType methods |
| `quarkmind-sc2/.../emulated/EmulatedGame.java` | The emulator — spending now deferred to completion |
| `quarkmind-sc2/.../emulated/EconomyTracker.java` | Spending classification + rate reporting |
| `quarkmind-sc2/.../replay/TrackerIntentExtractor.java` | Replay→intent pipeline |
| `quarkmind-sc2/.../replay/StandaloneEmulatedGameBaselineTest.java` | Validation gate |
| `quarkmind-sc2/.../replay/BuildingExpansionDiagnosticTest.java` | Building diagnostic |
| `docs/benchmarks/standalone-emulated-game-baseline.md` | Updated accuracy report |

### Next Session Recommendation

Continue with **#537** (Rate reporting — simulated income). This unblocks #538 and #539 (mineral/vespene balance). Alternatively, **#542** (SCV undercount) is small and now unblocked.

## Known Issues

- **DroolsDominanceWeightStrategyTest** — pre-existing ERROR (not from this work)
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
- Epic #516 (closed) — predecessor epic, all GT syncs eliminated
- Epic #534 — current epic, 100% accuracy target
