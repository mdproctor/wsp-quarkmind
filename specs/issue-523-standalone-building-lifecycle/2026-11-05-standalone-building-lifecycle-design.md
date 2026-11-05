# Standalone Building Lifecycle — Design Spec

**Issue:** #523 (child of #516 — Epic: Standalone EmulatedGame)
**Date:** 2026-11-05
**Branch:** issue-523-standalone-building-lifecycle

## Goal

Eliminate the `syncBuildings` GT sync from the replay validation harness. EmulatedGame must independently track building construction, completion, morphs, add-ons, and destruction from tracker event intents — no ground truth building injection.

## Background

The harness currently calls `syncBuildings()` every tick, which:
1. **Injects new buildings** (`injectReplayBuilding`) from GT state
2. **Syncs completion** (`syncReplayBuilding`) — updates incomplete→complete
3. **Marks reactors** (`markBuildingReactored`) from GT reactor tags
4. **Detects destruction** (standalone variant) — removes buildings that disappear from GT

`TrackerIntentExtractor` already sees building tracker events (UnitInit, UnitBorn, UnitDone, UnitTypeChange, UnitDied) but deliberately skips all of them for buildings. Only Drone consumption for Zerg building construction is handled (`consumeDroneForBuilding`).

EmulatedGame already has the building lifecycle machinery: `handleBuild()` with construction timers and `PendingCompletion`, `handleMorph()` for building morphs (Lair/Hive/GreaterSpire/OrbitalCommand/PlanetaryFortress), `removeDestroyedBuilding()`, and `markBuildingReactored()`.

## Approach

Emit building lifecycle intents from `TrackerIntentExtractor` and process them through EmulatedGame's existing `applyIntent()` path, extending it with two new intent types and modifications to the existing `BuildIntent`.

## New Intent Types

### BuildingBornIntent

For buildings that exist at game start (initial Nexus/CC/Hatchery). These appear as `UnitBorn` events at loop 0.

```java
public record BuildingBornIntent(
    String tag,           // tracker tag (r-tagIndex-tagRecycle)
    BuildingType type,
    Point2d position
) implements Intent {}
```

**EmulatedGame handler:** Adds a complete building directly to `PlayerState` — no construction timer, no mineral deduction (these buildings are not purchased). Registers town halls and gas buildings. Adds supply bonus. Records building spending for economy tracking.

### BuildingDiedIntent

For building destruction.

```java
public record BuildingDiedIntent(
    String tag,           // tracker tag
    BuildingType type
) implements Intent {}
```

**EmulatedGame handler:** Calls existing `removeDestroyedBuilding()` logic — subtracts supply bonus, records destruction in economy tracker, removes building from `PlayerState`.

### Extended BuildIntent

Add `replayTag` and `completionLoop` fields to support tracker-driven building construction:

```java
public record BuildIntent(
    String unitTag,           // worker tag (existing field)
    BuildingType buildingType,
    Point2d location,
    String replayTag,         // NEW: tracker tag for the building (null for non-tracker)
    long completionLoop       // NEW: pinned completion from UnitDone (-1 for computed)
) implements Intent {}
```

Existing callers use a convenience constructor: `BuildIntent(unitTag, buildingType, location)` which sets `replayTag=null` and `completionLoop=-1`.

**EmulatedGame handler modification:** When `replayTag` is non-null, use it as the building tag instead of generating "bldg-N". When `completionLoop >= 0`, pin completion to that loop (like TrainIntent's `pinnedCompletionLoop`). Skip mineral deduction for tracker-driven builds (the player already spent the resources).

## Intent sealed hierarchy update

`Intent.java` permits clause updated to include `BuildingBornIntent` and `BuildingDiedIntent`. Both `applyIntent` switch expressions in `EmulatedGame` (lines 333 and 349) updated with new cases.

## TrackerIntentExtractor Changes

### New building tracking map

Add `buildingTagToType` (Map<Long, BuildingType>) alongside the existing `unitTagToType`. Keyed by composite tag (tagIndex << 32 | tagRecycle). Populated from UnitInit and UnitBorn events for buildings.

### UnitBorn handler (buildings)

Currently: `if (Sc2ReplayShared.BUILDING_NAMES.contains(typeName)) {continue;}`

Change: When `typeName` is a building name, add to `buildingTagToType` and emit `BuildingBornIntent`:

```
BuildingBornIntent(tag, Sc2ReplayShared.toBuildingType(typeName), position)
```

Position comes from `IBaseUnitEvent.getXCoord()` / `getYCoord()`.

### UnitInit handler (buildings)

Currently: Calls `consumeDroneForBuilding()` and continues.

Change: Still call `consumeDroneForBuilding()`, then add to `buildingTagToType`. Track the tag in a `pendingBuildings` set. Emit `BuildIntent` with tracker tag:

```
BuildIntent("worker", buildingType, position, replayTag, -1)
```

The `completionLoop` is initially -1 (unknown). A second pass correlates UnitDone events.

### UnitDone handler

Currently: No-op.

Change: If the tag is in `pendingBuildings`, record the completion loop. Used to set `completionLoop` on the corresponding `BuildIntent`.

**Implementation detail:** Two-pass approach. First pass collects all events and builds a `pendingBuildingDoneLoops` map (tag → completionLoop). The `BuildIntent` is created with the pre-computed completionLoop from UnitDone. Since `extractFromReplay` already iterates all events and sorts intents at the end, a pre-pass for UnitDone loops is natural.

### UnitTypeChange handler (buildings)

Currently: Skips buildings (not in `unitTagToType`).

Change: Check `buildingTagToType` as well. When a building's type changes, emit a `MorphIntent`:

```
MorphIntent(tag, sourceName, targetName)
```

The existing `handleMorph()` in EmulatedGame already handles building morphs via `MORPH_BUILDING_TARGETS`.

### UnitDied handler (buildings)

Currently: Skips buildings (not in `unitTagToType`).

Change: Check `buildingTagToType` as well. When a building dies, emit a `BuildingDiedIntent`:

```
BuildingDiedIntent(tag, buildingType)
```

### Reactor/TechLab add-on detection

When `UnitInit` fires for a Reactor add-on (type name ends with "Reactor"):
1. Emit a normal `BuildIntent` for the Reactor building itself
2. Add a flag on the `BuildIntent` indicating it's a Reactor add-on (new boolean field `isReactor`, or detect from building type name at handling time)

When the Reactor building completes in EmulatedGame, the handler finds the nearest production building of the matching type (using position proximity, same logic as `ReplaySimulatedGame.findNearestProductionBuilding`) and calls `markBuildingReactored()` on it.

**Alternative considered:** Emitting a separate `ReactorMarkIntent`. Rejected because the Reactor detection happens naturally at building completion — no separate intent needed.

## Harness Integration

### ReplayValidationHarness

1. Remove `syncBuildings()` call from `runInternal()` (line 101)
2. Remove `injectedTags` tracking (line 88)
3. Remove `reactoredTags` from `ReplaySimulatedGame` (line 84-85) — no longer needed since Reactor detection happens from tracker events
4. Building intents flow through the same `applyIntent()` path as TrainIntents — they're part of the `intents` list from `TrackerIntentExtractor`

### StandaloneEmulatedGameBaselineTest

1. Remove `syncBuildings()` call from `runStandalone()` (line 276)
2. Remove `injectedBuildings` map and `reactoredTags` — no longer needed
3. Building accuracy now measured by comparing EmulatedGame's own building state against GT

### resetForHarness()

No change. The harness still uses `resetForHarness()` (no race model seed). All buildings come from tracker event intents — including the initial town hall via `BuildingBornIntent` from the loop-0 UnitBorn event.

## Acceptance Criteria

Per issue #523:
- Building accuracy 100% at all checkpoints without GT sync
- Production buildings available at correct times for unit training
- Tech tree progression correct

**Validation:** `TrackerDrivenAccuracyBaselineTest` and `StandaloneEmulatedGameBaselineTest` must pass with `syncBuildings` removed. Building counts (complete) must match GT at all checkpoint intervals.

## Edge Cases

### WarpGate morph (Gateway → WarpGate)

`toBuildingType` maps both "Gateway" and "WarpGate" to `BuildingType.GATEWAY`. The UnitTypeChange event fires but since both map to the same BuildingType, the morph is effectively a no-op at the EmulatedGame level. No special handling needed — `MORPH_BUILDING_TARGETS` doesn't include WarpGate.

### Zerg Drone consumption

`consumeDroneForBuilding()` already removes one Drone from `unitTagToType` when a non-morph Zerg building starts construction. This continues to work unchanged. EmulatedGame's `handleBuild()` deducts mineral costs — the Drone is consumed as part of Zerg building mechanics (the worker becomes the building).

### Flying buildings (Terran)

`toBuildingType` maps flying variants ("BarracksFlying", "FactoryFlying", etc.) to their landed counterparts. UnitTypeChange events for landing/lifting are effectively no-ops since both states map to the same BuildingType. No special handling needed.

### Building cancellation

In SC2, a building under construction can be cancelled, refunding 75% of minerals. Tracker data represents this as a UnitDied event (the incomplete building dies). The `BuildingDiedIntent` handler already handles this — it removes the building and subtracts any supply bonus. Resource refund is not explicitly tracked since the harness validates economy against GT.

### Supply Depot lowering

"SupplyDepot" and "SupplyDepotLowered" both map to `BuildingType.SUPPLY_DEPOT`. No special handling needed.

### CreepTumor, Nydus Canal

These are ability-placed structures with no construction time. They appear as UnitBorn (not UnitInit), so they get a `BuildingBornIntent` — complete immediately. This is correct because creep tumors and nydus worms function as complete buildings from the moment they're placed.

## Files Modified

| File | Change |
|------|--------|
| `intent/Intent.java` | Add `BuildingBornIntent`, `BuildingDiedIntent` to permits |
| `intent/BuildingBornIntent.java` | New record |
| `intent/BuildingDiedIntent.java` | New record |
| `intent/BuildIntent.java` | Add `replayTag`, `completionLoop` fields; convenience constructor |
| `emulated/EmulatedGame.java` | Handle `BuildingBornIntent`, `BuildingDiedIntent` in switch; modify `handleBuild` for tracker tags and pinned completion |
| `replay/TrackerIntentExtractor.java` | Emit building intents from UnitBorn/UnitInit/UnitDone/UnitTypeChange/UnitDied; two-pass for UnitDone completion loops |
| `replay/ReplayValidationHarness.java` | Remove `syncBuildings()` |
| `test/.../StandaloneEmulatedGameBaselineTest.java` | Remove `syncBuildings()` |
| `test/.../TechSpendingDiagnosticTest.java` | Remove `syncBuildings()` |

## References

- `EmulatedGame.java:693-724` — existing `handleBuild()` with construction timer
- `EmulatedGame.java:1060-1134` — existing `handleMorph()` with `MORPH_BUILDING_TARGETS`
- `EmulatedGame.java:1597-1603` — existing `removeDestroyedBuilding()`
- `EmulatedGame.java:1605-1618` — existing `injectReplayBuilding()`
- `EmulatedGame.java:1671-1698` — existing `syncReplayBuilding()`
- `TrackerIntentExtractor.java:74-204` — main extraction loop
- `TrackerIntentExtractor.java:232-249` — `buildBuildingTypeMap()` (already tracks buildings)
- `ReplayValidationHarness.java:202-214` — `syncBuildings()` (to be removed)
- `ReplaySimulatedGame.java:265-295` — GT `applyUnitInit()` (reference implementation)
- `ReplaySimulatedGame.java:388-402` — `findNearestProductionBuilding()` (Reactor detection)
- `StandaloneEmulatedGameBaselineTest.java:326-347` — standalone `syncBuildings()` (to be removed)
- Issue #523 — acceptance criteria
- Issue #516 — parent epic, GT sync elimination table
