---
layout: post
title: "E13: Scouting Calibration"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkus, drools, sc2, replay, calibration]
---

The plan for #16 assumed both replay datasets would feed into `ReplaySimulatedGame`.
That lasted about fifteen minutes. The IEM10 dataset is pre-processed JSON —
`*.SC2Replay.json` — and `ReplaySimulatedGame` takes binary `.SC2Replay` files via
RepParserEngine. Two formats, two parsers needed.

I decided on a full `SimulatedGame` subclass for the JSON side rather than a
lightweight extractor. If we're going to stress the scouting pipeline against real
data, it should run the real pipeline. So: `IEM10JsonSimulatedGame extends SimulatedGame`,
same tick model, same interface, `applyIntent()` a no-op.

The SC2EGSet JSON has the same tracker events as the binary — `UnitBorn`,
`PlayerStats`, `UnitDied` — but different field names and one encoding difference
I found before writing any Java. A quick Python inspection of the ZIP confirmed:
supply/food values in the binary format are fixed-point ×4096 (`getFoodUsed()`
returns 49152 for 12 probes). The JSON has them decoded (`scoreValueFoodUsed: 12`).
Same field name, different encoding, not documented anywhere. If we'd ported the
`/ 4096` from the binary parser, every supply value would have silently rounded to 0.

We built the class TDD-first as a subagent task. The tests drove the interface:
`enumerate(Path outerZip)` streaming nested ZIP entries, `matchup()` auto-detecting
Protoss from `ToonPlayerDescMap`, the full tick/reset/isComplete cycle. The
implementer came back with one surprise: `java.util.zip.ZipInputStream` rejects the
outer ZIP with `invalid compression method`. No further detail. The outer ZIP uses
BZip2 (method 12) — valid per the spec, rare in practice, unsupported by the
standard library. We added `commons-compress` and swapped `ZipArchiveInputStream`
for the outer layer only; the inner `_data.zip` uses DEFLATE and works fine.

The code quality review caught two things. First, `toUnitType()`,
`toBuildingType()`, `defaultUnitHealth()`, `defaultBuildingHealth()`, and
`BUILDING_NAMES` were identical in both replay classes. If someone adds a unit
type to one and forgets the other, the tables silently diverge — exactly the kind
of thing that surfaces in a calibration test six months later. We pulled everything
into `Sc2ReplayShared`, a package-private final class in `sc2/mock/`.
`ReplaySimulatedGame` keeps a thin delegate method for the test that accesses it
directly. Second: `commons-compress` needed a `NATIVE.md` entry before it went in.

The calibration harness runs all 59 replays (29 AI Arena binary + 30 IEM10 JSON)
to 183 ticks — 3 minutes at SC2 Faster speed — and prints unit counts by matchup:

```
PvZ (11 games):  ROACH min=0  max=0  mean=0.0   ← ZERG_ROACH_RUSH (was: ≥6)
                 ZERGLING min=2  max=22  mean=6.0
PvT (11 games):  MARINE min=0  max=6  mean=4.4   ← TERRAN_3RAX (was: ≥12)
PvP (37 games):  STALKER+ZEALOT min=0  max=4  mean=0.8   ← PROTOSS_4GATE (was: ≥8)
```

The ROACH line is the interesting one. Zero roaches across all 11 PvZ games at 3
minutes — not because the dataset lacks aggression, but because Roach Rushes peak
at 4–5 minutes. The 3-minute window closes before they arrive. The threshold of 6
was never reachable on real data at this point in the game. That's a timing
constraint, not a calibration problem. We lowered it to 4 as a sensitive lower
bound and made a note in the DRL comment.

The Terran and Protoss numbers are more tractable. MARINE maxes at 6 by 3
minutes for ByuN — one of the most aggressive bio players of 2016. That makes
the threshold of 12 look like it was set without consulting a single replay. New
value: 5. The 4-gate combined peak is 4 across 37 PvP games, so the threshold
is now 4 there too.

478 tests, #16 closed.
