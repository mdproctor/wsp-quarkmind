---
layout: post
title: "ZeroIndices Doesn't Mean Zero"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

Three issues in one branch: unit tests for `IEM10CommandExtractor.applySelectionDelta`
(#160), a `SimulatedGame.isComplete()` base-contract test (#161), and Terran ability IDs
in `AbilityMapping` (#140). The first two went exactly as planned. The third didn't, in
the right way.

`AbilityMapping.onSelection` processes `SelectionDeltaEvent` from Scelight — the binary
SC2 replay parser. The method tracks which buildings a player has selected so that when a
train command fires, we know the building tag to attach to the intent. I'd written a
similar method the day before for the IEM10 JSON extractor, so I wrote the plan spec from
that experience: four variants, `Mask` is an integer bitmask, `SweepToEnd` truncates,
`OneIndice` removes one item, `None` means add without removing.

I brought Claude in to implement it. What came back was a correction, not an implementation.

Claude had read the Scelight source. The actual API looks nothing like what I described:

- `Mask` payload is `BitArray` (`hu.belicza.andras.util.type.BitArray`), not a `Number`.
  Casting to `((Number) value2).intValue()` compiles fine and throws `ClassCastException`
  at runtime.
- `ZeroIndices` doesn't mean "zero indices to remove." It means retain exactly these
  indices — payload is `Integer[]` of positions to keep, with empty array meaning clear all.
- It's `OneIndices` (plural), not `OneIndice`.
- `SweepToEnd` doesn't exist in the binary protocol at all. That's IEM10 JSON only.

The plan was wrong on three out of four points. We rewrote `onSelection` against the
actual API.

The silent part is the bug this would have caused. Bot replays use almost exclusively
`ZeroIndices` — the dominant variant, 56K to 109K events per game. Human replays add
`Mask` events, around 1,800 per game. If we'd shipped the original spec, `AbilityMapping`
would have worked on every bot replay we tested against, then quietly produced wrong
building tags on human replays — zero training intents, no exception, no log. The kind
that goes undetected until you need the data it was corrupting.

The Terran ability IDs were simpler than expected. I'd assumed we needed new replay files,
but the existing `aiarena_protoss` directory already had five PvT games. Cross-referencing
no-target train commands by frequency across three Terran-wins games gave the candidates:
155 for SCV (single index only, consistent across all replays), 159 for Marine and Marauder
(idx 0 and idx 3 respectively). We validated those against a full 18-minute Terran replay.
177 Marine train commands in that game — plenty of signal.

The outstanding item is #162: extracting a shared `SelectionState` type to consolidate the
variant-handling logic between the binary and JSON extractors. Right now they're two
independent implementations with the same bug surface. That's next when the binary replay
work picks back up.
