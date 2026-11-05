---
layout: post
title: "The Simulator Learns to Build"
date: 2026-11-05
type: technical
entry_type: diary
subtype: deep-dive
projects: [QuarkMind]
---

EmulatedGame can now manage its own buildings. That sounds mundane until you
consider what it means: a simulated StarCraft II engine, running thousands of
ticks per second on commodity hardware, no longer needs a real game telling it
what structures exist on the map. It figures that out itself, from the same
input a human player would provide — build commands.

This is the single hardest sync to remove in the standalone fidelity programme,
and understanding why requires a detour into what buildings actually *do* in
an RTS economy.

## Why Buildings Are the Keystone

In StarCraft II, buildings are not passive decoration. They are load-bearing
infrastructure:

- **Production capability.** You can't train a Stalker without a Gateway. You
  can't build a Carrier without a Stargate and a Fleet Beacon. Get a building's
  completion time wrong by a few seconds and the entire downstream production
  timeline shifts.

- **Supply.** Pylons provide 8 supply each. Nexuses provide 15. Get the supply
  cap wrong and the simulator incorrectly blocks unit production — or worse,
  incorrectly allows it, creating phantom units that never existed in the real
  game.

- **Tech tree gating.** Cybernetics Core unlocks Stalker production. Twilight
  Council unlocks Charge research. A missing or late tech building doesn't just
  affect one unit type — it blocks an entire tier of the tech tree.

- **Economy.** Assimilators enable gas income. Nexuses define mining bases.
  Reactor add-ons double a building's production throughput.

Get any of this wrong and the errors cascade. A Pylon that completes two
seconds late delays every unit that was supply-blocked. A Gateway that
doesn't exist at the right moment means Zealots never get trained. The
divergence compounds geometrically with game time.

## What "Ground Truth Sync" Actually Means

Until now, EmulatedGame handled this problem by not handling it. Every tick,
the replay validation harness ran a function called `syncBuildings()`:

![Before: Ground Truth Sync](assets/gt-sync-before.svg)

Every tick, the harness asked the replay's ground truth "what buildings exist
right now?" and overwrote EmulatedGame's building state with the answer. New
buildings were injected. Completion status was synced. Reactor add-ons were
marked. Buildings that disappeared were removed.

This worked. The building state was always correct — because it was always
copied from a source that knew the answer. But it meant EmulatedGame's own
building logic was never exercised. The construction timers in `handleBuild`?
Never tested against real games. The morph completion logic for Lair or
OrbitalCommand? Never validated. The supply calculations? Always overwritten.

For a harness that measures *other* things — unit training accuracy, economy
tracking — this was acceptable. But for training an agent, it's a
non-starter. An agent training environment that can't independently track
buildings is an environment that can't run without SC2 installed. And if you
need SC2 installed, you're back to real-time game speed — one game per
minute, not thousands per second.

## The Intent-Driven Architecture

The fix follows the same pattern we've used for units, research, and morphs:
teach `TrackerIntentExtractor` to read building events from the replay's
tracker data, emit typed intents, and let EmulatedGame process them through
its own physics.

![After: Standalone Building Lifecycle](assets/gt-sync-after.svg)

The tracker events are authoritative. When a human player builds a Gateway,
the replay records a `UnitInit` event at the construction loop and a
`UnitDone` event at the completion loop. Initial buildings — the starting
Nexus, Command Center, or Hatchery — appear as `UnitBorn` events at loop
zero. Building morphs (Hatchery → Lair → Hive, Command Center →
OrbitalCommand) fire their own `UnitInit` events. Destruction appears as
`UnitDied`.

The extractor now emits four building-lifecycle intents from these events:

- **`BuildingBornIntent`** for initial buildings at game start. Complete
  immediately, no construction cost.

- **`BuildIntent`** (extended with a replay tag and pinned completion loop)
  for buildings constructed during the game. EmulatedGame places the
  incomplete building, schedules a completion callback at the exact loop
  the replay says it finished, and manages everything in between — supply
  bonus, town hall registration, gas building activation, Reactor detection.

- **`MorphIntent`** (extended with pinned completion) for building morphs.
  The tricky part was distinguishing morphs from fresh construction: both
  fire `UnitInit` events. A static `MORPH_BUILDING_SOURCES` map resolves
  which is which — if the tracker event name is "Lair", the source is a
  Hatchery. If it's "OrbitalCommand", the source is a Command Center.

- **`BuildingDiedIntent`** for building destruction.

One detail worth calling out: intent ordering matters at loop zero. The
initial Nexus must exist before the initial Probes can be assigned to it
for training purposes. A secondary sort key on intent type — buildings
before units — handles this without relying on tracker event ordering,
which varies across SC2 patch versions.

## What This Gives Us for Calibration

The immediate practical value is that EmulatedGame's building physics are
now exercised against every replay in the test suite. Previously, a bug in
`handleBuild`'s construction timing would be invisible — the harness
overwrote the result before anyone could measure it.

Now, building accuracy is measured directly. The harness compares
EmulatedGame's building counts against the replay's ground truth at every
checkpoint. A construction timer that's off by 10% will show as a
building that completes early or late, which cascades into supply divergence,
which cascades into unit count divergence. The signal is immediate and
actionable.

This also exposed a real bug: `handleBuild` wasn't deducting gas costs.
Seven Protoss tech buildings (Twilight Council, Robotics Facility, Stargate,
Dark Shrine, Templar Archives, Fleet Beacon, Robotics Bay) had incorrect
gas costs in `SC2Data`. The numbers were wrong in the data table, and the
handler wasn't reading the gas column at all. Both fixed.

## The Standalone Progress

Buildings were the fifth GT sync to fall. The progress chart for the
standalone fidelity programme:

![GT Sync Elimination Progress](assets/gt-sync-progress.svg)

Five of twelve syncs eliminated: buildings, supply cap, supply used,
spending, and workers. The remaining seven are all in the economy domain —
mineral balance, vespene balance, collection rates, worker distribution.
These are the mining model and gas income model, which represent the final
push to full standalone operation.

The end state is an EmulatedGame that receives a sequence of player commands
(build this, train that, research this upgrade) and independently produces
the full game state — units, buildings, economy, tech tree — with enough
accuracy to train an agent. No SC2 process. No real-time constraint.
Thousands of ticks per second.

We're not there yet, but the hardest piece is done. Buildings are the
keystone of the simulation because they gate everything else. Getting
mineral income right is important, but minerals are a continuous value
with natural tolerance. Getting *buildings* right is binary — either the
Gateway exists when the Stalker needs to train, or it doesn't. There's no
room for approximation, and no way to compensate for a miss.

That keystone now stands on its own.
