# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-23 — 3D unit models to replace sphere placeholders

**Priority:** medium
**Status:** active

Replace the current coloured-sphere 3D models with low-poly compound Three.js geometry per unit type (e.g. Zealot = torso box + shoulder spheres + blade cones). All geometry hand-built in JS — no external model pipeline needed. Hard gate: the isometric view must not become sluggish; run a `GameLoopBenchmarkTest` equivalent before and after to confirm no FPS regression.

**Context:** After E16 Zerg sprites the 2D directional cartoon sprites look solid across all three races. User wants to finish the remaining 2D sprite episodes first, then revisit 3D. Option 1 (stylised low-poly primitives) was agreed as the right starting approach.

**Promoted to:**

---

## 2026-04-15 — E8: Terrain height — high/low ground mechanics

**Priority:** high
**Status:** active

Extend E7's binary `WalkabilityGrid` to a three-value `TerrainGrid` (high/low/wall).
Ramps only walkable from specific directions. High ground gives vision advantage;
ranged attacks from low against high ground have 25% miss chance (real SC2 mechanic).
Visualizer shades tiles by height level.

**Context:** Came up during E7 brainstorm. Deliberately deferred to keep E7 focused on
walkability + A* pathfinding. The `WalkabilityGrid` API is designed so `TerrainGrid`
can extend or replace it without touching `AStarPathfinder`.

**Promoted to:**
