# Recalibrate Spatial Features Against Real Movement Data

**Issue:** #300
**Branch:** issue-300-recalibrate-spatial
**Date:** 2026-09-11
**Depends on:** #298 (UnitPositions tracker events — enemy units now move during replay)

## Problem

With #298 landed, enemy unit positions update during replay playback for the first time. Five downstream spatial features were previously operating against static birth positions — they produced results, but those results were derived from frozen data. Now that positions are correct, the spatial intelligence needs recalibration: verify accuracy improves, fix structural bugs exposed by real movement, and adjust thresholds where the data warrants.

Two structural issues surfaced during design:

1. **Posture stickiness:** `ENEMY_POSTURE` reverts to UNKNOWN after the 3-minute unit buffer eviction window, even though the build-order classification (ALL_IN or MACRO) is already established. This happens because `seenUnitTags` is permanent — once a unit tag is added, re-sighting it after eviction doesn't re-enter `unitBuffer`. The conclusion is retracted when its evidence expires.

2. **Dead event:** `EnemyPostureClassifiedEvent` is fired by `DroolsScoutingTask` but has zero `@Observes` consumers. The `StrategyTrustObserver` mentioned in ARC42STORIES.MD was never implemented — strategy routing uses `ScoutingIntelBroker` + `TacticalPosture` instead.

## Solution

### 1. Posture Persistence (D1)

Cache the last non-UNKNOWN posture in `DroolsScoutingTask`. When the DRL produces no posture decisions (empty buffer after eviction), use the cached value instead of falling back to UNKNOWN.

**In `DroolsScoutingTask`:**

Add a field:
```java
private String cachedPosture = "UNKNOWN";
```

Replace the current posture resolution (line ~249):
```java
// Before (wrong — evidence eviction retracts conclusion):
String posture = data != null && !data.getPostureDecisions().isEmpty()
    ? data.getPostureDecisions().get(0) : "UNKNOWN";

// After (conclusion persists until contradicted):
if (data != null && !data.getPostureDecisions().isEmpty()) {
    cachedPosture = data.getPostureDecisions().get(0);
}
String posture = cachedPosture;
```

Reset the cache on game restart — add to the existing reset path (wherever `prevPosture`, `prevBuildOrder`, etc. are reset):
```java
cachedPosture = "UNKNOWN";
```

**Why this is correct:** ENEMY_POSTURE is a build-order classification. ALL_IN/MACRO is about whether the enemy invested in army or economy — an early-game strategic choice. Once classified, the conclusion should persist until contradicted by new evidence. The ALL_IN → MACRO transition still works because expansion events are permanent (never evicted), so the DRL "Expansion: Macro" rule fires whenever expansions exist.

**What doesn't change:** `ScoutingSessionManager`, `unitBuffer`, `seenUnitTags`, and the DRL rules remain unchanged. The fix is entirely in how `DroolsScoutingTask` interprets an empty `postureDecisions` list. Pattern classification is unaffected.

### 2. Remove Dead Event (D3)

Delete `EnemyPostureClassifiedEvent.java` and remove its usage from `DroolsScoutingTask`:
- Remove `@Inject Event<EnemyPostureClassifiedEvent> postureClassified` field
- Remove the `postureClassified.fire(...)` call
- Remove the import

Update ARC42STORIES.MD §9 to remove the reference to `StrategyTrustObserver` consuming this event.

### 3. TacticalPosture Cascade Verification (D2)

The architecture separates economic posture (scouting: ALL_IN/MACRO) from military posture (summarisation: EARLY_AGGRESSION/DEFENSIVE_HOLD/MID_SKIRMISH/EARLY_MACRO). With #298's movement data, the cascade should automatically improve:

```
DroolsScoutingTask → ENEMY_POSTURE (fixed), TIMING_ATTACK_INCOMING (uses live positions)
  → MomentDetectionTask → GameMoment events (TIMING_PUSH, POSTURE_SHIFT)
    → GamePhaseSummariser → TacticalPosture
      → Strategy DRL, Dominance DRL, CBR, Commentary
```

No structural changes to the cascade. The calibration test (below) verifies whether TacticalPosture quality improves with real movement data.

### 4. SpatialCalibrationTest (D4)

A new `@Tag("benchmark")` test that replays AI Arena and IEM10 datasets, measuring three spatial metrics per replay:

**Package:** `io.quarkmind.plugin.scouting`
**Profile:** benchmark (excluded from default surefire run)

**Metrics collected per replay:**

| Metric | What it measures | Expected outcome |
|--------|-----------------|-----------------|
| **Posture UNKNOWN rate** | Percentage of frames where ENEMY_POSTURE is UNKNOWN after the first classification | Near 0% with D1 fix (was high without it) |
| **Timing attack fire rate** | Number of army-near-base events that fire during the replay | Non-zero for replays where the enemy attacks |
| **TacticalPosture transitions** | Count and sequence of TacticalPosture phase changes | More transitions than with static data (movement creates more moments) |

**Test structure:**

```java
@Tag("benchmark")
class SpatialCalibrationTest {

    static List<ReplayData> loadReplays() { /* AI Arena + IEM10 */ }

    @ParameterizedTest
    @MethodSource("loadReplays")
    void spatialMetrics(ReplayData replay) {
        // Drive replay through SimulatedGame
        // Collect ENEMY_POSTURE per tick after first classification
        // Count army-near-base events
        // Track TacticalPosture transitions
        // Print per-replay report
    }
}
```

Run with: `mvn test -pl quarkmind-sc2 -Pbenchmark -Dtest=SpatialCalibrationTest`

**Output:** A printed report per replay with the three metrics, plus aggregate statistics by matchup (PvT/PvZ/PvP). No hard pass/fail assertions initially — this is a measurement test. Hard thresholds can be added once baseline data is established.

### 5. Threshold Tuning

The following thresholds may need adjustment based on calibration test results:

| Threshold | Current | Location | What it controls |
|-----------|---------|----------|-----------------|
| `NEAR_BASE_DISTANCE` | 30f | `ScoutingSessionManager` | Distance from Nexus that counts as "near base" |
| `MIN_ARMY_NEAR_BASE` | 3 | `ScoutingSessionManager` | Minimum units to trigger army-near-base event |
| `ARMY_WINDOW_MS` | 10,000 | `ScoutingSessionManager` | Army-near-base event sliding window |
| `EXPANSION_DISTANCE_THRESHOLD` | 50f | `ScoutingSessionManager` | Distance from enemy base for expansion detection |

**Methodology (per protocol `sc2data-spatial-constants-require-calibration.md`):** Measure from replay data across multiple maps. Adjust only if the calibration test shows the current value produces incorrect results. Document the calibration source and sample size.

**What NOT to tune:** `UNIT_WINDOW_MS` (3 minutes) — this is the build-order classification window, not a spatial threshold. Changing it affects pattern classification accuracy.

### 6. Existing Calibration Suite

Run the full existing calibration suite before and after changes to verify no regressions:

```bash
mvn test -pl quarkmind-sc2 -Pbenchmark
```

Tests: `ScoutingCalibrationTest`, `MapControlCalibrationTest`, `PatternClassificationCalibrationTest`, `ExpansionLocationCalibrationTest`.

## Changes Required

| File | Change |
|------|--------|
| `DroolsScoutingTask.java` | Add `cachedPosture` field, replace UNKNOWN fallback with cache lookup, reset on game start |
| `EnemyPostureClassifiedEvent.java` | **Delete** |
| `SpatialCalibrationTest.java` | **New** — replay-driven spatial metrics test |
| `ARC42STORIES.MD` | Update §9 to remove StrategyTrustObserver reference |
| `ScoutingSessionManager.java` | Threshold adjustments if calibration data warrants (NEAR_BASE_DISTANCE, MIN_ARMY_NEAR_BASE, ARMY_WINDOW_MS) |

## Testing Strategy

### Unit tests (plain JUnit)

**DroolsScoutingTaskTest additions:**
- Posture persists as ALL_IN after unit buffer is evicted (empty buffer → cached ALL_IN, not UNKNOWN)
- Posture transitions ALL_IN → MACRO when expansion spotted after buffer eviction
- Posture resets to UNKNOWN on game restart
- Posture stays UNKNOWN when no enemies have ever been classified

**Existing test updates:**
- Tests that assert `EnemyPostureClassifiedEvent` firing → remove
- Tests that depend on posture being UNKNOWN after eviction → update expected value

### Integration tests (@QuarkusTest)

**DroolsScoutingTaskIT:**
- Verify `postureUnknownWhenNoEnemiesEverSeen` still passes (UNKNOWN is correct pre-classification)
- Add test: posture persists after eviction window

### Calibration tests (@Tag("benchmark"))

**SpatialCalibrationTest** (new):
- Per-replay spatial metrics (posture UNKNOWN rate, timing fire rate, TacticalPosture transitions)
- Aggregate stats by matchup

**Existing suite** (regression check):
- `ScoutingCalibrationTest` — verify still passes
- `MapControlCalibrationTest` — verify still passes
- `PatternClassificationCalibrationTest` — verify still passes

## Data Flow (after changes)

```
Game tick
  │
  ▼
ScoutingSessionManager.processFrame()
  │ army-near-base events (live positions ← #298 fix)
  │ unit buffer (first-seen, 3-min window)
  │ expansion events (permanent)
  ▼
DroolsScoutingTask.drl
  │ "Expansion: All-In" or "Expansion: Macro" rules
  ▼
DroolsScoutingTask.execute()
  │ postureDecisions non-empty → update cachedPosture
  │ postureDecisions empty    → use cachedPosture (D1 fix)
  │ write ENEMY_POSTURE, TIMING_ATTACK_INCOMING
  ▼
MomentDetectionTask
  │ PostureUpdate, TimingAlert → GameMoment events
  ▼
GamePhaseSummariser
  │ Batches moments → TacticalPosture (EARLY_AGGRESSION, etc.)
  ▼
Strategy DRL, Dominance DRL, CBR, Commentary, Advisory
```

## Known Limitations

1. **MACRO → ALL_IN pivot undetectable.** Expansion events are permanent — once MACRO is classified, it cannot revert to ALL_IN even if the enemy sacrifices their expansion. Pre-existing limitation, unaffected by this change.

2. **seenUnitTags stickiness persists for build-order classification.** Re-sighted units after eviction still don't re-enter `unitBuffer`. This is correct for build-order detection (prevents timestamp corruption) but means the unit buffer empties after 3 minutes. The posture cache (D1) handles the downstream consequence.

3. **TacticalPosture improvement is indirect.** The cascade depends on MomentDetectionTask receiving better ENEMY_POSTURE and TIMING_ATTACK_INCOMING inputs. If GamePhaseSummariser doesn't produce better TacticalPosture despite better inputs, a follow-up on the summariser's spatial sensitivity may be needed.

## References

- `DroolsScoutingTask.java:249-251` — posture resolution code (fix target)
- `ScoutingSessionManager.java:50-105` — buffer and threshold definitions
- `EnemyPostureClassifiedEvent.java` — dead event (removal target)
- `TacticalPosture.java` — military posture record (cascade target)
- `GamePhaseSummariser.java` — moment → TacticalPosture classification
- `DominanceWeightAdjustment.drl` — TacticalPosture consumer (dominance weights)
- `StarCraftStrategy.drl` — TacticalPosture consumer (strategy rules)
- Protocol `sc2data-spatial-constants-require-calibration.md` — calibration methodology
- Protocol `strategy-attack-under-unknown-posture.md` — posture semantics under UNKNOWN
- Issue #298 — UnitPositions tracker events (root cause fix)
- Issue #299 — Replay smoke tests (regression guard)
