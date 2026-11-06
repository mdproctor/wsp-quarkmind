# HANDOFF — quarkmind

## Current Work (2026-11-06)

Branch `issue-534-emulated-game-100-accuracy` — Epic #534: EmulatedGame 100% accuracy.

### Completed This Session

**#535 — Building accuracy: CC/Hatchery expansion undercount** (CLOSED)
- GT didn't handle UnitTypeChange → CC 15.7%→100%, Hatchery 64%→100%

**#536 — Technology spending: defer to completion** (CLOSED)
- recordBuildSpending at order time → 1-min anomaly eliminated, tech MAPE 46%→16.8%

**#537 — Rate reporting: simulated income** (CLOSED)
- Formula income → simulated income for rate reporting; mineral rate 21.5%→10.1%

**#540 — Economy spending undercount** (CLOSED)
- Starting buildings (BuildingBornIntent) never recorded spending → economy MAPE 27%→1.6%

**#544, #542, #543, #541** — All closed (negligible/noise/combat-model scope)

### Current Accuracy (post-session)

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| Units | 100% | 98.5% | 98.5% | 98.0% |
| Buildings | 100% | 98.8% | ~97% | 94.7% |
| Upgrades | 100% | 100% | 100% | 100% |
| foodMade | 1.0% | 1.5% | 2.5% | 3.1% |
| foodUsed | 4.1% | 3.4% | 6.0% | 4.0% |
| workersActiveCount | 3.8% | 3.3% | 3.8% | 3.2% |
| mineralsCollectionRate | 10.1% | 14.8% | 16.2% | 17.4% |
| vespeneCollectionRate | 181.6% | 34.5% | 37.0% | 48.3% |
| mineralsCurrent | 55% | 58% | 86% | 101% |
| vespeneCurrent | 245% | 47% | 68% | 105% |
| mineralsUsedCurrentArmy | 0% | 5.2% | 9.9% | 7.9% |
| mineralsUsedCurrentEconomy | 1.6% | 5.2% | 6.0% | 5.8% |
| mineralsUsedCurrentTechnology | 0% | 16.8% | 24.8% | 30.7% |
| vespeneUsedCurrentArmy | 0% | 4.5% | 19.7% | 10.9% |
| vespeneUsedCurrentTechnology | 0% | 46.1% | 47.8% | 46.6% |

### Queue — Epic #534

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #535 | Building accuracy — CC/Hatchery expansion undercount | M | DONE |
| #536 | Technology spending — defer recording to completion | M | DONE |
| #537 | Rate reporting — simulated income instead of formula | S | DONE |
| #540 | Economy spending undercount | S | DONE |
| #544 | Vespene economy spending — always zero | XS | CLOSED (negligible) |
| #541 | Army vespene spending — death timing variance | S | CLOSED (timing noise) |
| #542 | SCV unit count undercount | S | CLOSED (combat model) |
| #543 | foodUsed overcount | XS | CLOSED (timing noise) |
| #538 | Mineral balance — per-replay map geometry variance | M | Open |
| #539 | Vespene balance — income deficit compounds | M | Open |
| #545 | Building morph completion timing — OC/Lair undercount | S | Open |

### Remaining Gaps (largest to smallest)

1. **vespeneCollectionRate** 34-48% MAPE — gas income simulation deficit (honest after #537)
2. **mineralsCurrent/vespeneCurrent** 47-105% MAPE — balance drift from cumulative income/spending errors
3. **vespeneUsedCurrentTechnology** 46-47% MAPE — EM overcounts tech spending (174k vs 149k at 7-min)
4. **mineralsUsedCurrentTechnology** 17-31% MAPE — EM overcounts tech spending
5. **mineralsCollectionRate** 10-17% MAPE — mineral income simulation gap
6. **Building accuracy** 94.7% at 7-min — OC (97.9%) and Lair (85.4%) morph timing (#545)

### Key Files

| File | What it does |
|------|-------------|
| `quarkmind-sc2/.../mock/ReplaySimulatedGame.java` | GT — handles UnitTypeChange |
| `quarkmind-sc2/.../mock/SimulatedGame.java` | Base class |
| `quarkmind-sc2/.../emulated/EmulatedGame.java` | Emulator — spending deferred, simulated rates |
| `quarkmind-sc2/.../emulated/EconomyTracker.java` | Spending classification + rate reporting |
| `quarkmind-sc2/.../replay/TrackerIntentExtractor.java` | Replay→intent pipeline |
| `quarkmind-sc2/.../replay/StandaloneEmulatedGameBaselineTest.java` | Validation gate |
| `docs/benchmarks/standalone-emulated-game-baseline.md` | Current accuracy report |

### Next Session Recommendation

The remaining 3 open issues (#538, #539, #545) are M/S scale. The biggest accuracy gaps are vespene income simulation and balance drift — these are deep simulation issues (gas worker trip times, map geometry) rather than bug fixes. Consider whether deeper simulation work is worthwhile vs other epic priorities.

## Known Issues

- **DroolsDominanceWeightStrategyTest** — pre-existing ERROR
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid
