---
layout: post
title: "E12: Stalker Blink"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [quarkus, drools, tactics, goap, sealed-interface]
---

E12 turned out to be exactly the session I'd predicted: thorough, mechanical in parts, with a couple of genuinely non-obvious moments.

The first thing was the Unit record. Blink needs a `blinkCooldownTicks` field — a 9th argument to a constructor called in 70 places across 24 files. I had Claude work through them, appending `, 0` to every call site. Straightforward, except for one thing: adding `BlinkIntent` to the `Intent` sealed interface immediately broke the `applyIntent()` switch in `EmulatedGame` and the dispatch switch in `ActionTranslator`. Java's sealed interface exhaustiveness is a compile-time constraint across the whole codebase — adding a new permitted type breaks every existing switch over that interface, regardless of which file did the adding. The plan had split "create sealed type" and "wire it in" into separate steps. We patched it with placeholder cases in the same commit as the type, then wired the real implementation in the follow-up.

The blink mechanics themselves went cleanly. `EmulatedGame` already tracks weapon cooldowns via `unitCooldowns: Map<String, Integer>` — we added `blinkCooldowns` alongside it, with the same decrement-per-tick and cleanup-on-death pattern. The interesting piece is `blinkRetreatTarget()`: compute the direction away from the nearest enemy, try the direct 8-tile jump, then sweep ±45° increments if the destination is a wall tile. Same angular sweep `TerrainAwareKiteStrategy` uses for the 1-tile kite step, just scaled up.

`snapshot()` stamps the cooldown from the map — the stored Unit records always carry 0 for both weapon and blink cooldowns; the maps are the authority.

On the Drools side, the BLINKING group sits at salience 205 — above kiting (200) but below low-health retreat (210). The classification felt obvious in the design. The implementation revealed something less obvious: Drools salience controls firing *order*, not mutual exclusion. When a Stalker had low shields and was simultaneously on weapon cooldown and in range, both the blinking rule (205) and the kiting rule (200) fired. The unit received both a `BlinkIntent` and a `MoveIntent` in the same tick; whichever `applyIntent` handled last won.

Claude's final review caught it. The fix is a `not` guard on the kiting rule:

```drl
rule "Group: kiting"
    salience 200
when
    ...
    not /shieldsLowTags[ this == $u.tag() ]
then
```

Guard on `shieldsLowTags`, not on `blinkReadyTags` — you want to block kiting specifically when blink is the right response, not every time blink happens to be off cooldown.

The E2E survival test ran into a scenario problem. The obvious setup — Stalker at full health vs one Zealot — never triggered blink. The Stalker has a 5-tile range advantage and fires every tick; the Zealot closes to 0.5 tiles but takes damage the whole way. The shields never dropped below 25% before the Zealot died. We pre-set the Stalker's shields to 25 using `setShieldsForTesting` — one hit drops them to 18, blink fires, the unit retreats 8 tiles with 58 shields restored, and the control run ends noticeably worse.

446 tests. Issue #82 closed.

I'd flagged in E11 that GOAP planner interactions with two new WorldState keys (`shieldsLow`, `blinkReady`) might produce silent empty plans. They didn't. The planner routes correctly: BLINK → MOVE_TO_ENGAGE → ATTACK.
