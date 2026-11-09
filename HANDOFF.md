# HANDOFF — quarkmind

## Last Session

Closed 4 calibration issues (#578, #605, #603, #602). Updated `docs/sc2-physics-constants.md` throughout — EMA baseline numbers, real SC2 patch/geyser positions, race-specific production mechanics. Extended `CalibrationBaseline` with `mineralPatches` and `geyserPositions` fields. All 10 SC2 baselines re-recorded with 154 mineral patches and 32 geysers.

### What was done

- **#578 — Mineral rate regression (CLOSED):**
  Physics constants doc updated: §6 16w table fixed (6.9%→6.6%), §14 builder pause per-race + race production mechanics, §16 replaced stale simple-window baseline with EMA-based (mineralsCollectionRate 30.2%, mineralsCurrent 77.6%). All downstream fixes (#583-#601) resolved the original 4x regression.

- **#605 — Record mineral patch positions (CLOSED):**
  `CalibrationBaseline.PatchPosition(x, y, amount, rich)` inner record. `PlaybookSC2Adapter` captures at first tick. `DualRunCalibrationTest` injects via `setMineralPatchResources()`. TorchesAIE_v4 starting base patches: 3.0–5.9u from Nexus center (avg 4.3u).

- **#603 — Worker initialization delay (CLOSED):**
  Real patch positions from #605 resolved it. First-income tick matches (both tick 6). Tick 10 delta: -1 (target ±5). The +29 burst at tick 6 is synchronized worker deliveries, not distance error.

- **#602 — Record geyser positions (CLOSED):**
  `CalibrationBaseline.GeyserPosition(x, y, amount)` inner record. `EmulatedGame.setGeyserResources()` to override seeded geysers. DualRun filters to 12u radius from town hall (avoids far-map geysers breaking gas placement). TorchesAIE_v4 geysers: 5.7u and 8.5u (asymmetric, vs seeded 7.6u symmetric).

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 7 commits, ready for work-end | #578, #605, #603, #602 |

---

## Issue Map

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 578 | Mineral rate regression from acceleration | All downstream fixes resolved; physics doc updated with EMA baseline |
| 605 | Record mineral patch positions | CalibrationBaseline extended, 10 baselines re-recorded |
| 603 | Worker initialization delay | Resolved by real patch positions; tick 10 delta=-1 |
| 602 | Record geyser positions | CalibrationBaseline extended, 10 baselines re-recorded |

### Open — calibration roadmap

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|---|---|---|---|---|
| 606 | Gas harvest contention (6w) | S | Med | — | Unblocked by #602 |
| 604 | Re-record 24w/MULE baselines | S | Med | — | Debug-API timing artifacts |
| 598 | MULE isolation / expansion playbooks | M | High | — | Morph/calldown untested |

### Open — economy accuracy

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 579 | Technology spending timing | S | Med | InProgress/Current boundary |
| 580 | foodUsed MAPE | S | Med | Above structural floor |
| 581 | Army spending death-timing | S | Med | |
| 582 | Economy spending reporting-lag | S | Med | |

---

## Key Code Locations

| What | Where |
|------|-------|
| CalibrationBaseline (PatchPosition, GeyserPosition) | `quarkmind-sc2/.../playbook/CalibrationBaseline.java` |
| PlaybookSC2Adapter (patch/geyser capture) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| DualRunCalibrationTest (position injection) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| EmulatedGame (setMineralPatchResources, setGeyserResources) | `quarkmind-sc2/.../emulated/EmulatedGame.java` |
| SC2 physics constants | `docs/sc2-physics-constants.md` |
| Standalone baseline report | `docs/benchmarks/standalone-emulated-game-baseline.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
