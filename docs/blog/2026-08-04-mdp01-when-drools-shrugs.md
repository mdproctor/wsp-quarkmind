---
title: "When Drools Shrugs"
date: 2026-08-04
author: Mark Proctor
tags: [quarkmind, scouting, llm, drools, pattern-classification, casehub]
status: draft
---

The pattern classifier has a blind spot. Drools rules fire on known archetypes — a gateway rush looks like a gateway rush because the CEP pattern matches: early gateway, zealot production within a time window, no expansion. It works. But not everything the enemy does fits a known archetype. Some builds are transitional, some are improvised, and some are just weird. When nothing matches, every archetype sits below the confidence threshold, and the scouting pipeline reports nothing. Downstream consumers — strategy, tactics, advisory — get silence when they most need a signal.

The fix is an LLM fallback that fires when Drools can't reach 0.5 confidence on any archetype after three minutes of game time. Three minutes is enough for the evidence buffer to have real data — unit first-seen timestamps, army positions, expansion activity — but not so early that absence of signal is just noise.

## The Integration Problem

The interesting part isn't calling an LLM. It's how the result flows back into a pipeline that wasn't built for it.

The scouting pipeline has a dual-stack publishing path: `ScoutingIntelBroker` for latest-value caching and a qhorus channel for audit trail. Pattern assessments flow through both via `publishIntel()`. The LLM result needs to enter the same path — not as a separate stream that consumers have to know about, but as a `PatternAssessment` that looks identical to a Drools-produced one.

We built this as a standalone CaseHub capability. `DroolsScoutingTask` sets a trigger key on the CaseContext when all three gates pass — confidence below threshold, game time past the window, cooldown expired. CaseHub's `ContextChangeTrigger` fires a Worker that calls the LLM. The Worker writes its classification back to CaseContext keys. On the next tick, `DroolsScoutingTask` reads the result and overrides the cumulative confidence for that archetype.

The read-on-next-tick pattern avoids the alternative — sharing mutable state between CDI beans or introducing a new CDI event for a one-directional data flow. The CaseContext is already the communication channel between plugin phases. Using it here keeps the data flow consistent with how every other plugin communicates.

## The Prompt

The LLM gets two things: the chronological unit observation timeline (unit type and timestamp for every enemy unit first observed) and Drools' current confidence map (showing what it tried and how far it got). The archetype list is filtered to the enemy's race — a Protoss player can't be doing a zergling rush.

Constraining the output to the existing `StrategyArchetype` enum means consumers don't need to handle a new type. The LLM picks from the same vocabulary Drools uses. If it returns something invalid, the result is silently dropped.

## Decay

The override isn't permanent. Once the LLM's confidence is injected into the cumulative map, Drools' exponential decay (`DECAY_PER_FRAME = 0.99948`) applies on every subsequent tick. If new evidence contradicts the LLM — say the enemy abandons the rush and transitions to macro — the confidence erodes naturally. The LLM gave the system a starting point; the evidence either sustains or undermines it.

## The Reuse

The advisory infrastructure already has a `CompletionCallback` that fires `AdvisoryCompleted`, which feeds `AdvisoryChannelBroker` (audit trail) and `DeferredAdvisoryEvaluator` (outcome scoring). We reuse the same callback with a distinctive `advisorId` — `"llm-classifier:pattern-fallback"`. The LLM classification gets the same audit and evaluation treatment as a crisis or strategic advisory, without a single new CDI event type or observer.

The factory lives in `plugin/scouting/`, not `plugin/advisory/`. It classifies what the enemy is doing — that's scouting's domain. The fact that it calls an LLM is an implementation detail, the same way `DroolsScoutingTask` calling Drools doesn't put it in a `plugin/drools/` package.
