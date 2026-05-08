---
layout: post
title: "StarCraft Agent — Tactics Gets a Brain"
date: 2026-04-08
---
# StarCraft Agent — Tactics Gets a Brain

**Date:** 2026-04-08
**Type:** phase-update

---

## What I wanted: GOAP, built on top of Drools

`TacticsTask` has been a stub since Phase 0. The agent can decide macro
strategy with Drools, manage the economy with Quarkus Flow, but unit
micromanagement — attack, retreat, kite, focus — has been a switch statement
on the strategy string.

The obvious next step was gdx-ai. It has behaviour trees, pathfinding,
steering — everything I'd need. But it's JVM-only, no GraalVM metadata, and
I've always wanted to build GOAP on top of Drools. This was the chance.

The question was what "GOAP on Drools" actually means architecturally.
Drools is a forward-chaining rule engine. GOAP classically uses A* to search
over a graph of world states. They don't compose obviously.

## The architecture that emerged: action compiler, not planner

I took the design question to Claude. We worked through three options before
landing on one.

The instinctive move is to use Drools at every A* search node — insert a
hypothetical world state, fire rules, see which actions are applicable. Clean
separation. It also requires cloning the Drools session per search node, which
is expensive and increasingly impractical at planning depth.

The better model: Drools fires **once per tick**. It classifies units into
groups — low-health, in-range, out-of-range — and emits a list of applicable
action names. Java parses that output into `GoapAction` records with
preconditions, effects, and cost. Then a pure Java A* planner searches over
`WorldState` clones using those records. Drools and A* are decoupled at the
`GoapAction` boundary.

Drools as **action compiler**, not planner. One session per tick, not one per
search node. GE-0105 is in the garden.

Goal assignment uses two levels: the strategic posture from `DroolsStrategyTask`
(ATTACK/DEFEND/MACRO) sets the army-level goal. Drools Phase 1 rules then
decompose it into per-group sub-goals based on each unit's actual situation.
That policy sits in DRL, so it's hot-reloadable without a restart — you can
tune tactical aggression without touching a line of Java.

## The DataStore trap

Building the two-phase DRL rules surfaced a constraint I hadn't anticipated.
Phase 1 classifies units and writes group IDs to a `List<String>`. Phase 2
should fire based on which groups exist. But Drools doesn't know the list
changed — it has no hook into plain Java collections. Phase 2 rules were
silently never re-evaluated.

The fix: Phase 1 also inserts group IDs into a `DataStore<String>` called
`activeGroups`. DataStore insertions trigger Drools agenda re-evaluation.
Phase 2 pattern-matches on that DataStore instead:

```drl
rule "Action: Retreat available"
    salience 110
when
    /activeGroups[ this == "low-health" ]
then
    actionDecisions.add("RETREAT:1");
end
```

GE-0109 is in the garden. The kind of thing that takes a morning to diagnose
if you don't know to look for it.

## Five tasks, one code quality catch

We implemented this through a subagent pipeline — five tasks dispatched to
fresh Claude instances, with a two-stage review after each: spec compliance
first, then code quality.

The code quality reviewer caught something real on Task 1. `WorldState` wasn't
truly immutable. The constructor took a `Map<String, Boolean>` without copying
it — any caller holding the original map could mutate the record's internals
silently. The implementation compiled. All tests passed. The contract was still
broken.

The fix was a compact constructor:

```java
public WorldState {
    conditions = Map.copyOf(conditions);
}
```

That's exactly why the two-stage review exists.

152 tests now. `DroolsTacticsTask` replaced `BasicTacticsTask` as the active
CDI bean. Three of four plugin seams are real. ScoutingTask is next.
