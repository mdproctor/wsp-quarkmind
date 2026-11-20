---
layout: post
title: "Rectangles, Not Radii"
date: 2026-11-20
author: Mark Proctor
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [quarkmind, sc2, emulated-game, pathfinding, building-footprint, terrain]
---

# Rectangles, Not Radii

SC2 buildings sit on rectangular tile footprints. A Nexus occupies 5x5 tiles. A Barracks occupies 3x3. The pathing grid blocks those exact tiles and nothing else. Our EmulatedGame was using circular radii instead — a 5x5 Nexus got radius 2.5, and collision checks tested whether a unit's distance from centre was less than that radius. Close enough for most purposes, but wrong at the corners. A unit approaching diagonally hits the circle at distance 2.5 from centre, but the actual tile boundary at the corner is at distance 3.54. That's a 40% gap.

The fix was straightforward: `SC2Data.buildingFootprint(BuildingType)` returns `{width, height}` in tiles, and `enforceWall()` now checks whether a proposed position falls inside the axis-aligned rectangle rather than inside a circle. Static shared arrays keep the hot path allocation-free.

The interesting part was what broke. Two combat tests placed a Stalker at (10, 10) for a blink scenario, and a kiting test had units at (8, 12) retreating toward the Nexus at (8, 8). Both positions happened to sit inside the rectangular footprint but outside the old circular radius — the corners where the two shapes disagree. With circular collision, these units moved freely. With rectangular collision, they hit the building boundary. The fix was just moving the test positions to open ground, but it demonstrated exactly why the approximation was wrong: the old tests were only passing because units were phasing through building corners.

A related change landed first: wiring a default `TerrainGrid` in the `EmulatedGame` constructor. Previously, terrain was null unless explicitly set, which meant most tests ran workers in straight lines with no obstacle awareness. Now every `new EmulatedGame()` gets the emulated map's 64x64 grid, and workers navigate around obstacles from the start. This surfaced a coupling between terrain and fog of war — the snapshot method was gating enemy visibility on `terrainGrid != null`, which meant enabling terrain also enabled fog. I split these into independent flags: terrain for pathfinding, `fogEnabled` for visibility filtering. Production code enables both; tests get terrain without fog.

The mineral patch conversion was more revealing. I tried replacing the circular obstacle (radius 0.3, covering about one tile) with the correct 2x1 rectangular footprint. The MULE isolation playbook immediately regressed from +1.4% to +4.1% delta. The 2x1 rectangle blocks twice as many tiles as the old circle, fundamentally changing worker routing through the mineral line. The 0.3-radius value was empirically calibrated for the pathfinder — it represents the pathfinding obstacle, not the placement footprint. Converting it to a rectangle needs its own calibration pass, not a mechanical swap. I reverted the mineral patches to circular and left that for a dedicated issue.

The design rule holds: radii are correct for unit-unit collision (SC2 units genuinely have circular collision). Everything else — building collision, pathing grid obstacles, edge-point calculations — should use rectangular tile footprints. Buildings are done. Mineral patches need orientation and calibration work before they can follow.
