# HANDOFF — quarkmind

## Last Session

Closed 2 issues (#590, #591). Rate cadence alignment implemented — loop-based rate windowing in EconomyTracker. Mining efficiency factors recalibrated — factors > 1.0 needed post sub-tick fixes. Filed 3 new issues (#593-#595) for remaining rate accuracy vectors.

### What was done

- **#590 — Rate measurement cadence alignment (closed).** Replaced tick-counting (RATE_INTERVAL_TICKS=7, 154 loops) with loop-counting (rateIntervalLoops=160, fires every 8 ticks = 176 loops). Added EconomyConfig with `starcraft.economy.rate-interval-loops` property. Diagnostic confirmed PlayerStatsEvent cadence: 96.8% exactly 160 loops across 9183 intervals. MAPE comparison showed mixed effect: mineral rate -0.7% improvement, vespene rate +2.4% regression. Cadence alignment is architecturally correct but not the dominant MAPE factor.

- **#591 — Worker income direction analysis (closed).** Extended efficiency factor sweep from 0.85-1.0 to 0.85-1.30. Found model under-produces income post sub-tick — optimal rate factors all above 1.0: Protoss=1.18 (13.1% rate MAPE), Zerg=1.12 (12.2%), Terran=1.28 (18.1%). Applied via RaceModel.miningEfficiencyFactor() in EmulatedGame.tick(). mineralsCollectionRate MAPE improved from 22.1% to 14.5%. mineralsCurrent MAPE worsened (67.5% → 126.1%) — income increased but spending unchanged.

- **New issues filed:**
  - **#593** — SC2 rate algorithm reverse-engineering (blocked by #591)
  - **#594** — Sub-tick rate tracking — exact 160-loop window
  - **#595** — Gas rate regression analysis — 176-loop window impact

---

## Current Accuracy State

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

| Field | Previous Session | This Session | Status |
|---|---|---|---|
| mineralsCollectionRate | 22.8% | **14.5%** | Improved — efficiency factors |
| vespeneCollectionRate | 26.7% | **29.1%** | Regressed — cadence alignment |
| mineralsCurrent | 67.5% | **126.1%** | Worsened — income up, spending unchanged |
| vespeneCurrent | 65.4% | 65.4% | Unchanged |
| foodMade | 3.1% | 3.1% | At floor |
| foodUsed | 3.7% | 3.7% | At floor |
| workersActiveCount | 2.5% | 2.5% | At floor |
| mineralsUsedCurrentArmy | 7.1% | 7.1% | At floor |
| mineralsUsedCurrentEconomy | 5.8% | 5.8% | At floor |
| mineralsUsedCurrentTechnology | 17.0% | 17.0% | Near floor |
| vespeneUsedCurrentArmy | 7.4% | 7.4% | At floor |
| vespeneUsedCurrentTechnology | 42.0% | 42.0% | Structural |

### DivergenceRegressionTest — all green

100% units, buildings, upgrades. All matchup deltas within thresholds. Economy MAPE within 150% ceiling.

---

## Remaining Queue

```
- [ ] #592 — Overhead absorption limit investigation
- [ ] #593 — SC2 rate algorithm reverse-engineering
- [ ] #594 — Sub-tick rate tracking — exact 160-loop window
- [ ] #595 — Gas rate regression analysis — 176-loop window impact
```

Plus standing issues:
- #578 — Rate fields (14.5% MAPE, active calibration)
- #579 — Tech spending timing (42%, structural)
- mineralsCurrent 126.1% — needs spending accuracy or factor refinement

---

## Critical Insights for Next Session

### 1. The Terran factor 1.28 is suspiciously large

Workers producing 28% more per trip implies the effective cycle is 28% shorter than modeled. This likely compensates for MULE income under-modeling. Controlled SC2 playbooks (mineral-only, 12 workers, no MULEs) would isolate whether the factor is worker physics or MULE-related. The playbook infrastructure exists (PlaybookSC2RecorderTest, DualRunCalibrationTest).

### 2. mineralsCurrent MAPE worsened because income and spending are decoupled

The factors correctly increased income rate (14.5% vs 22.1%), but the extra income accumulates in balance because spending (from TrackerIntents) is unchanged. Two paths forward:
- **Fix spending accuracy** — the spending fields are at 5-17% MAPE, but the cumulative effect on balance is large
- **Balance-aware factor calibration** — optimize for a weighted combination of rate and balance MAPE, not rate alone. This would find smaller factors that improve rate less but don't blow up balance.

### 3. Gas factors not yet applied

vespeneCollectionRate MAPE is 29.1% but no gas efficiency factor was applied. A separate gas factor sweep (extending the same pattern) would likely improve gas rate. However, the gas rate regression from cadence alignment (#595) should be investigated first to avoid compounding corrections.

### 4. Playbook-based calibration is the next accuracy leap

Oracle replays have confounding variables (building, scouting, combat). Controlled playbooks against real SC2 would give precise per-worker rate measurements in isolation. The spec for #591 includes deferred Phase 1 (playbooks) for a session with SC2 available.

---

## Key Code Locations

| What | Where |
|------|-------|
| EconomyTracker (loop-based rate) | `quarkmind-sc2/.../emulated/EconomyTracker.java` |
| EconomyConfig (rate-interval-loops) | `quarkmind-sc2/.../emulated/EconomyConfig.java` |
| EconomyConfigProducer | `quarkmind-sc2/.../emulated/EconomyConfigProducer.java` |
| RaceModel (efficiency factors) | `quarkmind-sc2/.../emulated/RaceModel.java` |
| EmulatedGame.tick() (factor application) | `quarkmind-sc2/.../emulated/EmulatedGame.java:208-209` |
| MiningEfficiencyCalibrationTest (extended sweep) | `quarkmind-sc2/.../replay/MiningEfficiencyCalibrationTest.java` |
| RateCadenceDiagnosticTest | `quarkmind-sc2/.../replay/RateCadenceDiagnosticTest.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
