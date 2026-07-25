# Design Journal — issue-243-tactic-taxonomy

## §1 Domain Model Renames (2026-07-22 → 2026-07-25)

Three rename waves completed:
1. `EnemyArchetype` → `StrategyArchetype` — player-agnostic, gained `GamePhase phase` and `ArchetypeCategory category` fields
2. `EnemyPatternAssessment` → `PatternAssessment` — `ScoutingIntelPayload.PatternAssessment` inner type renamed to `PatternAssessmentPayload` to avoid collision
3. `summarisation.GamePhase` → `TacticalPosture` — resolved naming collision with new domain `GamePhase` enum. Record component `phase` → `posture`. Rule unit stores renamed: `phaseStore` → `tacticalPostureStore`, `postureStore` → `enemyPostureStore`

DRL files manually updated — `.name().contains("RUSH")` patterns replaced with `.category() == ArchetypeCategory.RUSH`.

**Design decision:** DominanceWeightRuleUnit `tacticalPostureStore` kept as `DataStore<String>` rather than upgrading to `DataStore<TacticalPosture>`. The type upgrade would cascade through `WeightContext`, `SituationalDominanceWeightStrategy`, and many tests for no functional gain at this stage.

## §2 YAML Strategy Taxonomy (2026-07-25)

`strategy-taxonomy.yaml` created with all 10 existing archetypes (`handAuthored: true`). `StrategyTaxonomy` CDI bean loads at startup with fail-fast validation — every enum value must have a YAML entry and vice versa.

New domain records: `CounterInfo`, `CounterEntry` for structured counter data (Protoss perspective). `StrategyTaxonomy.countersFor()` provides the standalone counter API.

## §3 Upstream API Breakage

Two rounds of upstream casehub-platform SNAPSHOT breaks required fixing:
1. `WorkerFunction.Sync` gained `outputType` + `BiFunction` signature; `SettingsScope.root()` gained `tenancyId`
2. `CaseHubRuntime.signalAndAwaitSync()` → `signalAndAwait()`; `startCase()` returns `UUID` directly; `signal()` returns `void`; `Preference.toSerializedValue()` added

A third issue (#262) — `ReactiveAgentIdentityVerificationService` CDI bean removed from platform — blocks all `@QuarkusTest` integration tests. Filed but unresolved.
