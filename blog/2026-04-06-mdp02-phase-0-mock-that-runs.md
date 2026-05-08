---
layout: post
title: "StarCraft II Quarkus Agent — Phase 0: The Mock That Runs"
date: 2026-04-06
---
# StarCraft II Quarkus Agent — Phase 0: The Mock That Runs

**Date:** 2026-04-06
**Type:** phase-update

---

## What we were trying to achieve: a fully operational architecture, without StarCraft II

Phase 0 had one goal: get the entire plugin stack working against a simulated game. No real SC2 required. Sixteen tasks, ten of them implementation, the rest review. By the end: 27 tests passing, CaseEngine cycling through four dummy plugins every 500ms, a QA REST harness for observing, directing, and interacting with the running agent.

We built it using subagent-driven development — each task dispatched to a fresh Claude instance, then reviewed by two more (spec compliance, then code quality) before moving on. The reviews caught more than I expected.

## The mock that became a specification: SimulatedGame

![SC2 Zealot](/quarkmind/blog/assets/sc2-zealot.jpg)

`SimulatedGame` sits at the centre of it all. It's a stateful, thread-safe Protoss game: 12 Probes, 1 Nexus, 50 minerals, 15/12 supply. It receives typed intents — `TrainIntent`, `BuildIntent`, `AttackIntent` — and mutates its own state. A `tick()` advances the game clock and completes one queued build action.

Four concrete test scenarios sit alongside it: `spawn-enemy-attack`, `set-resources-500`, `supply-almost-capped`, `enemy-expands`. Named mutations today; later the same names will trigger SC2 debug API calls against a real game. The mock is the specification, and the specification accumulates.

Code review found three synchronisation gaps. `reset()`, `snapshot()`, and `spawnEnemyUnit()` all needed `synchronized`. The `snapshot()` problem was the subtle one — five separate volatile field reads are individually safe but not atomic as a group:

```java
// BROKEN — each read is safe; the combination is not
public GameState snapshot() {
    return new GameState(minerals, vespene, supply, supplyUsed,
        List.copyOf(myUnits), List.copyOf(myBuildings),
        List.copyOf(enemyUnits), gameFrame.get());
}

// FIXED — all reads atomic within one critical section
public synchronized GameState snapshot() {
    return new GameState(minerals, vespene, supply, supplyUsed,
        List.copyOf(myUnits), List.copyOf(myBuildings),
        List.copyOf(enemyUnits), gameFrame.get());
}
```

A concurrent `tick()` between any two reads produces an inconsistent snapshot. Tests pass single-threaded. Production would have failed quietly.

## The annotation nobody documented: `@CaseType`

Wiring CaseHub's `CaseEngine` into the game loop was mostly mechanical. One thing wasn't.

`createAndSolve("starcraft-game", gameData, Duration.ofSeconds(5))` activates only `TaskDefinition` beans annotated with `@CaseType("starcraft-game")`. Without it, no plugins are found. The CaseEngine runs, nothing activates, the case quiesces silently.

The implementer subagent discovered this by reading the CaseHub source during Task 8. It's not documented anywhere in CaseHub yet — it went straight to the knowledge garden. Every pass-through plugin now carries it:

```java
@ApplicationScoped
@CaseType("starcraft-game")   // without this, CaseEngine can't find you
public class PassThroughStrategyTask implements StrategyTask {

    @Override public Set<String> entryCriteria() { return Set.of(StarCraftCaseFile.READY); }
    @Override public Set<String> producedKeys()  { return Set.of(); }

    @Override
    public void execute(CaseFile caseFile) {
        log.debugf("[STRATEGY] frame=%s minerals=%s",
            caseFile.get(StarCraftCaseFile.GAME_FRAME, Long.class).orElse(-1L),
            caseFile.get(StarCraftCaseFile.MINERALS, Integer.class).orElse(0));
    }
}
```

## Four plugin levels — the architecture that runs

![Plugin Architecture](/quarkmind/blog/assets/architecture-plugins.png)

*The four plugin levels and their hook points — all share CaseFile and IntentQueue as passive shared substrate.*

## What the research surfaced

After Phase 0 was working, we ran parallel research across the ecosystem — SC2 Java libraries, agent frameworks, Quarkus extensions. Two findings are worth noting here.

Drools has moved to the Apache Software Foundation as Apache KIE, version 10.1.0. The Maven coordinates shifted from `org.kie:*` to `org.drools:drools-quarkus-ruleunits:10.1.0`. More importantly: it now supports GraalVM native image via the Executable Model — rules compiled at build time, no runtime bytecode generation. That's the Phase 3 plugin.

The other finding was less encouraging. ocraft-s2client — the only Java SC2 API client — is a native image blocker. Its RxJava, Netty, and protobuf stack need extensive `reflect-config.json` entries. The SC2 protobuf schema alone has hundreds of generated message classes. The path forward is the GraalVM tracing agent run during a live game session, accumulating coverage across maps and races. Phase 1 task.

## What's there now

The architecture is operational. The QA endpoints — `GET /sc2/casefile`, `POST /sc2/debug/scenario/{name}`, `GET /sc2/intents/pending` — all work, all stripped from production via `@UnlessBuildProfile("prod")`. CaseHub cycles, the four dummy plugins fire and log what they see, the intent queue drains to the mock dispatcher.

Phase 1 is connecting it to a real game.

---
*SC2 unit image © Blizzard Entertainment — used for editorial purposes.*
