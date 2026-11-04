# HANDOFF — quarkmind

## Last Session (2026-11-04)

Completed #525 (UnitTypeChange handling in TrackerIntentExtractor). Filed #527 for the real foodUsed root cause (Zergling half-supply).

### What landed

- **#525** — TrackerIntentExtractor UnitTypeChange handling. Added `ID_UNIT_TYPE_CHANGE` handler to update `unitTagToType` when units morph or switch modes (SiegeTank→Sieged, Hellion→Hellbat, Roach→Ravager, etc.). Added 9 missing tracker type name mappings. Missed supply reduced from 1,437 to 475 (67% reduction). Added `UnitDiedCoverageDiagnosticTest` and morph death type accuracy test.

### Root cause correction

Investigation found that the foodUsed 4-11% MAPE overcount is **not** caused by missed UnitDied intents. UnitDied coverage diagnostic shows all supply-relevant deaths ARE captured (remaining gaps: Larva with 0 supply, Drone building morphs correctly skipped). The real root cause is **Zergling half-supply**: SC2 Zerglings cost 0.5 supply, our `supplyCost(ZERGLING)` = 1. Each Zergling adds 0.5 excess supply. Filed as #527.

Updated root cause table:

| Residual gap | MAPE | Root cause | Fix issue |
|---|---|---|---|
| foodMade | 0.9-4.4% | Tick quantization + duplicate seeded buildings | #526 |
| foodUsed | 4-11% | Zergling/Baneling half-supply (0.5 vs 1.0) | #527 |

### Issues filed

- **#527** (M/Med) — Half-supply tracking for Zergling/Baneling. Root cause of foodUsed 4-11% overcount. Switch to half-supply representation or use double for supply fields.

## Active Plan

Epic #516 — Standalone EmulatedGame. Queue updated:

1. #527 — Half-supply tracking for Zergling/Baneling (M/Med) — root cause of foodUsed overcount
2. #526 — Harness seed-state deduplication (S/Med) — reduces foodMade residual
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
