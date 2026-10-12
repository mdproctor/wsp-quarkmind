# HANDOFF — quarkmind

## Last Session

Completed #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. All 3 batches complete. Waypoint assessment posted.

### Completed This Session

1. **`"r-"` prefix fix** — TrackerIntentExtractor tags weren't triggering EmulatedGame's building auto-resolution fallback. Every TrainIntent and ResearchIntent was silently rejected. Fix: prefix tags with `"r-"`. 5-min unit accuracy: 29.9% → 93.6%.

2. **MULE exclusion** — MULEs are ability-spawned (Calldown MULE), not trained units. Excluded from both extraction and GT comparison. 93.6% → 94.5%.

3. **`trainedBy` completion** — 18 missing entries added (Sentry, Dark Templar, High Templar, Warp Prism, Reaper, Ghost, Siege Tank, Widow Mine, Cyclone, Thor, Liberator, Banshee, Raven, Battlecruiser, Swarm Host, Infestor, Corruptor, Viper). Correct building routing but no net accuracy change at 5-min due to Factory queue contention without Reactor modeling.

4. **Gap analysis** — Identified remaining 0.5% gap (32 units to 95%): Reactor add-on modeling (~15), mineral timing races (~10), Chrono Boost back-calculation (~5), larva timing (~5).

### Final Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 94.3% | 100% | 100% |
| 3-min | 95.7% | 100% | 25.0% |
| 5-min | 94.5% | 100% | 49.5% |
| 7-min | 93.5% | 100% | 69.7% |

### Key Finding

The tracker-driven path validates EmulatedGame's physics engine. 94.5% unit accuracy at 5-min with only 2 code fixes proves the physics model is sound. The remaining gap is distributed across many small causes, none individually blocking.

### What Remains

Issue recommends closing M2 and filing targeted issues for remaining gaps (Reactor add-ons, Chrono Boost timing, intent deferral) if they become blocking downstream.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 8 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
- Soredium: #442 filed (work-end queue stall bug)

### Plan

`plans/2026-10-12-tracker-emulated-physics.md` — 6 tasks in 3 batches. All complete.
