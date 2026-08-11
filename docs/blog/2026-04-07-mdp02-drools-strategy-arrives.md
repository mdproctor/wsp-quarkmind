---
layout: post
title: "StarCraft II Quarkus Agent — Drools Strategy Arrives"
date: 2026-04-07
---
# StarCraft II Quarkus Agent — Drools Strategy Arrives

**Date:** 2026-04-07
**Type:** phase-update

---

## What I was trying to achieve: the first real R&D integration

The whole point of this project is to use StarCraft II as a testbed for Drools, Quarkus Flow, and CaseHub. Every plugin so far has been hand-coded Java — good discipline, but not the experiment. Drools `StrategyTask` was the first real target: replace `BasicStrategyTask` with DRL rules.

## What I believed going in: two hours of wiring, then the interesting part

The library research had this pinned. `org.drools:drools-quarkus`, BOM at 10.1.0, Rule Units for native-compatible AOT compilation. I expected the rules themselves to be where the work was.

That was not accurate.

## Four things that bit us before the rules ran

I brought Claude in for the implementation. We added `drools-quarkus`, `drools-bom`, and `drools-ruleunits-api` to the pom.xml, wrote `StrategyRuleUnit implements RuleUnitData`, the DRL file, and `DroolsStrategyTask`. Compiled clean.

First Quarkus build error: `[ERR 101] Line 21:18 no viable alternative at input ']'`. The pattern `$b: /builders[]` — empty brackets — isn't valid OOPath. You write `/builders` with no brackets at all for a no-constraint match. Empty brackets are a parse error.

That fixed, the next: `type` is a reserved keyword in DRL. Writing `buildings[ type == BuildingType.GATEWAY ]` fails because the parser sees a keyword, not a property accessor. Same for `isComplete` — Drools applies JavaBean convention, mapping `isComplete()` to property `complete`, not `isComplete`. The fix for both: `this.method()` explicitly. `buildings[ this.type() == BuildingType.GATEWAY, this.isComplete() == true ]`.

Two down, two to go.

## The CaseHub gap: nobody home all along

Running the Quarkus build with DEBUG logging revealed something that had been wrong since the first plugin was written:

```
Removed unused CLASS bean [..., qualifiers=[@Any, @CaseType("starcraft-game")], target=org.acme.starcraft.plugin.BasicStrategyTask]
Removed unused CLASS bean [..., qualifiers=[@Any, @CaseType("starcraft-game")], target=org.acme.starcraft.plugin.BasicEconomicsTask]
```

All four `@CaseType` plugin beans were being removed by Quarkus Arc's dead-bean elimination. Nothing in the application was injecting them through CDI. `TaskDefinitionRegistry` was always empty. Every game tick called into CaseHub, CaseHub found no tasks, and returned immediately. The plugins had never actually run.

The fix: `StarCraftTaskRegistrar` is a startup bean that explicitly injects all four plugin seam interfaces. The injection keeps Arc from discarding the beans; the registrar then registers each with `TaskDefinitionRegistry`. CaseHub now runs the plugins.

## The classloader wall

Moving to the test phase, plain JUnit hit this immediately:

```
NullPointerException: Cannot invoke "DataSource$Factory.createStore()"
because "DataSource$FactoryHolder.get()" is null
```

`DataSource.createStore()` is initialized by the Quarkus extension at build time. Without Quarkus context, the factory is null. Drools Rule Unit tests require `@QuarkusTest`.

Adding `@QuarkusTest`, Claude came back with `ExceptionInInitializerError` during startup, caused by `ClassNotFoundException: org.acme.starcraft.agent.ResourceBudget`. The generated `StrategyRuleUnitRuleUnit_Bean` static initializer calls `drools-ruleunits-impl`'s `SimpleRuleUnitVariable`, which scans every field type of `StrategyRuleUnit` and calls `Class.forName()` on each. The plain fields `ResourceBudget budget` and `IntentQueue intentQueue` — application classes — weren't visible through the Drools classloader at that point.

The solution: `DataStore<T>` fields erase to `DataStore` at runtime — only the raw type gets loaded. `DataStore<ResourceBudget>` is safe; `ResourceBudget budget` is not. We redesigned `StrategyRuleUnit` to hold only JDK and Drools-known types as plain fields. Rules write string decisions to a `List<String>` — `"GATEWAY"`, `"CYBERNETICS_CORE"`, `"STALKER:gw-0"` — and `DroolsStrategyTask.execute()` reads them after `fire()` and handles budget and intent dispatch in Java.

The rules look like this:

```drl
rule "Build Gateway"
    salience 100
when
    /builders
    not /buildings[ this.type() == BuildingType.GATEWAY ]
    /buildings[ this.type() == BuildingType.PYLON, this.isComplete() == true ]
then
    buildDecisions.add("GATEWAY");
end
```

Declarative conditions. Budget and intents stay in Java. The separation is clean and testable.

## 107 tests

The session started at 94 tests. It ended at 107, all passing. The 13 new tests are in `DroolsStrategyTaskTest` — `@QuarkusTest`, injecting the active `StrategyTask` bean, calling `execute(CaseFile)` directly with crafted CaseFiles. `BasicStrategyTask` stays as a plain class for direct-instantiation testing, making the Drools tests the integration layer and the Basic tests the logic layer.

Two garden entries came out of the session: GE-0053 for the `DataSource.createStore()` plain-JUnit trap, and GE-0056 for the DRL OOPath gotchas — empty brackets, the `type` keyword clash, and the `isComplete` property naming convention. Three problems, one fix: `this.method()` throughout.

Quarkus Flow `EconomicsTask` is next.
