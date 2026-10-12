# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. 12 commits ahead of main. Pushed upgrade accuracy from 49.5% to 98.1% at 5-min (was the biggest gap). Unit accuracy unchanged at 94.9% — that is the remaining target.

### Completed This Session

1. **Type-aware research building resolution** (49.5% → 81.6% upgrades at 5-min) — Added `SC2Data.researchedAt()` mapping all 82 UpgradeType entries to their research BuildingType. Changed `handleResearch` fallback from "pick ANY idle building" to "pick idle building of CORRECT type via `canTrainFrom()`". Added multi-tick deferral for ResearchIntents (parallel to existing TrainIntent deferral) — failed research intents retry every tick until their building frees up.

2. **Original command loop timing** (81.6% → 98.1% upgrades at 5-min) — Added `commandLoop` field to `ResearchIntent` record (backward-compatible 2-arg constructor). Tracker-derived intents carry their back-calculated command loop, so deferred retries compute completion from the original time, not the retry tick. Eliminated timing drift at checkpoint boundaries.

### Current Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 94.9% | 100% | 100% |
| 3-min | 96.4% | 100% | 75.0% |
| 5-min | **94.9%** | 100% | **98.1%** |
| 7-min | 93.7% | 100% | **99.5%** |

### What Changed vs Last Session

| Dimension | 5-min before | 5-min after | Change |
|---|---|---|---|
| Units | 94.9% | 94.9% | unchanged |
| Buildings | 100% | 100% | unchanged |
| Upgrades | 49.5% | 98.1% | **+48.6pp** |

---

## Next Session: Push Unit Accuracy from 94.9% to Target

### The Gap: 344 units missed across 236 player runs at 5-min

| Unit Type | GT | Emulated | Delta | Accuracy | Root Cause | Fix Strategy |
|---|---|---|---|---|---|---|
| DRONE | 1724 | 1630 | -94 | 94.5% | Gas depletion within ticks; larva mechanics | See Zerg Worker section below |
| PROBE | 1421 | 1373 | -48 | 96.6% | Chrono Boost back-calculation offset | See Protoss Worker section below |
| QUEEN | 145 | 98 | -47 | 67.6% | Only 37% CmdEvent capture × 44% acceptance = 17% effective rate (from #386 diagnostic) | Queens are spawned by ability, not trained — needs ability-based extraction from tracker |
| ROACH | 65 | 35 | -30 | 53.8% | Insufficient vespene at command time; gas-heavy unit | Vespene sync timing — currently uses pre-tick GT vespene, but Roach commands may need within-tick gas |
| SCV | 1850 | 1824 | -26 | 98.6% | MULE calldown consuming CC training slots; no Reactor modeling | Check if OC vs CC distinction matters for training queue |
| REAPER | 36 | 21 | -15 | 58.3% | Missing Reactor modeling — Barracks with Reactor trains 2 at once | Reactor add-on modeling (see below) |
| ORACLE | 9 | 4 | -5 | 44.4% | Stargate contention (multiple Stargates, wrong one selected) | Better building selection for multi-building scenarios |
| WIDOW_MINE | 28 | 20 | -8 | 71.4% | Factory Reactor modeling | Same as Reaper — Reactor doubles throughput |
| SIEGE_TANK | 33 | 26 | -7 | 78.8% | Factory Tech Lab contention | Building queue depth check |
| HELLION | 52 | 44 | -8 | 84.6% | Factory Reactor modeling | Same as Reaper/Widow Mine |

### Root Cause Analysis — Ordered by Impact

#### 1. Zerg Workers (DRONE -94, QUEEN -47) — 141 units = 41% of total gap

**DRONE:** The `syncReplayBuilding` method calls `onBuildingComplete` every tick for complete buildings, which resets `hatcheryLarvaCount` to `MAX_LARVA`. Larva is never depleted in the harness, so the constraint on Drone production is purely economic (minerals/vespene), not biological. The 94 missed Drones are likely caused by:
- Gas depletion: Drones cost 50 minerals + 1 larva. If minerals are available but the larva spawn timing is wrong (always reset to max), training commands may succeed when they shouldn't, consuming minerals that should be available for other units. OR the opposite — Drones may be rejected due to the tracker-driven tag resolution, not larva.
- **Diagnostic step:** Add per-rejection-reason logging to `handleTrain` for `r-` prefix intents. Count "building not found", "building busy", "insufficient resources", "supply blocked" per unit type across all 118 replays. This tells us WHERE in the pipeline Drones are lost.

**QUEEN:** Queens are spawned via a Hatchery ability (Train Queen), not a standard train. The tracker shows them as `UnitBorn` events. TrackerIntentExtractor creates `TrainIntent("r-tag", QUEEN)` which routes to `handleTrain`. Queens require a specific Hatchery (each Hatch can only train one Queen at a time in SC2). The 67.6% accuracy suggests the building selection or queue-depth constraints are wrong.
- **Diagnostic step:** Check if multiple Queens are being assigned to the same Hatchery. In SC2, each Hatchery can only produce one Queen at a time (no queue). If the harness assigns 3 Queen TrainIntents to the same Hatch, 2 will fail.

#### 2. Terran Reactor Units (REAPER -15, WIDOW_MINE -8, HELLION -8) — 31 units = 9% of total gap

Reactors allow Barracks/Factory/Starport to train TWO units simultaneously. Without Reactor modeling, the building is marked busy after the first unit starts and the second is rejected. The harness has no concept of add-ons.

**Fix approach — simplest viable:**
- Don't model Reactors explicitly. Instead, for `r-` prefix TrainIntents where `trainedBy` returns BARRACKS/FACTORY/STARPORT, allow the building to have TWO concurrent training slots instead of one.
- This is an approximation — in real SC2, only buildings WITH a Reactor attachment get double slots. But since we're using GT intents (100% input coverage), the intent would only fire if the real player HAD a Reactor. So doubling all slots for tracker-driven intents is safe.
- Implementation: in `handleTrain`, when checking if a building is busy, for `r-` prefix intents check if the building has < 2 active training slots (instead of < 1).

#### 3. Protoss Workers (PROBE -48) — 14% of total gap

Probes are trained at the Nexus. 96.6% accuracy is already high. The 48 missed Probes could be:
- **Chrono Boost:** Protoss can Chrono Boost the Nexus to speed up Probe production. The back-calculated command loop uses the STANDARD train time, but Chrono-Boosted Probes complete faster. This means the intent fires TOO LATE — the GT Probe was born earlier than `completionLoop - standardTrainTime`.
- **Diagnostic step:** Compare the back-calculated command loop against the actual time window. If Probes are consistently born EARLIER than `commandLoop + trainTime`, Chrono Boost offset is confirmed.
- **Fix:** Detect Chrono Boost from tracker events (`UnitBorn` for Chrono Boost ability?) and adjust the train time accordingly. OR use a simpler heuristic: for Protoss workers, use a shorter back-calculation window.

#### 4. Gas-Heavy Units (ROACH -30) — 9% of total gap

Roach costs 75 minerals + 25 gas. At 53.8% accuracy, nearly half fail. The harness syncs vespene from pre-tick GT (`emulated.setVespeneForHarness(gtBefore.vespene())`), but multiple Roach intents in the same tick window may overdraw gas. Each Roach deducts 25 gas, but the GT vespene is only synced once per tick.
- **Diagnostic step:** Log vespene balance when Roach TrainIntents are rejected. If gas is the bottleneck, confirm whether per-intent gas deduction matches GT spending patterns.

### Recommended Session Plan

**Phase 1 — Diagnostics (30 min)**
Add per-rejection-reason logging to `handleTrain` for `r-` prefix intents. Run TrackerDrivenAccuracyBaselineTest and capture a breakdown: for each unit type that misses, count how many were rejected for "building not found", "building busy", "insufficient resources", "supply blocked". This replaces speculation with data.

**Phase 2 — Reactor Modeling (1 hr)**
Implement double training slots for `r-` prefix intents at Barracks/Factory/Starport. This is the simplest fix with the clearest impact: 31 units (Reaper+Hellion+Widow Mine) should recover. Test against the 118 replays.

**Phase 3 — Queen/Drone Larva (1 hr)**
Fix `syncReplayBuilding` larva reset. Queens need per-Hatchery queue depth. Drones need correct larva consumption. These are the largest single gaps.

**Phase 4 — Chrono Boost and Vespene (time permitting)**
Probe timing offset and Roach gas depletion. These are smaller gains but require more investigation.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 12 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
- Soredium: #442 filed (work-end queue stall bug)

### Pre-existing Test Failures

Drools tests are failing (pre-existing, not caused by this branch): `DroolsScoutingRulesTest` (13 errors), `DroolsStrategyL2L3Test` (7 errors), `DroolsDominanceWeightStrategyTest` (5 errors), `DroolsStrategyTaskTest` (22 errors), `ReplayEngineTest` (2 errors). All are `NullPointerException` or Drools initialization failures. Not related to tracker/emulated physics work.

### Key Code Locations

| File | What to read |
|---|---|
| `EmulatedGame.java:handleTrain` | Train intent processing — add diagnostic logging here |
| `EmulatedGame.java:handleResearch` | Research intent processing (just fixed) |
| `EmulatedGame.java:deferResearchIntent` / `processDeferredResearchIntents` | Multi-tick research deferral (new) |
| `SC2Data.java:researchedAt` | UpgradeType → BuildingType mapping (new, line ~1014) |
| `SC2Data.java:trainedBy` | UnitType → BuildingType mapping (existing) |
| `ReplayValidationHarness.java:syncBuildings` | Building sync from GT — larva reset issue is here |
| `TrackerIntentExtractor.java` | Tracker event → intent conversion |
| `ResearchIntent.java` | Now has 3-arg constructor with `commandLoop` field |
| `TrackerDrivenAccuracyBaselineTest.java` | The test to run after each fix |

### Plan

`plans/2026-10-12-tracker-emulated-physics.md` — original 6 tasks in 3 batches all complete. Next session should extend the plan with new tasks for the unit accuracy push.
