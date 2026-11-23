# HANDOFF — quarkmind

## Orientation

**Dual-run calibration is the primary accuracy measure.** Controlled SC2 playbooks on TorchesAIE, recorded via `PlaybookSC2RecorderTest`, compared tick-by-tick against the EM. Single-base economy: <3% error. Oracle replays are secondary (noise, wrong maps, not apples-to-apples).

**Immediate next step:** Modify `DualRunCalibrationTest.calibrate()` to use real map coordinates + cached `TerrainGrid` from `MapTerrainCache`. Currently offsets all coordinates to the 64×64 emulated map — the two-base economy test diverges at tick 43 because of this. The TorchesAIE 160×208 pathing grid is already recorded and cached.

**After that:** Validate two-base economy on real terrain. Record more multi-base playbooks (3-base, Terran/Zerg, different saturations). The goal is controlled, reproducible, apples-to-apples economy validation — not oracle replay MAPE.

---

## What's ready

- **TorchesAIE pathing grid:** 160×208, at `terrain/Torches AIE.pathgrid`
- **Two-base economy playbook:** `cal-two-base-economy.yaml` — 2 bases + gas, 450 ticks (~7.3 min), SC2 baseline recorded (131KB)
- **MapTerrainCache:** binary save/load, classpath lookup. `PlaybookSC2Adapter` captures terrain from SC2 API on game start. `resolveMapName` falls back to `replay.details.getTitle()`.
- **Saturation-aware worker assignment:** `onWorkerProduced` assigns to least-saturated base, not nearest
- **Spending leaks fixed:** `handleBuildingBorn` and tracker-driven `handleMorph` now deduct mineral/gas costs

---

## This Session (2026-11-23)

| Issue | What | Status |
|-------|------|--------|
| #756 | Terrain grid auto-sizing (64×64 too small for replay coords to 162×150) | Fixed |
| #757 | Saturation-aware worker assignment (expansion over-staffing +103%) | Fixed |
| #758 | Gas worker estimation | Closed not-planned — GT proximity counting unreliable |
| #759 | Baseline regeneration | Closed — done as part of #756/#757 |
| #760 | MapTerrainCache + real terrain recording | Infrastructure done, TorchesAIE recorded. `calibrate()` still uses offset coordinates. |
| #761 | Spending leaks: BuildingBornIntent + tracker morphs didn't deduct costs | Fixed |

**Physics doc (§16) rewritten:** dual-run calibration is primary, oracle baseline demoted to secondary with explicit methodology caveat.

---

## Branches

Stacked (pipeline bug soredium#617 — created manually):
- `issue-756-seed-expansion-mineral-patches` (from main)
- `issue-757-worker-transfer-count-varies` (on #756)
- `issue-758-gas-worker-over-assignment` (on #757)
- `issue-761-intent-rejection-diagnostic` (on #758)

---

## Notes

- **Pipeline bug (soredium#617):** Create branches manually, set queue state via plan_manager.
- **DroolsScoutingRulesTest/DroolsStrategyL2L3Test:** Pre-existing classpath issues.
- **DivergenceRegressionTest:** Passes with all changes.
