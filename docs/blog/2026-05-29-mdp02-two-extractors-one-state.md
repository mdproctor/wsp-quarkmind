---
layout: post
title: "Two Extractors, One State"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [selection-state, refactor, binary-replay, iem10]
---

Issue #162 was filed yesterday during the #160 review — not a bug, just a
duplication I didn't want to leave in place. `AbilityMapping.onSelection()`
and `IEM10CommandExtractor.applySelectionDelta()` both maintain an ordered
list of currently selected unit/building tags and apply removeMask deltas to
it. Same logic; different input types. Two implementations meant two places for
the next correctness fix to land.

The answer was a shared `SelectionState` class in `io.quarkmind.sc2`. Not in
`domain/` — it's mutable state, not a record. Not in `sc2.replay` or
`sc2.mock` — neither caller should own the shared abstraction. The root
package fits.

The API question was what operations to expose. The obvious set: `removeAt`,
`truncateTo`, `retainOnly`, `addTag`. The less obvious one that code review
surfaced: `removeIf(IntPredicate)`. Both Mask variants — binary BitArray and
JSON integer bitmask — need to iterate descending and remove where a bit is
set. That descending-loop invariant is non-trivial. Remove an element at
index 5 and indices 6+ shift down; if you then try to remove what was index 6
you're already one off. That logic belongs once. The callers become one-liners:

```java
// binary Mask (BitArray)
selection.removeIf(i -> i < bitArray.getCount() && bitArray.getBit(i));
// JSON Mask (int bitmask)
selection.removeIf(i -> (mask & (1 << i)) != 0);
```

The same review caught a real bug: `retainOnly` with duplicate indices in the
input array would silently produce duplicate tags. The spec hadn't addressed
that case. The fix is a tag-level dedup check in the kept list — consistent
with `addTag`'s own dedup semantics.

One discovery worth noting: `SelectionDeltaEvent.getDelta()` in Scelight always
returns non-null. The replay parser populates the delta field unconditionally;
the null return type is an artefact of the generic `get(field)` accessor pattern.
The `delta == null` guard in `AbilityMapping.onSelection()` is unreachable. I've
left it in — costs nothing and the protocol could change — but it's now in the
garden as undocumented behaviour.

One item filed for later as #163: `AbilityMappingTest` has no tests for
the delta variants. The binary path — ZeroIndices, Mask via BitArray,
OneIndices — is exercised only through real replay files. The IEM10 extractor
test is the template; the binary equivalent needs synthetic Scelight events to
give the same coverage.

#162 is closed. The two extractors share a single class, and the tricky
descending-removal loop lives exactly once.
