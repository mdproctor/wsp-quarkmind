# HANDOFF — quarkmind

## Last Session (2026-11-04)

Completed #525 (UnitTypeChange handling), #527 (half-supply tracking) from epic #516.

### What landed

- **#525** — TrackerIntentExtractor UnitTypeChange handling. Added `ID_UNIT_TYPE_CHANGE` handler, 9 missing tracker type mappings. Missed supply reduced 1,437→475 (67%). Root cause of foodUsed overcount was NOT missed deaths — it was Zergling half-supply.

- **#527** — Double supply tracking. Changed supply type from int to double. `supplyCost(ZERGLING)` = 0.5 (was 1), `supplyCost(BANELING)` = 0.5 (was 0). GT now preserves fractional supply from tracker events. foodUsed MAPE: 4.2-10.9% → **3.2-7.0%** (35-54% reduction). 19 files changed, 3,273 tests pass.

### Current standalone accuracy

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| foodMade | 0.9% | 1.5% | 4.2% | 4.4% |
| foodUsed | 4.2% | 3.2% | 7.0% | 5.1% |

### Remaining foodUsed 3-7% residual

Not from Zergling/Baneling supply. Likely causes:
- Morph supply deltas not tracked (Roach→Ravager +1 supply)
- Other supply accounting gaps
- Separate investigation needed

## Active Plan

Epic #516 — Standalone EmulatedGame. Queue:

1. #526 — Harness seed-state deduplication (S/Med) — reduces foodMade 0.9-4.4%
2. #522 — Standalone spending tracking (M/Med) — largest MAPE gap (40-87%)
3. #519 — Gas income ramp-up and worker assignment (M/Med) — 50-86% MAPE
4. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
5. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
6. #523 — Standalone building lifecycle (L/High) — structural, eliminates syncBuildings
7. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
