---
layout: post
title: "The Intent That Vanished"
date: 2026-10-31
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulation, calibration, playbooks, debugging]
series: issue-403-playbook-smoke-suite
---

# The Intent That Vanished

The upgrade-research playbook kept failing. 310 ticks of simulation, 413 vespene in the bank, a Forge sitting idle — and ProtossGroundWeaponsLevel1 never started researching. The playbook fired the research step at tick 122. The handler called `game.applyIntent(new ResearchIntent(...))`. The intent entered the game. And then nothing happened.

The problem was in how EmulatedGame handles research that can't start immediately. If the Forge isn't finished building or resources are short at the exact tick the intent arrives, `handleResearch()` calls `deferResearchIntent()` — adding it to an internal retry list. The game engine's main loop (`EmulatedEngine`) calls `processDeferredResearchIntents()` every tick to drain that list. But the playbook runner isn't the game engine. It has its own tick loop, and it never called the drain method.

![Deferred research intent lifecycle](images/deferred-intent-lifecycle.svg)

The research intent entered the deferred list and stayed there forever. The handler returned success (the intent was accepted), so the playbook marked the step as fired and never retried it. From the runner's perspective, the research was happening. From the game's perspective, it was sitting in a list nobody read.

The fix is one line in each loop:

```java
for (int tick = 0; tick < durationTicks; tick++) {
    game.tick();
    game.processDeferredResearchIntents();  // ← this was missing
    // ... fire playbook steps ...
}
```

The workaround we'd been using — `loop: continuous` on the research step, which re-fires the intent every tick until it sticks — masked the bug. It worked because each redundant intent was a fresh attempt at `handleResearch()`, and eventually one landed when the Forge was complete and resources were available. But it also meant hundreds of duplicate intents were being created and silently rejected per playbook run.

The real lesson isn't about the missing method call. It's about the gap between "the intent was accepted" and "the intent was processed." The delivery handler returned success because the intent was valid and queued. The game deferred it because preconditions weren't met. Neither component lied — they just had different definitions of "done." The playbook runner assumed acceptance meant execution. The game engine knew better, but nobody asked.

This is the same class of bug that shows up in any system with async processing and fire-and-forget delivery: message queues where the consumer isn't running, event buses where the listener was never registered, task schedulers where the executor pool is shut down. The symptom is always the same — the sender thinks it worked, the work never happens, and the gap is invisible until you go looking for the result.
