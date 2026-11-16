---
layout: post
title: "The Runner That Couldn't Save"
date: 2026-10-31
entry_type: note
subtype: diary
projects: [casehubio/quarkmind]
tags: [emulation, calibration, playbooks, design]
---

# The Runner That Couldn't Save

A StarCraft II expansion costs 400 minerals. A Probe costs 50. With continuous Probe production eating 50 minerals every training cycle, minerals never accumulate past ~50. The expansion build retries every tick, checks `minerals < 400`, fails, and waits. The Probes keep training. The minerals stay at ~50. The expansion never fires.

This is not a bug in the economy simulation. EmulatedGame's mining rates are correct. The Probe training cost is correct. The Nexus mineral cost is correct. The problem is that the playbook runner has no concept of saving — it fires every step that can fire, every tick, in declaration order. There is no mechanism for "stop spending on cheap things so you can afford the expensive thing."

Real SC2 players do this instinctively. You cut Probe production at 18 supply, bank minerals for the expansion, then resume training after the Nexus goes down. The playbook DSL had no equivalent.

## Three Ways to Solve It

I considered three approaches:

**Pause/resume** — explicit `pause: PROBE at: 18 supply` / `resume: PROBE at: 22 supply` steps in the YAML. Fully visible, mirrors human macro. But fragile: if supply triggers are wrong, training never resumes or resumes too early. Every expensive build needs its own pause/resume bookends. The playbook author carries the cognitive load.

**Priority ordering** — higher-priority steps fire first within each tick. Sounds clean, but solves nothing. The Nexus doesn't fail because the wrong step fires first within a tick — it fails because minerals never *accumulate* across ticks. Reordering within a single tick doesn't help when the problem is across hundreds of ticks.

**Mineral reservation** — the runner detects a pending expensive build and suppresses continuous spending until minerals accumulate. No YAML changes. The runner becomes smarter about allocation.

I went with reservation. It's implicit, which means harder to debug if something surprising happens. But it has a property the other two don't: existing playbooks get better without changing a line of YAML.

## The Threshold Problem

The first version reserved for any pending build. This broke Zerg immediately — the Hatchery costs 300 minerals, and the runner was suppressing Drone production to save for it. But a Hatchery at 300 minerals accumulates naturally during supply blocks (periods where supply is full and no units can train). The reservation was counterproductive: fewer Drones mining meant slower income, meaning it took *longer* to reach 300.

The insight: cheap builds don't need help. Supply blocks — the natural pauses when you're at 23/23 supply waiting for the next Pylon — create savings windows where minerals accumulate. A 100-mineral Pylon or 150-mineral Gateway fires during these windows without intervention. Only truly expensive builds (400-mineral Nexus, 400-mineral Command Center) need the runner to actively suppress spending.

![Mineral reservation mechanism](images/mineral-reservation-mechanism.svg)

The threshold landed at 350 minerals. Below that, the natural supply block mechanism handles it. Above it, the runner suppresses both continuous training and deferred supply-triggered builds until the expensive build fires.

## What It Looks Like in Practice

The `two-base-production` playbook was the test case. Before reservation, continuous Probe and Zealot training drained all minerals — the Nexus never fired even after 310 ticks (5 minutes of game time). After reservation:

```
tick 122: minerals=316  supply=23/31  buildings=4  ← saving, training suppressed
tick 183: minerals=369  supply=30/31  buildings=6  ← still saving
tick 200: minerals=400  supply=32/39  buildings=7  ← Nexus fires, training resumes
tick 244: buildings=8   NEXUS=2       GATEWAY=2    ← expansion complete
```

The reservation window is visible in the mineral curve: training stops, minerals climb linearly, the Nexus fires when minerals hit 400, then training resumes and minerals drop back to the normal ~50 equilibrium.

## The General Pattern

This is the same resource contention pattern that shows up in any system where cheap recurring operations starve expensive infrequent ones. A CI pipeline where fast linting jobs consume all the runner capacity, preventing slow integration tests from ever scheduling. A memory allocator where small allocations fragment the heap, preventing large contiguous allocations. A network queue where high-frequency small packets starve large-frame transmissions.

The solution shape is always the same: detect when an expensive operation is pending and temporarily suppress the cheap recurring ones until enough resource accumulates. The threshold — how expensive is "expensive enough" to warrant suppression — is the design decision that determines whether the mechanism helps or hurts.
