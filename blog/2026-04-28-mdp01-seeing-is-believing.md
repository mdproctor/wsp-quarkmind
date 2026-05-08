---
layout: post
title: "Seeing Is Believing"
date: 2026-04-28
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [visualizer, replay, testing, playwright, three-js]
---

I asked for visual verification three times. Each time Claude reported it done. When I sent a screenshot showing blank terrain where minerals should be, the gap was obvious.

The problem was architectural. All the Playwright tests ran in mock mode — a 64-tile map with camera distance 18. The Electron app runs in replay mode: a 160×208 map, camera distance 102. Geometry that renders visibly at 18 units disappears at 102. Claude's tests passed. The user saw nothing. Both facts were true simultaneously.

We spent too long on the wrong axis. `anyMineralOnScreen()` checked whether the mesh centre projected within the camera frustum. That's not visibility — that's mathematics. A two-pixel teal box buried inside a cliff tile satisfies NDC projection. It does not satisfy "the user can see it."

The fix was `ReplayVisualizerIT`. It starts the real replay jar as a subprocess on port 8082, opens headless Chromium at it, aims the camera at the first mineral in the scene using `window.__test.focusOnFirstMineral()`, then samples the actual WebGL pixel:

```java
var pixel = (Map<?,?>) page.evaluate(
    "() => window.__test.samplePixel(" + sx + ", " + sy + ")");
assertThat(b).as("mineral pixel must be cyan (B>R), not sandy terrain")
    .isGreaterThan(r);
```

Three bugs surfaced immediately. `preserveDrawingBuffer: false` (the Three.js default) means the GPU clears the drawing buffer between frames — `drawImage` from the WebGL canvas returns transparent black. Creep tiles at `y=0.02` were below the ground plane at `y=0.04` — depth testing hid them completely. And mineral `BoxGeometry` meshes, viewed from an isometric angle and depth-tested against cliff geometry, simply weren't visible from any angle. We switched minerals and geysers to `THREE.Sprite` objects, which always face the camera and are never occluded.

With `depthTest: false` and `renderOrder: 10` on creep tiles, and sprites replacing boxes for resources, all three pixel tests pass. Mineral cyan. Geyser green. Creep purple.

The rest of the session was cleaner. Building click-to-inspect for buildings (`/qa/building/{tag}`, raycaster extended to hit `buildingMeshes` and `enemyBuildingMeshes`) took about as long as the unit panel did. Camera auto-centre on the Protoss base was a small `onFrame` hook — `autocentreCamera(state)` fires once when `myBuildings` is non-empty, sets `camTarget` to the Nexus world position, marks itself done. The replay now opens looking at the right place.

Mineral depletion — `removeMineralPatchByTag` on `UnitDied` events — was already working. The validation test confirmed it. Not every investigation finds a bug.

The replay viewer is now actually usable: minerals visible, geysers visible, creep visible, buildings clickable, camera starts at the base. And we know it works because the tests sample real pixels, not scene graph counts.
