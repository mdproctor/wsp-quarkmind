# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. All 3 batches complete. Pushed from 29.9% to 94.9% unit accuracy at 5-min. Waypoint assessment posted. Next session: iterate to 100% across all dimensions.

### Completed This Session

1. **`"r-"` prefix fix** (29.9% → 93.6%) — TrackerIntentExtractor tags weren't triggering EmulatedGame's building auto-resolution fallback. Every TrainIntent and ResearchIntent was silently rejected. Fix: prefix tags with `"r-"`.

2. **MULE exclusion** (93.6% → 94.5%) — MULEs are ability-spawned (Calldown MULE), not trained units. Excluded from both extraction and GT comparison.

3. **`trainedBy` completion** (94.5% → 94.5%) — 18 missing entries added (Sentry, Dark Templar, High Templar, Warp Prism, Reaper, Ghost, Siege Tank, Widow Mine, Cyclone, Thor, Liberator, Banshee, Raven, Battlecruiser, Swarm Host, Infestor, Corruptor, Viper). Correct building routing but neutral net accuracy at 5-min.

4. **Intent deferral + autoResolved type check skip** (94.5% → 94.9%) — When building not found, production blocked, or insufficient resources for `r-` prefix intents, defer and retry next tick. Also skip post-resolution type check for auto-resolved buildings (was causing false rejections for town-hall-fallback units). Multi-tick retry tested and rejected — causes cascading mineral depletion.

### Final Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 94.9% | 100% | 100% |
| 3-min | 96.4% | 100% | 25.0% |
| 5-min | **94.9%** | 100% | 49.5% |
| 7-min | 93.7% | 100% | 69.7% |

### Key Findings

1. **Tag format was the big unlock** — a 2-character prefix fix (`"r-"`) took units from 29.9% to 93.6%. The physics engine was sound; intents were being silently rejected.
2. **Economy model is correct** — EmulatedGame has MORE minerals than GT (219% at 5-min). The issue is intent timing, not income.
3. **Multi-tick deferral is counterproductive** — retrying deferred intents across multiple ticks causes cascading mineral depletion as late-succeeding intents consume resources needed by later legitimate intents.
4. **syncReplayBuilding resets Zerg larva every tick** — `onBuildingComplete` is called every tick for complete buildings, resetting `hatcheryLarvaCount` to MAX_LARVA. Larva is never depleted in the harness.

### Next Session: Push to 100% Across All Dimensions

**Units (94.9% → target 100%):**
- Remaining gap is 344 units across 236 player runs. Distributed: Drone 91, Probe 53, Queen 49, SCV 33, Roach 29, Reaper 15, Hellion 8, Widow Mine 8, Siege Tank 7
- Root causes: gas depletion within ticks (Roach), supply timing, building queue contention without Reactor add-on modeling, Chrono Boost back-calculation offset
- Approaches to try: Reactor add-on modeling, Chrono Boost detection from tracker events, improved vespene handling, diagnostic logging per rejection reason

**Upgrades (49.5% at 5-min → target 100%):**
- ResearchIntent uses `"r-tracker"` — the fallback picks ANY idle complete building (not the correct type). Need type-aware research building resolution.
- zerglingmovementspeed at 25.7% (35 GT, 9 matched) — likely Spawning Pool not complete at back-calculated command time

**Economy (mineralsCollectionRate ~92% error → target correct):**
- Reporting units mismatch: EmulatedGame reports raw delta, SC2 reports minerals/minute. Need to align units.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 10 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
- Soredium: #442 filed (work-end queue stall bug)

### Plan

`plans/2026-10-12-tracker-emulated-physics.md` — 6 tasks in 3 batches. All complete. Next session extends with new tasks for 100% targets.
