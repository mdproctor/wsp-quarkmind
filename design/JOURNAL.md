# Design Journal — issue-193-casehub-engine-phase1-casefile-migration

### 2026-06-14 · §9.4·casehub-engine · #193

Phase 1 of the casehub-engine migration replaces the poc's `CaseFile` API with `CaseContext` across all plugin implementations. All four seam interfaces (`StrategyTask`, `TacticsTask`, `EconomicsTask`, `ScoutingTask`) now extend both `io.casehub.core.TaskDefinition` (poc, retained for Phase 1 bridge compatibility) and `io.quarkmind.agent.TaskDefinition` (QuarkMind's own). Every plugin implementation adds `execute(CaseContext)`, `activateIf()`, `requires()`, and `produces()` — the canonical API for Phase 2. Bridge methods (`execute(CaseFile)`, `canActivate(CaseFile)`, `entryCriteria()`) delegate to the new implementations so the existing poc `CaseEngine.createAndSolve()` path continues to work unchanged until Phase 2. This migration is purely additive: no plugin behaviour changes, no poc dependency removal; all 8 plugins and their test suites remain green.

### 2026-06-14 · §10 · #193

Two architectural decisions taken in Phase 1: (1) `io.quarkmind.agent.TaskDefinition` is QuarkMind's own plugin contract — not promoted to `casehub-engine-api` because it lacks cross-domain evidence. If AML, clinical, or devtown find it useful, it is promoted then. (2) `CaseFileContext` bridge adapter converts a poc `CaseFile` to `CaseContext` for the transition period by pre-extracting all 18 `QuarkMindCaseFile.ALL_KEYS` values at construction. It is explicitly labelled a Phase 1 shim — only used in bridge methods, deleted in Phase 2. `STRATEGY_SELECTED_ID = "agent.strategy.selected.id"` was added to `QuarkMindCaseFile` as a constant (not a raw literal) so Phase 2's `SequenceWorker` activation predicates can reference it compile-safely.

### 2026-06-14 · §9.4·Trust Routing · #193

`StrategyTrustRouter` now implements `io.quarkmind.agent.TaskDefinition` with `getId() = "trust-routing"` and `execute(CaseContext ctx)` that writes the currently selected strategy ID to context. This is a preparatory step — in Phase 1, `StrategyTrustObserver` (CDI event observer) still drives selection at game start and mid-game checkpoint; the new `execute()` just writes the result to context for observability. In Phase 2, when `SequenceWorker` (engine#484) is available, the observer is deleted and the sequence step takes over entirely, making trust routing structural and event-log-visible rather than hidden inside a CDI observer.
