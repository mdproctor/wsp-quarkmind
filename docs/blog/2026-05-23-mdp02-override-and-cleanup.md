---
layout: post
title: "Override and Cleanup"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [quarkmind]
---

Three code review nits and a follow-up feature. The nits are barely worth a paragraph.

We dropped `Math.sqrt` from the probe-to-nexus comparison — the assignment only needs relative ordering, and squared distance gives the same result. We cloned the varargs array in `setMiningProbesPerBase`, which was storing the reference directly — harmless today, wrong at the method boundary. And we added direct test coverage for the zero-nexus case in `countProbesPerBase`, making it package-private to allow it.

## The override problem

The previous entry landed `countProbesPerBase` on `ReplayValidationHarness` — ground-truth probe assignment for the replay harness. The next question: what about the emulated game running standalone, after the AI builds a second Nexus?

When the AI builds a second Nexus, `miningProbesPerBase` stays at `{12}`. The field doesn't update; nothing tells it to. Making `tick()` auto-compute from buildings and units is the obvious fix.

The complication is the replay harness. It injects ground-truth probe counts before each tick via `setMiningProbesPerBase`. If `tick()` auto-computes, it clobbers what the harness just set.

Two bad alternatives: a persistent flag callers are supposed to clear (easy to forget), or a separate harness-specific setter (adds a call site that doesn't need to exist). The clean solution: a one-shot override. `setMiningProbesPerBase` sets the array and a boolean. `tick()` checks it — if set, uses the stored value and clears it; if not, auto-computes. The harness calls `setMiningProbesPerBase` before every tick, so the flag is always set when the harness is in control. Standalone mode never calls it.

```java
if (!miningProbesOverridden) {
    miningProbesPerBase = countProbesPerBase(friendly.buildings, friendly.units);
}
miningProbesOverridden = false;
```

## The algorithm moves

With `tick()` and the harness both needing `countProbesPerBase`, we moved it to `EmulatedGame` as a public static method. The harness delegates to it. One copy, correct home.

## The test that didn't read the new contract

`zeroProbesYieldsNoMineralGain` set the override to zero probes, then ticked 100 times. First tick consumes the override. Ticks 2–100 auto-compute from 12 probes on the initial Nexus — and earn income. The assertion failed.

The test was written when the override persisted indefinitely. It no longer does. One call, one tick, assert.
