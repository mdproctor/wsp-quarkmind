---
layout: post
title: "Kiting and the planner's perfect mistakes"
date: 2026-04-17
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
---

E9 gave the agent eyes. The enemy staging area went dark behind the fog,
and the scout finally had something to actually discover. But when the
Zealots came through the ramp, the Stalkers still stood exactly where they
were, took everything, and hoped the damage tables were in their favour.

E10 fixes that. Three things, in order of how obvious they were.

**The first was a bug.** `DroolsTacticsTask` had a hardcoded constant:
`STALKER_RANGE = 6.0`. The real Stalker attack range is 5.0 tiles. It's
been wrong since E-series began — slightly too generous, which meant units
were counted as "in range" when they weren't, and the error compounds in a
mixed-type army. One line, `SC2Data.attackRange(unit.type())`, per unit
instead.

**The second was focus-fire.** Before this, each Stalker attacked
independently — whatever enemy was nearest to it. With three Stalkers
against four Zealots, damage spreads and nothing dies. Focus-fire means
the whole group targets whichever enemy has the lowest combined health
and shields:

```java
static Optional<Unit> selectFocusTarget(List<Unit> enemies) {
    return enemies.stream()
        .min(Comparator.comparingInt(e -> e.health() + e.shields()));
}
```

First enemy dies faster. Fewer units attacking back.

**The third was kiting.** Stalkers have a 5-tile ranged attack and a
3-tick weapon cooldown. Zealots are pure melee. The opportunity is obvious:
fire, back up during the cooldown, fire again. The Zealot never catches up.

This required getting weapon cooldown into the snapshot. In real SC2,
`weapon_cooldown` is a first-class field in the observation protobuf —
every serious bot framework reads it to drive kiting decisions. I brought
Claude in for the implementation. We added `weaponCooldownTicks` to the
`Unit` record and stamped it from EmulatedGame's internal cooldown map at
snapshot time. Drools uses it to classify a new group: healthy units that
are in attack range but on cooldown. GOAP dispatches a `MoveIntent`
stepping one tile away from the nearest enemy. Next tick, off cooldown and
still in range — `AttackIntent`. Back and forth.

Two GOAP bugs surfaced.

The first: the kiting group was assigned goal `"unitSafe"`. The only action
that sets `unitSafe` to true is RETREAT, which requires `lowHealth: true`.
Kiting units are healthy. The planner searched, found nothing, returned an
empty list. The dispatch loop checks `if (!plan.isEmpty())` before acting,
so the units did nothing at all. No exception, no warning. Claude flagged it
during review: the goal should be `"enemyEliminated"`, reachable as
`KITE → ATTACK`.

The second was subtler. Without `"onCooldown", false` in ATTACK's
preconditions, the planner could apply ATTACK directly to the kiting
WorldState — because `WorldState.get()` returns false for absent keys, and
ATTACK didn't declare `onCooldown` at all. ATTACK costs 2; KITE followed
by ATTACK costs 3. The planner correctly chose the cheaper path. Claude
caught it writing the integration test: the test asserted a `MoveIntent`
for an on-cooldown Stalker and came back with an `AttackIntent`. The fix
is one entry in the precondition map:

```java
"ATTACK", new GoapAction("ATTACK",
    Map.of("inRange", true, "enemyVisible", true, "onCooldown", false),
    Map.of("enemyEliminated", true), 2)
```

`WorldState.get()` returning false for absent keys, combined with
applicability depending only on declared preconditions — a missing entry
makes an action over-applicable. Once you see it, obvious. Getting there
required the test.

430 tests. The Stalkers now back away when they fire and converge on the
weakest target.
