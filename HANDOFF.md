# HANDOFF — quarkmind

## Last Session (2026-11-04)

Completed #517 (baseline measurement) and #521 (standalone supply tracking) from epic #516. Filed #525 and #526 for residual supply gaps. Posted baseline data to all child issues.

### What landed

- **#517** — Baseline: measure standalone EmulatedGame accuracy without GT syncs. Created `StandaloneEmulatedGameBaselineTest` running 118 oracle replays with all 10 economy GT syncs disabled (buildings still injected from GT). Units/buildings/upgrades: 100%. Economy: 20-470% MAPE depending on field. Baseline doc at `docs/benchmarks/standalone-emulated-game-baseline.md`. Posted per-field MAPE to epic #516 and all child issues (#518-#522).

- **#521** — Standalone supply tracking. Fixed `SC2Data.supplyBonus` for town halls (NEXUS/CC=15, was 0). Added `SC2Data.unitSupplyProvided` for Overlord/Overseer (8). Added supply tracking to `injectReplayBuilding`/`syncReplayBuilding`/`markReplayBuildingComplete`. Unified Overlord supply from ZergRaceModel to universal path. foodMade: 37-57% → **0.9-4.4%** MAPE. foodUsed: 24-78% → **4-11%** MAPE.

### Issues filed

- **#525** (S/Med) — TrackerIntentExtractor UnitDied coverage. Root cause of foodUsed residual: extractor misses UnitDied for units whose UnitBorn wasn't tracked. Each missed death leaves stale supply in accumulator.
- **#526** (S/Med) — Harness seed-state deduplication. Root cause of foodMade residual: race model seeds initial units/buildings that overlap with GT injections. Currently worked around by zeroing supply after reset.

### Root cause analysis

| Residual gap | MAPE | Root cause | Fix issue |
|---|---|---|---|
| foodMade | 0.9-4.4% | Tick quantization + duplicate seeded buildings | #526 |
| foodUsed | 4-11% | Missed UnitDied intents in TrackerIntentExtractor | #525 |

### Decisions

- Supply tracking is now universal in EmulatedGame — removed race-model-specific Overlord supply from ZergRaceModel, replaced with `SC2Data.unitSupplyProvided` called at unit spawn/death for all races.
- Standalone test zeroes supply after `reset()` to avoid double-counting with GT-injected buildings. This is a workaround — #526 addresses the root cause.

## Active Plan

Epic #516 — Standalone EmulatedGame. Queue ordered for maximum independence:

1. #525 — TrackerIntentExtractor UnitDied coverage (S/Med) — quick win, reduces foodUsed residual
2. #526 — Harness seed-state deduplication (S/Med) — quick win, reduces foodMade residual
3. #522 — Standalone spending tracking (M/Med) — largest MAPE gap (40-87%)
4. #519 — Gas income ramp-up and worker assignment (M/Med) — 50-86% MAPE
5. #518 — Distance-aware mineral mining model (L/High) — 22-54% MAPE
6. #520 — Standalone worker lifecycle tracking (M/Med) — blocked by #518, #519
7. #523 — Standalone building lifecycle (L/High) — structural, eliminates syncBuildings
8. #524 — Final gate: remove all GT syncs (S/Low) — blocked by all above

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files
- **blizzard_ladder_4.9.2 reconstituted data deleted** — needs re-export
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — standalone accuracy (economy MAPE without GT syncs)
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — GT-synced accuracy (all 0.0%/100%)
- Epic #516 — standalone EmulatedGame issue tracker
- Epic #481 — economy fidelity (complete)
