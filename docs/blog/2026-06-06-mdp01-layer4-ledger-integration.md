---
layout: post
title: "Layer 4: the ledger watches the plugins"
date: 2026-06-06
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [casehub-ledger, cdi, quarkus, layer4]
---

Two things shipped today. The first was small — a null correctness fix in `InMemoryCaseFile` that we've known about since #172 surfaced the problem. The second was the Layer 4 milestone: every agent plugin now reports its decisions to casehub-ledger.

The `InMemoryCaseFile` fix looked like one line. It was four. `get()` used `Optional.of(null)` when the stored value was null, which NPEs — the documented symptom. But the real fix was also guarding `put()` and the constructor. The constructor silently stored null values from its `initialState` map, which meant `put(key, null)` followed by `put(key, "new")` would call `fireEvent` with `previous.value() == null`, hitting the same `Optional.of(null)` on a different path. Design principle: if the invariant is "null never enters the store", enforce it at every entry point, not just the one that surfaced the symptom. `putIfAbsent` and `putIfVersion` needed the same guard.

The ledger integration took most of the session because the design had to be exactly right before we touched code. I wanted the plugins clean — no knowledge of ledger infrastructure — so the architecture was CDI async events: plugins fire `PluginDecisionEvent` via `Event.fireAsync()` when they detect a state transition, and `PluginOutcomeAuditor` `@ObservesAsync` picks it up and writes to `OutcomeRecorder`. Transition-level, not tick-level — firing every 500ms per plugin would be 88 ledger entries a second of noise.

The spec went through five review rounds and each one found something real. Round one: I had the `AttestationVerdict` enum values completely wrong — `SUCCESS` and `UNCLEAR` don't exist. The actual values are `SOUND`, `FLAGGED`, `ENDORSED`, `CHALLENGED`. SOUND for any legitimate plugin output including "nothing to do", FLAGGED for genuine errors only. Using FLAGGED for "no enemies visible this tick" would have systematically penalised plugins for doing their job. Round two: `DefaultOutcomeRecorder` throws `IllegalStateException` unless `casehub.ledger.outcome.default-attestor-id` is configured — silent at compile time, fatal at first invocation. Round three: eight `@Alternative @Priority(1)` beans from `casehub-ledger-memory`, all requiring explicit `quarkus.arc.selected-alternatives` in every non-JPA profile (test, mock, emulated, replay). Classpath presence alone does nothing.

The implementation revealed two more things worth knowing. First: `Event.fire()` does not deliver to `@ObservesAsync` observers. Not a configuration problem, not a CDI version thing — the spec is explicit that synchronous delivery reaches `@Observes` and async delivery reaches `@ObservesAsync` and the two are separate channels. We fired and the observer never ran, no error, nothing. Switching to `fireAsync()` fixed it. Second: `Unit` is a Java record, `record Unit(String tag, UnitType type, Point2d position, ...)`. Record `equals()` and `hashCode()` include all components. Position changes every tick. Using `unit.equals(other)` or a `List<Unit>.hashCode()` for transition detection fires a ledger event on every tick for every moving unit. The fix is always to compare by `unit.tag()` — a stable string identifier. I got this wrong in the spec and the reviewer caught it.

For `LedgerLifecycleAdapter` — the component that clears `InMemoryLedgerEntryRepository` when the game stops — the `clear()` method doesn't exist on the SPI, only on the concrete in-memory implementation. Injecting the concrete type via `@Any Instance<InMemoryLedgerEntryRepository>` and calling `isUnsatisfied()` before `get().clear()` means the adapter is a no-op when the JPA implementation is active in the `%sc2` profile. No profile annotation, no null check, no conditional compile. CDI handles it cleanly.

Layer 4 is closed. The plugins now produce a ledger trail. Layer 6 (trust routing, #158) can read from it once the trust scorer runs.
