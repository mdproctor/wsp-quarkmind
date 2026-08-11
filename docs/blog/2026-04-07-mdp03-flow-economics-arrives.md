---
layout: post
title: "StarCraft II Quarkus Agent — Flow Economics Arrives"
date: 2026-04-07
---
# StarCraft II Quarkus Agent — Flow Economics Arrives

**Date:** 2026-04-07
**Type:** phase-update

---

## What I was trying to achieve: Quarkus Flow as the second R&D integration

With Drools running strategy, the next target was Quarkus Flow for economics. The library research had flagged them as complementary — CaseHub for per-tick reactive decisions, Flow for multi-step orchestration. I wanted to find out if that boundary held in practice.

## Design before code: three options, one ADR

The Drools session had jumped to implementation fairly quickly. This time I front-loaded the thinking. I brought Claude in to brainstorm the architecture first. We worked through three distinct placements for Flow in the system — per-tick blocking (Option A), stateful multi-tick within the plugin seam (Option B), and Flow as match lifecycle orchestrator (Option C). Option B won, documented in ADR-0001 alongside the other two options so the decision can be revisited.

We also expanded the economics scope. Basic had covered probes and pylons. But Protoss economics has a natural workflow shape: probe saturation → gas transition → expansion → multi-base. Each phase has clear entry conditions. That's a workflow problem, not a rules problem. So we widened the scope to all four phases, transferred assimilator building from Drools (where it had always been economically misplaced), and added expansion signalling.

Then spec, plan, subagent-driven implementation.

## The listen task that didn't listen

Most of the implementation went cleanly. Task 7 — the Flow itself — didn't.

The plan called for `listen("tick", toOne("org.acme.starcraft.economics.tick"))` to receive ticks from the SmallRye in-memory channel. A subagent came back: `listen` doesn't receive from SmallRye. The `toOne(type)` argument is a CloudEvent type, not a channel name. SmallRye in-memory carries plain Java objects. The two worlds don't connect.

The workaround, once you know it, is clean:

```java
@Incoming("economics-ticks")
public Uni<Void> processTick(GameStateTick tick) {
    return startInstance(tick).replaceWithVoid();
}
```

Declare `@Incoming` on the `Flow` subclass and call `startInstance(payload)`. Each message starts a fresh workflow instance that runs the FuncDSL `consume` pipeline. This shifts the model from the spec's long-lived stateful instance to per-tick instances — functionally equivalent for the current scope, and worth revisiting once Drools CEP and match lifecycle work begins.

An earlier subagent had already hit a related constraint when wiring the emitter: SmallRye requires a downstream consumer for every `@Channel` emitter at boot time. It created a drain stub unprompted to keep the build healthy. The stub was exactly right — it got replaced when the real flow landed.

## Jackson makes consume steps independent

The second surprise came in the integration test phase. Each `consume` step in a Quarkus Flow `descriptor()` receives a fresh Jackson deserialization of the original workflow input. Mutations don't propagate.

In practice: `ResourceBudget.spendMinerals(100)` in `checkSupply` doesn't affect what `checkProbes` sees. Every step starts from the original mineral count. This is silent — no error, tests pass, but cross-step budget arbitration within a single tick simply doesn't work. SC2 rejects unaffordable commands at the game layer anyway, so nothing breaks in play. But the assumption was wrong.

Also: passing a plain mutable Java class as workflow input throws `FAIL_ON_EMPTY_BEANS` at runtime — Jackson can't serialize it. `GameStateTick` became a record. `ResourceBudget` got `@JsonAutoDetect(fieldVisibility = ANY)`.

## 128 tests, four garden entries

The session started at 107. We ended at 128 — 16 new unit tests in `EconomicsDecisionServiceTest`, 4 integration tests in `EconomicsFlowTest` that emit a tick and assert the queue after 300ms.

Four garden entries: GE-0059 (consume steps re-deserialize input), GE-0060 (POJO `FAIL_ON_EMPTY_BEANS`), GE-0061 (`listen` silent with SmallRye), GE-0062 (`@Incoming` + `startInstance` bridge). First quarkus-flow entries in the garden.

Tactics next. gdx-ai behaviour trees or a GOAP planner.
