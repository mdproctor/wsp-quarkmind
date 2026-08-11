---
layout: post
title: "Data Tables Aren't Dispatch Tables"
date: 2026-07-09
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [domain-model, sc2data, design-patterns]
---

SC2Data had a quiet bug. Three cost methods — `mineralCost`, `gasCost`, `supplyCost` — used switch expressions with `default` clauses. When the `UnitType` enum grew from 6 types to 67, most new entries silently got the default. A fleet of 4 Carriers, 3 Colossi, and 2 Void Rays should value at 4,700 resources. SC2Data computed 900. An 81% undervaluation, invisible because the compiler was satisfied.

The `default` clause is the trap. Java switch expressions over enums are exhaustive by default — add a new enum constant and the compiler forces you to handle it. But a `default` case disables this. Every method that had one was a gap waiting to accumulate.

The obvious fix: remove the defaults, list every constant explicitly. That's what we did for the sealed `Intent` pattern in the harness layer — each Intent subtype gets a distinct switch arm because each arm contains distinct behaviour. The compiler catches new subtypes at compile time. Clean.

But unit costs aren't behaviour. They're data. Three co-varying values per type (mineral, gas, supply) scattered across three parallel switch expressions, each with identical arm structure. Adding a new unit means editing three separate switches. Worse, you can fix one and forget the others because they're independent edits.

The right pattern for a data table over an enum is an `EnumMap` with a completeness check in the static initialiser. One `UnitCosts` record per type — `(int mineral, int gas, int supply)` — on a single line. One entry to add, one entry to verify, one place to read a unit's full cost profile.

The tradeoff: enforcement moves from compile time to class-load time. A missing entry throws `ExceptionInInitializerError` when SC2Data loads. In practice, that fires before any test can run — every test in the suite transitively touches SC2Data.

The adversarial spec review earned its cost on this one. It caught three data errors before a line of implementation was written: Colossus supply was 4 in my spec — Liquipedia says 6, and it's been 6 since Wings of Liberty. Tempest supply was 5 — patch 5.0.14 dropped it to 4. And HELLBAT was missing from the transformation forms category entirely, even though it's the same reversible pattern as SIEGE_TANK_SIEGED and VIKING_ASSAULT.

The Zergling needed its own paragraph in the Javadoc. SC2 charges 50 minerals for a training command that produces 2 zerglings. Our record stores per-individual costs: 25 minerals each, 1 supply per command. Consumers that deduct costs per training command need to multiply by `trainCount`. That convention was implicit before — now it's explicit on the record.

The `MultiFactorDominanceAssessor` sees the real army now. A late-game Protoss deathball doesn't look like 900 minerals of default-valued units. It looks like what it is: 4,700 resources of Carriers, Colossi, and Void Rays.
