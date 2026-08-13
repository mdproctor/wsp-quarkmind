# Map Control Dominance Factor Design

**Issue:** #228
**Date:** 2026-08-13
**Status:** Approved
**Depends on:** #223 (multi-factor DominanceAssessor), #256 (ExpansionLocation clustering)

## Problem

`MultiFactorDominanceAssessor` has four factors: economy, army value, tech tier, base count. Map control — territorial coverage relative to available expansion locations — was descoped from #223 pending base-location clustering. That clustering now exists via `ExpansionLocation` (#256). The current base count factor measures absolute count delta but not spatial coverage across the map.

## Design Decisions

1. **5th factor alongside bases** — `mapControl` added as a 5th field in `DominanceWeights`, not a replacement. Runtime weight configuration enables testing both modes: set `bases=0.05, mapControl=0.10` for coexistence, or `bases≈0.0, mapControl=0.15` for effective replacement. Note: the Drools strategy enforces `MINIMUM_WEIGHT=0.05` per factor; full replacement (bases=0.0) requires the temporal strategy (current default).
2. **Expansion-site occupation** — an expansion location is "controlled" if a completed base building sits within `EXPANSION_CONTROL_RADIUS` (10.0 map units) of its centroid. Expansions with no nearby base are "uncontrolled." The factor name `mapControl` is the extension point — richer territorial signals (army presence, watchtowers, creep) can enrich it later.
3. **Consistent normalisation** — `clamp((ownControlled - enemyControlled) / maxExpectedMapControlDelta)`, matching the pattern of all other factors (fixed configurable constant, not variable per map).
4. **Fog-of-war guard** — `enemyBuildings.isEmpty() → 0.0`, consistent with bases and tech factors. Acknowledged asymmetry: own controlled expansions are always known but enemy expansions behind fog are invisible, systematically inflating perceived advantage mid-game. Same limitation as bases factor, bounded by dead zone threshold and milestone confidence.
5. **Default weights** — economy=0.30, army=0.35, tech=0.20, bases=0.05, mapControl=0.10. Budget taken primarily from bases (0.15→0.05). Both factors overlap significantly for standard play. Bases at 0.05 sits at the Drools MINIMUM_WEIGHT floor — asymmetric modifier responsiveness (can go up, not down). Uncalibrated defaults.
6. **Calibration** — `EXPANSION_CONTROL_RADIUS=10.0` is an uncalibrated starting value. A calibration test validates against IEM10 + AI Arena replays per the spatial constants protocol (`sc2data-spatial-constants-require-calibration`).

## Factor Calculation

### Map Control (`mapControl`)

```java
static final double EXPANSION_CONTROL_RADIUS = 10.0;

private double mapControlFactor(GameState state) {
    if (state.enemyBuildings().isEmpty()) return 0.0;
    List<ExpansionLocation> expansions = state.mapInfo().expansions();
    if (expansions.isEmpty()) return 0.0;

    long ownControlled = expansions.stream()
        .filter(exp -> controlledBy(state.myBuildings(), exp)).count();
    long enemyControlled = expansions.stream()
        .filter(exp -> controlledBy(state.enemyBuildings(), exp)).count();
    return clamp((double) (ownControlled - enemyControlled) / maxExpectedMapControlDelta);
}

private static boolean controlledBy(List<Building> buildings, ExpansionLocation exp) {
    return buildings.stream().anyMatch(b ->
        b.isComplete() && SC2Data.isBase(b.type())
        && b.position().distanceTo(exp.position()) <= EXPANSION_CONTROL_RADIUS);
}
```

Data source: `GameState.mapInfo().expansions()` — populated at game start from resource clustering (#256). `Building.position()` and `ExpansionLocation.position()` provide the proximity check inputs.

`maxExpectedMapControlDelta` configurable (default: 4). Typical SC2 maps have 8-16 expansions; controlling 4 more than the opponent is a significant territorial lead.

## Type Changes

### `DominanceWeights`

```java
public record DominanceWeights(double economy, double army, double tech, double bases, double mapControl) {
    public DominanceWeights {
        double sum = economy + army + tech + bases + mapControl;
        if (Math.abs(sum - 1.0) > 0.001) {
            throw new IllegalArgumentException("Weights must sum to 1.0, got " + sum);
        }
    }
}
```

### `WeightModifier`

```java
public record WeightModifier(
    double economyDelta, double armyDelta, double techDelta,
    double basesDelta, double mapControlDelta, String reason) {}
```

### `MilestoneConfig.Dominance`

Add:
```java
@WithName("max-expected-map-control-delta")
@WithDefault("4")
int maxExpectedMapControlDelta();
```

### `MilestoneConfig.Dominance.WeightAnchor`

Add:
```java
@WithName("map-control-weight") double mapControlWeight();
```

## Assessor Changes

### `MultiFactorDominanceAssessor.assess()`

- Compute `double mapControl = mapControlFactor(state)` alongside the other four factors
- Include in weighted sum: `+ mapControl * weights.mapControl()`
- Add to factors map: `factors.put("mapControl", mapControl)`
- Update NEUTRAL constant: add `"mapControl", 0.0`
- Update weight-shift logging: include `mapControl` weight in debug format string

### Constructor

- Add `maxExpectedMapControlDelta` field (CDI constructor reads from config, test constructor accepts parameter)

## Weight Strategy Changes

All three implementations return 5-field `DominanceWeights`:

### `TemporalDominanceWeightStrategy`

- `WeightAnchor` gains `mapControlWeight` field
- Interpolation includes mapControl dimension

### `SituationalDominanceWeightStrategy`

- Phase-to-weights map entries gain mapControl field

### `DroolsDominanceWeightStrategy`

- `applyModifiers()` processes `mapControlDelta` from modifiers
- Clamps at `MINIMUM_WEIGHT` per factor (mapControl included)
- Renormalization includes 5th field

## Drools Rule Changes

All 19 rules in `DominanceWeightAdjustment.drl` gain a 5th argument `0.00` in `new WeightModifier(...)`:

```java
// Before:
modifiers.add(new WeightModifier(-0.10, +0.15, -0.05, 0.00, "..."));
// After:
modifiers.add(new WeightModifier(-0.10, +0.15, -0.05, 0.00, 0.00, "..."));
```

MapControl weight adjustments per archetype/phase deferred until we have runtime data on how map control interacts with tactical situations.

## Config Changes

### `application.properties`

Default weight anchors updated across all profiles:

```properties
quarkmind.milestones.dominance.max-expected-map-control-delta=4
# Weight anchors — example for temporal strategy:
# economy=0.30, army=0.35, tech=0.20, bases=0.05, mapControl=0.10
```

## Testing

### New tests

| Test | Type | What it validates |
|------|------|------------------|
| `mapControlFactor_ownControlsMore_positive` | Unit | Own bases at 2 expansions, enemy at 1 → positive score |
| `mapControlFactor_noExpansions_returnsZero` | Unit | Empty expansions list → 0.0 |
| `mapControlFactor_enemyBuildingsEmpty_returnsZero` | Unit | Fog guard returns 0.0 |
| `mapControlFactor_equalControl_zero` | Unit | Symmetric occupation → ~0.0 |
| `MapControlCalibrationTest` | Benchmark (`@Tag("benchmark")`) | Validates EXPANSION_CONTROL_RADIUS=10.0 against IEM10 + AI Arena replays — measures base-building-to-expansion-centroid distances |

### Updated tests

| Test | Change |
|------|--------|
| `assess_containsAllFiveFactors` | Assert 5 keys (was 4) |
| `assess_overallIsWeightedSum` | Include mapControl in weighted sum calculation |
| `assess_overallClampedToOne` | Update constructor call with 5th parameter |
| `gameState()` helpers | Add overload accepting `MapInfo` with expansion locations |
| `DominanceWeightsTest` | 5-field construction, sum-to-1.0 with 5 fields |
| All weight strategy tests | Return and assert 5-field weights |
| `DroolsDominanceWeightStrategyTest` | 5-field `WeightModifier` assertions |
| `DominanceWeightRuleUnitTest` | 5-field modifier in rule firing assertions |

### Test fixture design

Existing `gameState()` factory passes `null` for `mapInfo`. Tests that don't need map control keep using this — `mapControlFactor` returns 0.0 when `expansions.isEmpty()` or `mapInfo` is null-safe. New map control tests use an overload that constructs `MapInfo` with expansion locations at known positions.

## Changed Classes

| Class | Change |
|-------|--------|
| `DominanceWeights` | Add `mapControl` field (5th), sum invariant includes it |
| `WeightModifier` | Add `mapControlDelta` field (5th) |
| `MilestoneConfig.Dominance` | Add `maxExpectedMapControlDelta` |
| `MilestoneConfig.Dominance.WeightAnchor` | Add `mapControlWeight` |
| `MultiFactorDominanceAssessor` | Add `mapControlFactor()`, `controlledBy()`, `EXPANSION_CONTROL_RADIUS`, `maxExpectedMapControlDelta` field; update `assess()`, constructors, NEUTRAL, logging |
| `TemporalDominanceWeightStrategy` | 5-field weights, interpolation |
| `SituationalDominanceWeightStrategy` | 5-field weights |
| `DroolsDominanceWeightStrategy` | 5-field modifier processing, clamping, renormalization |
| `DominanceWeightAdjustment.drl` | All 19 rules — 5th arg `0.00` |
| `AnchorInterpolator` | 5-field interpolation (if separate from TemporalDominanceWeightStrategy) |

## New Classes

| Class | Package | Purpose |
|-------|---------|---------|
| `MapControlCalibrationTest` | `io.quarkmind.agent` (test) | Benchmark — validates EXPANSION_CONTROL_RADIUS against replay datasets |

## Deferred

- Game-design tuning of Drools mapControl weight modifiers per archetype/phase — needs runtime data
- Richer map control signals: army presence, watchtower control, creep spread, static defense
- Replay-validated weight calibration for all 5 factors (#227)
