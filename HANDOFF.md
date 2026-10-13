# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. 22 commits ahead of main. Pushed unit accuracy from 99.4% to 99.5% through per-building Reactor detection, capacity-aware queue routing, and Chrono Boost extraction from game events.

### Completed This Session

1. **First-principles analysis of the gap** — The previous session's HANDOFF claimed side-by-side SC2 API was required for all further progress. Analysis proved 3 of 4 blockers were solvable from replay data alone:
   - Per-building Reactor: UnitInit position correlation (not SC2 `add_on_tag`)
   - Chrono Boost: game events in oracle replays (not SC2 `buff_ids`)
   - MULE/OC: misdiagnosis — MULE uses energy, not the production queue

2. **Per-building Reactor detection** (`63197567`) — Replaced type-level Reactor detection (ALL buildings of a type marked reactored) with per-building detection. When a Reactor UnitInit event arrives, `findNearestProductionBuilding` finds the parent building by position from a `productionBuildings` map that excludes add-ons. More correct but revealed that type-level detection was accidentally compensating for routing imprecision.

3. **Capacity-aware queue routing** (`63197567`) — Updated `shortestQueue` to use `depth = active + queue - maxSlots` (signed). Reactored buildings (maxSlots=2) have negative depth when slots are free, so they're preferred. Achieves proportional intent distribution (2:1:1 for 3 Barracks with 1 Reactor).

4. **Chrono Boost abilLink discovery** (`62c8337f`) — `ChronoBoostAbilLinkDiscoveryTest` found abilLink=716 (444 occurrences across 65 Protoss player-replays). Added to `TrackerIntentExtractor`.

5. **Chrono Boost extraction from game events** (`62c8337f`) — `TrackerIntentExtractor` now parses `RepContent.GAME_EVENTS` alongside tracker events. Chrono Boost CmdEvents (abilLink=716) are converted to `AbilityIntent("r-NEXUS", "CHRONO_BOOST", decodedTargetTag)`. Fixed `resolveBuilding` to try direct tag match before type-based resolution. Protoss gains at 5-min: Probe +1, Zealot +1, Stalker +1, Observer +1.

### Current Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 100.0% | 100% | 100% |
| 3-min | 99.5% | 100% | 75.0% |
| 5-min | **99.5%** | 100% | **98.1%** |
| 7-min | 99.7% | 100% | **99.5%** |

### Session Progression

| Step | 5-min units | Change |
|---|---|---|
| Start of session | 99.4% | — |
| Per-building Reactor + capacity routing | 99.4% | neutral (more correct, same headline) |
| Chrono Boost extraction | 99.5% | +0.1pp |

---

## Next Session: Closing the Last 0.5% to 100%

### Goal: 100% unit, building, and upgrade accuracy at all checkpoints

### The Remaining Gap — 36 Missing Units at 5-min

| Type | GT | Em | Delta | Root Cause | Confirmed By |
|---|---|---|---|---|---|
| HELLION | 52 | 39 | -13 | Queue timing — all catch up by 7-min (100%) | 7-min shows 0 delta |
| WIDOW_MINE | 28 | 20 | -8 | Same as Hellion — Factory queue timing | 7-min still -8 (slower recovery) |
| MARINE | 372 | 366 | -6 | Barracks routing imprecision | Per-building Reactor worsened from -5 to -6 |
| PROBE | 1421 | 1417 | -4 | Insufficient Chrono Boost — only +1 of expected +5 | Energy model may reject some casts |
| PHOENIX | 8 | 6 | -2 | Stargate queue timing | Small count magnifies delta |
| MEDIVAC | 9 | 8 | -1 | Starport timing | Catches up by 7-min |
| STALKER | 51 | 50 | -1 | Gateway timing | Catches up by 7-min |
| REAPER | 36 | 35 | -1 | Barracks timing | Catches up by 7-min |

### The Remaining Gap — Upgrades

| Checkpoint | Accuracy | Missing |
|---|---|---|
| 3-min | 75.0% | 1 of 4 (TunnelingClaws) |
| 5-min | 98.1% | 2 of 103 |
| 7-min | 99.5% | 1 of 185 |

### Three Categories of Gap

**Category A: Timing at checkpoint (24 units — Hellion, Medivac, Stalker, Reaper, some Marine)**
Units are trained but complete 1-2 ticks AFTER the checkpoint. Confirmed by 7-min showing 100% for these types. The EmulatedGame trains them at the right speed, but queue entry is delayed by routing heuristics.

**Category B: Sustained throughput (8 units — WidowMine, some Marine)**
Units never fully catch up. The EmulatedGame's throughput is genuinely slower than the real game for these types, likely because the wrong building is processing the intent.

**Category C: Ability coverage (4 units — Probe)**
Chrono Boost extraction works (abilLink=716) but only recovered 1 of 5 expected Probes. The Nexus energy model is likely rejecting some casts.

### Approaches to 100%

#### Approach 1: Calibrate Terran Train Times (Quick Win — Category A)

**Impact estimate: 10-15 units** — the biggest bang for the buck.

Most Terran unit train times are uncalibrated (all set to `672 = 30s × 22.4`). The real times are shorter:

| Unit | Current (loops) | Expected (loops) | Source |
|---|---|---|---|
| HELLION | 672 | ~470 (21s) | SC2 game data |
| WIDOW_MINE | 672 | ~470 (21s) | SC2 game data |
| REAPER | 672 | ~1008 (45s) | SC2 game data |
| MEDIVAC | 672 | ~672 (30s) | Already correct |
| SIEGE_TANK | 672 | ~717 (32s) | SC2 game data |
| VIKING | 672 | ~672 (30s) | Already correct |

**Why this helps even though back-calculation cancels out:** The back-calculated `commandLoop = bornLoop - trainTime` changes when `trainTime` changes. With the correct (shorter) time, intents arrive LATER — closer to their completion — which reduces the time they spend in the queue. Less queue time = fewer ticks of delay = more units complete before the checkpoint.

**How to calibrate:**
1. Run `SC2TrainTimeCalibrationTest` — check if it covers Terran (may need oracle replays with Terran games)
2. If the test doesn't cover Terran, derive times from the SC2 game data files (Liquipedia has canonical values for LotV patch 4.9.3)
3. Update `SC2Data.UNIT_TRAIN_TIMES` with calibrated values
4. Re-run accuracy baseline

#### Approach 2: Fix Chrono Boost Energy Budget (Quick Win — Category C)

**Impact estimate: 3-4 units (Probes)**

The Chrono Boost extraction works but most casts are rejected by the energy model. Diagnosis needed:

1. **Add diagnostic counters** to `ProtossRaceModel.handleAbility()` — count accepted vs rejected Chrono Boosts and the rejection reason (insufficient energy)
2. **Check Nexus energy regen rate** — `SC2Data.CASTER_ENERGY_REGEN_PER_LOOP` might be wrong
3. **Check starting energy** — `SC2Data.NEXUS_STARTING_ENERGY` (currently 50.0) — correct for LotV
4. **Check multi-Nexus routing** — the `r-NEXUS` caster resolution uses `resolveBuildingAll` which tries each Nexus until one has enough energy. If the first Nexus always gets tried first, energy distribution is uneven. Consider round-robin or energy-ordered selection.
5. **Skip energy check for tracker-driven Chrono Boost** — same principle as the economy bypass: if the CmdEvent exists, the real player DID cast it. Energy checks measure economy accuracy, not physics accuracy. For tracker-driven intents, bypass the energy check and always apply the speed multiplier.

Option 5 is the most consistent with the existing philosophy (tracker intents bypass resource checks). The `AbilityIntent` from game events should be treated the same way as `TrainIntent` from tracker events — the real player did it, don't second-guess.

**Implementation:** In `applyAbilityResolved`, when the AbilityIntent has `r-` prefix tags, skip the energy check in `model.handleAbility` — apply the Chrono Boost unconditionally. May need a new method like `model.forceAbility(state, casterTag, ability, targetTag, gameLoop)`.

#### Approach 3: Queue Routing from Game Events (Medium — Category B)

**Impact estimate: 5-8 units (WidowMine, Marine)**

CmdEvents contain the selected unit when the player issued a train command. By tracking `SelectionDeltaEvent`s (which AbilityMapping already does), we can determine which specific building the player had selected when they issued each train command.

**How:**
1. In `TrackerIntentExtractor`, add `SelectionDeltaEvent` tracking (same pattern as `AbilityMapping.onSelection`)
2. When extracting train intents from tracker events, cross-reference with any CmdEvent within a ±5 loop window to find which building was selected
3. Encode the building tag in the `TrainIntent` instead of using `r-` auto-resolution
4. EmulatedGame routes to the exact building

**Challenge:** Only ~43% of CmdEvents are captured in stripped replays, but oracle replays have 100% game events. For oracle replays this should work. For non-oracle replays, fall back to `r-` auto-resolution.

**Complexity:** Medium — requires correlating tracker events with game events by timing, and building a selection-state tracker into the extractor.

#### Approach 4: Uncalibrated Terran Build Times (Medium — Category A)

If Approach 1 (train time calibration) doesn't fully close the gap, also calibrate **building** completion times. `SC2BuildTimeCalibrationTest` exists but may not cover all building types. Incorrect building completion times delay when the building is available for production, cascading into late unit births.

#### Approach 5: Side-by-Side SC2 Calibration (Heavy — last resort)

The previous HANDOFF's approach. Only needed for the final ~5-8 units that require exact per-building queue state from the SC2 observation API. **Do not attempt until Approaches 1-4 are exhausted.** The infrastructure exists (`QuarkusSC2Transport`, `ObservationTranslator`, `SC2BotAgent`) but requires SC2 running locally.

### Recommended Order

1. **Approach 2** (Chrono Boost energy bypass) — 15 minutes, +3-4 Probes
2. **Approach 1** (Terran train time calibration) — 30-60 minutes, +10-15 units
3. Re-assess. If < 100%, then:
4. **Approach 3** (Queue routing from game events) — 2-4 hours, +5-8 units
5. **Approach 5** (SC2 side-by-side) only if genuinely stuck

### Key Code Locations

| File | What to read |
|---|---|
| `EmulatedGame.java:shortestQueue` | Capacity-aware routing — `depth = active + queue - maxSlots` |
| `EmulatedGame.java:resolveBuilding` | Direct tag match then type-based resolution for r- prefix tags |
| `EmulatedGame.java:applyAbilityResolved` | Chrono Boost dispatch — caster resolution, energy check, speed multiplier |
| `ProtossRaceModel.java:handleAbility` | Chrono Boost energy model — deducts 50 energy, sets chronoBoostedUntil |
| `ProtossRaceModel.java:trainingSpeedMultiplier` | Returns CHRONO_BOOST_MULTIPLIER (0.5) when boost is active |
| `ReplaySimulatedGame.java:applyUnitInit` | Per-building Reactor via position correlation + productionBuildings map |
| `ReplayValidationHarness.java:syncBuildings` | Uses reactoredBuildingTags (per-building) not reactoredBuildingTypes |
| `TrackerIntentExtractor.java` | Extracts train/research from tracker events + Chrono Boost from game events |
| `ChronoBoostAbilLinkDiscoveryTest.java` | Diagnostic: abilLink=716 discovery from 65 Protoss replays |
| `SC2Data.java:UNIT_TRAIN_TIMES` | Terran entries mostly uncalibrated at 672 (30s × 22.4) |
| `TrackerDrivenAccuracyBaselineTest.java` | Accuracy test — 99.4% threshold, writes docs/benchmarks/ |
| `TrackerTrainDiagnosticTest.java` | Per-unit-type rejection breakdown diagnostic |

### Pre-existing Test Failures

Drools tests are failing (pre-existing, not caused by this branch): `DroolsScoutingRulesTest`, `DroolsStrategyL2L3Test`, `DroolsDominanceWeightStrategyTest`, `DroolsStrategyTaskTest`, `DominanceWeightRuleUnitTest`, `PatternClassificationRuleUnitTest`, `PatternClassificationCalibrationTest`, `ReplayEngineTest`. All are Drools initialization or NullPointerException failures. Not related to tracker/emulated physics work.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 22 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
