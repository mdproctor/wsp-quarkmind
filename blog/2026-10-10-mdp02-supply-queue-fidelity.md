---
layout: post
title: "Supply belongs to the unit that's training, not the one waiting in line"
date: 2026-10-10
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulated-game, fidelity, terran, supply, economy]
---

EmulatedGame had a supply accounting error that's been quietly dragging down Terran and Protoss economy fidelity since the production queue was added. When a unit was queued behind another — not training, just waiting — its supply cost was consumed immediately. In real SC2, supply is committed when a unit starts training, not when it enters the queue.

The effect was subtle enough to miss. With queue depth 5 and two production buildings, up to 10 supply slots could be locked by units that hadn't started training yet. The Terran economy playbook was hitting ~42 SCVs at tick 305 instead of the 43-44 you'd expect from dual-OC production. Not a dramatic miss, but the kind of persistent 5% gap that compounds through a game.

The fix is three changes that work together. First, `addSupplyUsed` moves from `handleTrain` to `startTraining` — supply is consumed only when a unit actually begins production. Second, the supply check at queue time now includes *projected* supply — iterating all building queues to count their committed supply cost. This prevents over-commitment: you can't queue five Marines when you only have supply for two, because the check knows three are already waiting. Third, `drainBuildingQueues` gets its own supply gate — before popping the next unit from a queue, it checks whether supply is actually available. If not, the unit waits.

The projected supply check is the piece that kept this from being a trivial move-the-line fix. Without it, the queue would accept unlimited units (since supply wasn't consumed on queue entry anymore), burn their minerals, then have them stuck indefinitely waiting for supply that was never going to arrive. The check prevents that by including queued units in the supply calculation at queue time without actually consuming their supply.

Protoss was the unexpected beneficiary. The calibration jump was dramatic — 49 to 54 Probes at tick 305. With two Nexuses producing continuously, the old code was locking 4-5 supply slots in queued Probes, effectively capping production several Probes early. Freeing that supply headroom lets the second Nexus actually use the supply that Pylons provide.

A second fix landed in the same branch: `resolveBuilding` for ability casters. When the playbook fires a MULE calldown via `AbilityIntent("r-orbital_command", ...)`, the resolver was returning the first matching Orbital Command regardless of energy. With two OCs, OC-1 would drain its energy on calldowns while OC-2 accumulated energy it never spent. The fix iterates all matching buildings and tries `handleAbility` on each until one succeeds — the first OC with sufficient energy wins. `handleAbility` returns false without side effects when energy is insufficient, so the iteration is safe.

A side effect of testing this surfaced a gap in `spawnBuildingForTesting` — it wasn't calling `onBuildingComplete`, so OCs spawned for tests had no starting energy and Hatcheries had no larva. Normal game flow calls `onBuildingComplete` when a building finishes construction; the test helper was silently skipping it. Adding the callback brought test buildings in line with real game state.

The economy model still has headroom before it matches real SC2 numbers. The Terran playbook gets 43 SCVs where a real game with identical build order would hit 48-50. Some of that gap is MULE calldown timing, some is build order optimisation the playbook doesn't attempt yet. But the supply accounting is no longer the bottleneck.
