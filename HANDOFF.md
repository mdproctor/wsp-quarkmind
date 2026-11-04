# HANDOFF — quarkmind

## Last Session (2026-11-04)

Completed #525, #527, #526 from epic #516 (Standalone EmulatedGame).

### What landed

- **#525** — TrackerIntentExtractor UnitTypeChange handling. Added `ID_UNIT_TYPE_CHANGE` handler, 9 missing tracker type mappings. Missed supply reduced 1,437→475. Root cause of foodUsed overcount identified as Zergling half-supply, not missed deaths.

- **#527** — Double supply tracking. Changed supply from int to double. `supplyCost(ZERGLING)` = 0.5 (was 1), `supplyCost(BANELING)` = 0.5 (was 0). foodUsed MAPE: 4.2-10.9% → **3.2-7.0%** (35-54% reduction).

- **#526** — Harness seed-state deduplication. Added `resetForHarness()` to skip race model seed. Removed supply zeroing workaround. foodMade MAPE unchanged (0.9-4.4%) — residual is GT timing mismatch, not deduplication.

### Current standalone accuracy

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| foodMade | 0.9% | 1.5% | 4.2% | 4.4% |
| foodUsed | 4.2% | 3.2% | 7.0% | 5.1% |

### Remaining work for 100% accuracy

Supply tracking is largely solved. Remaining high-MAPE fields:
- mineralsCurrent: 22% — needs distance-aware mining (#518)
- vespeneCurrent: 41% — needs gas income model (#519)
- workersActiveCount: 45% — needs worker lifecycle (#520, blocked by #518/#519)
- spending fields: 40-87% — needs spending tracking (#522)
- building state: needs standalone lifecycle (#523)

## Active Plan

Epic #516 — Standalone EmulatedGame. Remaining queue:

1. #522 — Standalone spending tracking (M/Med) — largest MAPE gap (40-87%)
2. #519 — Gas income ramp-up and worker assignment (M/Med) — 50-86% MAPE
3. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
4. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
5. #523 — Standalone building lifecycle (L/High) — structural, eliminates syncBuildings
6. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy
- Epic #516 — standalone EmulatedGame issue tracker
