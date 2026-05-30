---
layout: post
title: "Three races, one seam"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulated-game, race-model, terran, zerg, scelight]
---

The `EmulatedGame` has been Protoss since the day it was built. Twelve probes, a Nexus, 15 supply — all hardcoded. Adding Terran and Zerg (#138) forced a decision I'd been putting off: where does race-specific logic actually belong?

Claude's first instinct was inline switch branches — a case for Protoss seeding here, a case for Terran MULE handling there. The class is already 800 lines; what's another few hundred of conditionals? I pushed back. I've watched SC2-specific code grow unpleasant before, but the more important argument is #74: the long-game vision of pluggable races means SC2 has to be a plugin eventually, not baked into the engine. Building inline today creates a bigger excavation later.

So we went with a plugin seam instead. By the time we had three options laid out — inline switches, a `RaceModel` interface, or subclasses — the third fell away immediately (can't have both players use the same engine as different subclasses), leaving `RaceModel` as the right call. Six methods: seed initial state, run passive per-tick mechanics (larva regen, MULE expiry, Queen energy), check if production can proceed, commit it, respond when a unit spawns, and report how many units one training order produces. `EmulatedGame` holds a `ProtossRaceModel` by default and delegates everything race-specific through those six methods.

## The spec review caught the most interesting bug before implementation started

The original design had a single `consumeProductionResource()` method that would decrement larva and spawn an Egg — called before the resource check. For Zerg this means: the larva is gone and an Egg is sitting in the unit list before we know if the player has minerals. If they don't, the Egg is stranded and the larva is lost. The ordering was backwards and I hadn't noticed it.

Claude split the method into two phases: `canProduce()` (read-only, called before resource deduction) and `onProductionCommitted()` (mutates state, called after). This isn't just cleaner — it's why the tests pass correctly. `trainRoach_noLarva_rejected_noResourceDeducted` asserts minerals are unchanged after a larva-blocked rejection; without the split, that assertion would catch a real bug on the first run.

## The three implementations ended up with satisfying personalities

Protoss is almost entirely empty — seeds twelve probes and a Nexus, and most interface methods are no-ops. That's the design validation: if adding a race takes under fifty lines, the seam is in the right place.

Terran is the most self-contained. MULEs are called down via `canProduce` returning HANDLED — the model spawns the unit immediately, registers an expiry loop, and the main handler exits without touching the queue. Each tick, `tickPassive` checks for expired MULEs, removes them, and adds flat income for the survivors. MULE income rate confirmed from SC2 research: 3.45× SCV rate. Worker income is race-invariant across all three races — all three workers mine 5 minerals per trip on the same cycle — so `mineralIncomePerTick()` stays untouched.

Zerg is the most structural. The larva counter per hatchery is a map inside `ZergRaceModel`, not a field on `PlayerState` — the physics engine doesn't need to know how larvae work, only that when `canProduce` returns PROCEED, a unit can be morphed. The tricky part was the Egg queue. Multiple units can be in-flight from the same hatchery simultaneously; the first attempt used a single `Map` from hatchery tag to egg tag, which overwrote on the second morph. The fix was a `Deque` per hatchery in FIFO order; `onUnitSpawned` polls the head. Queen inject fires automatically in `tickPassive` when energy crosses 25 — this deviates from real SC2 (it's player-commanded and frequently delayed or missed) but is documented as a known divergence.

## The AbilityMappingSelectionDeltaTest construction trap

The other thread from last session — synthetic Scelight event coverage for `onSelection` (#163) — had its own construction surprise. The `SelectionDeltaEvent` constructor takes a `baseBuild` parameter I'd been passing as `0`. Below build 16561, the constructor runs an old-format compatibility block that rewrites the `removeMask` by path-navigating through the delta struct. With a synthetic struct, this silently replaced every variant with a null-payload Mask. All the `ZeroIndices` and `OneIndices` tests behaved as carry-forward with no error and no exception.

The fix is passing `16561` instead of `0`. Finding it required decompiling `Delta.class`, reading the `SelectionDeltaEvent` source, and tracing why a `Map` stored where a `Pair` was expected caused silent null reads rather than a `ClassCastException` — Java's unchecked generic cast means the type mismatch is invisible at the assignment and only surfaces as garbage field access. Two garden entries from that.

828 tests passing. `countProbesPerBase` is now `countWorkersPerBase`. ADR-0007 recorded for the `RaceModel` seam decision. #138, #140, and #163 all closed.
