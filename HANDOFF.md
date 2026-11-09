# HANDOFF — quarkmind

## Last Session

Closed 6 issues (#578, #605, #603, #602, #606, #604). Filed 2 follow-ups (#608, #609). Extended CalibrationBaseline with mineralPatches, geyserPositions, basePosition. Fixed coordinate translation bug and ActionTranslator Protoss-only limitation. Recalibrated gas harvest 22→26 loops. Physics constants doc revised throughout.

### Accuracy summary

**Dual-run (controlled SC2):** all structural divergences resolved, gas within ±5%, minerals within ±0.3%.

**Oracle (118 replays, 7-min):**
| Field | Before | After | Direction |
|-------|--------|-------|-----------|
| mineralsCurrent | 126.1% | 77.4% | improved |
| mineralsCollectionRate | 22.1% | 30.1% | EMA more honest, not regression |
| vespeneCollectionRate | 29.1% | 38.1% | regressed — gas harvest increase amplifies worker allocation gap (#608) |
| vespeneCurrent | 65.4% | 74.7% | regressed — follows from rate (#608) |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 12 commits, ready for work-end | #578, #605, #603, #602, #606, #604 |

---

## Issue Map

### Open — next in queue

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 598 | MULE isolation + expansion transfer playbooks | M | High | |
| 579 | Technology spending timing | S | Med | |
| 580 | foodUsed MAPE | S | Med | |
| 581 | Army spending death-timing | S | Med | |
| 582 | Economy spending reporting-lag | S | Med | |
| 608 | Oracle vespene MAPE regression | S | Med | Gas efficiency factor — recover 38.1%→≤30% |
| 609 | Worker delivery desynchronization | S | Low | Stagger initial walks — tick 6 burst ±10 |

---

## Key Code Locations

| What | Where |
|------|-------|
| CalibrationBaseline | `quarkmind-sc2/.../playbook/CalibrationBaseline.java` |
| PlaybookSC2Adapter | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| ActionTranslator (all-race train abilities) | `quarkmind-sc2/.../real/ActionTranslator.java` |
| DualRunCalibrationTest | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| WorkerPhysicsConfig (gas=26.0) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
