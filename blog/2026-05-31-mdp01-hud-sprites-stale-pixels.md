---
layout: post
title: "The Pixel Buffer That Didn't Know The Camera Had Moved"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [visualizer, three-js, playwright, testing]
---

The visualizer has been working since April. It renders probes, geysers,
minerals, buildings, fog — the full SC2 picture. But four things sat on the
deferred list since then: HUD mineral formatting, probe overlap, resource
sprites that looked like placeholder rectangles, and tests that expressed
timing in tick counts when game-seconds are the natural unit.

I brought Claude in to close all four. Most of it was mechanical. The
interesting part happened inside a `focusOnFirstGeyser()` call.

## The minerals number

The HUD was showing raw integers: `Minerals: 1234`. Simple enough to fix —
`toLocaleString('en-US')` gives you the comma, a span with a colour class
for the severity tier.

I thought about what the tiers should mean. Below 50 minerals you can't
train anything regardless of race — a probe costs 50, a marine costs 50, a
drone costs 50. So < 50 is zero-agency by definition. The amber tier (50–149)
is one basic unit away from zero. The thresholds aren't arbitrary; they come
directly from the `SC2Data` training costs that the economics model already
uses.

## Probes orbiting the patch

When multiple probes harvest the same mineral patch they converge on the same
world position. The visualizer was stacking them pixel-exact — one sprite
indistinguishable from twelve.

The fix is a post-pass over the unit sprite map after each sync: group
sprites within TILE * 0.5 of each other, distribute the group in a uniform
ring at TILE * 0.32 radius. One call to `applyUnitSpread(unitSprites)` at
the end of `syncUnits()`.

Testing this was tricky. `window.__test.sprite()` returns world coordinates
as `Math.round()` integers. A ring spread of TILE * 0.32 ≈ 0.224 world units
vanishes when rounded — three probes that are genuinely separated all report
the same integer position. We had to read raw floats directly from
`sp.position.x` and `sp.position.z` via `page.evaluate()` to see the spread
at all.

## The factory pattern for resource sprites

The geyser and mineral patch sprites were solid-colour rectangles — green
square, blue rectangle. They worked fine functionally but looked like
development placeholders.

We replaced both with canvas-drawn textures: teal-green radial gradient with
concentric rings for the geyser (gas venting upward), blue radial gradient
with horizontal crystal streaks for the mineral patch. The draw functions
live in `drawGeyser(ctx, S)` and `drawMineralPatch(ctx, S)`.

The natural scope for this would have been just the geyser. But updating
geysers to canvas textures while leaving mineral patches as solid-colour
`SpriteMaterial` calls would have created an inconsistency — the same
primitive used two different ways with no reason for the difference. We built
`makeResourceMaterial(type)` instead, called once at startup for each type
with the result stored as `GEYSER_MAT` and `MINERAL_MAT`. Both sync
functions clone from those. Consistent lifecycle, consistent visual quality.

## The pixel buffer that didn't know the camera had moved

This is the part I didn't see coming.

We added Playwright pixel assertions to verify the canvas textures were
actually rendering: focus the camera on the geyser, sample the centre pixel,
assert the green channel dominates. The first run returned terrain colour —
sandy brown, nowhere near green. `focusOnFirstGeyser()` was returning valid
screen coordinates. The geyser was in the scene. `preserveDrawingBuffer:
true` was set on the renderer.

Claude traced it down: `updateCamera()` repositions the camera in JavaScript
memory, but does not update the WebGL pixel buffer. The buffer reflects the
last rendered frame. With `preserveDrawingBuffer: true` the buffer persists
between frames — but only until `renderer.render()` is called next. In a
`page.evaluate()` call, no animation frames fire, so `samplePixel()` was
reading pixels rendered under the previous camera transform, before the focus
happened.

The fix is one line in the focus helper: `renderer.render(scene, camera)`
after `updateCamera()`, before computing and returning screen coordinates.
The `animate()` loop will overwrite this render on its next rAF cycle, but it
renders identical content (no game state changed), so there's no visual side
effect.

It's in the garden now as GE-20260531-b7c259. The symptom — correct
coordinates, wrong colour — reads as a coordinate error. It isn't.

## Time in ticks vs time in seconds

The HUD tests expressed timing as raw tick counts: run 5 ticks, check
minerals increased. Functional, but tick counts are an implementation detail.
Game-seconds are what you'd think in if you were watching the game.

There's a naming trap here I nearly walked into. `gameFrame` sounds like it
counts game loops — the 22.4-per-second SC2 clock. It doesn't. In
`SimulatedGame`, `gameFrame.incrementAndGet()` fires once per outer tick
call. One outer tick is 22 game loops. So `gameTimeSeconds = gameFrame /
22.4` gives a number 22× too small.

The correct formula is `(gameFrame * 22) / 22.4`. Five seconds of game time
is `ceil(5 * 22.4 / 22) = 6` outer tick calls. After 6 ticks `gameTimeSeconds
≈ 5.89`. The `window.__test.gameTimeSeconds()` accessor uses this formula,
with a comment naming the Java constants it mirrors so the literals don't
look like magic numbers.

We landed with six new Playwright tests across the four sub-tasks and the
visualizer is showcase-ready. The canvas sprites in particular are a clear
visual step up — geysers no longer look like traffic lights.
