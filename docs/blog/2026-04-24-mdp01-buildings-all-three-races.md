---
layout: post
title: "Buildings, All Three Races"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

The visualizer had sprites for 65 multiplayer units across Terran, Protoss, and
Zerg. Buildings were next — nine of them, all Protoss, rendered as coloured
boxes. The goal was to replace the boxes with canvas sprites to match the unit
aesthetic.

We replaced them. The `BuildingType` enum was Protoss-only, so we drew nine
functions — `drawNexus`, `drawPylon`, `drawGateway` and so on — created a
`BUILDING_MATS` map alongside `UNIT_MATS`, and updated `syncBuildings()` to
render sprites instead of Three.js `BoxGeometry`. The boxes had always been
temporary; they just stayed temporary longer than intended.

Then I asked: what about Terran and Zerg buildings?

The honest answer was that there weren't any. The `BuildingType` enum had nine
Protoss types and `UNKNOWN`. The replay parser mapped only Protoss building
names. `ObservationTranslator` detected only `PROTOSS_*` unit types as
buildings. The whole building layer assumed a Protoss player.

This wasn't always wrong. An earlier decision — start Protoss, expand later —
had locked it in. But the platform was supposed to support all three races.
Later had arrived.

The expansion touched every layer the domain crosses. `BuildingType` grew from
9 values to 48: six more Protoss (Photon Cannon, Shield Battery, Dark Shrine,
Templar Archives, Fleet Beacon, Robotics Bay), fifteen Terran (Command Center
through Refinery), eighteen Zerg (Hatchery through Extractor). `SC2Data` got
health values, mineral costs, build times, and supply bonuses for all of them
— Hive gives 6 supply, Supply Depot gives 8, Hatchery gives 6, in case that
ever matters. `Sc2ReplayShared` learned to map SC2 string names across all
three races, including flying variants like `BarracksFlying` and state variants
like `SupplyDepotLowered` and `SpineCrawlerUprooted`. `ObservationTranslator`
swapped its `PROTOSS_BUILDINGS` set for `ALL_BUILDINGS` covering everything.

One gap in the ocraft library: `Abilities.BUILD_TEMPLAR_ARCHIVES`,
`Abilities.RESEARCH_LURKER_LEVEL`, and `Abilities.BUILD_NYDUS_CANAL` don't
exist in ocraft 0.4.21. Not documented anywhere — compilation failure is how
you find out. Those three map to `null` in `ActionTranslator.mapBuildAbility()`
for now; no Terran or Zerg agent logic calls them yet.

The 39 draw functions follow the same canvas 2D pattern as unit sprites, with
race-appropriate aesthetics: Protoss in blue and gold crystal, Terran in
industrial grey with red accents, Zerg in organic dark browns with
bioluminescent greens. The Hive is darker and more imposing than the Lair,
which is darker than the Hatchery. The Fusion Core gets a hexagonal body with
a glow at the centre. The Nydus Canal is a worm emerging from a ground crack,
teeth visible.

The showcase layout needed some working out. I put the new building rows at
game tile z = −2, −4, −6 to fit them below the unit grid. They appeared
outside map bounds in the Playwright test — world coordinates −23.8, −26.6,
−29.4. The visualizer maps tile position to world space as
`worldZ = tileZ × TILE − HALF_H`; for a 64×64 grid, tile z=0 maps to
world −22.4. Negative tile z goes further negative, past the ±23 bound. The
valid tile range is 0 to 63; the unit grid sits at z=2 to 20. We moved the
building rows to z=22 through 34 — six rows at 2-tile intervals — which map
to small positive world-z values with plenty of headroom.

The showcase now seeds 49 buildings: the original nine Protoss at z=22, six
new Protoss at z=24, two rows of Terran at z=26 and 28, two rows of Zerg at
z=32 and 34.
