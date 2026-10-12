# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. 26 commits ahead of main. Pushed unit accuracy from 99.4% to 99.6% through Chrono Boost energy bypass, Terran train time calibration, and Phoenix calibration.

### Completed This Session (second session on #406)

1. **Chrono Boost energy bypass** (`2ec59731`) — Added `RaceModel.forceAbility()` that applies ability effects without resource checks. `ProtossRaceModel.forceAbility()` sets `chronoBoostedUntil` directly, skipping energy validation. Used in `applyAbilityResolved` when caster has `r-` prefix. No accuracy change — energy wasn't the bottleneck; Chrono Boosts were already being applied successfully.

2. **Terran train time calibration** (`639ce9b0`) — Replaced uncalibrated 672-loop default (30s) with LotV 4.9.3 values: Hellion/Hellbat/WidowMine 470 (21s), Reaper/SiegeTank/Cyclone 717 (32s), Ghost 650 (29s), Thor/Raven/Banshee/Liberator 963 (43s), Battlecruiser 1434 (64s). **Hellion +6 (39→45), WidowMine +5 (20→25). 99.5% → 99.6%.**

3. **Phoenix calibration** (`c1713bcc`) — Phoenix 672→560 (25s). +1 Phoenix.

4. **Previous session work (preserved):** Per-building Reactor detection (`63197567`), capacity-aware queue routing (`63197567`), Chrono Boost extraction from game events (`62c8337f`).

### Current Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 100.0% | 100% | 100% |
| 3-min | 99.6% | 100% | 75.0% |
| 5-min | **99.6%** | 100% | **98.1%** |
| 7-min | 99.8% | 100% | **99.5%** |

### Cumulative Progression (across both sessions)

| Step | 5-min units | Missing | Change |
|---|---|---|---|
| Start (Milestone 2 beginning) | 94.9% | ~350 | — |
| Economy bypass | 99.2% | ~55 | +4.3pp |
| Reactor dual-training | 99.4% | ~39 | +0.2pp |
| Per-building Reactor + capacity routing | 99.4% | 36 | neutral (more correct) |
| Chrono Boost extraction | 99.5% | 36 | +0.1pp |
| Terran train time calibration | 99.6% | 26 | +0.1pp |
| Phoenix calibration | 99.6% | 25 | +1 unit |

---

## Next Session: 99.6% → 100%

### The Remaining 25 Missing Units at 5-min

| Type | GT | Em | Delta | Catches up by 7-min? | Root cause |
|---|---|---|---|---|---|
| HELLION | 52 | 45 | -7 | Yes (100% at 7-min) | Queue timing — intents route to wrong Factory, delay at checkpoint |
| MARINE | 372 | 366 | -6 | Partially (-5 at 7-min) | Barracks routing — some Marines go to non-Reactored Barracks |
| PROBE | 1421 | 1417 | -4 | Yes (100% at 7-min) | Chrono timing — boost applied but inter-Probe queue spacing differs |
| WIDOW_MINE | 28 | 25 | -3 | No (-4 at 7-min) | Sustained throughput loss — wrong Factory assignment |
| PHOENIX | 8 | 7 | -1 | Yes | Stargate timing at checkpoint |
| MEDIVAC | 9 | 8 | -1 | No (-4 at 7-min) | Starport throughput — gets worse at 7-min, not better |
| STALKER | 51 | 50 | -1 | Yes | Gateway timing |
| REAPER | 36 | 35 | -1 | Yes | Barracks timing |

### Two Categories

**Category A — Timing at checkpoint (16 units):** Hellion, Probe, Phoenix, Stalker, Reaper. These units ARE trained — they complete 1-2 ticks after the 5-min cutoff and show 100% by 7-min. These are the easiest wins since they need only minor timing adjustment.

**Category B — Sustained throughput (9 units):** Marine (partially), WidowMine, Medivac. These don't catch up by 7-min. The EmulatedGame genuinely produces fewer because intents are routed to the wrong building. The real player sent commands to specific buildings; our heuristic distributes differently.

### What's Been Tried and Didn't Work

- **Chrono Boost energy bypass** — Energy wasn't the bottleneck. All casts were already succeeding.
- **Type-level Reactor** — Over-counted (marked all buildings of a type), but accidentally compensated for routing imprecision. Per-building is more correct but slightly worse on Marines.

### Approaches to Close the Gap

#### Approach A: Medivac Train Time Investigation (Quick — Category B)

**Impact estimate: 1-4 Medivac**

Medivac shows -1 at 5-min but -4 at 7-min — WORSENING over time. This suggests the train time is WRONG in the opposite direction: currently 672 but real may be LONGER (Liquipedia shows 30s = 672 for Medivac, which should be correct). Alternatively, the issue may be that Medivac's production building (Starport) has other units competing for queue time (Liberator, Viking).

**Diagnosis:** Run `TrackerTrainDiagnosticTest` filtered to Medivac and check rejection reasons. If Medivacs are being deferred or queued, the issue is building competition, not train time.

#### Approach B: Queue Routing from CmdEvent Selection State (Medium — Category B)

**Impact estimate: 5-9 units (Marine, WidowMine, Medivac)**

This is the main lever for Category B. CmdEvents in oracle replays tell us which building was selected when the player issued a train command. By cross-referencing tracker UnitBorn events with nearby CmdEvents:

1. Parse `SelectionDeltaEvent`s alongside CmdEvents (existing `AbilityMapping` pattern)
2. When a tracker UnitBorn at loop N correlates with a CmdEvent at loop N-trainTime (±tolerance), use the CmdEvent's selected building tag
3. Encode the specific building tag in the `TrainIntent` for direct routing

**Key insight:** The oracle replays have 100% game events. The 43% CmdEvent capture rate only applies to stripped/restored replays.

**Complexity:** Medium — needs selection state tracking and timing correlation. Risk: CmdEvent timing might not align perfectly with tracker back-calculation.

#### Approach C: Adjust Intent Spacing for Timing-at-Checkpoint (Quick — Category A)

**Impact estimate: 5-10 units (Hellion, Probe, Phoenix)**

The back-calculated command time uses `bornLoop - trainTime`. If we use `bornLoop - trainTime + offset` where offset is a small per-unit correction (calibrated from the systematic timing error), we can nudge units to complete earlier.

**How to calibrate:** Run a diagnostic that measures, for each unit type, the average tick offset between EmulatedGame birth and ground truth birth. If Hellions consistently complete 1 tick late, subtract 1 tick from the offset.

**Risk:** This is a fudge factor — it masks the real issue (queue routing) with a timing correction. It works for 5-min checkpoint but might regress at other checkpoints.

#### Approach D: Per-Type Minimum Queue Depth (Quick — Category A + B)

Instead of `shortestQueue` distributing evenly, guarantee that at least one intent per type always goes to the building with the most capacity. This helps Factory-produced units (Hellion, WidowMine) where a single Reactored Factory should get most intents.

**Implementation:** In `handleTrain` for auto-resolved intents, if the intent's unit type is produced from a building that has a Reactor, route ALL intents of that type to the Reactored building first (until it's full), then overflow to others.

### Recommended Order

1. **Approach D** (Reactor-first routing) — 30 min, addresses Category B
2. **Approach C** (timing offset) — 30 min, addresses Category A
3. **Approach B** (CmdEvent selection routing) — 2-4 hours, ultimate fix
4. **Approach A** (Medivac investigation) — 15 min, targeted fix

### Key Code Locations

| File | What to read |
|---|---|
| `SC2Data.java:UNIT_TRAIN_TIMES` | Train times — Terran calibrated, Protoss/Zerg partially |
| `EmulatedGame.java:shortestQueue` | Capacity-aware routing: `depth = active + queue - maxSlots` |
| `EmulatedGame.java:handleTrain` | Auto-resolution for r- prefix intents, economy bypass |
| `EmulatedGame.java:applyAbilityResolved` | Chrono Boost — forceAbility for r- prefix, speed multiplier |
| `ProtossRaceModel.java` | handleAbility (energy check) + forceAbility (bypass) + trainingSpeedMultiplier |
| `RaceModel.java:forceAbility` | Default delegates to handleAbility; Protoss overrides |
| `ReplaySimulatedGame.java:applyUnitInit` | Per-building Reactor via position correlation |
| `ReplayValidationHarness.java:syncBuildings` | Uses reactoredBuildingTags (per-building) |
| `TrackerIntentExtractor.java` | Train/research from tracker + Chrono Boost from game events |
| `TrackerDrivenAccuracyBaselineTest.java` | Accuracy test — 99.4% threshold |

### Pre-existing Test Failures

Drools tests (pre-existing): `DroolsScoutingRulesTest`, `DroolsStrategyL2L3Test`, `DroolsDominanceWeightStrategyTest`, `DroolsStrategyTaskTest`, `DominanceWeightRuleUnitTest`, `PatternClassificationRuleUnitTest`, `PatternClassificationCalibrationTest`, `ReplayEngineTest`. All Drools init or NPE.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 26 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
