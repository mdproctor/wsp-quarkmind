# Dynamic PhaseResolver — State-Based Phase Detection

**Issue:** #258
**Date:** 2026-07-30
**Status:** Approved

## Problem

`TimeBasedPhaseResolver` uses fixed time thresholds (5min/12min) to label game phases. Real game phases depend on game state: a player who fast-expands enters mid-game earlier; a player under rush pressure stays in early-game longer. The phase label feeds pattern classification, coaching context, and strategic reasoning — an inaccurate label cascades.

## Decision Summary

- Change `PhaseResolver.resolve(double gameTimeMinutes)` to `resolve(GameState)`
- Add `GameState` as a CaseFile key alongside decomposed fields
- Implement `StateBasedPhaseResolver` using expansion count, tech tier, and supply
- CDI producer selects implementation via config property (default: `state-based`)
- `StrategyTaxonomy.activeSignatures` keeps independent time-based phaseWindow filtering (different concern — detection windows vs phase labels)

## Interface Change

```java
// domain/PhaseResolver.java
public interface PhaseResolver {
    GamePhase resolve(GameState gameState);
}
```

`PhaseResolver` and `GameState` are both in `domain/` — no dependency issues. Time is derived internally via `SC2Data.GAME_LOOPS_PER_SECOND` (the existing domain constant at 22.4 fps). `GameState` gains a convenience method:

```java
public double gameTimeMinutes() {
    return gameFrame() / SC2Data.GAME_LOOPS_PER_SECOND / 60.0;
}
```

`DroolsScoutingTask.FRAMES_PER_SECOND` and `PatternClassificationCalibrationTest.FRAMES_PER_SECOND` are updated to reference `SC2Data.GAME_LOOPS_PER_SECOND` (existing constant consolidation, not a new constant).

## GameState on CaseFile

`GameStateTranslator.toMap()` adds the full snapshot:

```java
data.put(QuarkMindCaseFile.GAME_STATE, state);
```

New constant:
```java
public static final String GAME_STATE = "game.state";  // game.* namespace — observation state
```

`GAME_STATE` is added to `QuarkMindCaseFile.ALL_KEYS` for completeness. (`ALL_KEYS` currently has zero references — tracked as #264 for cleanup evaluation.)

**Pre-existing bug (out of scope):** `GameStateTranslator.toMap()` filters workers as `u.type() == UnitType.PROBE` — Protoss-only. Terran SCVs and Zerg Drones are misclassified as army. The fix (`u.type().isWorker()`) is tracked as #265; it should land as a prerequisite PR before this change.

Decomposed fields remain for plugins that read individual keys. `DroolsScoutingTask` retrieves the full state:

```java
GameState gameState = ctx.getAs(QuarkMindCaseFile.GAME_STATE, GameState.class);
ctx.set(QuarkMindCaseFile.GAME_PHASE, phaseResolver.resolve(gameState).name());
```

## StateBasedPhaseResolver

Location: `agent/StateBasedPhaseResolver.java`. Plain class (no `@ApplicationScoped` — producer owns scope).

### Signals

**Expansion count** — main buildings in `myBuildings`:
- Protoss: NEXUS
- Terran: COMMAND_CENTER, ORBITAL_COMMAND, PLANETARY_FORTRESS
- Zerg: HATCHERY, LAIR, HIVE

**Tech tier** — presence of buildings in `myBuildings`:
- Tier-2: ROBOTICS_FACILITY, STARGATE, TWILIGHT_COUNCIL, FACTORY, STARPORT, LAIR, HYDRALISK_DEN, ROACH_WARREN
- Tier-3: FLEET_BEACON, ROBOTICS_BAY, TEMPLAR_ARCHIVES, DARK_SHRINE, FUSION_CORE, GHOST_ACADEMY, HIVE, GREATER_SPIRE, ULTRALISK_CAVERN

**Supply** — `supplyUsed` from GameState.

All building signals count buildings regardless of `Building.isComplete` status. An under-construction building represents a committed resource investment — the transition signal is the decision to build, not the completion. Time floors (below) prevent misclassification from premature construction.

### Phase Rules

Evaluated top-down, first match wins:

1. **LATE** if: (3+ expansions AND has tier-3 tech) OR supply ≥ 150
2. **MID** if: 2+ expansions OR has tier-2 or tier-3 tech OR supply ≥ 60
3. **EARLY** otherwise

### Time Floors

The resolver won't return a phase earlier than physically possible — prevents misclassification from incomplete observation data:

- LATE requires gameFrame ≥ 8 minutes (clamp to MID)
- MID requires gameFrame ≥ 3 minutes (clamp to EARLY)

These floors are conservative — legitimate game states can't reach those phases before those times.

## TimeBasedPhaseResolver

Updates to `resolve(GameState)`. Extracts time from `gameState.gameFrame()`, applies existing 5.0/12.0 thresholds. No behavior change. Becomes a plain class (producer owns scope).

## CDI Selection

```java
// agent/PhaseResolverProducer.java
@ApplicationScoped
public class PhaseResolverProducer {
    @Produces @ApplicationScoped
    PhaseResolver phaseResolver(
            @ConfigProperty(name = "quarkmind.phase-resolver.strategy",
                           defaultValue = "state-based") String strategy) {
        return "time-based".equals(strategy)
            ? new TimeBasedPhaseResolver()
            : new StateBasedPhaseResolver();
    }
}
```

The producer instantiates directly — both implementations are plain classes with no injected dependencies. Default: `state-based`. Config `quarkmind.phase-resolver.strategy=time-based` reverts to old behavior.

## What Doesn't Change

- **`GamePhase` enum** — EARLY, MID, LATE unchanged
- **`StrategyArchetype`** — phase field on each archetype unchanged
- **`StrategyTaxonomy.activeSignatures(double)`** — keeps per-archetype time-based phaseWindow filtering. Detection windows (when could pattern X physically manifest?) are a different question from phase labels (what developmental stage is the game in?). Per-archetype time windows handle transition overlap; hard phase boundaries would create cliff edges.
- **`GamePhaseSummariser`** — produces TacticalPosture, separate concept
- **`GamePhaseTrigger`** — milestone system, reads TacticalPosture
- **`CoachingWorkerFactory`** — reads GAME_PHASE string from CaseFile, no change needed
- **Scouting dispatch, pattern classification, posture detection** — unchanged

## Test Plan

### StateBasedPhaseResolverTest (new, plain JUnit)

Parameterized tests with constructed `GameState` records:

- **Early game:** 1 base, no tech, low supply → EARLY
- **Mid game via expansion:** 2 bases, no tech → MID
- **Mid game via tech:** 1 base, tier-2 building → MID
- **Mid game via supply:** supply ≥ 60 → MID
- **Late game via expansion + tech:** 3+ bases, tier-3 building → LATE
- **Late game via supply:** supply ≥ 150 → LATE
- **Time floor (MID):** state signals say MID, frame < 3 min → EARLY
- **Time floor (LATE):** state signals say LATE, frame < 8 min → MID
- **Empty buildings:** game start, no buildings → EARLY
- **Mixed signals:** high supply but 1 base → tests supply-based transition
- **All races:** Protoss, Terran, Zerg building types for expansion and tech detection
- **Zerg morph chain:** HIVE present, no surviving tier-2 buildings → MID (not EARLY)
- **Under-construction buildings:** counted toward signals (committed investment)

### TimeBasedPhaseResolverTest (updated)

Same behavior, new signature — pass `GameState` with appropriate `gameFrame`.

### PhaseResolverProducerTest (new, plain JUnit)

Config property routes to correct implementation.

### Existing test updates

DroolsScoutingTask tests referencing `phaseResolver` update to pass `GameState`. No behavior changes expected.

## Files Changed

| File | Change |
|------|--------|
| `domain/PhaseResolver.java` | Interface: `resolve(GameState)` |
| `domain/GameState.java` | Add `gameTimeMinutes()` convenience method |
| `domain/QuarkMindCaseFile.java`* | New `GAME_STATE` constant, added to `ALL_KEYS` |
| `agent/TimeBasedPhaseResolver.java` | Update signature, remove `@ApplicationScoped` |
| `agent/StateBasedPhaseResolver.java` | **New** — state-based implementation |
| `agent/PhaseResolverProducer.java` | **New** — CDI producer with config selector |
| `agent/GameStateTranslator.java` | Add `GAME_STATE` key |
| `plugin/scouting/DroolsScoutingTask.java` | Retrieve `GameState`, pass to resolver |
| Tests (4 files) | New + updated tests |

*`QuarkMindCaseFile` is in `agent/`, not `domain/`.