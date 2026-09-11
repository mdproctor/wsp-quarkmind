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

**Post-confirmation phase** (at least one confirmed expansion has been observed): Confirmed tier takes precedence. Posture is resolved directly in Java — `MACRO` if any confirmed expansion is alive, `ALL_IN` if all confirmed expansions are retracted. The DRL expansion rules are bypassed for posture classification in this phase because posture is deterministic from the confirmed expansion state alone — the DRL's dependency on `unitEvents` (which empties after 3 minutes due to `seenUnitTags` permanence) makes it unreliable for late-game posture resolution.

The suspected tier continues to feed pattern classification (strategy archetypes), where permanent historical evidence is correct — an enemy who expanded and then sacrificed is not a rusher. Posture and pattern classification intentionally diverge: posture reflects current tactical state (retractable), pattern classification reflects historical strategic evidence (permanent).

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

### ObservationTranslator (prerequisite)

The real SC2 profile (`%sc2`) does not populate `enemyBuildings` in `GameState`. `ObservationTranslator.translate()` maps ALL enemy observations to `Unit` records in `enemyUnits` and passes `List.of()` for `enemyBuildings`. Without this fix, `gameState.enemyBuildings()` is empty in production, making Tier 2 non-functional.

**Fix:** Separate enemy observations the same way self observations are already separated (mirroring lines 113-119):

```java
// Current:
List<Unit> enemies = enemyUnits.stream()
    .map(ObservationTranslator::toUnit)
    .toList();
// ... later: List.of() for enemyBuildings

// Fixed:
List<Unit> enemies = enemyUnits.stream()
    .filter(u -> !isBuilding(toUnitsEnum(u)))
    .map(ObservationTranslator::toUnit)
    .toList();

List<Building> enemyBuildings = enemyUnits.stream()
    .filter(u -> isBuilding(toUnitsEnum(u)))
    .map(ObservationTranslator::toBuilding)
    .toList();
```

Then pass `enemyBuildings` instead of `List.of()` in the `GameState` constructor call.

**Downstream impact:** `ENEMY_UNITS` (populated from `state.enemyUnits()` via `GameStateTranslator`) will no longer include buildings. Consumers affected:
- `DroolsScoutingTask`: `enemies` list used for army size, nearest threat, `processFrame()`. Buildings had `UnitType.UNKNOWN` and didn't match any DRL rules — no functional impact on rule firing. Army size and threat position become more accurate without buildings.
- `MomentDetectionTask`, `BasicScoutingTask`, `DroolsTacticsTask`: all read `ENEMY_UNITS` — removing buildings from the unit list is semantically correct (buildings aren't army units).
- `buildSnapshot` feature extraction: `gs.enemyBuildings()` loop (`DroolsScoutingTask.java:~290`) was previously empty in the `%sc2` profile — now correctly populated. This fixes a pre-existing gap in enemy building feature counts.

This is a clean separation that makes the real SC2 profile consistent with mock/replay profiles.

### ScoutingSessionManager

**New constant:**
```java
public static final float CONFIRMED_EXPANSION_DISTANCE = 25f;
```

Tighter than `EXPANSION_DISTANCE_THRESHOLD` (50 tiles) because once we observe the actual enemy main base building, we have its exact position rather than a heuristic estimate. Flagged for calibration per `sc2data-spatial-constants-require-calibration.md`.

**New fields:**
```java
private final Map<String, Point2d> confirmedExpansions = new LinkedHashMap<>();
private boolean hasEverConfirmed = false;
private Point2d confirmedMainBase = null;
```

`confirmedExpansions` maps building tag → position for each confirmed expansion building. `hasEverConfirmed` tracks whether the post-confirmation phase has been entered. `confirmedMainBase` records the actual enemy main base position once observed — used for precise expansion classification (see `processBuildings` below).

**New method: `processBuildings(List<Building> enemyBuildings, Point2d estimatedEnemyBase, List<Unit> friendlyUnits, List<Building> friendlyBuildings)`**

Called each tick between `evict()` and `buildRuleUnit()` within the `if (needsCep)` block. Logic:

1. Filter enemy buildings to base types only (`NEXUS`, `HATCHERY`, `COMMAND_CENTER`, `LAIR`, `HIVE`, `ORBITAL_COMMAND`, `PLANETARY_FORTRESS`).

1a. **Main base identification** (if `confirmedMainBase` is null): Among all base-type buildings within `EXPANSION_DISTANCE_THRESHOLD` of `estimatedEnemyBase`, select the one **closest** to `estimatedEnemyBase` as `confirmedMainBase`. This makes identification deterministic regardless of list iteration order — when multiple base buildings are visible in the same tick (e.g., main base at (224,224) and natural at (200,200) both within 50 tiles of the estimate), the closest is always the main base.

1b. **Expansion classification** — for each remaining base-type building:
   - If `confirmedMainBase` is not null AND position is beyond `CONFIRMED_EXPANSION_DISTANCE` of `confirmedMainBase`:
     - Add to `confirmedExpansions` by tag, set `hasEverConfirmed = true`
   - If `confirmedMainBase` is null AND position is beyond `EXPANSION_DISTANCE_THRESHOLD` of `estimatedEnemyBase`:
     - Add to `confirmedExpansions` by tag, set `hasEverConfirmed = true` (expansion seen before main base)

2. For each previously confirmed expansion (iterate `confirmedExpansions`):
   - If the building tag is NOT in the current `enemyBuildings` list:
     - Check if we have vision of the expansion location via `hasVisionOf(location, friendlyUnits, friendlyBuildings)`
     - If we have vision AND the building is absent → retract (remove from `confirmedExpansions`)
     - If we don't have vision → keep (sticky)

`buildRuleUnit()` is **unchanged** — it always populates `expansionEvents` from `expansionBuffer`. The DRL expansion rules continue to operate on suspected (Tier 1) data. Post-confirmation posture bypasses DRL entirely (see §DroolsScoutingTask changes below).

**Updated `reset()`:** Clear `confirmedExpansions`, reset `hasEverConfirmed`, and clear `confirmedMainBase`.

**New testability accessors:**
```java
public int confirmedExpansionCount() { return confirmedExpansions.size(); }
public boolean hasEverConfirmed() { return hasEverConfirmed; }
```

### DroolsScoutingTask

**Updated `execute()`:** Two changes within the `if (needsCep)` block:

**1. processBuildings call** — between `evict()` and `buildRuleUnit()`:
```java
sessionManager.processFrame(enemies, gameTimeMs, ourNexus, estimatedBase);
sessionManager.evict(gameTimeMs);
// NEW: Tier 2 building-based expansion detection
GameState gameState = ctx.getAs(QuarkMindCaseFile.GAME_STATE, GameState.class);
if (gameState != null) {
    sessionManager.processBuildings(gameState.enemyBuildings(), estimatedBase,
                                    gameState.myUnits(), gameState.myBuildings());
}
data = sessionManager.buildRuleUnit();
```

**2. Post-confirmation posture bypass** — replace the current posture resolution:
```java
// Post-confirmation: posture is deterministic from confirmed expansion state.
// Bypasses DRL because unitEvents empties after 3 min (seenUnitTags permanence),
// making the DRL expansion rules unreliable for late-game posture.
if (sessionManager.hasEverConfirmed()) {
    cachedPosture = sessionManager.confirmedExpansionCount() > 0 ? "MACRO" : "ALL_IN";
} else if (data != null && !data.getPostureDecisions().isEmpty()) {
    cachedPosture = data.getPostureDecisions().get(0);
}
String posture = cachedPosture;
```

No changes to the DRL rules — the two-tier logic is handled by the Java posture bypass. The DRL expansion rules continue to operate on `expansionBuffer` data for build-order classification and pattern evidence.

### SpatialCalibrationTest

**Updated `measureReplay()`:** Add `processBuildings()` call after `processFrame()` using `state.enemyBuildings()`. This enables the confirmed tier in the calibration test.

**New test: `macroRevertsToAllInOnExpansionSacrifice()`:** SimulatedGame scenario:
1. Spawn enemy Nexus at expansion location (confirmed expansion → MACRO)
2. After N ticks, remove the expansion building (sacrifice)
3. Spawn friendly unit near the expansion location (vision → retraction fires)
4. Assert posture transitions: UNKNOWN → MACRO → ALL_IN

**New test: `macroRevertsToAllInOnExpansionSacrificeLateGame()`:** SimulatedGame scenario:
1. Spawn enemy Nexus at expansion location (confirmed expansion → MACRO)
2. Run for 200+ ticks (past the 3-minute unit buffer eviction window — unitBuffer is empty)
3. Remove the expansion building (sacrifice)
4. Spawn friendly unit near the expansion location (vision → retraction fires)
5. Assert posture transitions: UNKNOWN → MACRO → ALL_IN
6. Verify transition occurs despite empty unitBuffer (post-confirmation Java bypass)

**New test: `macroStableWhenExpansionPersists()`:** SimulatedGame scenario:
1. Spawn enemy Nexus at expansion location → MACRO
2. Run 900 ticks with the building present
3. Assert posture stays MACRO (no false retraction)

### Protocol update

The protocol `strategy-attack-under-unknown-posture.md` referenced in the original spec does not exist (also referenced in #300 spec). A new protocol needs to be created describing posture semantics — including the new MACRO → ALL_IN transition when confirmed expansions are retracted. This is a separate task to be tracked as a GitHub issue.

## Testing Strategy

### Unit tests (plain JUnit)

**ScoutingSessionManager tests:**
- `processBuildings` adds base-type buildings at non-starting locations to `confirmedExpansions`
- `processBuildings` ignores main-base buildings (within EXPANSION_DISTANCE_THRESHOLD of enemy base)
- `processBuildings` retracts confirmed expansion when building absent + vision available
- `processBuildings` retains confirmed expansion when building absent + no vision (fog)
- `hasEverConfirmed` transitions pre→post confirmation on first building confirmation
- `confirmedMainBase` is set to the **closest** base-type building within `EXPANSION_DISTANCE_THRESHOLD` of enemy base estimate (deterministic when multiple candidates visible)
- When main base and natural visible simultaneously, closest to estimate wins regardless of list order
- Expansions classified relative to `confirmedMainBase` using tighter `CONFIRMED_EXPANSION_DISTANCE`
- `buildRuleUnit()` always uses `expansionBuffer` (unchanged)

**ObservationTranslator tests:**
- Enemy buildings are separated from enemy units (not in `enemyUnits`, present in `enemyBuildings`)
- `GameState.enemyBuildings()` populated with correct building types and positions
- Self-unit separation behavior unchanged (regression guard)

**SpatialCalibrationTest additions:**
- `macroRevertsToAllInOnExpansionSacrifice` — UNKNOWN → MACRO → ALL_IN (within unit buffer window)
- `macroRevertsToAllInOnExpansionSacrificeLateGame` — UNKNOWN → MACRO → ALL_IN (after 3-min eviction, verifies Java bypass)
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
ObservationTranslator.translate()  ← PREREQUISITE FIX
  │ enemyUnits = non-building enemy observations
  │ enemyBuildings = building enemy observations (was List.of())
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
  │ expansionEvents ← expansionBuffer (always, unchanged)
  ▼
DroolsScoutingTask.drl
  │ "Expansion: Macro" / "Expansion: All-In" rules ← unchanged
  ▼
DroolsScoutingTask.execute()
  │ if hasEverConfirmed: posture ← Java bypass (confirmedExpansions state)
  │ else:                posture ← DRL postureDecisions + cachedPosture
  │ ENEMY_POSTURE written to CaseFile
  ▼
MomentDetectionTask → GamePhaseSummariser → TacticalPosture
  (downstream cascade unaffected)
```

## Vision Check for Retraction

To determine if we have vision of an expansion location, check whether any friendly unit or building is within vision range. A simple distance check:

```java
private boolean hasVisionOf(Point2d location, List<Unit> friendlyUnits, List<Building> friendlyBuildings) {
    float visionRange = 9.0f; // conservative: Stalker vision range (most common combat unit)
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

1. **Vision range is approximate.** Different units have different vision ranges in SC2 (Observer: 11, Stalker: 10, Probe: 8). Using a single conservative constant (9.0, Stalker range) may delay retraction when only Observers are present, but avoids premature retraction when low-vision units (Probes at 8) are the only units nearby. False retraction (premature ALL_IN) is worse than delayed retraction (stale MACRO).

2. **Building type detection scope.** Only base-type buildings are checked (Nexus/Hatchery/CC and their upgrades). Proxy structures (forward Pylon, proxy Barracks) are not tracked as expansions. This is correct — proxy aggression is a different signal than economic expansion.

3. **Multi-expansion tracking.** `confirmedExpansions` tracks all confirmed expansion buildings. ALL_IN reversion only fires when ALL confirmed expansions are retracted. An enemy with 3 expansions who loses 1 stays MACRO.

## References

- `ObservationTranslator.java:122-137` — enemy building separation (prerequisite fix)
- `GameStateTranslator.java:61` — ENEMY_UNITS populated from state.enemyUnits()
- `ScoutingSessionManager.java` — expansion buffer and unit heuristic (fix target)
- `DroolsScoutingTask.java:249-254` — cachedPosture and posture resolution
- `DroolsScoutingTask.drl` — "Expansion: Macro" and "Expansion: All-In" rules
- `Building.java` — domain record with tag, type, position, health
- `BuildingType.java` — enum with base types (NEXUS, HATCHERY, COMMAND_CENTER, etc.)
- Protocol for posture semantics — needs creation (GitHub issue to be filed; referenced in both #300 and #305 specs)
- Protocol `sc2data-spatial-constants-require-calibration.md` — calibration methodology for vision range and `CONFIRMED_EXPANSION_DISTANCE`
- Issue #300 — posture persistence (cachedPosture fix)
- Issue #304 — ALL_IN calibration (expansion heuristic false-positive finding)
