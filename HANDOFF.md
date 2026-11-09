# HANDOFF — quarkmind

## Last Session

Closed 5 issues (#578, #605, #603, #602, #606). Extended CalibrationBaseline with mineralPatches, geyserPositions, and basePosition. Fixed critical coordinate translation bug. Recalibrated gas harvest duration from 22→26 loops. All baselines re-recorded 3× against real SC2.

### What was done

- **#578 — Mineral rate regression (CLOSED):** Physics constants doc updated with EMA baseline, race mechanics, builder pause.

- **#605 — Record mineral patch positions (CLOSED):** CalibrationBaseline.PatchPosition. SC2 TorchesAIE_v4 starting base: 3.0–5.9u from Nexus (avg 4.3u).

- **#603 — Worker initialization delay (CLOSED):** Resolved by real patch positions. First-income tick matches. Tick 10 delta: -1.

- **#602 — Record geyser positions (CLOSED):** CalibrationBaseline.GeyserPosition. Translated distance: 7.6u symmetric (matching seeded).

- **#606 — Gas harvest contention (CLOSED):** Reframed from contention penalty to baseline gas rate. Recalibrated gas_harvest_duration from 22→26 loops. Dual-run: 3w=-2.2%, 6w=+4.0%.

- **Coordinate translation fix:** CalibrationBaseline.basePosition records SC2 town hall position. calibrate() translates all coordinates by (emBase - sc2Base) offset. Without this, the 12u proximity search found nothing — injection was functionally inert.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 9 commits, ready for work-end | #578, #605, #603, #602, #606 |

---

## Issue Map

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 578 | Mineral rate regression | Physics doc updated; all downstream fixes resolved |
| 605 | Record mineral patch positions | CalibrationBaseline extended, coord translation |
| 603 | Worker initialization delay | Resolved by real patch positions |
| 602 | Record geyser positions | CalibrationBaseline extended, coord translation |
| 606 | Gas harvest contention | Recalibrated 22→26 loops; was rate, not contention |

### Open — next in queue

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 604 | Re-record 24w/MULE baselines | S | Med | Debug-API timing artifacts |
| 598 | MULE/expansion playbooks | M | High | |
| 579 | Technology spending timing | S | Med | |
| 580-582 | Economy spending floors | S | Med each | |

---

## Key Code Locations

| What | Where |
|------|-------|
| CalibrationBaseline (PatchPosition, GeyserPosition, basePosition) | `quarkmind-sc2/.../playbook/CalibrationBaseline.java` |
| PlaybookSC2Adapter (capture patches/geysers/base) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| DualRunCalibrationTest (coord translation + injection) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| EmulatedGame (setMineralPatchResources, setGeyserResources) | `quarkmind-sc2/.../emulated/EmulatedGame.java` |
| WorkerPhysicsConfig (gas_harvest=26.0) | `quarkmind-sc2/.../emulated/WorkerPhysicsConfig.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
