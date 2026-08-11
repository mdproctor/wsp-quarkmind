---
layout: post
title: "StarCraft II Quarkus Agent — Phase 1: Wiring Real SC2"
date: 2026-04-06
---
# StarCraft II Quarkus Agent — Phase 1: Wiring Real SC2

**Date:** 2026-04-06
**Type:** phase-update

![Protoss Stalker](/quarkmind/blog/assets/sc2-stalker.jpg)

---

## What we were trying to achieve: prove the real SC2 wiring matches the mock contract

Phase 1 had one job: implement `sc2.real` — the real ocraft-backed implementations of the three CDI interfaces (`SC2Client`, `GameObserver`, `CommandDispatcher`) — and prove they satisfy the same contracts as the mock. No intelligence changes. No new plugin logic. Just swap the implementation family and see if the pipeline still runs.

We used the same subagent-driven approach as Phase 0: fresh Claude instance per task, two-stage review (spec then code quality) before moving on.

## The two-loop architecture — and why it matters

Before writing a line of code I had to settle one architectural question. ocraft-s2client is callback-driven: `S2Agent.onStep()` fires every SC2 frame (~22fps) and you *must* send commands within that callback. Our `@Scheduled` intelligence pipeline runs at 500ms. These two clocks can't share a thread.

The solution: `IntentQueue` as the handoff. The `@Scheduled` pipeline reads game state, runs CaseEngine, and writes typed intents to the queue. ocraft's `onStep()` drains the queue and sends those intents to SC2 — within the callback, as required. Two independent loops, one thread-safe queue between them.

`RealCommandDispatcher.dispatch()` is a no-op. The dispatching happens in `onStep()`, not in our scheduler. This was the clearest evidence the architecture was right: the mock dispatcher mutated `SimulatedGame` directly, and the real dispatcher doing nothing was the correct behaviour.

## What ocraft actually looks like

The plan was written from documentation and the Phase 0 mock. Implementation found four surprises in the actual 0.4.21 API.

`obs.getUnits()` returns `List<UnitInPool>` — not `List<Unit>`. The actual unit is accessed via `.unit()` on the wrapper. `unit.getType()` returns a `UnitType` interface, not the `Units` enum. `unit.getHealth()` and `getHealthMax()` return `Optional<Float>`. `unit.getTag()` returns a `Tag` object needing `.getValue()`.

The translation ends up looking like this:

```java
for (UnitInPool uip : obs.getUnits()) {
    Unit unit = uip.unit();

    // getType() returns UnitType interface — pattern match to get the enum
    Units unitEnum = (unit.getType() instanceof Units u) ? u : Units.INVALID;

    // getHealth()/getHealthMax() return Optional<Float>
    int health    = (int) unit.getHealth().orElse(0f);
    int maxHealth = (int) unit.getHealthMax().orElse(0f);

    // getTag() returns Tag wrapper — call getValue()
    String tag = String.valueOf(unit.getTag().getValue());
}
```

None of these are in the ocraft README. All four surprises are now in the knowledge garden as one entry.

## The debug restriction that changed SC2BotAgent

The most interesting discovery came from `SC2DebugScenarioRunner`. The plan said: get the `SC2BotAgent` reference and call `agent.debug().debugCreateUnit(...)`. The subagent hit a runtime NPE immediately.

`agent.debug()` accesses an internal `controlInterface` that is only initialised inside the ocraft game loop. Calling it from a REST handler — even after the game has started — throws a NullPointerException with no helpful message.

The fix emerged cleanly:

```java
// SC2BotAgent — queue commands externally, flush in onStep()
private final ConcurrentLinkedQueue<Runnable> pendingDebugCommands
    = new ConcurrentLinkedQueue<>();

public void enqueueDebugCommand(Runnable command) {
    pendingDebugCommands.add(command); // safe from any thread
}

@Override
public void onStep() {
    Runnable cmd;
    boolean hadCommands = false;
    while ((cmd = pendingDebugCommands.poll()) != null) {
        cmd.run();       // debug() is safe here — we're in the game loop
        hadCommands = true;
    }
    if (hadCommands) debug().sendDebug();
    // ... observation storage, intent dispatch ...
}

// REST handler / scenario runner — no knowledge of onStep() required:
agent.enqueueDebugCommand(() ->
    agent.debug().debugCreateUnit(Units.PROTOSS_STALKER, 1, Point2d.of(22f, 20f), 2));
```

The queue is the interface. The caller doesn't need to know about the constraint. Claude flagged this as DONE_WITH_CONCERNS, which is exactly what that status is for.

One remaining limitation: ocraft 0.4.21's `DebugInterface` has no `debugSetMinerals(int)`. Only `debugGiveAllResources()` exists at the high-level API. The `set-resources-500` scenario maxes out resources rather than setting exactly 500 minerals. Precise resource setting would require raw protobuf — deferred.

## What's there now

Seven classes in `sc2/real/`. All mock beans gained `@UnlessBuildProfile("sc2")` — in the sc2 profile, exactly one implementation of each interface is active. `AgentOrchestrator` uses `Instance<SimulatedGame>` so it compiles in both profiles. All Phase 0 tests still pass.

Task 9 is still open: start SC2, run `mvn quarkus:dev -Dquarkus.profile=sc2`, and watch it connect. That one requires actual hands on a keyboard.

---
*SC2 unit image © Blizzard Entertainment — used for editorial purposes.*
