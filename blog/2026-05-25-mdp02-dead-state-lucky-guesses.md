---
layout: post
title: "Dead State, Lucky Guesses, and a Layer That Didn't Exist"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

Five small issues. The word "small" proved relative.

## The field that survived two sessions too long

After auto-engage shipped, `resolveCombat()` stopped reading `attackingUnits`. It fires on `nearestInRange` and cooldown alone — no set membership check. But `attackingUnits` kept getting written: `setTarget` added or removed tags on each intent, blink cancelled attack mode by removing a tag, test-spawned enemies added theirs by default.

The kiting test had a comment explaining the mechanism: "MoveIntent clears `attackingUnits` so no fire this tick." That was wrong since April.

ADR-0003 had justified the Set explicitly — "cheap to query in `resolveCombat()`." We'd written that ADR while the field was still active. By the time we removed it, the ADR described code that hadn't existed for weeks.

When the code reviewer came back, it flagged one more thing: `setTarget(String, Point2d, boolean, PlayerState)` still took an `isAttack` flag that now only served a log statement. Dead field carries dead parameter. Both removed.

ADR-0003 is Superseded. DESIGN.md had a paragraph describing the field as retained dead state "pending cleanup" — that's fixed too.

## FACTORY confirmed, accidentally

The previous entry covered the addon contamination bug: 32 calibration observations for Factory build time, all addon completions at 400 loops, the whole set discarded. We replaced the measurement with a 960-loop estimate.

With the addon filter in place, 16 clean structure completions came back at exactly 960 loops.

The estimate happened to be right. Contaminated data produced a confident, incorrect modal — and we'd replaced it with a formula value that matched the ground truth anyway.

## Freezing enemies in a geometry test

Testing focus-fire convergence required stationary enemies. `spawnEnemyForTesting` sets `enemy.unitTargets` to the nexus position, so enemies drift each tick.

The fix: override with the unit's current position. `moveEnemyUnits` calls `stepToward(pos, pos)`, distance is zero, the unit stays put. This works because `moveEnemyUnits` does not remove the target when the unit arrives — unlike `moveFriendlyUnits`, which does. One is an open loop; the other isn't. Neither behaviour is documented anywhere. Reading both methods is the only way to know.

## The layer that never shipped

Writing LAYER-LOG entries for Layers 1 and 2 — naive loop then CaseHub blackboard — hit a problem immediately: Layer 1 never existed. QuarkMind's first `AgentOrchestrator` already used `CaseEngine`. The naive baseline was always a conceptual starting point, not a committed phase.

The Layer 1 entry documents the pattern Layer 2 replaced — gap comments naming the specific coordination requirements each subsequent layer closes. Same structure as the AML reference; different status: described, not deployed.

Layers 3 through 7 have issues and a definition of done: no layer closes without its LAYER-LOG entry written first.
