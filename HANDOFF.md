# HANDOFF — quarkmind

## Orientation for Next Session

**Economy calibration methodology pivot required.** The standalone baseline against oracle replays is not apples-to-apples (oracle replays have combat, scouting, variable builds). The right approach: controlled SC2 playbooks on TorchesAIE with the real pathing grid, validated via DualRunCalibrationTest.

**Start with:** Modify `DualRunCalibrationTest.calibrate()` to use real map coordinates + cached `TerrainGrid` from `MapTerrainCache` instead of offsetting to the 64×64 emulated map. The two-base economy test (`dualRunCalTwoBaseEconomyAgainstSC2`) diverges at tick 43 because of this coordinate offset.

**What's ready:**
- TorchesAIE 160×208 pathing grid recorded (`terrain/Torches AIE.pathgrid`)
- Two-base economy playbook + SC2 baseline recorded (450 ticks, 131KB)
- MapTerrainCache infrastructure (save/load/classpath lookup)
- `PlaybookSC2Adapter` captures terrain on game start

---

## This Session (2026-11-23)

**Issues completed:** #756, #757, #758 (closed), #759 (closed), #761

**Key fixes:**
- #756 — Terrain grid auto-sizing (64×64 too small for replay coordinates)
- #757 — Saturation-aware worker assignment (expansion over-staffing fix, mineral rate 38.2%→13.7%)
- #761 — Spending leaks: BuildingBornIntent and tracker-driven morphs didn't deduct costs

**Infrastructure built (#760):**
- MapTerrainCache: binary save/load for TerrainGrid
- PlaybookSC2Adapter: captures pathing grid from SC2 API
- TorchesAIE 160×208 pathing grid recorded
- Two-base economy playbook (cal-two-base-economy.yaml) + SC2 baseline
- resolveMapName: falls back to replay.details.getTitle() when initData is empty

**Key finding — methodology:**
- Oracle replays (118, blizzard ladder) are on dozens of different ladder maps (Thunderbird LE, etc.) — NOT TorchesAIE
- Oracle replays have combat/scouting/harass noise — not controlled economy
- The 13.7% rate MAPE is measured against this noise — not meaningful for calibration
- DualRunCalibrationTest against controlled playbooks on TorchesAIE is the right comparison
- BUT: `calibrate()` offsets all coordinates to fit the 64×64 emulated map instead of using real terrain
- Next step: use real coordinates + real terrain in calibrate()

---

## Stacked Branches

Branches stacked (pipeline bug soredium#617):
- `issue-756-seed-expansion-mineral-patches` (from main)
- `issue-757-worker-transfer-count-varies` (stacked on #756)
- `issue-758-gas-worker-over-assignment` (stacked on #757)
- `issue-761-intent-rejection-diagnostic` (stacked on #758)

`work end` will need to handle the stack.

---

## Notes

- **Pipeline bug (soredium#617):** Workaround: create branches manually, set queue state via plan_manager.
- **DroolsScoutingRulesTest/DroolsStrategyL2L3Test:** Pre-existing classpath issues.
- **DivergenceRegressionTest:** Passes with all changes.
- **Oracle replays are NOT on TorchesAIE** — they're on dozens of ladder maps. Recording pathing grids for all of them isn't practical. Use controlled playbooks instead.
