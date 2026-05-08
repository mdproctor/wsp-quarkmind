---
layout: post
title: "High Ground"
date: 2026-04-16
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulation, terrain, combat, testing]
---

The wall works. E7 fixed pathfinding through the chokepoint, with a physics
constraint as backstop. But the emulated map was still flat — every tile either
walkable or wall, no elevation, no tactical geometry. SC2 isn't flat.

E8 adds height.

The terrain model extends from two values to four: `HIGH`, `LOW`, `RAMP`, and
`WALL`. The 64×64 emulated map now has real topology — enemy staging above the
wall line (y > 18), nexus side below (y < 18), chokepoint gap as a ramp rather
than just a hole in the wall.

I made one deliberate tactical choice: enemies attack from high ground. My ranged
units defending from below have a 25% miss chance when firing uphill. It's the
harder scenario — no natural defensive advantage — but it's the correct one. High
ground matters in SC2 because of exactly this penalty.

The miss check lives in `resolveCombat()`, not in `DamageCalculator`. The
calculator stays pure — damage formula only. The guard fires before
`computeEffective()`:

```java
private boolean missesHighGround(Point2d attackerPos, Point2d targetPos, UnitType attackerType) {
    if (terrainGrid == null) return false;
    if (SC2Data.attackRange(attackerType) <= 1.0f) return false; // melee — never penalised
    Height ah = terrainGrid.heightAt((int) attackerPos.x(), (int) attackerPos.y());
    Height th = terrainGrid.heightAt((int) targetPos.x(),   (int) targetPos.y());
    if (ah != Height.LOW || th != Height.HIGH) return false;
    return random.nextDouble() < 0.25;
}
```

Testing probability logic without a mocking framework: inject an anonymous
`Random` subclass with a fixed `nextDouble()` return — `0.0` for always-miss,
`1.0` for always-hit. `Random.nextDouble()` isn't `final`. No Mockito, no
seed-hunting — the boundary condition is explicit.

## One subagent per task, two reviews each

I had Claude execute E8 as subagent-driven development — seven tasks, each
dispatched as a fresh subagent so context didn't bleed between them. After each:
spec compliance review, then code quality review, before moving on.

The review gates caught things. A quality reviewer flagged `java.util.Random`
written fully qualified when `java.util.*` was already imported — inconsistent
with every other `java.util` type in the file. Another caught `@Tag("browser")`
on the new Playwright test but not on two existing pixel-sampling tests, meaning
those tests ran in the default surefire suite but were excluded from the playwright
profile. Both fixed before the next task began.

378 surefire tests. The wall-physics and pathfinding tests from E7 still pass.

## The map now looks like something

[![Visualizer showing high ground (warm brown, top) separated from low ground (dark navy, bottom) by a wall line with a ramp gap](/quarkmind/blog/assets/e8-high-ground.png)](/quarkmind/blog/assets/e8-high-ground.png)

Warm tan/brown above the wall line — HIGH ground. Dark canvas below — LOW ground.
The wall stripe with the ramp gap visible through it. It looks like a proper
topographic map now.

The next step is fog of war. I parked it out of E8 deliberately — full fog of war
is a system-level feature, and half-measures leave an inconsistent game model. It
gets its own epic.
