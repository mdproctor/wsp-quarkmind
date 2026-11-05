# HANDOFF — quarkmind

## Last Session (2026-11-05)

Completed #530, #528, major progress on #529. Filed #531, #532, #533.

### What landed

- **#530** — Archon supply tracking. `supplyCost(ARCHON)` = 4. HT merge now supply-neutral.

- **#528** — Morph cost tracking. TrackerIntentExtractor emits MorphIntent on ID_UNIT_TYPE_CHANGE. EmulatedGame records spending for tracker-driven morphs. EconomyTracker.recordUnitDeath subtracts total investment via morphSourceType.

- **#529** — SC2 spending classification + timing (major). Four root causes identified and fixed:
  1. **Timing**: SC2's `scoreValueMineralsUsedCurrentArmy` counts COMPLETED items only. We recorded at training START. Moved `recordTrainSpending` and `recordResearchSpending` to completion callbacks.
  2. **Building injection timing**: `injectReplayBuilding` now records spending only for complete buildings. `syncReplayBuilding` records when buildings transition to complete.
  3. **Overlord classification**: SC2 classifies Overlords as economy (supply providers). Added `unitSupplyProvided > 0` check.
  4. **UNKNOWN building cost**: 218 UNKNOWN buildings (add-ons, creep tumors) × 100m = phantom 21,800m in technology. Set to 0.

### Accuracy improvements (5-min checkpoint)

| Field | Before | After |
|-------|--------|-------|
| mineralsUsedCurrentArmy | 62.5% | **10.4%** |
| mineralsUsedCurrentEconomy | 10.3% | **4.8%** |
| mineralsUsedCurrentTechnology | 63.2% | **23.9%** |
| vespeneUsedCurrentArmy | 58.4% | **21.2%** |
| vespeneUsedCurrentTechnology | 82.6% | **57.1%** |
| foodMade | 2.6% | 2.6% |
| foodUsed | 7.0% | 7.0% |

### Branch

`issue-530-archon-supply-tracking` — 4 commits, needs work-end.

### Issues filed this session

- **#531** — Track Terran add-on buildings (Tech Lab, Reactor) as BuildingType (XS/Low)
- **#532** — Vespene spending variance — high per-replay error (S/Med)
- **#533** — Technology mineral spending remaining gap — 24% MAPE at 5-min (S/Med)

### Queue status

Epic #516 — Standalone EmulatedGame. Updated queue:

1. ~~#530 — Archon supply tracking~~ ✅
2. ~~#528 — Morph cost tracking~~ ✅
3. ~~#529 — SC2 spending classification~~ ✅ (classification done, MAPE 63%→24%)
4. #531 — Track Terran add-ons (XS/Low) — NEW, quick win
5. #533 — Tech mineral remaining gap (S/Med) — NEW, diagnostic
6. #532 — Vespene spending variance (S/Med) — NEW, diagnostic
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

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy (post-#529)
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
