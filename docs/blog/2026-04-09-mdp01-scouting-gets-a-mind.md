---
layout: post
title: "QuarkusMind — Scouting Gets a Mind"
date: 2026-04-09
---
# QuarkusMind — Scouting Gets a Mind

**Date:** 2026-04-09
**Type:** phase-update

---

## What I was trying to achieve: closing the fourth plugin loop

Four plugin seams, four frameworks. Strategy got Drools forward-chaining rules.
Economics got Quarkus Flow. Tactics got Drools plus a hand-rolled GOAP planner.
Scouting was the last one, and the plan was Drools Fusion CEP — temporal event
accumulation for build-order detection. Can we tell a Zerg roach rush from a
Terran 3-rax by watching unit sightings accumulate over time?

`BasicScoutingTask` was already working: passive intel (army size, nearest
threat) and an active probe scout. The new layer was the intelligence.

## What we believed going in: Drools Fusion would handle the time windows

The natural path is `window:time(3m)` in the DRL, STREAM event processing
mode, a `KieSession` with a `PseudoClock`. That's the documented CEP approach.

I brought Claude in to work through the implementation. We hit the wall
quickly: the `drools-quarkus` extension builds around the rule unit model —
`RuleUnitInstance`, `DataStore<T>` — not traditional `KieSession`. STREAM
mode needs `kmodule.xml` configuration that conflicts with how the extension
compiles its KieBase. They don't coexist cleanly.

## Drools as accumulator, Java as timekeeper

So we pivoted. Instead of Drools managing the time windows, Java does.
`ScoutingSessionManager` maintains three `Deque<Event>` buffers — unit
first-seen events (3-minute window), army-near-base events (10-second window),
expansion sightings (permanent). Each tick: evict expired events, build a
fresh `ScoutingRuleUnit` from the current buffers, fire a `RuleUnitInstance`.
The rules themselves are simple accumulators:

```drl
rule "Zerg Roach Rush"
when
    accumulate(
        /unitEvents[ this.type() == UnitType.ROACH ];
        $count : count();
        $count >= 6
    )
then
    detectedBuilds.add("ZERG_ROACH_RUSH");
end
```

The temporal logic stays in Java — easy to test, easy to reason about.
The rules stay stateless. We lost the elegance of `window:time()` but
gained a pattern that actually compiles.

One thing the code review caught: the first implementation only wrote
`ENEMY_BUILD_ORDER` to the CaseFile when a build was detected — absent
otherwise. That's inconsistent with `ENEMY_POSTURE` (always written, with
"UNKNOWN" as fallback) and with what `producedKeys()` advertised. Claude
flagged it as the only Important issue. Fixed to always write with "UNKNOWN"
as the fallback.

## QuarkusMind

Four plugins done — and the project still had a placeholder name. "starcraft"
was the repo folder, the Maven artifact, the package root. Fine for a
testbed; wrong for something that's becoming a platform.

I wanted a name that worked across races (we started Protoss, but that won't
stay true), signalled the tech stack, and wasn't purely a StarCraft reference.
After some back-and-forth: **QuarkusMind**. Quarkus in full — no abbreviation.
Mind for the intelligence layer, the CaseHub blackboard, and a quiet nod to
the Zerg OverMind that the multi-agent architecture has always been channelling.

The rename was a full refactor: `org.acme.starcraft` → `io.quarkmind`,
`StarCraftCaseFile` → `QuarkMindCaseFile`, the GitHub repo, the Maven
coordinates. SC2-specific code kept its SC2 references — the `sc2` sub-package,
`@CaseType("starcraft-game")`, replay paths. Those things are factually about
StarCraft 2. Renaming them would just be wrong.

One gotcha from the rename: `mv /starcraft /quarkmind` exits 0 and the shell's
cwd becomes invalid — every subsequent bash call fails with "Working directory
no longer exists." Do the folder rename last.
