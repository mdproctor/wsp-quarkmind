# HANDOFF — quarkmind

## Last Session

Branch `issue-367-oracle-accuracy-baseline` — **landed on main** (4 squashed commits, ff-merge).

### What was built

1. **Oracle accuracy baseline (#367)** — `OracleAccuracyBaselineTest` measuring StrippedReplayFeatureExtractor accuracy across 118 oracle replays. T1-T4 observability tier model. Calibrated MULE abilLink (171→90), suppressed WarpGate phantom UnitInit.

2. **CmdEvent detection gap investigation (#374)** — Root cause found: Blizzard's ladder replay API downloads contain only ~43% of production CmdEvents. The production building multiplier is the correct compensating mechanism, not a hack. `CmdEventGapDiagnosticTest` documents the evidence.

3. **TrackerEventFeatureExtractor (#377)** — Reads UnitBorn/UnitInit/UnitDone/UnitDied/Upgrade/PlayerStats directly from restored replay tracker events. 100% ground-truth accuracy by definition. Auto-detecting `ReplayFeatureExtractor` wrapper routes to tracker path when available, falls back to stripped path. Separate `gameCommands` array for movement/order data from CmdEvents.

4. **Comparison report** — `TrackerVsStrippedComparisonTest`: tracker=29,625 vs stripped=23,150 UnitBorn events (78.1%). Major stripped distortions eliminated: Baneling 0%→100%, Sentry 910%→100%, Probe 74.7%→100%.

### Issues closed

| # | Title | Resolution |
|---|-------|------------|
| 367 | Oracle accuracy baseline | Done |
| 373 | Unit/building extraction accuracy | Done (multiplier deferred as correct) |
| 374 | CmdEvent detection gap | Root cause: Blizzard API data limitation |
| 375 | Baneling morph detection | Superseded by #377 |
| 376 | Per-type accuracy metric | Superseded by #377 |
| 377 | TrackerEventFeatureExtractor | Done |

### Suggested next work

**EmulatedGame physics calibration** — the replay validation harness now compares against ground truth instead of ~78% approximation. Running `DivergenceBaselineReportTest` will surface real emulator divergences previously masked by extraction noise.

## References

| What | Where |
|------|-------|
| Commits on main | `08e746c3`, `217b2eea`, `2b8e3dd7`, `2e74afb9` |
| TrackerEventFeatureExtractor | `quarkmind-sc2/.../replay/TrackerEventFeatureExtractor.java` |
| ReplayFeatureExtractor | `quarkmind-sc2/.../replay/ReplayFeatureExtractor.java` |
| Comparison test | `TrackerVsStrippedComparisonTest.java` |
| CmdEvent gap diagnostic | `CmdEventGapDiagnosticTest.java` |
| Oracle accuracy baseline | `OracleAccuracyBaselineTest.java` |
| Baseline report | `docs/benchmarks/oracle-accuracy-baseline.md` |
