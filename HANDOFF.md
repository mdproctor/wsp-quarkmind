# HANDOFF — quarkmind

## Last Session

Working on #406 (Milestone 2: Tracker-event-driven EmulatedGame physics validation). Branch: `issue-406-tracker-emulated-physics`. 15 commits ahead of main. Pushed unit accuracy from 94.9% to 99.2% at 5-min by isolating physics from economy in the tracker-driven path.

### Completed This Session

1. **Diagnostic infrastructure** — Added per-rejection-reason counters to `EmulatedGame.handleTrain` for tracker-driven (r- prefix) intents. `TrackerTrainDiagnosticTest` (@Tag("diagnostic")) runs all 118 oracle replays and prints per-unit-type rejection breakdown: received, trained, building_not_found, production_blocked, insufficient_resources, queue_full, defer_overflow.

2. **Key diagnostic finding** — `insufficient_resources` is the dominant rejection reason (57%+ of all rejections), not building contention. Zero building_not_found. Mineral sync from GT made things WORSE (94.9% → 90.4%) because it removed the compensating mineral accumulation from failed intents.

3. **Economy bypass for tracker-driven intents** (94.9% → 99.2%) — First-principles insight: tracker-driven intents represent units the real player DID produce. Resource/production checks measure economy accuracy, not physics accuracy. Changed `handleTrain` to skip `canProduce`, resource checks, and resource deduction for `autoResolved` (r- prefix) intents. Non-tracker intents retain all checks.

4. **ReplayValidationHarness refactor** — All public `run` overloads now delegate to a single `runInternal` method. Added 5-param overload `run(groundTruth, intents, tickLimit, emulated, race)` for diagnostic access to EmulatedGame after the run. Added `playerRaceModel()` getter and `setMineralsForHarness()` to EmulatedGame.

### Current Results (118 oracle replays, 236 player runs)

| Checkpoint | Units | Buildings | Upgrades |
|---|---|---|---|
| 1-min | 100.0% | 100% | 100% |
| 3-min | 99.5% | 100% | 75.0% |
| 5-min | **99.2%** | 100% | **98.1%** |
| 7-min | 99.5% | 100% | **99.5%** |

### What Changed vs Last Session

| Dimension | 5-min before | 5-min after | Change |
|---|---|---|---|
| Units | 94.9% | 99.2% | **+4.3pp** |
| Buildings | 100% | 100% | unchanged |
| Upgrades | 98.1% | 98.1% | unchanged |

### Waypoint Target Status

| Target | Value | Status |
|--------|-------|--------|
| Unit accuracy ≥ 95% at 5-min | 99.2% | **Exceeded** |
| Building accuracy ≥ 99% | 100% | **Exceeded** |
| Upgrade accuracy ≥ 90% | 98.1% | **Exceeded** |
| Economy MAPE ≤ 15% | not assessed | Economy metrics diverge by design (resource checks bypassed for physics isolation) |
| Per-type timing ≥ 95% within ±1 tick | not assessed | Needs timing-specific test |

---

## Remaining Physics Gaps (0.8% at 5-min = 55 units)

| Unit Type | GT | Em | Delta | Root Cause |
|---|---|---|---|---|
| MARINE | 372 | 354 | -18 | Barracks Reactor — no dual-training |
| HELLION | 52 | 38 | -14 | Factory Reactor — no dual-training |
| WIDOW_MINE | 28 | 20 | -8 | Factory Reactor — no dual-training |
| PROBE | 1421 | 1414 | -7 | Chrono Boost back-calculation timing |
| STALKER | 51 | 49 | -2 | Building queue timing |
| Other | — | — | -6 | Minor timing and queue effects |

### Fix Strategies for >99%

1. **Reactor dual-training** (~40 units) — Model Reactor add-ons allowing 2 concurrent training slots on Barracks/Factory/Starport. For r- prefix intents, allow up to 2 active training slots per building.

2. **Chrono Boost timing** (~7 units) — Adjust back-calculated command loop for Protoss units when Chrono Boost tracker events are detected.

### Branch State

- Branch: `issue-406-tracker-emulated-physics`
- 15 commits ahead of main
- Paused: #387 (reconstitute ladder replays) on pause stack
- Soredium: #442 filed (work-end queue stall bug)

### Pre-existing Test Failures

Drools tests are failing (pre-existing, not caused by this branch): `DroolsScoutingRulesTest` (13 errors), `DroolsStrategyL2L3Test` (7 errors), `DroolsDominanceWeightStrategyTest` (5 errors), `DroolsStrategyTaskTest` (22 errors), `ReplayEngineTest` (2 errors). All are `NullPointerException` or Drools initialization failures. Not related to tracker/emulated physics work.

### Key Code Locations

| File | What to read |
|---|---|
| `EmulatedGame.java:handleTrain` | Train intent processing — economy bypass for r- prefix intents |
| `EmulatedGame.java:trackerTrainDiagnostics` | Diagnostic counters per UnitType per rejection reason |
| `EmulatedGame.java:handleResearch` | Research intent processing |
| `ReplayValidationHarness.java:runInternal` | Unified harness loop — all run overloads delegate here |
| `TrackerIntentExtractor.java` | Tracker event → intent conversion |
| `TrackerDrivenAccuracyBaselineTest.java` | The accuracy test to run after each fix |
| `TrackerTrainDiagnosticTest.java` | Per-unit-type rejection breakdown diagnostic |
