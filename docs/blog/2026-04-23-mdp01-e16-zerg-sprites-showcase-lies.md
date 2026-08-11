---
layout: post
title: "E16: Zerg Sprites and the Showcase That Lied"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [visualizer, three-js, zerg, sprites, quarkus, testing]
---

E16 is done. Zergling, Roach, Hydralisk, Mutalisk — four draw functions, four distinctly different body plans, all following the palette and dispatch pattern that E14 and E15 established.

The Zerg palette is classic SC2: dark purple carapace (`#2a0a3a`), fleshy midtones (`#5c1a6e`), yellow eyes (`#ffe066`). Team colour — blue for friendly, red for enemy — shows up as a glowing bio-sac: a radial gradient with `shadowBlur` on the belly, shoulder, or wing-joint depending on the unit. It reads clearly at sprite scale and feels genuinely biological rather than just coloured armour.

Mutalisk gets the flying treatment — `TILE * 1.1` above terrain rather than `TILE * 0.5`, same elevation behaviour as Medivac. All four directions use the same manta silhouette with a slight canvas rotation per direction. Ground units have genuinely different body plans per facing; a flying unit mostly just tilts.

Then I asked to see the showcase.

E15 ended with a lesson: write invariant tests, don't debug by looking at the browser. We managed to break that rule twice before I'd finished congratulating myself on the sprite work.

Claude started the server in emulated mode and seeded the showcase. The instruction was right; the profile was wrong — and Claude didn't check before telling me to look. I saw units in fog, on the far side of walls, nothing where it should be. We spent three attempts moving units closer to the Nexus before realising we were solving the wrong problem entirely.

`ShowcaseResource` injects `SimulatedGame` and calls `simulatedGame.spawnEnemyUnit(...)`. In emulated mode, `SC2Engine` resolves to `EmulatedEngine`, which broadcasts `EmulatedGame` — a completely separate CDI bean. The seeded units never appear. No warning, no error. The emulated game's own AI units render wherever the simulation happened to place them, and that's what I was looking at.

The fix is simple: showcases run in mock mode. The prevention is the Playwright test `showcaseRendersAllUnitsAboveTerrainSurface`, which now runs before any "come and look" — asserting all ten enemies render, sprite Y above terrain surface, no objects outside map bounds.

The second problem was the terrain colour. I wanted light sandy brown for the visible ground, replacing the dark blue-grey the visualiser had carried since E14. We changed the tile material to `0xb8956a`. Still grey.

The sun was the problem. Three.js `MeshLambertMaterial` computes `result = material × light`. The directional light was `0xaabbff` — a strong blue-white that overwhelms any warm tone in the material. Changing tile colour is futile if the light is fighting you. We shifted the sun to neutral white, the ambient to a warm `0x706050`, and the brown came through immediately.

Both the sprite Y and 3D model Y are now anchored to `TERRAIN_SURFACE_Y` — a module-level variable updated once in `loadTerrain()`:

```js
const groundY = TERRAIN_SURFACE_Y + TILE * 0.5;
const flyingY = TERRAIN_SURFACE_Y + TILE * 1.1;
```

In mock mode `TERRAIN_SURFACE_Y` is 0.08; in emulated mode it's `TILE`. Units sit at the right height in both profiles without a single profile check at spawn time.

Eleven Protoss units still fall back to the UNKNOWN blob, ten Terran, nine Zerg. I want all of them done before touching 3D models. After the 2D sprite pass is complete the question of replacing the sphere placeholders with low-poly Three.js geometry becomes interesting — but not before.
