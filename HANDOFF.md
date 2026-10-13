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

| Type | GT | Em | Delta | 7-min trend |
|---|---|---|---|---|
| MARINE | 372 | 365 | -7 | -5 at 7-min (worsening) |
| HELLION | 52 | 50 | -2 | +2 over-production at 7-min |
| STALKER | 51 | 49 | -2 | Recovers to 100% at 7-min |
| MEDIVAC | 9 | 8 | -1 | -4 at 7-min (worsening) |
| REAPER | 36 | 35 | -1 | Recovers to 100% at 7-min |
| WIDOW_MINE | 28 | 27 | -1 | -4 at 7-min (worsening) |

### What We Now Know

The remaining deficits fall into two categories:
1. **Reactor timing gaps** (Stalker, remaining Hellion, Reaper): Same root cause as the fixed Hellions — Reactor pair detection may need tuning, or these have slightly different timing patterns.
2. **Sustained throughput loss** (Marine, Medivac, WidowMine): These worsen at 7-min. The issue is `shortestQueue` distributing intents to different buildings than the real player chose. CmdEvent-based routing (Approach B) is the only remaining lever.

### Approach B: CmdEvent Selection Routing (2-4 hours)

Oracle replays have 100% game events. CmdEvents include building selection state. Cross-reference tracker UnitBorn with nearby CmdEvents to determine which building the player selected, then route the intent to that specific building.

### Key Code Locations

| File | What it does |
|---|---|
| `EmulatedGame.java:handleTrain` | Auto-resolution, Reactor pair detection, queue routing |
| `EmulatedGame.java:shortestQueue` | Capacity-aware building selection |
| `ReplayValidationHarness.java:syncBuildings` | GT building injection and Reactor marking |
| `TrackerIntentExtractor.java` | Intent generation from tracker events |

### Pre-existing Test Failures

Drools (init/NPE): `DroolsScoutingRulesTest`, `DroolsStrategyL2L3Test`, `DroolsDominanceWeightStrategyTest`, `DroolsStrategyTaskTest`, `DominanceWeightRuleUnitTest`, `PatternClassificationRuleUnitTest`, `PatternClassificationCalibrationTest`, `ReplayEngineTest`. Also `MomentDetectionTaskTest`, `GameStateRoundTripTest`.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 34 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
