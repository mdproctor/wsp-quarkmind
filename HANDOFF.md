# HANDOFF — quarkmind

## Last Session (2026-11-04)

Completed #525, #527, #526, #522 from epic #516 (Standalone EmulatedGame).

### What landed

- **#525** — TrackerIntentExtractor UnitTypeChange handling. Added `ID_UNIT_TYPE_CHANGE` handler, 9 missing type mappings. Root cause of foodUsed identified as Zergling half-supply, not missed deaths.

- **#527** — Double supply tracking. Changed supply from int to double. `supplyCost(ZERGLING)` = 0.5 (was 1), `supplyCost(BANELING)` = 0.5 (was 0). foodUsed MAPE: 10.9% → **7.0%** at 5-min.

- **#526** — Harness seed-state deduplication. Added `resetForHarness()` to skip race model seed.

- **#522** — Standalone spending tracking. Unit death subtracts from spending (SC2 "InCurrentArmy" semantics). Building costs on GT injection. Corrected classification: `isEconomyBuilding` (bases/supply/gas only) replaces `isTechBuilding`. Building destruction detection. mineralsUsedCurrentEconomy MAPE: **40% → 10%**. foodMade: **4.2% → 2.6%** at 5-min.

### Current standalone accuracy (5-min checkpoint)

| Field | MAPE |
|-------|------|
| foodMade | 2.6% |
| foodUsed | 7.0% |
| mineralsUsedCurrentEconomy | 10.3% |
| mineralsUsedCurrentArmy | 62.5% |
| mineralsUsedCurrentTechnology | 63.2% |
| vespeneUsedCurrentArmy | 58.4% |
| vespeneUsedCurrentTechnology | 82.6% |
| mineralsCollectionRate | 22.1% (3-min) |
| vespeneCollectionRate | 25.5% (5-min) |
| workersActiveCount | 3.9% (1-min), 44.7% (5-min) |

### Remaining spending gaps

Army spending overcounted (62%) — morph costs not fully tracked. Technology spending overcounted (63%) — SC2 classification of production buildings needs research. Both need deeper investigation.

## Active Plan

Epic #516 — Standalone EmulatedGame. Remaining queue:

1. #530 — Archon supply tracking (XS/Low) — quick win, 4 supply on merge
2. #528 — Morph cost tracking (S/Med) — fixes army spending gap from morphs
3. #529 — SC2 spending classification research (S/Med) — fixes technology spending overcount
4. #519 — Gas income ramp-up and worker assignment (M/Med) — 25-50% MAPE
5. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
6. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
7. #523 — Standalone building lifecycle (L/High) — structural, eliminates syncBuildings
8. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
