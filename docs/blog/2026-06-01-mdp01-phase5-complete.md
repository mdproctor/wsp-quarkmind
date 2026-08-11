---
layout: post
title: "Phase 5 Complete: EmulatedGame Accuracy Gaps Closed"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [emulatedgame, phase5, sc2, replay-validation]
---

Phase 5 is done. I expected to branch and start coding. The test suite said otherwise: all four child issues closed, 828 unit and integration tests passing.

The verification was the work. Before opening a branch, we ran the full suite to confirm Phase 5's definition of done. It was satisfied. Then we cleaned up the docs — DESIGN.md had stale test counts scattered across three locations. The Playwright count had drifted: 251 in the docs, 288 in the codebase. Claude caught it during the doc review.

Phase 5 covered four accuracy gaps in EmulatedGame:

**Parallel training queues** — each building now maintains its own queue. Supply is reserved at queue time, not on completion. The real SC2 consequence: you can't queue into over-supply; the emulation now correctly blocks it at reservation.

**Auto-engage** — friendly units attack enemies in range without a dispatcher sending an AttackIntent. It's easy to forget real SC2 combat is automatic. Without this, units stood next to enemies and did nothing.

**Mineral saturation** — flat +5/tick replaced by the actual saturation model: linear up to 16 workers per base, diminishing returns through 24, flat above that. This closes the biggest single source of income divergence in replay validation.

**Visualizer work** — the last entry covers this in detail: mineral HUD with colour tiers, probe spread, canvas sprites for patches and geysers, time-based test API.

EmulatedGame is now a credible closed-loop testbed. Divergence in Phase 6 replay validation should reflect real physics gaps, not missing mechanics.
