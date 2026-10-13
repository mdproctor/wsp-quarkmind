# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. 34 commits ahead of main. Pushed accuracy from 99.6% to **99.8%** through Reactor pair detection.

### Completed This Session (third session on #406)

1. **Root cause investigation** — Through systematic diagnostics (train funnel, tick timing, per-replay analysis, Reactor tag tracing), identified the true root cause of the remaining deficit: back-calculated `commandLoop = bornLoop - trainTime` places Reactor pair intents BEFORE the Reactor addon exists in the GT, causing sequential production (1 slot) instead of parallel (2 slots).

2. **Reactor pair detection** (`e4c38d3f`) — When two auto-resolved intents arrive within 10 loops of each other on the same building, detected as Reactor pair and forced 2-slot concurrent production. The building is permanently marked Reactored for future intents. **99.6% → 99.8% (+10 units).**

3. **Six diagnostic tests added** — TrainDiagnosticTest, TickTimingDiagnosticTest, HellionDeficitDiagnosticTest, ReactorTagTraceDiagnosticTest, HellionTickTraceDiagnosticTest, ReactorStateAtIntentTest. All `@Tag("diagnostic")`.

4. **Approaches ruled out** — Reactor-first shortestQueue tiebreaker (neutral), Reactor-only candidate filtering (regressed to 98.9%), pre-tick intent application (neutral), town-hall-fallback removal (regressed to 99.4% — deferred intents lost by processDeferredTrainIntents re-deferral guard).

### Current Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 100.0% | 100% | 100% |
| 3-min | 99.6% | 100% | 75.0% |
| 5-min | **99.8%** | 100% | **98.1%** |
| 7-min | 99.8% | 100% | **99.5%** |

### Cumulative Progression (across all sessions)

| Step | 5-min units | Missing | Change |
|---|---|---|---|
| Start (Milestone 2 beginning) | 94.9% | ~350 | — |
| Economy bypass | 99.2% | ~55 | +4.3pp |
| Reactor dual-training | 99.4% | ~39 | +0.2pp |
| Per-building Reactor + capacity routing | 99.4% | 36 | neutral (more correct) |
| Chrono Boost extraction | 99.5% | 36 | +0.1pp |
| Terran train time calibration | 99.6% | 26 | +0.1pp |
| Phoenix calibration | 99.6% | 25 | +1 unit |
| **Reactor pair detection** | **99.8%** | **16** | **+0.2pp** |

---

## Next Session: 99.8% → higher

### Remaining 16 Missing Units at 5-min

| Type | GT | Em | Delta | 7-min trend | Category |
|---|---|---|---|---|---|
| MARINE | 372 | 365 | -7 | -5 at 7-min (worsening) | Queue routing |
| HELLION | 52 | 50 | -2 | +2 over-prod at 7-min | Reactor timing or routing |
| STALKER | 51 | 49 | -2 | Recovers to 100% | Reactor timing |
| MEDIVAC | 9 | 8 | -1 | -4 at 7-min (worsening) | Queue routing |
| REAPER | 36 | 35 | -1 | Recovers to 100% | Reactor timing |
| WIDOW_MINE | 28 | 27 | -1 | -4 at 7-min (worsening) | Queue routing |

### Two Remaining Fix Paths

#### Path 1: Tune Reactor pair detection (quick — 30 min)

The current pair detection triggers when two intents arrive within 10 loops on the same resolved building. Some remaining deficits (Stalker -2, Hellion -2, Reaper -1) may need:
- **Wider tolerance**: increase from 10 to 20-30 loops for races with different event timing
- **Cross-building detection**: pairs might route to different buildings if shortestQueue distributes them. Track recent intents per unit type globally, not per building
- **Protoss Warp Gate pairs**: Stalkers from Warp Gates produce differently than Reactor pairs but look similar (simultaneous warp-ins)

**How to test**: Run `HellionDeficitDiagnosticTest` variant for Stalker/Reaper to see their intent spacing patterns. If they show pairs within 10-30 loops, widen the tolerance. If not, it's pure queue routing — proceed to Path 2.

#### Path 2: CmdEvent-based building routing (medium — 2-4 hours)

This is the definitive fix for the "sustained throughput" deficits (Marine -7, Medivac -1, WidowMine -1) that worsen at 7-min. These units are routed to wrong buildings by `shortestQueue`, and no pair detection can fix this — the player simply uses specific buildings in a specific order.

**The existing infrastructure** supports this. Here's the implementation plan:

##### Step 1: Enhance TrackerIntentExtractor to correlate CmdEvents with UnitBorn

`TrackerIntentExtractor.java` already parses game events for Chrono Boost (lines 99-113). Extend this to track the current building selection and correlate train CmdEvents with tracker UnitBorn events.

**Existing pattern to follow:** `ReplayCommandExtractor.java` (lines 51-76) shows the full CmdEvent + SelectionDelta processing pipeline:
```java
for (Event raw : events) {
    if (raw instanceof SelectionDeltaEvent sel) {
        mapping.onSelection(sel);          // tracks which building is selected
    } else if (raw instanceof CmdEvent cmd) {
        for (ReplayCommand rc : mapping.process(cmd)) {
            // rc tells you what the player did and on which building
        }
    }
}
```

`AbilityMapping.java` (line 391+) already handles SelectionDelta state accumulation — use it directly, don't reimplement.

##### Step 2: Build a CmdEvent→UnitBorn correlation map

For each player, iterate game events and build a map:
```
Map<CorrelationKey, String> cmdBuildingTag
```
Where `CorrelationKey` = (unitType, commandLoop ±tolerance) and the value is the selected building tag from the CmdEvent's selection state.

The correlation logic:
1. When a CmdEvent fires with an abilLink that maps to a train command (use `AbilityMapping.process()` which returns `ReplayCommand.IntentCommand` for train commands)
2. The `AbilityMapping` tracks the current selection — `mapping.selectionSnapshotForTest().get(0)` gives the first selected building tag
3. Record: at loop L, unit type U was queued on building tag B
4. Match with tracker UnitBorn at loop L + trainTime ±10 for type U

##### Step 3: Encode the building tag in TrainIntent

Currently, `TrackerIntentExtractor` creates:
```java
String tag = "r-" + ue.getUnitTagIndex() + "-" + ue.getUnitTagRecycle();
intents.add(new TimedIntent(commandLoop, new TrainIntent(tag, unitType)));
```

When a CmdEvent correlation exists, use the BUILDING tag instead:
```java
String buildingTag = cmdBuildingTag.get(new CorrelationKey(unitType, commandLoop));
if (buildingTag != null) {
    tag = buildingTag;  // exact building tag — handleTrain finds it directly
}
```

When `handleTrain` receives an intent with a real building tag (not `r-` prefix), it finds the building by exact tag match (line 365-367) and uses it directly — no `shortestQueue` needed.

##### Step 4: Handle the tag format

The CmdEvent selection tags use `GameEventStream.decodeTag(rawTag)` format. The GT buildings injected via `syncBuildings` use `Sc2ReplayShared.makeTag(unitTagIndex, unitTagRecycle)` format. These must match.

Check: `GameEventStream.decodeTag` and `Sc2ReplayShared.makeTag` — if they produce the same tag format for the same unit, this works. If not, add a conversion.

**Key file:** `GameEventStream.java` — has `decodeTag(long rawTag)` static method. Compare with `Sc2ReplayShared.makeTag(int tagIndex, int tagRecycle)`.

##### Step 5: Fallback for unmatched intents

Not all UnitBorn events will correlate with CmdEvents (timing mismatches, edge cases). Keep the existing `r-` prefix auto-resolution as fallback — it's already at 99.8% accuracy. CmdEvent routing is an enhancement, not a replacement.

##### Key files and line numbers

| File | Lines | What to modify |
|---|---|---|
| `TrackerIntentExtractor.java` | 51-76 | Add CmdEvent+SelectionDelta processing loop (parallel to tracker event loop) |
| `TrackerIntentExtractor.java` | 73-75 | Use correlated building tag when available |
| `AbilityMapping.java` | 391+ | Already handles SelectionDelta — use as-is |
| `ReplayCommandExtractor.java` | 51-76 | Reference implementation for CmdEvent processing pattern |
| `EmulatedGame.java` | 365-367 | Exact tag match path already exists — CmdEvent-routed intents use this |
| `GameEventStream.java` | `decodeTag()` | Tag format from game events |
| `Sc2ReplayShared.java` | `makeTag()` | Tag format from tracker events |

##### Testing

- Run `TrackerDrivenAccuracyBaselineTest` (`-Preport`) after each step
- Expected improvement: +5 to +15 units (Marine is the big target at -7)
- Run `HellionDeficitDiagnosticTest` (adapt for Marine) to verify per-replay improvement
- Existing unit tests should not regress — the change only affects oracle replay intents

### Approaches That Were Tried and Ruled Out

| Approach | Result | Root cause of failure |
|---|---|---|
| shortestQueue Reactor tiebreaker | Neutral | No Reactored Factories in deficit replays |
| Reactor-only candidate filtering | 98.9% (regressed) | Overloaded Reactored buildings, starved plain ones |
| Pre-tick intent application | Neutral | Not a timing-at-checkpoint issue |
| Town-hall-fallback removal | 99.4% (regressed) | `processDeferredTrainIntents` has `processingDeferred` guard that drops re-deferred intents permanently |

### Key Architecture Understanding

**Why intents arrive before buildings exist:** `commandLoop = bornLoop - trainTime` back-calculates when training should start. But the building might not exist yet in the EmulatedGame at that loop because `syncBuildings` only injects buildings after `groundTruth.tick()`. The town-hall fallback (`handleTrain` lines 380-384) catches these by routing to a CC/Nexus/Hatch. This is NECESSARY — removing it causes intents to be permanently lost via the deferred-intent overflow path.

**Why pair detection works:** Reactor pairs are born ~4 loops apart (2 slots completing simultaneously). The back-calculated commandLoops are also ~4 loops apart. The 10-loop tolerance catches this. When detected, the building is permanently marked Reactored via `reactoredBuildings.add(resolvedTag)`, so future intents on the same building also get 2 slots.

**SCV over-production (48 vs GT 33):** Starting SCVs are both seeded by `RaceModel.seedInitialState` AND generated as TrainIntents from tracker UnitBorn at loop 0. This fills the CC queue. Not a blocking issue — `Math.min(emVal, gtVal)` caps accuracy. But it means the CC has a deep queue, so any intents routed to it via town-hall fallback get delayed.

### Pre-existing Test Failures

Drools (init/NPE): `DroolsScoutingRulesTest`, `DroolsStrategyL2L3Test`, `DroolsDominanceWeightStrategyTest`, `DroolsStrategyTaskTest`, `DominanceWeightRuleUnitTest`, `PatternClassificationRuleUnitTest`, `PatternClassificationCalibrationTest`, `ReplayEngineTest`. Also `MomentDetectionTaskTest`, `GameStateRoundTripTest`.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 34 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
