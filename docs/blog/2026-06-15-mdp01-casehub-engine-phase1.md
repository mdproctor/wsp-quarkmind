---
layout: post
title: "The Bridge Between Two Engines"
date: 2026-06-15
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [casehub, casehub-engine, migration, cdi, plugin-api, java]
---

The poc dependency had been on borrowed time for months. `casehub-core:1.0.0-SNAPSHOT` — the retired proof-of-concept — was still running every game tick, dispatching plugins via `createAndSolve()` and returning a `CaseFile` that every plugin read and wrote. The production engine uses `CaseContext` and a fundamentally different execution model. Getting there was a two-phase problem; we just shipped Phase 1.

Phase 1's scope was deliberate: migrate the plugin API without touching the dispatch infrastructure. Every plugin gets `execute(CaseContext)`, `activateIf()`, `requires()`, and `produces()` — but the poc's `createAndSolve()` still calls the plugins, just through bridges. The bridges aren't workarounds; they're the migration path. `execute(CaseFile)` delegates to `execute(CaseContext)` via a `CaseFileContext` adapter, then syncs any outputs back. The poc engine never knows anything changed.

The `activateIf()` / `requires()` split took some iteration to get right. The intent is clear: `requires()` declares which context keys must be present (the engine evaluates this first), and `activateIf()` adds extra gates — CDI-injected state like `StrategySelector.isSelected()` or whether the broker has a threat position. The mistake we caught in review was that every plugin's `activateIf()` was also re-checking the keys from `requires()`. Redundant and wrong: in Phase 2, the engine handles the key check; `activateIf()` only needs the extra gates. We cleaned that up across all eight plugins.

The bridge in `canActivate(CaseFile)` needed updating too. It was calling `activateIf()` directly, which, after stripping the key checks from `activateIf()`, would no longer gate on key presence at all. The fix was a default method on `TaskDefinition`:

```java
default boolean testActivation(CaseContext ctx) {
    return requires().stream().allMatch(ctx::contains) && activateIf().test(ctx);
}
```

Every bridge then becomes one line: `return testActivation(new CaseFileContext(caseFile))`. More importantly, it mirrors exactly what Phase 2's SequenceWorker will do — `requires()` first, then `activateIf()`. The bridge and the production path are semantically identical.

Two CDI surprises along the way.

First: adding `@CaseType("starcraft-game")` to `StrategyTrustRouter` broke its injection into `StrategyTrustObserver`. The `@CaseType` annotation carries `@jakarta.inject.Qualifier` — any custom qualifier removes `@Default` from the bean. `StrategyTrustObserver` injects `StrategyTrustRouter` without a qualifier, which expects `@Default`. The error named the concrete type and `[@Default]`, making it look like the bean didn't exist. Removed the annotation for Phase 1; it goes back in Phase 2 when `@CaseType` becomes plain metadata.

Second: adding `casehub-engine-blackboard` as a test dependency to get `CaseContextImpl` for unit tests triggered 34 CDI deployment failures. The blackboard looks like a model/API module from the outside — it has `DefaultCasePlanModel`, `PlanItem`, and so on. But it ships CDI beans that inject engine SPIs: `EventLogRepository`, `JobScheduler`, `CaseInstanceRepository`. None of those are on the classpath without the full engine runtime. The fix was to drop the dependency and use `CaseFileContext` for test construction instead. Same `CaseContext` interface, no engine runtime needed.

`StrategyTrustRouter` now implements `TaskDefinition` with `getId() = "trust-routing"` and writes the currently selected strategy ID to context. For now, `StrategyTrustObserver` still drives the selection logic — the `execute()` method just surfaces the result to context for observability. In Phase 2, when the SequenceWorker is available, the observer goes away and the sequence step takes over.

Phase 2 waits on two engine features: a bulk `signalAndAwaitSync()` that can update all game-state keys atomically and wait for quiescence, and a `SequenceWorker` with explicit skip-and-continue semantics. The plugin code is ready.
