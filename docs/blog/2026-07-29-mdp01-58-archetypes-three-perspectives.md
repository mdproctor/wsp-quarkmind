---
layout: post
title: "58 Archetypes, Three Perspectives"
date: 2026-07-29
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [taxonomy, counters, detection]
series: issue-257-fill-strategy-taxonomy
---

The two-track detection architecture from the previous session gave us the rails. This session filled them in.

Twenty archetypes grew to fifty-eight — enough to cover all three races across early, mid, and late game. The existing nine early-game entries already had Terran and Zerg covered in broad strokes (marine rush, zergling rush, roach rush, the macro openings). What was thin was mid-game and late-game: only eight mid and three late. The real diversity of SC2 strategy lives in the mid-game transitions, where a player commits to a composition that defines the rest of the game. Ling-bane-muta. Roach-ravager. Bio-mine. Ghost-bio. Lurker contain. These aren't exotic builds — they're what you see in every competitive match. Leaving them out of the taxonomy meant the coaching system went blind after five minutes.

The thirty-eight new entries went in as pure YAML. Each one needs a detection signature (which unit types at what counts within what time window), counter recommendations, and human-readable detection signals. The generic pipeline handles the rest — no DRL rules needed for any of them. The `PatternClassificationRuleUnit` matches `SignatureSpec` records against unit counts and fires evidence markers automatically. That's the payoff of the two-track design: adding coverage is an authoring task, not an engineering task.

## The Counter Schema Had to Break

The original counter data was Protoss-only. Every archetype's `strongCounters` and `weakCounters` listed Protoss units — Stalkers, Immortals, Colossus, Storm. That made sense when QuarkMind was a single-race agent. It stopped making sense the moment the taxonomy claimed to describe what the *enemy* was doing. A Terran player facing a Zergling rush doesn't care that Stalkers are good against Lings. They need to know that a Bunker at the natural with Marines inside trades four-to-one.

The fix was a schema migration: flat `strongCounters`/`weakCounters` lists became a `counters` map keyed by player race — `PROTOSS`, `TERRAN`, `ZERG`, each with its own `strong` and `weak` sections. The Java side gained a `countersFor(archetype, race)` overload, backed by an `EnumMap<Race, CounterInfo>` in `ArchetypeEntry`. The existing single-argument method delegates to `countersFor(arch, Race.PROTOSS)`, so the coaching pipeline — which currently only coaches Protoss players — works unchanged.

Fifty-eight archetypes times three races is a hundred and seventy-four counter sections. Each one needs SC2-accurate advice: which units to build, why they work, what to avoid and why. Terran counters lean on Siege Tanks, Vikings, and Ghosts. Zerg counters lean on Queens, Corruptors, and Vipers. The patterns are different enough that you can't generate one race's counters from another's — the advice is genuinely matchup-specific.

One gotcha surfaced during authoring: I wrote `BUNKER` as a Terran counter unit and the taxonomy blew up at init time. `UnitType` only models mobile units — buildings aren't in the enum. The error was a bare `IllegalArgumentException` from `Enum.valueOf()` with no indication which of the two thousand YAML lines caused it. The fix was obvious (use `MARAUDER` instead), but the diagnostic path — binary-searching a large YAML file for an unnamed offending entry — is the kind of thing worth writing down so the next person authoring counter data doesn't waste the same fifteen minutes.
