---
layout: post
title: "E14: A New Visualizer"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkus, three-js, visualizer, sprites, sc2]
---

The existing visualizer was a flat 32×32 PixiJS grid — top-down, static
camera, unit portraits fetched from Liquipedia. For the replay viewer it
worked fine. For cartoon sprites with directional art, it didn't.

The trigger was a conversation about all-race cartoon units — something in
the spirit of the StarCrafts mod, which turns SC2 into something genuinely
charming. Directional sprites need a camera that actually moves. A flat grid
doesn't have one.

The design question was how much 3D to commit to. I didn't want to spec
anything until I'd seen it. Claude built an interactive Three.js demo in the
brainstorm visual companion — drag-to-orbit, fog of war, a toggle between 3D
sphere models and 2D billboarded sprites, three art styles side by side.
The 3D sphere models with cartoon eyes were surprisingly good. The directional
2D sprites were more characterful. We settled on 3D terrain with 2D sprites:
the depth and camera feel of a real RTS, with hand-drawn characters readable
from any angle.

[![E14 visualizer — isometric view with directional cartoon sprites](/quarkmind/blog/assets/e14-3d-visualizer-iso.png)](/quarkmind/blog/assets/e14-3d-visualizer-iso.png)

## The directional sprite mechanism

Each unit type has four canvas textures generated at startup — one per facing
direction. Each frame, a single function computes which direction the camera
is looking relative to the unit's facing angle:

```javascript
function getDir4(facingAngle, unitPos, camPos) {
  const camAngle = Math.atan2(-(camPos.x - unitPos.x), camPos.z - unitPos.z);
  let rel = camAngle - facingAngle;
  while (rel < 0)          rel += Math.PI * 2;
  while (rel >= Math.PI*2) rel -= Math.PI * 2;
  return Math.round(rel / (Math.PI/2)) % 4;
}
```

The negated `camPos.x` is a Three.js handedness correction — without it,
left and right are inverted. Then `sprite.material = mats[dir]`. No
spritesheet pipeline, no texture atlas. The art is Canvas 2D drawing
functions: `drawProbe(ctx, S, dir)`, `drawZealot(ctx, S, dir)`, front/side/back
variants with the eye and weapon always at the front of the profile.

## Three bugs worth naming

`THREE.SpriteMaterial` defaults to `depthWrite: false`. Fog planes with
`renderOrder=5` rendered straight through unit sprites at low camera angles.
The symptom looked like a renderOrder problem; the actual cause was that
sprites weren't writing to the depth buffer at all. Fix: `depthWrite: true`
and `alphaTest: 0.1` on the SpriteMaterial.

The staging mesh leak was subtler. `syncUnitLayer` was called for staging
units with `new Map()` as the mesh tracking argument. The removal path checked
`meshMap instanceof Map` — true, because it is a map — then called
`meshMap.get(tag)`, which always returned `undefined` because the map was
empty on every call. 3D meshes accumulated in the scene forever. A code
reviewer caught it.

The final code review flagged a hardcoded `64` in the fog decode loop:
`visibility.charAt(gz * 64 + gx)`. The grid is 64 tiles wide today, so
everything worked. If the terrain endpoint ever returns a different width, the
fog decodes silently against the wrong stride. Changed to `gz * GRID_W + gx`.

[![Low-angle view showing 3D terrain and fog of war](/quarkmind/blog/assets/e14-3d-visualizer-low.png)](/quarkmind/blog/assets/e14-3d-visualizer-low.png)

## What shipped

Three.js replaces PixiJS client-side only — the server WebSocket protocol is
unchanged. The new renderer has 3D terrain with raised walls and ramps, an
orbiting camera with drag/scroll/pan and three angle presets, per-tile fog of
war, and directional cartoon sprites for Probe, Zealot, Stalker, and a generic
enemy. Buildings stay as geometry for now. Playwright test count went from 6
to 17; the non-Playwright suite held steady at 475.

All-race art — Terran and Zerg — is E15 and E16.
