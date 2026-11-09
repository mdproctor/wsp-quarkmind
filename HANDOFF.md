# HANDOFF — quarkmind

## Last Session

Closed #590 (rate cadence). Investigated #591 (income direction) — major finding from SC2 playbook calibration: per-worker mining physics is only 5% off, the 22% aggregate MAPE is from worker allocation, not physics. Efficiency factors (1.18/1.12/1.28) were wrong — reverted to 1.0. Filed #593-595 for rate accuracy. Built SC2 calibration infrastructure: 10 playbooks, adapter tick/loop fix, sc2 Maven profile.

### What was done

- **#590 — Rate cadence alignment (closed).** EconomyTracker switched from tick-counting (154 loops) to loop-counting (160 loops, configurable). Empirically confirmed 96.8% of PlayerStatsEvents at exactly 160 loops. Mixed MAPE effect (+/-1-2%). Added EconomyConfig with `starcraft.economy.rate-interval-loops`.

- **#591 — Worker income direction (in progress).**
  - Extended efficiency factor sweep to 0.85-1.30. Oracle-derived factors: Protoss=1.18, Zerg=1.12, Terran=1.28 — all above 1.0, suggesting model under-produces.
  - Applied factors via RaceModel.miningEfficiencyFactor() in EmulatedGame.tick(). Rate MAPE improved (22.1% → 14.5%) but mineralsCurrent MAPE exploded (67.5% → 126.1%).
  - **SC2 playbook calibration disproved the factors.** Recorded 3 baselines against real SC2: Protoss=56.9, Terran=56.2, Zerg=56.2 minerals/min/worker. Model produces 60.0. The model OVER-produces per-worker income by 5%. The oracle-derived factors > 1.0 were compensating for worker allocation gaps, not physics.
  - **Factors reverted to 1.0.** The infrastructure (RaceModel methods, EmulatedGame wiring) stays for future use.

- **SC2 calibration infrastructure built:**
  - 10 targeted playbooks in `src/test/resources/playbooks/cal-*.yaml`
  - Fixed PlaybookSC2Adapter: tick/loop conversion, isRunning() race condition
  - Added `sc2` Maven profile (was impossible to run @Tag("sc2") tests before)
  - 9 self-baseline tests + 9 SC2 recorder tests in PlaybookSC2RecorderTest
  - 3 SC2 baselines recorded in `src/test/resources/calibration/`

- **New issues filed:** #593 (SC2 rate algorithm RE), #594 (sub-tick rate tracking), #595 (gas rate regression)

---

## Current Accuracy State

### Standalone EmulatedGameBaselineTest — 7-min checkpoint

Factors at 1.0 — same as before this session for most fields:

| Field | Value | Status |
|---|---|---|
| mineralsCollectionRate | 22.1% | Cadence alignment, no net change from factors |
| vespeneCollectionRate | 29.1% | Unchanged — gas factors not applied |
| mineralsCurrent | 67.5% | Unchanged |
| vespeneCurrent | 65.4% | Unchanged |
| All other fields | Same as previous session | At floor or structural |

### Key calibration data (NEW — from SC2 playbooks)

| Measurement | SC2 Real | EM Model | Delta |
|---|---|---|---|
| Protoss per-worker mineral rate | 56.9/min | 60.0/min | +5.5% over |
| Terran per-worker mineral rate | 56.2/min | 60.0/min | +6.8% over |
| Zerg per-worker mineral rate | 56.2/min | 60.0/min | +6.8% over |

---

## Remaining Queue

```
- [ ] #592 — Overhead absorption limit investigation
- [ ] #593 — SC2 rate algorithm reverse-engineering
- [ ] #594 — Sub-tick rate tracking — exact 160-loop window
- [ ] #595 — Gas rate regression analysis — 176-loop window impact
```

---

## Critical Insights for Next Session

### 1. Per-worker physics is calibrated — allocation is the gap

SC2 playbooks prove per-worker rate is within 5%. The 22% aggregate MAPE in oracle replays comes from worker ALLOCATION — too many EM workers are non-productive (building pauses, saturation, transit). This is #572's original domain (worker activity modeling). The fix isn't physics constants — it's modeling which workers are actively mining vs paused/scouting/transferring.

### 2. Oracle replays calibrate behavior, SC2 playbooks calibrate physics

Don't fit physics constants against oracle data — the confounding variables lead to wrong answers (1.18 factor that should be 0.95). Use SC2 playbooks for physics, oracle replays for full-system validation.

### 3. Seven SC2 recordings remain — run in parallel with calibration

**Completed (3):** cal-mineral-baseline-12w (Protoss), cal-mineral-baseline-terran, cal-mineral-baseline-zerg
**Remaining (7):** saturation (16w/24w), gas (3w/6w), MULE isolation, expansion transfer, worker speed

Run each with `-Psc2` (the sc2 Maven profile added this session):
```bash
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalMineralSaturated16w
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalMineralSaturated24w
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalGasBaseline3w
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalGasDual6w
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalMuleIsolation
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalExpansionTransfer
mvn test -pl quarkmind-sc2 -Psc2 -Dtest=PlaybookSC2RecorderTest#recordCalWorkerSpeedSingle
```

**Apply data as it arrives.** Each recording writes a JSON baseline to `src/test/resources/calibration/`. As each lands, immediately compare against EmulatedGame (load the JSON, run EM with same playbook, diff per-tick minerals/vespene). Apply findings to physics constants or the model without waiting for all 7 to complete.

**Priority order:** saturation (16w/24w) first — reveals the saturation curve; gas (3w) next — isolates gas per-worker rate; MULE isolation — explains Terran's large oracle deficit.

### 4. Gas efficiency factor sweep hit the ceiling

Oracle-derived gas factors (1.28-1.30) hit the sweep ceiling at 1.30. Like mineral factors, these are likely wrong — compensating for allocation, not physics. SC2 gas baseline recording will reveal the true per-worker gas rate.

### 5. The efficiency factor infrastructure stays

`RaceModel.miningEfficiencyFactor()` / `gasEfficiencyFactor()` and the EmulatedGame.tick() wiring are committed with default 1.0. Ready for calibrated values once the allocation gap is understood and the factors can be correctly decomposed into physics vs behavior.

---

## Key Code Locations

| What | Where |
|------|-------|
| EconomyTracker (loop-based rate) | `quarkmind-sc2/.../emulated/EconomyTracker.java` |
| EconomyConfig (rate-interval-loops) | `quarkmind-sc2/.../emulated/EconomyConfig.java` |
| RaceModel (efficiency factor interface) | `quarkmind-sc2/.../emulated/RaceModel.java` |
| EmulatedGame.tick() (factor application) | `quarkmind-sc2/.../emulated/EmulatedGame.java:208-209` |
| PlaybookSC2Adapter (tick/loop conversion) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| PlaybookSC2RecorderTest (10 scenarios) | `quarkmind-sc2/.../real/PlaybookSC2RecorderTest.java` |
| Calibration playbooks | `quarkmind-sc2/src/test/resources/playbooks/cal-*.yaml` |
| SC2 baselines (3 recorded) | `quarkmind-sc2/src/test/resources/calibration/cal-*.json` |
| MiningEfficiencyCalibrationTest (extended) | `quarkmind-sc2/.../replay/MiningEfficiencyCalibrationTest.java` |
| RateCadenceDiagnosticTest | `quarkmind-sc2/.../replay/RateCadenceDiagnosticTest.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |
| Maven sc2 profile | `quarkmind-sc2/pom.xml` (run with `-Psc2`) |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
