# Train Timing Precision — Sub-tick CmdEvent Offset Fix

**Issue:** #142  
**Date:** 2026-05-19  
**Branch:** `issue-142-tick-rounding`

---

## Problem

`ReplayValidationHarness` shows `firstUnitDivergenceTick = 36` and a growing unit
delta (up to 15 over a full replay) regardless of mining model. The root cause is
train completion timing, not resource accumulation.

`EmulatedGame.startTraining` computes:

```java
long completesAt = gameFrame + SC2Data.trainTimeInTicks(unitType);
```

`trainTimeInTicks` uses integer outer-tick resolution (1 tick = 22 loops). A Probe's
real SC2 train time is 268.8 loops (12 seconds × 22.4 loops/sec) = 12.218 outer ticks.
When a `CmdEvent` arrives at loop offset ≥ 18 within a 22-loop tick window, the real
unit completes 1 outer tick later than EmulatedGame predicts — producing a systematic
1-tick-early completion that accumulates across the replay.

The harness compounds this by stripping the loop from `TimedIntent` before passing
the intent to `EmulatedGame`, so sub-tick information is lost at the call boundary.

---

## Design

### SC2Data — domain constants and derived timing

Three additions, one change:

**New constants:**
```java
public static final int    LOOPS_PER_TICK         = 22;   // moved from Sc2ReplayShared
public static final double GAME_LOOPS_PER_SECOND  = 22.4; // Faster speed
```

**New method:**
```java
public static double trainTimeInLoops(UnitType type) {
    return switch (type) {
        case PROBE    -> 12 * GAME_LOOPS_PER_SECOND;  // 268.8
        case ZEALOT   -> 28 * GAME_LOOPS_PER_SECOND;  // 627.2
        case STALKER  -> 31 * GAME_LOOPS_PER_SECOND;  // 694.4
        case IMMORTAL -> 40 * GAME_LOOPS_PER_SECOND;  // 896.0
        case OBSERVER -> 22 * GAME_LOOPS_PER_SECOND;  // 492.8
        default       -> 30 * GAME_LOOPS_PER_SECOND;  // 672.0
    };
}
```

**`trainTimeInTicks` derived** (eliminates future drift between the two methods):
```java
public static int trainTimeInTicks(UnitType type) {
    return (int)(trainTimeInLoops(type) / LOOPS_PER_TICK);
}
```

All five Protoss values are unchanged by this derivation. `Sc2ReplayShared` and
`ReplayValidationHarness` reference `SC2Data.LOOPS_PER_TICK` instead of their
own copies.

---

### TimedIntent — move to `sc2/intent/`

`TimedIntent` moves from `sc2.replay` to `sc2.intent`, alongside `TrainIntent`,
`BuildIntent`, `MoveIntent`, `AttackIntent`, `BlinkIntent`.

Declaration is unchanged:
```java
public record TimedIntent(long loop, Intent intent) {}
```

Rationale: `TimedIntent` is a general SC2 concept — an intent paired with the game
loop at which it was issued. SC2's API exposes game loops in live play as well as
replay. There is no replay-specific knowledge encoded in the type itself. Placing it
in `sc2.intent` removes the package coupling that would otherwise arise from
`EmulatedGame` accepting it.

Import sites in `sc2.replay` (`AbilityMapping`, `ReplayCommandStream`,
`ReplayValidationHarness`) update their import. No other callers exist.

---

### EmulatedGame — loop-aware training

New public overload:
```java
public void applyIntent(TimedIntent ti) {
    switch (ti.intent()) {
        case TrainIntent  t -> handleTrain(t, friendly, ti.loop());
        case MoveIntent   m -> setTarget(m.unitTag(), m.targetLocation(), false, friendly);
        case AttackIntent a -> setTarget(a.unitTag(), a.targetLocation(), true,  friendly);
        case BuildIntent  b -> handleBuild(b, friendly);
        case BlinkIntent  b -> executeBlink(b.unitTag(), friendly);
    }
}
```

`startTraining` gains an `absLoop` parameter and computes the correct completion tick:
```java
private void startTraining(String buildingTag, UnitType unitType, PlayerState state, long absLoop) {
    int loopOffset = (int)(absLoop % SC2Data.LOOPS_PER_TICK);
    long completesAt = gameFrame
        + (int)((loopOffset + SC2Data.trainTimeInLoops(unitType)) / SC2Data.LOOPS_PER_TICK);
    // ... rest unchanged
}
```

The existing `applyIntent(Intent intent)` remains unchanged and passes `absLoop = 0`
internally — offset 0 produces the same result as the current integer arithmetic,
so `SimulatedGame` and all agent/mock callers are unaffected.

---

### ReplayValidationHarness — pass TimedIntent directly

```java
// Before
emulated.applyIntent(intents.get(cursor).intent());

// After
emulated.applyIntent(intents.get(cursor));
```

The local `LOOPS_PER_TICK` constant is removed; `SC2Data.LOOPS_PER_TICK` is used
throughout.

---

## Acceptance Criteria

`ReplayValidationTest` tightened from the current `maxUnitDelta ≤ 2` bound to:

- `firstUnitDivergenceTick == -1` — no unit count divergence across the 3-minute window
- `maxUnitDelta == 0`

---

## Tests

**`SC2DataTest`**
- `trainTimeInLoopsDefinedForProtossUnits` — Probe=268.8, Zealot=627.2, Stalker=694.4, Immortal=896.0, Observer=492.8
- `trainTimeInLoopsConsistentWithTicks` — for each Protoss type, `(int)(trainTimeInLoops / LOOPS_PER_TICK) == trainTimeInTicks`

**`EmulatedGameTest`**
- `probeCompletesOnTimeWithZeroLoopOffset` — `applyIntent(TimedIntent)` at offset 0; unit appears after exactly 12 ticks
- `probeCompletesOneLaterWithLateLoopOffset` — offset 18 (≥ rollover threshold); unit appears after 13 ticks

**`ReplayValidationTest`** — assertions updated to `firstUnitDivergenceTick == -1` and `maxUnitDelta == 0`

---

## Files Touched

| File | Change |
|------|--------|
| `domain/SC2Data.java` | Add `LOOPS_PER_TICK`, `GAME_LOOPS_PER_SECOND`, `trainTimeInLoops`; derive `trainTimeInTicks` |
| `sc2/replay/TimedIntent.java` | Delete (moved) |
| `sc2/intent/TimedIntent.java` | Create (moved from `sc2.replay`) |
| `sc2/emulated/EmulatedGame.java` | Add `applyIntent(TimedIntent)`; update `startTraining` with `absLoop` |
| `sc2/replay/ReplayValidationHarness.java` | Pass `TimedIntent` directly; use `SC2Data.LOOPS_PER_TICK` |
| `sc2/mock/Sc2ReplayShared.java` | Reference `SC2Data.LOOPS_PER_TICK` |
| `sc2/replay/AbilityMapping.java` | Update import |
| `sc2/replay/ReplayCommandStream.java` | Update import |
| `domain/SC2DataTest.java` | Add 2 tests |
| `sc2/emulated/EmulatedGameTest.java` | Add 2 tests |
| `sc2/replay/ReplayValidationTest.java` | Tighten assertions |
