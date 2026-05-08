---
layout: post
title: "QuarkusMind — Making the Bot Play"
date: 2026-04-09
---
# QuarkusMind — Making the Bot Play

**Date:** 2026-04-09
**Type:** phase-update

---

## What I was trying to achieve: closing the loop from intent to game

Four plugins working. All four emit intents from real game state — `BuildIntent`, `TrainIntent`, `AttackIntent`, `MoveIntent`. But `SC2BotAgent.onStep()` drained the queue and logged every entry as `[SC2] Intent (Phase 1 no-op)`. The bot could observe StarCraft II and reason about what to do. It couldn't do anything.

The goal was to change that.

## What I assumed going in: this would be the easy part

After three sessions building Drools rule units, Quarkus Flow, a GOAP planner, and CEP event buffers, a mapping table from `BuildingType` to `Abilities.BUILD_PYLON` felt like a rest stop. I was wrong about the timeline.

## The design that made testing free

Before writing a line of implementation I made one choice that shaped everything: `ActionTranslator.translate()` would return `List<ResolvedCommand>` rather than calling `ActionInterface` directly.

The ocraft `ActionInterface` has twelve-plus `unitCommand` overloads. Testing a class that calls it directly requires mocking — and mocking twelve overloads is enough to make you reconsider writing the test. By making the translator a pure function that returns data, the test is plain AssertJ with no mocking framework at all:

```java
var commands = ActionTranslator.translate(List.of(new BuildIntent("456", GATEWAY, p)));
assertThat(commands.get(0).ability()).isEqualTo(Abilities.BUILD_GATEWAY);
assertThat(commands.get(0).target()).contains(Point2d.of(30f, 40f));
```

`SC2BotAgent.onStep()` applies the results with a two-overload `ifPresentOrElse` — position-targeted intents get one overload; train commands (building → unit, no position) get the other.

I brought Claude in to implement this. We built fourteen tests across three groups — ability maps, dispatch, error cases — before writing any implementation. The full roster: nine `BuildingType` mappings, eleven trainable `UnitType` mappings including the `ARCHON` null case (two-Templar merge; single-tag `TrainIntent` can't express it), four intent dispatch assertions, five error and edge cases.

## The CaseHub surprise: three failures nested inside one

Halfway through, we ran the full test suite. Zero of 173 tests compiled.

The error pointed at `DefaultCaseFile`. CaseHub had been refactored: the constructor changed, the class moved from `casehub-core` to a new `casehub-persistence-memory` module, and that module had no Jandex index — so Quarkus's CDI scanner silently ignored all its `@ApplicationScoped` beans. Three distinct failures nested inside one `Unsatisfied dependency` message.

The fix was three parts. Update eight test files to use `InMemoryCaseFile`. Add `casehub-persistence-memory` as a dependency. And add this to `application.properties`:

```properties
quarkus.index-dependency.casehub-persistence-memory.group-id=io.casehub
quarkus.index-dependency.casehub-persistence-memory.artifact-id=casehub-persistence-memory
```

Without the third part, the Jandex-less jar is invisible to CDI even with every bean correctly annotated. `quarkus.index-dependency` is documented — in the class loading reference, not the CDI guide. Everyone looks in the CDI guide first.

One more: `mvn install -DskipTests` on a Quarkus library module still runs augmentation. `mvn install -DskipTests -Dquarkus.build.skip=true` is what you actually need.

## What the code reviewer caught: the unsealed Intent

With ActionTranslator working and all fourteen tests green, a final code review found one thing worth acting on. Claude flagged it: `Intent` was a plain interface, not sealed. The `translate()` switch had a `default` arm that warned on unknown types — which sounds defensive but is actually a silent miss. Add a new intent subtype later and the compiler won't notice that `ActionTranslator` needs updating.

Sealing it removes the `default` arm entirely and converts that future runtime warning into a compile error. Five lines, but it changes the failure mode from silent to loud.

## Where we are

187 tests pass. The bot can connect to SC2, observe game state, run all four plugins, and issue real commands. The live smoke test requires an actual SC2 process — that's next when one is available. The pipeline from observation to action is end to end.
