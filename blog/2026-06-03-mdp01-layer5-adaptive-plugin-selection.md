---
layout: post
title: "Layer 5: The Gate That Works"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [casehub-engine, adaptive-dispatch, binding-conditions, l5]
---

Layer 5 is done. `TacticsTask` now activates only when scouting has found a threat
and strategy has set a course of action. In the early game — no enemy contact — tactics
never runs.

The mechanism is `entryCriteria()` on each plugin. The CaseEngine checks which keys are
present in the CaseFile before scheduling a task; any missing key means the task is
skipped. So `TacticsTask` declaring `{READY, STRATEGY, NEAREST_THREAT}` as its criteria
means it sits out until scouting writes `NEAREST_THREAT`. Scouting only writes that when
enemies are visible. The gate is real.

`StrategyTask` got `{READY, ENEMY_ARMY_SIZE}` — not a genuine gate (scouting always
writes `ENEMY_ARMY_SIZE`, even as 0), but an ordering commitment. Strategy runs after
scouting every tick. When C2 lands and strategy migrates to scouting-derived intel
(`ENEMY_POSTURE`, `ENEMY_BUILD_ORDER`) instead of raw `ENEMY_UNITS`, the ordering
becomes a proper data dependency. The stub read is in both strategy implementations,
tracked as #169.

Two things the foundation didn't do that we had to work around.

First: `TaskDefinition.canActivate()` in the installed snapshot returns `true`
unconditionally. The interface has the method; the default implementation ignores
`entryCriteria()` entirely. Bytecode confirmed: `iconst_1; ireturn`. Updating
`entryCriteria()` alone had no runtime effect. We overrode `canActivate()` explicitly
on all four affected plugins — the correct implementation the default should have had.
When casehub-core fixes the default, the overrides are removable. Garden entry filed
(GE-20260603-e98633). Protocol written.

Second: `createAndSolve()` returns the CaseFile immediately — before plugins have run.
The duration parameter goes to a `PropagationContext` timeout, not to a blocking await.
So the CaseFile we get back from the tick contains only translator-written keys
(`MINERALS`, `ARMY`, `ENEMY_UNITS`). Plugin-written keys (`NEAREST_THREAT`, `STRATEGY`)
are absent. Garden entry (GE-20260603-a944d1). Tests use `canActivate()` directly on
the injected CDI beans to verify gate semantics — cleaner than waiting on async solve
completion, and it tests the right thing.

Also extracted `GameTickExecutor` from `AgentOrchestrator.gameTick()`. The original
discarded the `CaseFile` returned by `createAndSolve()`. Now `AgentOrchestrator` exposes
`getLastTickResult()` — the test surface the integration tests needed.

The distinction between ordering dependencies and conditional gates turned out to be
important enough to document explicitly in `ARC42STORIES.MD` and the LAYER-LOG. They
look identical in the code (`entryCriteria()` returning a set of keys) but serve
different architectural purposes. `ENEMY_ARMY_SIZE` on strategy is a sequencing
constraint that will become a data dependency at C2. `NEAREST_THREAT` on tactics is
a genuine adaptive gate today.
