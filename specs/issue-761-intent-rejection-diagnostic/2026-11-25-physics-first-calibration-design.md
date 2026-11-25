# Physics-First Calibration Methodology

**Branch:** issue-761-intent-rejection-diagnostic
**Issue:** #763
**Date:** 2026-11-25

## Problem

The EmulatedGame has 7 constants that were derived by parameter sweeping (curve-fitting against SC2 baselines) rather than by measuring the SC2 mechanism they claim to model. When one constant changes, swept constants need re-sweeping — a compensation treadmill. The project rule is "no approximations, replicate SC2's actual mechanisms."

**Swept constants identified:**

| Constant | Value | What it claims to model | How it was derived |
|----------|-------|------------------------|-------------------|
| `APPROACH_ANIMATION_LOOPS` | 20.0 | Deceleration/positioning at patch | 8w cadence fitting #679 |
| `GAS_APPROACH_DOCKING_LOOPS` | 11.5 | Gas entry animation | Dual-run sweep #689 |
| `LOW_SATURATION_DODGE_SCALE` | 0.25 | Reduced collision at ≤2 w/p | Dodge sweep #656 |
| `INITIAL_WORKER_STAGGER` | 6.1 | Desynchronised spawn timing | Dual-run sweep #578 |
| `GAS_DEPOSIT_LOCK_LOOPS` | 8.0 | Gas deposit serialisation | 6w sweep #682 |
| `MINERAL_PATCH_OBSTACLE_RADIUS` | 0.5 | Mineral patch collision geometry | Tile-marking on TorchesAIE #766 |
| `MIN_APPROACH_AFTER_WAIT` | 0 (TBD) | Irreducible pickup animation | Unmeasured — needs SC2 data |

## Solution

A reusable methodology for replacing swept constants with measured physics, executed in phases from lowest-noise to highest.

### Constant Classification

Every EM constant is classified into one of four categories:

| Category | Meaning | Fix path |
|----------|---------|----------|
| **Data editor** | Value from SC2 game files | Look it up |
| **Observable** | Measurable from SC2 API | Record from SC2, measure, replace |
| **Emergent** | Should not be a constant — should emerge from correct simulation | Fix the physics, delete the constant |
| **Unknown** | We don't know if SC2 has this mechanism | Investigate first |

New confidence level **Swept** added to `sc2-physics-constants.md` for constants derived by parameter sweep.

### SC2 Worker Profiler

New `@Tag("sc2")` test: `WorkerPhysicsProfilerTest` in `io.quarkmind.sc2.playbook`.

**Mechanism:** Drives real SC2 via the bot API at per-loop resolution (`RequestStep` with `step_size=1`). Runs short controlled windows on TorchesAIE.

**Scenarios:**
- 8w economy-only, 100 ticks — zero contention, foundation constants
- 16w economy-only, 50 ticks — moderate contention
- 24w economy-only, 50 ticks — high contention  
- 36w economy-only, 50 ticks — extreme contention
- 3w gas single-geyser, 50 ticks — gas cycle
- Game start, 10 ticks — initial worker stagger

Each scenario reaches the target worker count via playbook, then switches to per-loop stepping for the measurement window.

**Per-loop recording per worker:**
- Tag (worker identity)
- Position (x, y)
- Cargo (mineral/gas count)
- Orders (ability ID, target position)

**Output:** JSON file per scenario in `src/test/resources/profiler/`. One entry per game loop per worker.

### Worker Physics Analyser

Plain Java class `WorkerPhysicsAnalyser` in `io.quarkmind.sc2.emulated` (test source). Reads profiler JSON and computes constants from state transitions.

**State inference:**
- Position stable at patch for 2+ loops → arrived at patch
- Cargo 0→5 → mining complete
- Position starts moving from patch → departed patch
- Position stable at TH for 2+ loops → arrived at TH
- Cargo 5→0 → deposit complete

**Derived measurements per worker per cycle:**
- `walkToPatch` — loops from TH departure to patch arrival
- `approachAndMine` — loops from patch arrival to cargo flip
- `walkToBase` — loops from patch departure to TH arrival
- `deposit` — loops from TH arrival to TH departure
- `waitAtPatch` — loops from arrival to mining start (contention scenarios)
- `totalCycle` — sum, must equal measured cycle length

**Key comparison:** At 8w `approachAndMine` = approach + mining (no waits). At 36w, compare waiting vs non-waiting workers' `approachAndMine`. The delta IS the approach absorption — measured from SC2, not swept.

**Output:** Report with measured constants, mean ± σ, sample size per constant.

### Phased Execution

**Phase 1 — Zero contention (8w), foundation constants:**
- Approach animation (from `approachAndMine` - known mining duration)
- Mining duration (direct measurement, validates existing 22-loop value)
- Deposit duration (direct measurement, validates existing 12-loop value)
- Full cycle decomposition (every loop accounted for)

**Phase 2 — Contention tiers (16w, 24w, 36w):**
- Wait-to-mine transition time for waiting workers
- Approach duration for waiting vs non-waiting workers
- Walk time growth per saturation level (reveals collision physics)
- `MIN_APPROACH_AFTER_WAIT` measured, not swept

**Phase 3 — Startup and gas:**
- Initial worker stagger from first-10-tick observation
- Gas cycle decomposition at per-loop resolution
- Gas deposit lock investigation (does SC2 have this mechanism?)

**Phase 4 — Emergent constant validation:**
- Run EM with all measured constants from Phases 1-3
- Check if `LOW_SATURATION_DODGE_SCALE` is still needed (remove if physics is now correct)
- Check if `INITIAL_WORKER_STAGGER` is still needed (remove if SC2's natural desync is now modeled)
- Any remaining gap reveals a missing mechanism to investigate

### Validation Rule

**Never revert a measured constant to a swept value.** A measured constant is what SC2 does. If aggregate accuracy worsens after replacing a swept value with a measured one, another constant was swept to compensate for the wrong value. Investigate what else is wrong — don't re-sweep.

### Constant Registry

Extend `sc2-physics-constants.md`:
- Add **Swept** confidence level
- Reclassify constants identified in this audit
- Add new §20 "Measurement Backlog" tracking each Swept constant's category, phase assignment, and measurement status

## Scope

**In scope:** Mineral worker physics constants. The 7 swept constants identified, plus validation of existing "Calibrated" constants.

**Out of scope:** Combat stats, unit HP, train times, upgrade costs — these use different measurement methods (replay analysis, data editor) and are not part of the worker physics calibration.

## References

- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhysicsConfig.java` — all swept constants
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — tryStartMining absorption, collision model
- `docs/sc2-physics-constants.md` — existing constant documentation
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/PlaybookSC2RecorderTest.java` — existing SC2 recorder (per-tick)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerPhaseListener.java` — EM-side phase tracking
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/real/ObservationTranslator.java` — SC2 API data extraction
- Session sweep data (2026-11-25): wait-time absorption, dodge scale, approach animation sweeps
- CLAUDE.md design rules: "No approximations. Replicate SC2's actual mechanisms."
