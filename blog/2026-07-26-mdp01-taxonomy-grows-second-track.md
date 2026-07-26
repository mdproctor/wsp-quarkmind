---
layout: post
title: "The Taxonomy Grows a Second Track"
date: 2026-07-26
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [drools, detection, taxonomy, coaching]
series: issue-243-tactic-taxonomy
---

The strategy detection system started with ten archetypes — all early game, all hand-authored Drools rules. Zergling rush, marine rush, cannon rush, gateway pressure. The kind of thing you can recognise from a three-minute replay: count the units, check the clock, call it. That was enough for the coaching pipeline to give advice about what was happening right now. It was nowhere near enough for mid-game or late-game.

The design question was where the new knowledge should live. Fifty to ninety compositions across all game phases and matchups is a different problem than ten hand-crafted rules. Hand-authored DRL is precise but doesn't scale — each archetype needs its own evidence rules, counter-indication rules, and calibration. The taxonomy needed a data layer: structured YAML defining what each archetype looks like (unit signatures, time windows, counter recommendations), with generic Drools rules that match the data rather than encoding the knowledge.

The result is a two-track architecture. Track one: the original hand-authored rules. Ten archetypes with bespoke DRL evidence logic — marines before four minutes, medivacs in mid-game, siege tanks after five. These stay because their detection logic is complex (multi-unit boolean combinations, army-near-base spatial checks, expansion counter-indication cascades). Track two: a generic pipeline where `SignatureSpec` records from the YAML taxonomy are loaded into the rule unit's `signatureStore` at each tick, and two data-driven rules handle the matching. "Does this unit type exceed the minimum count within the time window?" and "Is the no-expansion flag set with no expansion events?" That's it. Every new archetype added to the YAML file is automatically detectable — no DRL required.

The plan originally called for a separate `SignatureFact` record and a `SignatureFactBuilder` CDI bean. I dropped both. `SignatureSpec` already existed in `StrategyTaxonomy` with identical fields — creating a copy would have been unnecessary abstraction. And the builder was a one-method filter that belongs on the taxonomy itself: `activeSignatures(double gameTimeMinutes)` returns the specs for non-hand-authored archetypes whose time window overlaps the current game time.

That decision surfaced a Drools gotcha worth recording. `SignatureSpec` was originally a nested record inside `StrategyTaxonomy`. Java compiles fine, IntelliJ shows no errors, the DRL `import io.quarkmind.agent.StrategyTaxonomy.SignatureSpec` is valid syntax. But the Drools Quarkus code generator translates nested types to `StrategyTaxonomy$SignatureSpec` in its generated Java source — and the KieMemoryCompiler can't resolve the `$` separator. The error only appears at Quarkus augmentation time, long after you've stopped looking at the code that caused it. The fix was promoting `SignatureSpec` to a top-level record in `io.quarkmind.domain`. Clean, but only obvious after the fact.

The coaching pipeline now gets counter data for free. When a pattern assessment identifies an archetype, `CoachingWorkerFactory` looks up `STRONG COUNTERS` and `WEAK COUNTERS` from the taxonomy and threads them into the LLM prompt. The coach can now say "they're going Roach-Hydra — you need Colossus splash" instead of just "enemy pattern detected." `TimeBasedPhaseResolver` classifies the game into EARLY/MID/LATE by elapsed time and writes `GAME_PHASE` to the CaseFile, so the prompt also carries the current phase.

Twenty archetypes now — ten hand-authored, ten data-driven. The hand-authored ones cover early-game rushes and pressure builds where detection logic needs boolean complexity. The data-driven ones cover mid and late-game compositions where the signature is simpler: Roach-Hydra is three roaches and two hydralisks after five minutes. Battlecruiser transition is one BC after ten. The generic rules handle these without writing a single line of DRL per archetype.

The fifty-to-ninety target from the original issue is a YAML editing task now, not a rules engineering task. That's the real delivery — not the twenty archetypes themselves, but the infrastructure that makes the next thirty trivial.
