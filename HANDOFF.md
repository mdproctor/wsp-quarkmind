# HANDOFF — quarkmind

## Last Session

Branch `issue-380-restoration-coverage-audit` — **closed**, landed on main as `c07c8c8f` (3 squashed commits from 7).

### What happened

Completed three issues from the Phase 2.5 epic (#366):

**#380 — Restoration coverage audit:**
- New `RestorationCoverageAuditTest` (`@Tag("report")`) scans all 13 replay datasets (152,109 replays)
- Results: 632 (0.4%) route through TrackerEventFeatureExtractor (ground-truth), 151,477 (99.6%) use StrippedReplayFeatureExtractor fallback
- Report at `docs/benchmarks/restoration-coverage.md`

**#371 — Cross-patch regression suite:**
- Extended `DivergenceRegressionTest` with economy MAPE category and cross-patch baselines
- Calibrated for AI Arena (79% units), IEM PyeongChang (22%), ASUS ROG (22%)

**#381 — Race hardcoding fix:**
- `ReplayValidationHarness` hardcoded `ProtossRaceModel` — two-line fix tripled accuracy (15% → 46.8%)

### Current accuracy baselines (5-min checkpoint, 118 oracle replays)

| Category | Value | Notes |
|----------|-------|-------|
| Units | 46.8% | Post-race-fix |
| Buildings | 100.0% | Harness-synced from GT |
| Upgrades | 0.0% | No ResearchIntents applied |
| Economy | 375.8% MAPE | Measured across all races |

### Suggested next work

`work start #372` — Re-reconstitute training data (pipeline execution, ~2 hours).

## References

| What | Where |
|------|-------|
| Commits on main | `c07c8c8f` (3 squashed) |
| Coverage report | `docs/benchmarks/restoration-coverage.md` |
| Epic status | `casehubio/quarkmind#366` |
