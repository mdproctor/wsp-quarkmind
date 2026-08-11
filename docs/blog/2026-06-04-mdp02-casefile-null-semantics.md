---
title: "CaseFile null semantics and three small fixes"
date: 2026-06-04
type: note
tags: [quarkmind, casehub, drools, maintenance]
---

Three small issues, one branch. #172 (NEAREST_THREAT null guard), #173 (Assimilator never built), #174 (stale doc refs).

The interesting one was #172. The issue assumed `put(NEAREST_THREAT, null)` would make `contains()` return false — the same behaviour as `CaseContextImpl.set(null)`, which is a no-op on absent keys. But `InMemoryCaseFile.put()` always writes an `ItemEntry(null, version)`, and `contains()` checks whether the `ItemEntry` object exists, not the value stored inside it. So `contains()` returns true, `get()` is called, and `Optional.of(null)` throws NPE. Two CaseFile implementations, opposite null semantics, nothing in the docs. Filed as casehub #175.

The quarkmind fix is `orElseThrow()` — makes the contract explicit. The structural fix lives in casehub (`Optional.of` → `Optional.ofNullable` in `InMemoryCaseFile.get()`).

I missed two things in #173 that Claude and I caught in code review. First: the original ASSIMILATOR handler called `budget.spendMinerals(75)` before confirming a free geyser existed. Geyser availability must be confirmed first, minerals spent after — otherwise 75 minerals vanish silently when the geyser list happens to be empty. Second: I'd written the `firstFreeGeyser()` unit tests inside `DroolsStrategyTaskTest` because it was the nearest test file. Wrong home — a pure static method has no business booting Quarkus. Moved to `DroolsStrategyTaskStaticTest`.

Small session. ARC42STORIES.MD corrected (L5 complete, ledger#114 no longer a blocker), handover skill updated with an arc42 stale scan step in the wrap checklist.
