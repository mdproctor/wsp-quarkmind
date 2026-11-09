# HANDOFF — quarkmind

## Last Session

Closed 4 issues (#578, #605, #603, #602). Fixed critical coordinate translation bug — real SC2 mineral/geyser positions were injected but never found due to EM Nexus at (8,8) vs SC2 map coordinates. All 10 baselines re-recorded 3× (patches, geysers, then basePosition+translation). Updated `docs/sc2-physics-constants.md` throughout.

### What was done

- **#578 — Mineral rate regression (CLOSED):**
  Physics constants doc updated: §6 16w table fixed (6.9%→6.6%), §14 builder pause per-race + race production mechanics, §16 replaced stale simple-window baseline with EMA-based (mineralsCollectionRate 30.2%, mineralsCurrent 77.6%).

- **#605 — Record mineral patch positions (CLOSED):**
  `CalibrationBaseline.PatchPosition(x, y, amount, rich)`. Capture at first tick via PlaybookSC2Adapter. Inject via `setMineralPatchResources()` with coordinate translation (offset = emBase - sc2Base).

- **#603 — Worker initialization delay (CLOSED):**
  Real patch positions resolved it. First-income tick matches. Tick 10 delta: -1 (target ±5). Burst at tick 6 (+29) is synchronized worker delivery — structural.

- **#602 — Record geyser positions (CLOSED):**
  `CalibrationBaseline.GeyserPosition(x, y, amount)`. `EmulatedGame.setGeyserResources()`. DualRun filters to 12u radius. Translated geyser distances confirmed at 7.6u (matching seeded).

- **Coordinate translation fix (#605/#602):**
  The EM Nexus is at (8,8) but SC2 patches are at map coordinates (e.g. 124.5, 159.5). `CalibrationBaseline.basePosition` records the SC2 town hall position. `calibrate()` translates all patch/geyser coordinates by `(8 - baseX, 8 - baseY)` before injection. Without this, the 12u proximity search found nothing — injection was functionally inert.

### Key finding — gas rate exposed by translation

With coordinate translation working, the dual-run tests reveal:
| Config | SC2 rate | EM rate | Delta |
|--------|---------|---------|-------|
| 3w/1g | 41.4/min/w | 49.4/min/w | +19% |
| 6w/2g | 38.6/min/w | 49.2/min/w | +27% |

The EM over-produces equally at both scales — it's a baseline gas rate issue (harvest_duration=22 too short), not contention. The 7% SC2 drop from 3w→6w is real contention but secondary.

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | 10 commits, ready for work-end | #578, #605, #603, #602, coord translation |

---

## Issue Map

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 578 | Mineral rate regression | All downstream fixes resolved; physics doc updated |
| 605 | Record mineral patch positions | CalibrationBaseline extended, coord translation |
| 603 | Worker initialization delay | Resolved by real patch positions |
| 602 | Record geyser positions | CalibrationBaseline extended, coord translation |

### Open — next in queue

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| 606 | Gas harvest contention | M | High | Reframed: baseline gas rate +19-27%, not contention scaling. Re-sweep gas_harvest_duration against dual-run. |
| 604 | Re-record 24w/MULE baselines | S | Med | Debug-API timing artifacts — SCV count diverges |
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
| SC2 physics constants | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
