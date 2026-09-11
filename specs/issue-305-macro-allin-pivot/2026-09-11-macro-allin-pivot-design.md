# Detect MACRO to ALL_IN Pivot (Expansion Sacrifice)

**Issue:** #305
**Branch:** issue-305-macro-allin-pivot
**Date:** 2026-09-11
**Depends on:** #300 (posture persistence), #304 (ALL_IN calibration)

## Problem

`ScoutingSessionManager` detects enemy expansions by checking whether enemy units are >50 tiles from the estimated enemy base (`EXPANSION_DISTANCE_THRESHOLD`). This heuristic has two problems:

1. **False positives from attacking armies.** An enemy army attacking the player's base is 200+ tiles from the enemy's main — far exceeding the 50-tile threshold. Every attack triggers expansion detection, making ALL_IN posture impossible to detect during combat (#304 finding).

2. **Permanent expansion evidence.** `expansionBuffer` is a `List` with no eviction. Once MACRO is classified, it persists forever — even if the enemy sacrifices their expansion for an all-in pivot. This is documented as Known Limitation #1 in the #300 spec.

## Solution

### Two-tier expansion detection (D1)

Replace the single-source unit-distance heuristic with a two-tier system:

**Tier 1 — Suspected (existing unit heuristic, unchanged):** Enemy unit observed >50 tiles from the estimated enemy base. Early signal — fires before expansion buildings are visible. Permanent (no eviction). This is the existing behavior, preserved for early-game coverage.

**Tier 2 — Confirmed (new building check):** Enemy base-type building (NEXUS, HATCHERY, COMMAND_CENTER, LAIR, HIVE, ORBITAL_COMMAND, PLANETARY_FORTRESS) observed at a non-starting location. Sticky until contradicted (D3) — persists until we observe the location and the building is gone.

### Confirmed overrides suspected (D2)

The DRL posture rules operate in two phases:

**Pre-confirmation phase** (no base building ever observed at a non-starting location): Uses the existing suspected-only logic. Posture = MACRO if suspected expansion exists, ALL_IN if units seen but no suspected expansion. Identical to current behavior.

**Post-confirmation phase** (at least one confirmed expansion has been observed): Confirmed tier takes precedence. Posture = MACRO if any confirmed expansion is alive, ALL_IN if all confirmed expansions are destroyed/absent. The suspected tier is ignored for posture classification.

The transition from pre-confirmation to post-confirmation is one-way — once building data exists, it's the authoritative source.

### Fog of war — sticky until contradicted (D3)

A confirmed expansion persists until we observe the expansion location without the building present. This requires tracking:
- Which locations have confirmed expansion buildings (by building tag)
- Whether those locations are currently visible (building tag present in current observation)
- Whether a previously-confirmed building has disappeared while its location is in our vision

**Retraction condition:** A confirmed expansion is retracted when:
1. We have vision of the area (any friendly unit or building within vision range of the expansion location), AND
2. The expansion building tag is absent from the current `enemyBuildings()` list

**Not retracted when:** The expansion building simply leaves vision (fog). The building may still exist — we just can't see it.

## Changes Required

### ScoutingSessionManager

**New fields:**
```java
private final Map<String, Point2d> confirmedExpansions = new LinkedHashMap<>();
private boolean hasEverConfirmed = false;
```

`confirmedExpansions` maps building tag → position for each confirmed expansion building. `hasEverConfirmed` tracks whether the post-confirmation phase has been entered.

**New method: `processBuildings(List<Building> enemyBuildings, Point2d estimatedEnemyBase)`**

Called each tick after `processFrame()`. Logic:

1. For each enemy building with a base type (`NEXUS`, `HATCHERY`, `COMMAND_CENTER`, `LAIR`, `HIVE`, `ORBITAL_COMMAND`, `PLANETARY_FORTRESS`):
   - If position is NOT within `EXPANSION_DISTANCE_THRESHOLD` of `estimatedEnemyBase` (i.e., it's not the main base):
     - Add to `confirmedExpansions` by tag
     - Set `hasEverConfirmed = true`

2. For each previously confirmed expansion (iterate `confirmedExpansions`):
   - If the building tag is NOT in the current `enemyBuildings` list:
     - Check if we have vision of the expansion location (any friendly unit or building within a vision radius — use `NEAR_BASE_DISTANCE` or a new constant)
     - If we have vision AND the building is absent → retract (remove from `confirmedExpansions`)
     - If we don't have vision → keep (sticky)

**Updated `buildRuleUnit()`:**

Populate `expansionEvents` differently based on tier:
- If `hasEverConfirmed`: populate from `confirmedExpansions` (non-empty = MACRO)
- If `!hasEverConfirmed`: populate from `expansionBuffer` (existing behavior)

**Updated `reset()`:** Clear `confirmedExpansions` and reset `hasEverConfirmed`.

**New testability accessors:**
```java
public int confirmedExpansionCount() { return confirmedExpansions.size(); }
public boolean hasEverConfirmed() { return hasEverConfirmed; }
```

### DroolsScoutingTask

**Updated `execute()`:** After calling `mgr.processFrame(enemies, gameTimeMs, ourNexus, estimatedBase)`, also call:
```java
GameState gameState = ctx.getAs(QuarkMindCaseFile.GAME_STATE, GameState.class);
if (gameState != null) {
    sessionManager.processBuildings(gameState.enemyBuildings(), estimatedBase);
}
```

No changes to the DRL rules — the two-tier logic is handled entirely in `ScoutingSessionManager.buildRuleUnit()` by choosing which data source populates `expansionEvents`.

### SpatialCalibrationTest

**Updated `measureReplay()`:** Add `processBuildings()` call after `processFrame()` using `state.enemyBuildings()`. This enables the confirmed tier in the calibration test.

**New test: `macroRevertsToAllInOnExpansionSacrifice()`:** SimulatedGame scenario:
1. Spawn enemy Nexus at expansion location (confirmed expansion → MACRO)
2. After N ticks, remove the expansion building (sacrifice)
3. Spawn friendly unit near the expansion location (vision → retraction fires)
4. Assert posture transitions: UNKNOWN → MACRO → ALL_IN

**New test: `macroStableWhenExpansionPersists()`:** SimulatedGame scenario:
1. Spawn enemy Nexus at expansion location → MACRO
2. Run 900 ticks with the building present
3. Assert posture stays MACRO (no false retraction)

### Protocol update

Update `strategy-attack-under-unknown-posture.md` to reflect that MACRO → ALL_IN is now possible when a confirmed expansion is destroyed.

## Testing Strategy

### Unit tests (plain JUnit)

**ScoutingSessionManager tests:**
- `processBuildings` adds base-type buildings at non-starting locations to `confirmedExpansions`
- `processBuildings` ignores main-base buildings (within EXPANSION_DISTANCE_THRESHOLD of enemy base)
- `processBuildings` retracts confirmed expansion when building absent + vision available
- `processBuildings` retains confirmed expansion when building absent + no vision (fog)
- `hasEverConfirmed` transitions pre→post confirmation on first building confirmation
- `buildRuleUnit()` uses confirmed tier when `hasEverConfirmed`, suspected tier otherwise

**SpatialCalibrationTest additions:**
- `macroRevertsToAllInOnExpansionSacrifice` — UNKNOWN → MACRO → ALL_IN
- `macroStableWhenExpansionPersists` — MACRO stays MACRO

### Integration tests (@QuarkusTest)

**DroolsScoutingTaskIT additions:**
- Posture transitions MACRO → ALL_IN when confirmed expansion building removed with vision
- Posture stays MACRO when confirmed expansion building removed without vision (fog)

### Calibration tests (@Tag("benchmark"))

Re-run `SpatialCalibrationTest` with the updated `processBuildings()` call. Expected changes:
- Replays with visible enemy expansion buildings: confirmed tier activates, PostTrn unchanged (MACRO from building, not unit heuristic)
- Replays where enemy expansion is destroyed: PostTrn may increase (MACRO → ALL_IN transition)
- No replay should show false retraction (building persists → MACRO stable)

## Data Flow (after changes)

```
Game tick
  │
  ▼
ScoutingSessionManager.processFrame()
  │ unit buffer (first-seen, 3-min window) ← unchanged
  │ army buffer (near-base, 10-sec window) ← unchanged
  │ expansion buffer (unit heuristic, permanent) ← Tier 1: suspected
  │
ScoutingSessionManager.processBuildings()  ← NEW
  │ confirmed expansions (building tag → position)  ← Tier 2: confirmed
  │ retraction check (vision + building absent)
  │
ScoutingSessionManager.buildRuleUnit()
  │ if hasEverConfirmed: expansionEvents ← confirmedExpansions
  │ else:                expansionEvents ← expansionBuffer (existing)
  ▼
DroolsScoutingTask.drl
  │ "Expansion: Macro" / "Expansion: All-In" rules ← unchanged
  ▼
DroolsScoutingTask.execute()
  │ cachedPosture logic ← unchanged
  │ ENEMY_POSTURE written to CaseFile
  ▼
MomentDetectionTask → GamePhaseSummariser → TacticalPosture
  (downstream cascade unaffected)
```

## Vision Check for Retraction

To determine if we have vision of an expansion location, check whether any friendly unit or building is within vision range. A simple distance check:

```java
private boolean hasVisionOf(Point2d location, List<Unit> friendlyUnits, List<Building> friendlyBuildings) {
    float visionRange = 11.0f; // approximate Protoss unit vision range in map units
    for (Unit u : friendlyUnits) {
        if (u.position().distanceTo(location) < visionRange) return true;
    }
    for (Building b : friendlyBuildings) {
        if (b.position().distanceTo(location) < visionRange) return true;
    }
    return false;
}
```

The `visionRange` constant should be calibrated from replay data (per protocol `sc2data-spatial-constants-require-calibration.md`).

## Known Limitations

1. **Vision range is approximate.** Different units have different vision ranges in SC2 (Observer: 11, Stalker: 10, Probe: 8). Using a single constant is an approximation. Acceptable for this purpose — the question is "can we see the expansion location?", not "what is the exact vision boundary?"

2. **Building type detection scope.** Only base-type buildings are checked (Nexus/Hatchery/CC and their upgrades). Proxy structures (forward Pylon, proxy Barracks) are not tracked as expansions. This is correct — proxy aggression is a different signal than economic expansion.

3. **Multi-expansion tracking.** `confirmedExpansions` tracks all confirmed expansion buildings. ALL_IN reversion only fires when ALL confirmed expansions are retracted. An enemy with 3 expansions who loses 1 stays MACRO.

## References

- `ScoutingSessionManager.java` — expansion buffer and unit heuristic (fix target)
- `DroolsScoutingTask.java:249-254` — cachedPosture and posture resolution
- `DroolsScoutingTask.drl` — "Expansion: Macro" and "Expansion: All-In" rules
- `Building.java` — domain record with tag, type, position, health
- `BuildingType.java` — enum with base types (NEXUS, HATCHERY, COMMAND_CENTER, etc.)
- Protocol `strategy-attack-under-unknown-posture.md` — posture semantics (needs update)
- Protocol `sc2data-spatial-constants-require-calibration.md` — calibration methodology for vision range
- Issue #300 — posture persistence (cachedPosture fix)
- Issue #304 — ALL_IN calibration (expansion heuristic false-positive finding)
