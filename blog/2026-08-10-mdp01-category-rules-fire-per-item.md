---
layout: post
title: "When Category Rules Fire Per-Item"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [drools, dominance-weights, category-matching, design-review]
---

The dominance weight system started with hand-picked rules — specific archetypes matched by enum value, with delta estimates pulled from domain knowledge rather than data. BANSHEE_HARASS got its own rule. BIO_TIMING and MECH_PUSH shared another. Four of six archetype categories had no weight response at all.

The gap was structural, not just coverage. RUSH and MACRO already matched by category — `archetype().category() == ArchetypeCategory.RUSH` — so any new rush archetype automatically got a weight response. But TIMING only covered two of six archetypes via specific enum checks, and HARASS covered one of nine. Every new archetype added to these categories would silently produce no weight adjustment.

We normalised the entire rule set to category-level matching. Six categories, two confidence tiers each (high >=0.6, moderate >=0.3), twelve archetype rules total. The old push and banshee rules were deleted — their high-confidence deltas became the TIMING and HARASS category rules. TECH and COMPOSITION got new rules with deltas derived from SC2 domain knowledge: tech threats demand counter-tech investment; formed army compositions demand army commitment and deferred expansion.

The combined signal rules followed the same normalisation — the old "Push + MID_SKIRMISH" rule matched `StrategyArchetype.TERRAN_BIO_TIMING` by enum; the new version matches `ArchetypeCategory.TIMING`. We added five new combinations where SC2 domain knowledge justified them: harass under defensive pressure demands economy recovery and detection tech, a tech race during macro play calls for doubling down, and a formed composition at your base demands maximum army commitment.

The interesting part was what the design review caught. Drools DataStore pattern matching fires per item in the store, not per category. A rule matching `archetype().category() == ArchetypeCategory.COMPOSITION` creates one activation per `PatternAssessment` whose category is COMPOSITION — not one activation for the COMPOSITION category as a whole. If three composition archetypes were detected simultaneously (MARINE_TANK at 0.7, BIO_MINE at 0.6, GHOST_BIO at 0.5), the category rule fires three times, combined rules fire three times, and the resulting army weight hits 67% after normalisation.

The mental model — "category rule fires once per category" — is wrong. It fires once per matching DataStore item. The DRL syntax reads as a filter, but the semantics are per-item activation. In single-assessment scenarios, which is what most tests exercise, the behaviour looks correct. The bug only surfaces with multiple same-category assessments, which the pattern classifier legitimately produces.

The fix is Java-side dedup before DataStore insertion: keep the highest-confidence assessment per category using `Collectors.toMap` with a merge function, then insert the deduplicated list. The DRL rules stay simple — they can assume one assessment per category because the strategy layer guarantees it. Trying to solve this in DRL with `accumulate` would have added complexity to every rule and interacted badly with the combined signal rules.

The confidence threshold change was deliberate. The old push rule fired at >=0.5; the new TIMING high tier fires at >=0.6. An assessment at 0.55 now gets the moderate-tier response instead of the old full-strength one. At mid-confidence the classifier is uncertain, and a weaker response is appropriate. The old single-tier thresholds were initial estimates — the two-tier system is the calibrated replacement.

The rule set went from thirteen to twenty-five rules covering the full archetype taxonomy. The per-category dedup means the rule count doesn't affect stacking risk — every category produces exactly one modifier regardless of how many archetypes the classifier detects. The combined signals add domain-specific fine-tuning where category plus tactical posture together justify a different response than either alone.
