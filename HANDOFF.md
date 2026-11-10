# HANDOFF — quarkmind

## Last Session

Closed 4 issues (#609, #598, #608, #610). Filed 6 follow-ups (#611-616). Worker stagger, SC2 adapter morph/calldown, gas efficiency override. Physics constants doc revised throughout.

### Accuracy summary

**Dual-run (controlled SC2):** all structural divergences resolved, gas within ±5%, minerals within ±0.5%.

**Oracle (118 replays, 7-min):**
| Field | Before this session | After | Direction |
|-------|-------------------|-------|-----------|
| mineralsCollectionRate | 30.1% | 30.1% | unchanged |
| mineralsCurrent | 77.4% | 77.4% | unchanged |
| vespeneCollectionRate | 38.1% | 26.1% | recovered via gas efficiency override 1.20 (#608) |
| vespeneCurrent | 74.7% | 58.0% | recovered (#608) |

### Key changes

1. **Worker stagger (#609):** `walkStartLoop = i × 6` delays initial mineral walks. Tick 6 delta +29→+4. Configurable via `starcraft.worker.initial-stagger-loops`.
2. **SC2 adapter morph (#598):** Debug morph (DebugKillUnit + DebugCreateUnit) — ability-based morph silently rejected by SC2 when CC has production queue. `matchesBuildingType()` now handles morphed variants (CC↔OC, Hatch↔Lair↔Hive).
3. **Gas efficiency override (#608):** `EmulatedGame.setGasEfficiencyOverride(1.20)` applied only in standalone oracle test. Global RaceModel factor breaks dual-run (50% over-production). Worker estimation adjustment (GAS_PER_WORKER_PER_MIN) hits a 32.5% MAPE floor from integer discretization.
4. **Config bug (#610):** WorkerPhysicsConfigProducer gas harvest default was still 22 (should be 26 after #606).

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 18 commits, ready for work-end | #578, #605, #603, #602, #606, #604, #609, #598, #608, #610 |

---

## Queue fix needed

The `.queue` file has two issues from an accidental `advance`:
1. **#579 marked as `[x]`** — still open, needs unchecking
2. **Order:** #611-614 (quick wins) should come before #579-582 (spending issues), with #615-616 after

Intended order:
```
[x] #608, #609 (closed this session)
[ ] #611 — Mining efficiency recalibration (S/Low) ← next
[ ] #612 — Doc refresh (S/Low)
[ ] #613 — MULE income analysis (S/Med)
[ ] #614 — Expansion transfer analysis (S/Low)
[ ] #579 — Technology spending timing (S/Med)
[ ] #580 — foodUsed MAPE (S/Med)
[ ] #581 — Army spending death-timing (S/Med)
[ ] #582 — Economy spending reporting-lag (S/Med)
[ ] #615 — Gas worker allocation (M/Med)
[ ] #616 — SC2 morph investigation (S/Med)
```

---

## Issue Map

### Open — next in queue

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 611 | Mining efficiency recalibration | S | Low | Quick win — absorb stagger's −0.44% offset |
| 612 | Dual-run baseline doc refresh | S | Low | §6 numbers stale after #604 re-recording + #609 stagger |
| 613 | MULE income isolation analysis | S | Med | Data recorded, validate EM's 277.8 minerals/min |
| 614 | Expansion transfer delay analysis | S | Low | Data recorded, validate MAX_TRANSFER_PER_EXPANSION |
| 579 | Technology spending timing | S | Med | InProgress/Current timing boundary |
| 580 | foodUsed MAPE | S | Med | 6.1% vs target 2-3% |
| 581 | Army spending death-timing | S | Med | 7% MAPE, tick vs frame granularity |
| 582 | Economy spending reporting-lag | S | Med | 5.8% MAPE |
| 615 | Gas worker allocation modeling | M | Med | Replace 1.20× override with proper allocation |
| 616 | SC2 morph via ability API | S | Med | Research — why MORPH_ORBITAL_COMMAND is rejected |

---

## Key Code Locations

| What | Where |
|------|-------|
| Worker stagger | `SimulatedWorker.walkStartLoop` + `WorkerSimulation.onInitialWorkers()` |
| Gas efficiency override | `EmulatedGame.setGasEfficiencyOverride()` |
| Debug morph | `PlaybookSC2Adapter.translateMorphDebug()` |
| Building type matching | `PlaybookSC2Adapter.matchesBuildingType()` |
| CalibrationBaseline | `quarkmind-sc2/.../playbook/CalibrationBaseline.java` |
| PlaybookSC2Adapter | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| DualRunCalibrationTest | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| WorkerPhysicsConfig | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |
| StandaloneEmulatedGameBaselineTest | `quarkmind-sc2/.../replay/StandaloneEmulatedGameBaselineTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
