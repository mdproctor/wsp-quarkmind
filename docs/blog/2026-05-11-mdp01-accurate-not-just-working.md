---
layout: post
title: "Accurate, Not Just Working"
date: 2026-05-11
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [architecture, documentation, quarkus, java]
excerpt: "A tier-4 health check before Phase 6 finds the README still naming replaced stubs and the emulated mode guide completely absent — accuracy over progress."
---

Phase 6 was the plan. I didn't start it.

Not because it wasn't ready — the infrastructure exists, the plan is written. But pausing before a new phase to ask "what's unfinished?" has saved more time than it's cost. We ran a tier-4 project health check instead.

Tier-4 is the full sweep: documentation accuracy, internal consistency, architecture integrity, dependency health, user journey, git state. Takes fifteen minutes; occasionally finds something real. This time it did.

The README's plugin table still listed `PassThroughTacticsTask` and `PassThroughScoutingTask` as the current implementations. Those stubs were replaced by `DroolsTacticsTask` and `BasicScoutingTask` months ago. A reader following the README would get a false picture of what actually runs.

`docs/running.md` had no emulated mode section. None. The `%emulated` profile — physics simulation, live Three.js visualizer, enemy AI — was completely absent from the user guide. We'd built the whole thing and never documented it.

The architectural finding was more interesting. Claude flagged that `GameStateTick` lived in `domain/` but imported `ResourceBudget` from `agent/`. Domain is supposed to be the bottom of the stack: plain Java records, no imports from layers above. This was an upward dependency violation hiding in plain sight.

`GameStateTick` was never a domain class. It's a flow-specific DTO — an immutable snapshot carrying game state plus a resource budget, emitted once per tick by `FlowEconomicsTask`. The consequence of its misplacement isn't just aesthetic: any code treating `domain/` as a clean, dependency-free foundation would silently pull in `agent/` through it. The layer separation is what makes the domain model reusable across engine implementations.

Moving it to `plugin/flow/` exposed the wildcard import trap. Tests using `import io.quarkmind.domain.*` stopped resolving `GameStateTick` silently. The moved class also lost implicit access to `Unit`, `Building`, and `Resource` — types it previously used without imports because they were in the same package. Two separate compilation failures from one move, both reporting `cannot find symbol`, neither looking different from a missing class.

DESIGN.md had three different test counts, all wrong. The scouting calibration entry in Next Steps referenced closed issues. `pixi.min.js` was listed as "pending removal" despite being deleted weeks ago.

Small things. But a project where the docs don't reflect reality is a project where you stop trusting the docs.

`pom.xml` had `mockito-junit-jupiter` pinned at 5.11.0. Quarkus BOM 3.34.2 manages it at 5.21.0. Maven takes the project-level declaration without warning — the BOM version is silently overridden. We dropped the pin.

Phase 5 completion is next: parallel training queues, friendly auto-engage, a proper mineral saturation model, and the deferred visualizer work. The project was working before. Now it's accurate.
