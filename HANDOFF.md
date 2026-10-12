# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. Batches 1-2 complete (TrackerIntentExtractor + baseline test). Batch 3 (physics calibration) remaining.

### Completed This Session

1. **Recovered stale .plan** — #386 close left the plan in `closing:stamped` with old branch reference. Root cause filed as Hortora/soredium#442 (3 independent failures in work-end pipeline). Manually advanced queue to #406.

2. **TrackerIntentExtractor** — new class in `io.quarkmind.sc2.replay` that converts oracle tracker events into `TimedIntent`s:
   - `UnitBorn` → `TrainIntent` with back-calculated command loop (completion loop - train time)
   - `Upgrade` → `ResearchIntent` via `UpgradeType.fromTrackerName()` (new alias for `fromPythonName`)
   - Buildings are NOT converted to intents — the harness syncs them from ground truth
   - Tracker path: 240 TrainIntents vs CmdEvent: 42 (471% more coverage on a single replay)

3. **TrackerDrivenAccuracyBaselineTest** — `@Tag("report")` test across 118 oracle replays (236 player runs):
   - 1-min: 74.8% units, 100% buildings
   - 3-min: 47.2% units, 100% buildings
   - 5-min: 29.9% units, 100% buildings
   - 7-min: 21.4% units, 100% buildings
   - Unit accuracy degrades over time — economy model can't sustain 100% production load
   - Report: `docs/benchmarks/tracker-driven-accuracy-baseline.md`

4. **Made `Sc2ReplayShared` public** — class and `BUILDING_NAMES`, `toUnitType`, `toBuildingType` promoted to public for cross-package access.

### Key Finding

With 100% input coverage, the **economy model is the bottleneck**, not missing inputs. EmulatedGame's mineral income can't sustain the full production load from tracker intents — units get rejected due to insufficient minerals, and accuracy drops from 75% at 1-min to 21% at 7-min. Fixing the economy model (mineral income rates, worker saturation curves) is the critical path for Batch 3.

### Tracker vs CmdEvent Baseline Comparison (5-min)

| Metric | CmdEvent (43% inputs) | Tracker (100% inputs) |
|--------|----------------------|----------------------|
| Units | 76.7% | 29.9% |
| Buildings | 100% | 100% |

Tracker is LOWER than CmdEvent because: with fewer intents, EmulatedGame spends fewer minerals and stays closer to ground truth counts. With 100% intents, it overspends and runs out of minerals, blocking subsequent production.

### What Remains — Batch 3 (Physics Calibration)

1. **Task 5: Gap analysis** — compare tracker vs CmdEvent baselines per-type, identify where EmulatedGame's economy model diverges, post analysis to #406
2. **Task 6: Fix physics gaps** — iterative TDD cycle per gap: mineral income calibration, worker saturation curves, race-specific mechanics (Chrono Boost, MULE, Inject Larva)

### Plan

`plans/2026-10-12-tracker-emulated-physics.md` — 6 tasks in 3 batches. Tasks 1-4 complete.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 5 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
- Soredium: #442 filed (work-end queue stall bug)
