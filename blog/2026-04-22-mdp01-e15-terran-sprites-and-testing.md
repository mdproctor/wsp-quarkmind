---
layout: post
title: "E15: Terran Sprites and the Cost of Looking Instead of Testing"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkus, three-js, visualizer, sprites, sc2, testing]
---

E15 delivered the Terran side of the cartoon roster: Marine, Marauder, and
Medivac. With E14's Protoss already in — Probe, Zealot, Stalker — we now have
the foundations of a real all-race visualizer.

The design decision that took the most conversation was team colour. The
original approach used a single material per unit type, no distinction between
friendly and enemy. That works fine when you're always playing Protoss. I said
we'd eventually support all races, and the design fell apart immediately: a
Marine sprite has to look different on your team versus the enemy's.

The obvious fix is palette-swapping the whole model. SC2 doesn't do that. The
real game applies player colour to specific zones — shoulder pads, visor stripe,
knee plates — while the body stays in the unit's canonical colours. We matched
that. `makeDirTextures(drawFn, teamColor, size)` now generates two material sets
per unit type, keyed as `MARINE_F` and `MARINE_E`. The dispatch in
`syncUnitLayer` is a single string lookup with a fallback to `UNKNOWN_F`/`UNKNOWN_E`
for any race not yet drawn.

The three Terran sprites follow the same Canvas 2D architecture as the Protoss
set. The Marine gets visor stripe and shoulder pads in teamColor. The Marauder
gets a narrower visor slit and knee armour plates. The Medivac was the most
interesting: it's a flying unit, so it floats above the ground plane. A
`FLYING_UNITS = new Set(['MEDIVAC'])` constant drives the height offset; adding
future air units is one line.

[![E15 showcase — Terran and Protoss sprites in the isometric visualizer](/quarkmind/blog/assets/e15-showcase-sprites.png)](/quarkmind/blog/assets/e15-showcase-sprites.png)

Then the debugging session. I wanted to see all the sprites together in the
visualizer, so we built a `ShowcaseResource` endpoint that seeds one of every
type across the map. This took far longer than it should have.

The symptoms kept pointing at the wrong causes. Units appeared as tiny specks
far from the camera — I assumed a camera angle problem. Units were covered by
fog — I assumed a fog plane initialisation issue. Purple rectangles floating
1200 world units off the map — I went through three explanations before the
right one surfaced.

Claude ran `scene.traverse()` across the full Three.js scene and collected the
world position of every mesh. The result was immediate: `BasicEconomicsTask.pylonPosition()`
had no upper bound on its row index. With `buildingCount` reaching 2317 after
the AI had been running unchecked, tile y was 1752. World z was 1204. Purple
rectangles.

The duplicate-Pylon bug was the same pattern — no check whether a Pylon was
already under construction, so the AI queued a new one on every game tick once
supply pressure hit. Trivial to fix once found; expensive to find by looking at
the visualizer.

What would have caught both: a parameterized test on `pylonPosition()` with
`buildingCount` values of 100, 1000, 10000. At buildingCount=100 the tile
y-coordinate is already 45 — suspicious. At 1000 it's 270 — obviously off the
64-tile map. We had tested it with values of 0, 1, and 4.

The fix was `GameStateInvariantTest` — plain JUnit, no Quarkus, no browser.
Positions within map bounds after N ticks. Building count within a plausible
range. Tag uniqueness. The Playwright equivalent is `allSceneObjectsAreWithinMapBounds`,
which traverses the full scene graph and fails if anything is outside ±23 world
units. It would have failed on the first run after the overflow was introduced.

The rule isn't new: test invariants at the code level, not by opening a browser.
The lesson is that I let visual debugging run for hours before asking whether
the geometry was wrong rather than the camera.
