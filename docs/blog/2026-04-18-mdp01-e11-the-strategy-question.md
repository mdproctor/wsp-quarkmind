---
layout: post
title: "E11: The Strategy Question"
date: 2026-04-18
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkus, cdi, tactics, strategy-pattern]
---

Before touching E11, we ran the post-E10 benchmark baseline. Stalker kiting, focus-fire, and cooldown tracking all in — game loop still at 0ms mean, 1ms p95. Good.

Then I got distracted.

The E11 brainstorm started with the obvious deferred items: terrain-aware kiting, multi-target focus-fire, Blink. But somewhere in the middle I found myself asking whether unit definitions should be configurable. Not Java enums and static switch statements, but YAML. New unit type, new file, no code changes.

The argument: Java RTS engine space is nearly empty. A configurable RTS plus Drools CEP plus Quarkus Flow, all running as a GraalVM native binary, would be genuinely unusual. It could matter to the Quarkus and GraalVM communities as an R&D showcase — the kind of work that pushes investment in native support. I parked it as GitHub #74 rather than let it derail the session. But it coloured everything that followed: E11 became about extensibility, not just tactics.

**The strategy seam**

The core of E11: extract `kiteRetreatTarget()` and `selectFocusTarget()` from `DroolsTacticsTask` into CDI strategy interfaces. `@Named("direct")` preserves current behaviour. Select by config property at startup.

The mechanism is clean:

```java
kiteStrategy = kiteStrategies.select(NamedLiteral.of(kiteStrategyName)).get();
```

Fails fast at startup if the name is wrong, discovers new implementations automatically, and extending it later — a `@Named("lua")` implementation that executes a script, say — requires zero changes to `DroolsTacticsTask`.

We also added `TerrainProvider`, an `@ApplicationScoped` bean that routes terrain from `EmulatedEngine` to whichever kite strategy is active. One thing worth flagging: `EmulatedGame` is not a CDI bean — it's owned by `EmulatedEngine`. Adding `@Inject` to `EmulatedGame` fields silently does nothing. Injection must go on the owning bean.

**The cooldown revelation**

The Stalker's attack cooldown was 3 ticks — an early approximation. Real SC2 is closer to 1 tick at 500ms per tick. Changing it was a one-liner, but the fallout was instructive.

The E10 kiting test failed. I assumed it needed a threshold adjustment. Claude came back with the actual root cause: the test's "standing still" baseline was using `AttackIntent(enemy.position)`. In `EmulatedGame`, that does two things — moves the unit toward the target, and keeps it in `attackingUnits` across ticks, firing every tick with no cooldown gap. The kiting unit, by contrast, issues `MoveIntent` on cooldown ticks, which clears the attack state. The baseline had been getting double DPS and walking into melee. The test had passed with cooldown=3 only because the long kiting windows masked the flaw.

The fix: issue `MoveIntent(unit.position())` on cooldown ticks in both scenarios. Now both fire at the same rate; the only variable is the retreat movement. Stalker retains 132 HP vs roughly 90. Kiting works, and the test proves something real.

**Terrain and multi-target**

`TerrainAwareKiteStrategy` uses an angular sweep: try the ideal retreat direction first, then ±45°, ±90°, ±135°, 180°, return the first walkable tile. No A* for a single-step retreat.

`OverkillRedirectFocusFireStrategy` assigns the minimum attackers needed to kill the primary target, redirects the rest to the secondary. Simple greedy loop, but it expresses genuine tactical intent.

**Blink: deferred**

Blink was the conditional item. After landing everything else, I had Claude assess the scope: 70 constructor call sites for the Unit record migration, non-trivial EmulatedGame physics integration, two new GOAP WorldState keys with potential silent-plan interactions. Full session minimum.

E12 it is.
