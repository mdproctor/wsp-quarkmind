# HANDOFF — quarkmind

## Orientation for Next Session

**Expansion income deficit phase complete.** Issues #756-#761 resolved. mineralsCollectionRate improved from 38.2% to 13.7% (-24.5pp). Two real bugs found: terrain grid too small for replay coordinates, and spending leaks in BuildingBornIntent/tracker morphs.

**Start with:** `work end` to land the stacked branches (#756→#757→#758→#761), or pivot to other work.

**Key open issue:** #760 — MapTerrainCache infrastructure is built but needs SC2 running to record real map pathing grids. Until recorded, the EM runs on flat terrain — the rate comparison is not apples-to-apples with real SC2 maps.

**Key reference:** `docs/sc2-physics-constants.md` §16 — economy baseline with methodology caveat.

---

## This Session (2026-11-23)

**Completed 6 issues:**
- #756 (M/Med) — Fixed: 64×64 emulated map too small for replay coordinates (patches to 162×150). A* pathfinding clamped expansion positions to grid edge. `setMineralPatchResources()` auto-sizes terrain grid.
- #757 (S/Med) — Fixed: `onWorkerProduced` assigned to nearest base → expansion over-staffing (+103%). Now assigns to least-saturated base. **This was the big win: mineral rate 36.6%→13.7%.**
- #758 (S/Low) — Closed not-planned: GT proximity-based gas worker counting is unreliable. Rate-based estimation works correctly.
- #759 (XS/Low) — Closed: already done as part of #756/#757 work.
- #760 (M/Med) — Infrastructure complete: MapTerrainCache for recording/loading real SC2 map pathing grids. `PlaybookSC2Adapter` captures terrain on game start. Needs SC2 to record TorchesAIE_v4 grid.
- #761 (M/Med) — Fixed: `handleBuildingBorn` and tracker-driven `handleMorph` never deducted mineral/gas costs from balance. mineralsCurrent 132.4%→123.0%.

**Rate improvement:**

| Field | Start | End | Delta |
|-------|-------|-----|-------|
| mineralsCollectionRate (7-min) | 38.2% | 13.7% | **-24.5pp** |
| vespeneCollectionRate (7-min) | 18.7% | 12.3% | **-6.4pp** |
| mineralsCurrent (7-min) | 84.9% | 123.0% | +38.1pp (rate fixed → more income, then spending leaks partially fixed) |
| vespeneCurrent (7-min) | 61.1% | 67.2% | +6.1pp |

**Key findings:**
1. The 64×64 emulated map was a fundamental flaw — replay coordinates range to (162, 150). Every expansion base had garbage A* paths.
2. Worker distribution was inverted: EM over-staffed expansions by 103% at 7-min while under-staffing main by 38%. Root cause: `onWorkerProduced` assigned to nearest base, not least-saturated.
3. Spending leaks: `BuildingBornIntent` and tracker-driven building morphs (CC→OC, Hatch→Lair) never deducted costs. Buildings were free in standalone mode.
4. The rate comparison is NOT apples-to-apples: EM runs on flat terrain, SC2 uses real maps. Recording real map pathing grids (#760) is the prerequisite to validating whether the rate is truly at the floor.

**Updated:** `docs/sc2-physics-constants.md` (§14 cumulative drift, §15 terrain auto-sizing, §16 full economy baseline revision), `docs/benchmarks/standalone-emulated-game-baseline.md`

---

## Queue

#761 is the active item (closed on GitHub). Queue is drained.

---

## Stacked Branches

Branches are stacked (pipeline bug soredium#617 — branches created manually):
- `issue-756-seed-expansion-mineral-patches` (from main)
- `issue-757-worker-transfer-count-varies` (stacked on #756)
- `issue-758-gas-worker-over-assignment` (stacked on #757)
- `issue-761-intent-rejection-diagnostic` (stacked on #758)

`work end` will need to handle the stack — may need manual rebasing or sequential landing.

---

## Notes

- **Pipeline bug (soredium#617):** Still present. Workaround: create branches manually, set queue state via plan_manager. Alignment detection also fails for stacked branches.
- **DroolsScoutingRulesTest/DroolsStrategyL2L3Test:** Pre-existing classpath issues in full test suite — unrelated to this session's changes.
- **DivergenceRegressionTest:** Passes with all changes.
