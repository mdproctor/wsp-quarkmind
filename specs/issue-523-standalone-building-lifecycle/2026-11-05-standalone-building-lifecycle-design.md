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

**EmulatedGame handler:** Adds a complete building directly to `PlayerState` — no construction timer, no mineral deduction, no spending recording (these buildings are gifts at game start, not purchased). Registers town halls and gas buildings. Adds supply bonus.

### BuildingDiedIntent

For building destruction.

```java
public record BuildingDiedIntent(
    String tag,           // tracker tag
    BuildingType type
) implements Intent {}
```

**EmulatedGame handler:** Calls existing `removeDestroyedBuilding()` logic — subtracts supply bonus, records destruction in economy tracker, removes building from `PlayerState`.

### Extended MorphIntent

Add `completionLoop` field to support pinned morph completion from tracker data (R2-04):

```java
public record MorphIntent(
    String unitTag,
    String sourceName,
    String targetName,
    long completionLoop       // NEW: pinned completion from UnitDone (-1 for computed)
) implements Intent {}
```

Existing callers use a convenience factory: `MorphIntent.of(unitTag, sourceName, targetName)` which sets `completionLoop=-1`. All 3 production sites (SC2DeliveryHandler, ReplayCommandExtractor, TrackerIntentExtractor unit morphs) and 13 test sites updated to use this factory.

**EmulatedGame handler modification:** In `handleMorph`, when `completionLoop >= 0`, pin the morph completion to that loop (convert to tick: `completionLoop / LOOPS_PER_TICK`) instead of computing `gameFrame + buildTimeInLoops(targetBt) / LOOPS_PER_TICK`. This is the same pattern as BuildIntent's pinned completion and TrainIntent's `pinnedCompletionLoop`.

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

Existing callers use a compact constructor or static factory: `BuildIntent.of(unitTag, buildingType, location)` which sets `replayTag=null` and `completionLoop=-1`. All 11 existing call sites must be updated: ProtobufToIntent, ReplayCommandExtractor, EnemyBehavior, SC2DeliveryHandler, EconomicsDecisionService (3 sites), BasicEconomicsTask, DroolsStrategyTask (3 sites).

**EmulatedGame handler modification:**
- When `replayTag` is non-null, use it as the building tag instead of generating "bldg-N".
- When `completionLoop >= 0`, pin completion to that loop (like TrainIntent's `pinnedCompletionLoop`).
- **Tracker-driven affordability bypass (R1-06):** When `replayTag` is non-null, skip the affordability check and deduct costs unconditionally (may go negative — correct, matches `injectReplayBuildingWithCost`'s documented behavior). For non-tracker builds, keep the existing affordability check.
- Always deduct mineral and gas costs from PlayerState — consistent with handleTrain's tracker-driven path. While the mineral GT sync is active, the deduction is overwritten. When the mineral GT sync is later removed (#516), the deduction is essential.
- **Gas cost gap (R1-07):** `SC2Data.gasCost(BuildingType)` currently returns incorrect values for Protoss tech buildings — several return 0 when they should have significant gas costs, and two (Dark Shrine, Templar Archives) return 100 when they should be higher. Fix `SC2Data.gasCost()` as part of this change:

  | Building | Current | Correct |
  |----------|---------|---------|
  | Twilight Council | 0 | 100 |
  | Robotics Facility | 0 | 100 |
  | Stargate | 0 | 150 |
  | Fleet Beacon | 0 | 200 |
  | Robotics Bay | 0 | 150 |
  | Dark Shrine | 100 | 150 |
  | Templar Archives | 100 | 200 |

  Then add gas deduction to `handleBuild` and use the 3-arg `economyTracker.recordBuildSpending(type, mineralCost, gasCost)`.
- Add the building tag to `trackerDrivenBuildings` when processing a tracker-driven BuildIntent.

## Intent sealed hierarchy update

`Intent.java` permits clause updated to include `BuildingBornIntent` and `BuildingDiedIntent`. Three exhaustive switch expressions updated with new cases:

1. `EmulatedGame.applyIntent(TimedIntent)` (line 333) — dispatch to handlers
2. `EmulatedGame.applyIntent(Intent, PlayerState, PhysicsState)` (line 349) — dispatch to handlers
3. `ActionTranslator.translate()` (line 44) — map to SC2 protocol commands (new types → `null`, buildings are not translated to real SC2 actions)

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

**Loop-0 ordering constraint (from decision review):** `BuildingBornIntents` at loop 0 must sort before `TrainIntents` at loop 0. The initial town hall must exist before worker trains resolve. Add a `thenComparingInt` secondary sort key by intent type priority in the final `intents.sort()` (the `Comparator.comparingLong(TimedIntent::loop)` call at the end of `extractFromReplay`):

| Intent type | Priority | Rationale |
|------------|----------|-----------|
| BuildingBornIntent | 0 | Initial buildings must exist before anything trains from them |
| BuildingDiedIntent | 1 | Remove old buildings before creating replacements (morph flows) |
| BuildIntent | 2 | Construction starts before training |
| MorphIntent | 3 | Morphs happen on existing buildings |
| TrainIntent | 5 | Training requires buildings to exist |
| ResearchIntent | 5 | Research requires buildings to exist |
| AbilityIntent | 5 | Abilities target existing entities |
| UnitDiedIntent | 8 | Deaths are final events |
| All others | 10 | Default — no ordering dependency |

### UnitInit handler (buildings)

Currently: Calls `consumeDroneForBuilding()` and continues.

Change: Still call `consumeDroneForBuilding()` for non-morph buildings, then add to `buildingTagToType`. Use the `buildingDoneLoops` pre-pass map (see UnitDone handler below) to get the pinned `completionLoop`.

**Morph vs fresh construction (R1-04, R2-02):** UnitInit fires for BOTH fresh buildings AND building morphs (Lair, Hive, GreaterSpire, OrbitalCommand, PlanetaryFortress). Detect morph buildings using `MORPH_BUILDING_SOURCES.containsKey(typeName)` — NOT `NON_DRONE_BUILDINGS`, which is a "skip drone consumption" set with different membership (includes LurkerDen, SpineCrawlerUprooted, SporeCrawlerUprooted — none of which are morph buildings; excludes OrbitalCommand, PlanetaryFortress — which ARE morph buildings). For morph buildings, emit a `MorphIntent` instead of a `BuildIntent`:

```
// For morph buildings (MORPH_BUILDING_SOURCES.containsKey(typeName)):
//   {Lair, Hive, GreaterSpire, OrbitalCommand, PlanetaryFortress}
MorphIntent(tag, sourceType.name(), trackerTypeName, completionLoop)
// e.g. MorphIntent("r-5-1", "HATCHERY", "Lair", 4320)

// For fresh construction (all other buildings):
BuildIntent("worker", buildingType, position, replayTag, completionLoop)
```

The `sourceType` for morph buildings is resolved in two steps:
1. **Tag lookup (preferred):** Check `buildingTagToType` for the compositeTag — if the morph target reuses the source building's tag, the source type is already registered. E.g., a Lair UnitInit at the Hatchery's compositeTag finds HATCHERY in the map.
2. **Static fallback:** If no entry exists (morph target gets a new tag), use a static `MORPH_BUILDING_SOURCES` map:
   ```java
   Map.of("Lair", BuildingType.HATCHERY, "Hive", BuildingType.LAIR,
          "GreaterSpire", BuildingType.SPIRE,
          "OrbitalCommand", BuildingType.COMMAND_CENTER,
          "PlanetaryFortress", BuildingType.COMMAND_CENTER)
   ```
   This is deterministic — each morph target has exactly one source type.

Update `buildingTagToType` to the new type after emitting the MorphIntent. The `handleMorph` auto-resolve mechanism (line 1092) uses `BuildingType.valueOf(sourceName)` to find the first complete building of the source type, so the MorphIntent works correctly regardless of whether the tracker tag matches the existing building's tag.

### UnitDone handler (pre-pass)

Currently: No-op.

Change: A pre-pass before the main extraction loop builds a `buildingDoneLoops` map (composite tag → UnitDone loop). The main UnitInit handler uses this to set `completionLoop` on both `BuildIntent` (fresh construction) and `MorphIntent` (building morphs). Since `extractFromReplay` already iterates all events and sorts intents at the end, a pre-pass is natural.

**Build-time validation diagnostic (from decision review):** Optionally, compute `SC2Data.buildTimeInLoops(type)` and compare against the actual duration (UnitDone loop - UnitInit loop). Log a diagnostic on mismatch. This validates build-time constants against actual replays without affecting completion timing — the pinned value is always used.

### UnitTypeChange handler (buildings)

Currently: Skips buildings (not in `unitTagToType`).

Change: Check `buildingTagToType` as well. When a building's type changes, emit a `MorphIntent`:

```
MorphIntent.of(tag, sourceType.name(), rawTrackerTypeName)
// e.g. MorphIntent.of("r-5-1", "HATCHERY", "Lair")  → completionLoop = -1 (computed)
```

**Naming convention (R1-02):** `sourceName` must use `BuildingType.name()` format (`"HATCHERY"`, `"COMMAND_CENTER"`) to match `BuildingType.valueOf()` at handleMorph line 1097. `targetName` must use the raw SC2 tracker event type name (`"Lair"`, `"OrbitalCommand"`) to match `MORPH_BUILDING_TARGETS` keys. Evidence: `ZergEmulatedGameTest:476` uses `MorphIntent(tag, "HATCHERY", "Lair")`, `TerranEmulatedGameTest:259` uses `MorphIntent(tag, "COMMAND_CENTER", "OrbitalCommand")`.

**Pre-existing unit morph naming bug (companion fix):** The existing unit morph code at `TrackerIntentExtractor.java:148` uses `oldType.name()` / `newType.name()` — producing `"BANELING"` instead of `"Baneling"`. This fails silently because `MORPH_UNIT_TARGETS` and `MORPH_SOURCE_TYPES` keys use tracker event names (e.g., `"Baneling"`, `"Zergling"`). Currently harmless (GT sync compensates), but must be fixed before unit GT sync removal. Fix in this change: use `raw.get("unitTypeName")` for targetName (the raw tracker event string), and the `MORPH_SOURCE_TYPES` key-name convention for sourceName (currently inconsistent — building morphs use enum names, unit morphs should use tracker event names since `MORPH_SOURCE_TYPES` is keyed by tracker event names).

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

**Self-match prevention (R1-03):** The Reactor building maps to the same `BuildingType` as its parent (e.g., `BarracksReactor` → `BARRACKS`). The nearest-building search must exclude the Reactor's own tag from candidates, otherwise it self-matches at distance 0. Pass the Reactor's tag into the search method and skip it.

**Alternative considered:** Emitting a separate `ReactorMarkIntent`. Rejected because the Reactor detection happens naturally at building completion — no separate intent needed.

**TechLab add-ons (R1-09):** TechLab add-ons (BarracksTechLab, FactoryTechLab, StarportTechLab) also map to their parent's BuildingType. They should be treated as regular buildings for counting purposes but are NOT production buildings (same exclusion as Reactors — `if (!isAddOn)`). TechLab gating for unit production (Marauder requires TechLab) is out of scope for this issue — file a follow-up if needed for 100% accuracy.

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

**Follow-up:** When the mineral/vespene GT sync is removed (#518/#519), cancellation refunds will need explicit tracking. File issue linked to the economy GT sync removal epic.

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
| `intent/MorphIntent.java` | Add `completionLoop` field; convenience constructor |
| `domain/SC2Data.java` | Fix `gasCost(BuildingType)` for 7 Protoss buildings |
| `emulated/EmulatedGame.java` | Handle `BuildingBornIntent`, `BuildingDiedIntent` in switch; modify `handleBuild` for tracker tags and pinned completion; modify `handleMorph` for pinned morph completion |
| `replay/TrackerIntentExtractor.java` | Emit building intents from UnitBorn/UnitInit/UnitDone/UnitTypeChange/UnitDied; two-pass for UnitDone completion loops |
| `real/ActionTranslator.java` | Add `BuildingBornIntent` and `BuildingDiedIntent` cases (→ `null`, not translated to SC2 actions) |
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
