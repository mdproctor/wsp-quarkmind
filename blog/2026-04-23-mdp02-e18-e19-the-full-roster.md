---
layout: post
title: "E18–E19: The Full Roster"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [quarkmind]
tags: [visualizer, three-js, sprites, quarkus, testing, showcase]
---

When we added Zerg sprites and looked at the replay parser, there was a gap nobody had noticed: SCV, Drone, Overlord, Baneling, Phoenix, Oracle, Tempest, Mothership, Warp Prism, Locust, Broodling, Infested Terran, Changeling, Auto Turret — all falling through to UNKNOWN. Every replay had been silently discarding units for months.

The audit was simple in hindsight: read the replay parser switch statement and compare it against the SC2 multiplayer unit list. What's in the parser is what you handle; everything else becomes a grey blob. I'd been focused on units the AI needed to reason about and had never asked the more basic question: what appears in a real match? Workers appear in every match. Overlords are visible from second one in Zerg games. Banelings might be the most decisive unit in ZvT. All of them were UNKNOWN.

Some sprites were straightforward. The Hellbat is the Hellion in bipedal mech form — a different silhouette entirely from the wheeled vehicle, so it earns its own draw function. Viking Assault is the Viking landed on two mechanical legs, wings folded. Neither required any design thought beyond accurately describing the model.

Others required actual decisions. The Archon has no solid body — it's pure psionic energy. The draw function has no filled shapes: two overlapping thick ring strokes in team colour, arc discharges radiating outward, a bright core at the centre. Team colour is the entire sprite. The Changeling is a shapeshifter that appears disguised in replays, so there's no "correct" form to draw. I went with an amorphous blob with pseudopods — like it's mid-transformation, still deciding what to become. That felt honest to what it is.

The Infested Terran was satisfying: same humanoid silhouette as the Marine, but dark degraded armour, Zerg tendrils from the torso, a Zerg eye glowing through the cracked visor in team colour. Recognisably humanoid, obviously wrong.

One failure mode worth recording: after adding twenty-four new unit types, `mvn quarkus:dev` started returning HTML error pages instead of JSON. The error was `ClassTooLargeException: Class too large: io/quarkus/runner/recorded/LogStreamProcessor$handler586886279`. The application had accumulated enough enum values and switch cases that a Quarkus-generated startup handler exceeded the JVM's 64KB constant pool limit. `mvn compile` was clean. Tests passed. Only the dev server broke. The fix is `mvn clean` before `mvn quarkus:dev` — it removes the stale augmentation cache and regenerates. It's the kind of failure that costs time because nothing you own is broken.

The showcase needed rethinking with sixty-five units. Units placed outside the Nexus sight radius are fogged in the browser, but `window.__test.enemyCount()` counts scene graph objects — not visually revealed ones. The Playwright test kept passing while the browser showed nothing. The solution: scatter friendly units as fog observers. `spawnFriendlyUnitForTesting` existed on `EmulatedGame` but not `SimulatedGame`, so we added it. Four Probes at strategic map positions cover the full grid. The lesson is that counting objects in a scene is not the same as asserting they're visible — a useful distinction if you're ever testing a renderer.

The visualiser now covers everything that appears in a multiplayer match. The long tail was longer than expected.
