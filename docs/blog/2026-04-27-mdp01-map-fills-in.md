---
layout: post
title: "The Map Fills In"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [sc2, replay, visualizer, three-js, playwright, testing]
---

The replay was navigable but half-blind. Units moved. Buildings appeared. But the economy was invisible — no mineral patches, no geysers, no enemy Hatcheries expanding across the map. For a Protoss vs Zerg game, that's half the story missing.

We started with neutrals. SC2's tracker events mark every unit with a `controlPlayerId`. Minerals, geysers, rocks — they carry `ctrlId == 0`. The existing `applyUnitBorn()` dropped them entirely. The fix was a classification branch:

```java
if (ctrlId != null && ctrlId == 0) {
    if (Sc2ReplayShared.isGeyser(unitName))
        addGeyser(new Resource(tag, pos, defaultGeyserAmount(unitName)));
    else if (Sc2ReplayShared.isMineralPatch(unitName))
        addMineralPatch(new Resource(tag, pos, defaultMineralAmount(unitName)));
}
```

Enemy buildings had the same problem but a different cause — the building path had no `else` branch. Everything that wasn't a friendly building was silently dropped. One more condition, and Hatcheries and Spires started appearing.

Rendering all of this meant new Three.js sync functions: `syncMineralPatches()` and `syncEnemyBuildings()` alongside the existing `syncGeysers()`. Minerals got flat teal boxes. Enemy buildings reused the building sprite path with a red team colour. The visibility toggle — a button that flips `.visible` on all enemy sprites and meshes — was the easiest piece; satisfying in its simplicity.

Creep we approximated. No per-tile state comes from the tracker events. We paint all tiles within `CREEP_RADIUS = 10` of every Hatchery, Lair, or Hive as translucent purple. Not precise, but it gives the right impression of base shape.

---

Then came the unit inspect panel. The infrastructure existed — click a sprite, fire a raycaster, fetch `/qa/unit/{tag}`, populate the panel. We just had no test for it.

The physical click approach failed first. Fixed-position buttons at the viewport top intercept headless Chromium clicks before they reach the canvas `mouseup` listener. No error — just silence.

The second problem: `worldToScreen(wx, wz)` hardcoded `y=0`. Unit sprites sit at `TERRAIN_SURFACE_Y + TILE * 0.5 ≈ 0.43` world units above the ground. At ~14 world-unit camera distance in isometric view, that shifts the projection ~27 pixels — just enough to miss the sprite's raycaster bounding box.

The third was subtler. The portrait draw call in `showUnitPanel` had wrong argument count: `drawX(ctx, 32, 32, 0, tColor)` instead of `drawX(ctx, 32, 0, tColor)`. The `.then()` chain swallowed the resulting `TypeError`. The panel never appeared, no error anywhere. It had been broken from the day the portrait drawing was added.

We found it by refactoring to `async/await`. Claude surfaced the TypeError immediately once the rejection had somewhere to propagate. For the click reliability problem, we gave up on physical mouse events and wrote `window.__test.clickUnit(tag)` — an async function that fires the raycaster programmatically, awaits the full fetch+DOM pipeline, and resolves when the panel is visible. `page.evaluate("async () => window.__test.clickUnit(...)")` blocks until it's done.

Three tests now cover the full inspect lifecycle. The panel works.
