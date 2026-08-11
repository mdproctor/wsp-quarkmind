---
layout: post
title: "What the probe sees"
date: 2026-04-16
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

Before E9, the visualizer showed everything. Every Zealot waiting in the staging
area, every enemy unit moving down the ramp — all of it fully visible regardless
of where your probes were. As a debugging view that made sense. As a simulation
it was nonsense.

E9 adds fog of war. The real SC2 version: three tile states, server-side
filtering, the high-ground rule.

**The high-ground rule is what makes it interesting.** In SC2, units on HIGH
ground are invisible to units on LOW ground regardless of range. Your probes and
Zealots on the LOW side cannot see the enemy staging area on HIGH, even if it's
fifteen tiles away. The only way to reveal it is to get a unit up there. Which
means scouting. Which means DroolsScoutingTask's active probe dispatch suddenly
matters.

I wanted fog enforced server-side, not in the visualizer. Filtering in JavaScript
would be dishonest — the agent would still receive the full enemy list and make
decisions from it. `GameState.enemyUnits` now only contains what your units can see.

The CDI wiring was the design challenge. `EmulatedEngine` is `@IfBuildProfile("emulated")` —
it only exists in that mode. `GameStateBroadcaster` runs in all non-prod profiles.
The engine produces a `VisibilityGrid` each tick; the broadcaster needs to attach
it to the WebSocket payload. Expanding the `SC2Engine` interface would be wrong —
real SC2 handles fog natively and would never implement it.

The solution is a thin `@ApplicationScoped` bridge with no profile guard:

```java
@ApplicationScoped
public class VisibilityHolder {
    private volatile VisibilityGrid current;
    public void set(VisibilityGrid g) { current = g; }
    public VisibilityGrid get() { return current; }
}
```

`EmulatedEngine` writes to it each tick; the broadcaster reads it. In mock or
replay mode, `get()` returns null and the payload sends `"visibility":null`. Clean
profile separation without interface pollution.

The grid recomputes each tick using a standard circle check, with one extra
condition:

```java
// SC2 high-ground rule: LOW or RAMP observer cannot illuminate HIGH tiles
if (tileH == TerrainGrid.Height.HIGH
        && observerH != TerrainGrid.Height.HIGH) continue;
```

Everything else is straightforward — squared-distance check, WALL tiles skipped,
null terrain treated as all-LOW for backward compatibility with tests that don't
configure terrain.

I ran this as subagent-driven development — nine tasks, each built by a fresh
Claude instance then reviewed twice: once for spec compliance, once for code quality.
Two things came out of those reviews that I wouldn't have caught otherwise.

The first was in the visualizer's fog-draw loop. The initial implementation
iterated `y` from 0 to 63 — the full 64-tile grid. But the canvas viewport is
only 32 rows high. Tiles `y=32..63` compute to negative canvas Y and are clipped
silently by the renderer. No error, visually correct, 50% of the draw work wasted
every frame. The quality reviewer caught it. One-word fix: `VIEWPORT_H` instead of
`64`.

The second was in `GameStateBroadcaster`. An existing unit test instantiated the
class with `new` — no CDI — leaving `visibilityHolder` null at the field level.
The implementing agent handled it with `visibilityHolder != null ? ... : null`.
Pragmatic, but it encodes test setup into production logic: an `@Inject` field is
never legitimately null in a running container, so the guard says nothing true about
production and something false about the tests. The fix: give the unit test a real
`new VisibilityHolder()` and remove the outer guard.

The result is 409 tests passing. The visualizer goes dark at the high-ground
boundary the moment the emulated game starts. The enemy staging area is invisible
until a probe makes it up the ramp. DroolsScoutingTask dispatches one, toward
(56,56), which is now inside the emulated map bounds after the scouting fix earlier
in this session.

Three tile states: black means never seen, dimmed grey means seen but no longer
in vision, clear means currently visible. What the probe sees is all the agent knows.

[![Fog of war — the probe's vision circle lit, HIGH ground in memory, solid black beyond](/quarkmind/blog/assets/e9-fog-of-war.png)](/quarkmind/blog/assets/e9-fog-of-war.png)

*Frame 133: the Nexus and probes illuminate a circle of LOW ground. The HIGH terrain above the ramp is visible in grey-memory (previously entered, currently dark). Everything else is unseen.*
