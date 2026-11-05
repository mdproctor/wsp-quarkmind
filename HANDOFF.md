# HANDOFF — quarkmind

## Last Session (2026-11-05)

Completed #530, #528, #529, #531 from epic #516 (Standalone EmulatedGame).

### What landed

- **#530** — Archon supply tracking. `supplyCost(ARCHON)` = 4.

- **#528** — Morph cost tracking. TrackerIntentExtractor emits MorphIntent on ID_UNIT_TYPE_CHANGE. EmulatedGame records spending for tracker-driven morphs. EconomyTracker.recordUnitDeath subtracts total investment via morphSourceType.

- **#529** — SC2 spending timing + classification. Four root causes fixed:
  1. Timing: moved recordTrainSpending and recordResearchSpending to completion callbacks (SC2's "Current" counts completed items only)
  2. Building injection: only record spending for complete buildings; syncReplayBuilding records at completion transition
  3. Overlord classification: supply-providing units (Overlord/Overseer) classified as economy, not army
  4. UNKNOWN building cost: set to 0 (was 100m, phantom spending)

- **#531** — Missing building mappings + warp-in guard. Added toBuildingType for CreepTumor, CreepTumorQueen, CreepTumorBurrowed, OracleStasisTrap, AssimilatorRich, LurkerDenMP. Added BUILDING_NAMES guard in applyUnitInit — warp-in units were being added as UNKNOWN buildings. Calibrated LURKER_DEN and CREEP_TUMOR build times.

### Accuracy improvements (5-min checkpoint)

| Field | Before | After |
|-------|--------|-------|
| mineralsUsedCurrentArmy | 62.5% | **10.4%** |
| mineralsUsedCurrentEconomy | 10.3% | **4.7%** |
| mineralsUsedCurrentTechnology | 63.2% | **23.9%** |
| vespeneUsedCurrentArmy | 58.4% | **21.2%** |
| UNKNOWN buildings | 218 | **0** |

### Branch

`issue-530-archon-supply-tracking` — 5 commits, not yet merged. No work-end run (queue preservation).

### Queue status

Epic #516 — Standalone EmulatedGame:

1. ~~#530 — Archon supply tracking~~ ✅
2. ~~#528 — Morph cost tracking~~ ✅
3. ~~#529 — SC2 spending classification~~ ✅
4. ~~#531 — Track missing building types~~ ✅
5. #533 — Technology mineral remaining gap — 24% MAPE at 5-min (S/Med)
6. #532 — Vespene spending variance (S/Med)
7. #519 — Gas income ramp-up and worker assignment (M/Med) — 25% MAPE
8. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
9. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
10. #523 — Standalone building lifecycle (L/High) — structural
11. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy (post-session)
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
