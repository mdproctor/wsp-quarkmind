# HANDOFF — quarkmind

## Last Session

Landed four issues on Epic #534 (EmulatedGame 100% accuracy): worker activity modelling (#572), mining duration calibration (#575), acceleration model (#574), gas movement (#577 — resolved by #574). One issue remaining: spending timing (#576).

### What was done

- **#572 — Worker activity modelling (landed).** Added SCOUTING and TRANSFERRING states to WorkerTask. Scouting lifecycle follows the pauseWorkerForBuilding pattern: `onWorkerScouting(tag, returnAtLoop)` pulls worker off minerals, auto-return at scheduled loop. Transit modelling replaces instant base transfers with physical walk time via moveToward. EmulatedGame exposes scouting hooks for replay harness and strategy layer callers.

- **#575 — Mining duration calibration (landed).** MINING_DURATION_LOOPS corrected from 16 to 44.576 (Liquipedia: 1.99 seconds × 22.4 loops/sec). MAX_MINERS changed from 2 to 1 — only one worker can occupy a mineral patch at a time. Together these produce a saturation curve where the 3rd worker per patch earns ~59% of the 2nd worker's rate (SC2 published: ~50-55%).

- **#574 — Per-loop acceleration model (landed).** Replaced constant-speed moveToward with per-loop physics simulation. Each of 22 game loops within a tick: update speed (accelerate at 2.5 u/s², decelerate near target, clamp to max 3.94) and move. Constants from SC2 data editor (SC2Mapster wiki). At 6-unit patch distance, workers never reach max speed — triangular velocity profile.

- **#577 — Gas movement model (closed, resolved by #574).** The acceleration model applies to gas workers via the same moveToward method. Root cause identified in #577 ("constant-speed linear movement") was the same as #574.

- **Blog: "The Physics That Was Always There"** — published as mdp04 in the series. Covers all three physics corrections with SVG diagrams.

### Design principle reinforced

**Simulation over formulas — no exceptions.** Per-loop stepping, not analytical trip times. Formulas (even physics equations evaluated once) don't work because they skip the simulation and miss edge cases. The user explicitly rejected the analytical arrival time approach: "we've spent a whole week trying to get formulas right and we cannot get 100% accuracy." The fix is always to simulate at the right granularity.

---

## Current Accuracy State

### Saturation curve (WorkerSimulationTest diagnostic)

| Workers | Total min/min | Per worker | SC2 published |
|---------|--------------|------------|---------------|
| 8 | 254.5 | 31.8 | ~55 |
| 12 | 344.1 | 28.7 | ~48 |
| 16 | 418.5 | 26.2 | ~41 |
| 20 | 483.6 | 24.2 | ~33 |
| 24 | 540.7 | 22.5 | ~22 |

Per-worker rates are lower than SC2 across the board because the test uses flat 6-unit patch distances. Real SC2 has close patches at 4-5 units. Replay mode uses GT mineral patch positions and will be more accurate.

Efficiency drop 16→24: 13.9% (SC2 reference: ~20% average, ~50% marginal for workers 17-24).

### Key changes to constants

| Constant | Before | After | Source |
|----------|--------|-------|--------|
| MINING_DURATION_LOOPS | 16 (int) | 44.576 (double) | Liquipedia: 1.99s × 22.4 |
| MAX_MINERS | 2 | 1 | SC2 — one worker per patch at a time |
| Worker acceleration | N/A (constant speed) | 2.5 u/s² | SC2Mapster data editor |
| Worker deceleration | N/A | 2.5 u/s² | SC2Mapster (defaults to acceleration) |

---

## Remaining Queue

```
- [x] #572 — Worker activity modelling
- [x] #575 — Mining duration playbook validation
- [x] #574 — Worker movement acceleration model
- [x] #577 — Gas movement model (resolved by #574)
- [ ] #576 — Spending timing model ← active
```

### #576 — Spending timing model (S/Med)

Spending category fields (mineralsUsedCurrentArmy etc.) have 5-35% error. Root cause: EM deducts spending at intent-execution time, GT reports via PlayerStatsEvent at ~160-loop cadence. May be a measurement artifact rather than a simulation bug. Different domain from the physics work this session — reporting cadence alignment, not movement or mining physics.

---

## Critical Insights for Next Session

1. **Run baselines FIRST.** Three major physics changes landed without standalone baseline validation (oracle data unavailable). Run `StandaloneEmulatedGameBaselineTest` and `DivergenceRegressionTest` before any new work. File issues from whatever the data reveals.

2. **Per-worker income halved.** The acceleration model dropped per-worker mineral income from ~52 to ~26 in unit tests. Gas from ~183 to ~132 gas/min. This could be over-correction — the flat 6-unit patch distance in tests is longer than real SC2 close patches (4-5 units). Replay mode with GT positions may show better numbers. But if the standalone baseline shows regression, diagnose whether the acceleration constant (2.5) or the patch distance model is the cause.

3. **Scouting hooks are infrastructure only.** `onWorkerScouting`/`onWorkerReturnFromScout` exist on EmulatedGame but no caller invokes them. The replay harness needs GT-position-based scout detection (worker far from any base = scouting). Without this, standalone mode still has all workers mining continuously.

4. **Analytical ≠ simulation.** The user explicitly rejected computing trip times from d=½at². Even physics equations evaluated once are formulas. The arrival time must emerge from per-loop stepping. This is a hard constraint — don't propose analytical shortcuts.

5. **The saturation curve test is the mining validation tool.** `WorkerSimulationTest#saturationCurve_thirdWorkerEarnsLessThanSecond` prints per-worker income at 8/12/16/20/24 workers. Run it after any mining or movement change.

6. **File issues aggressively.** Every finding from baseline runs should become an issue. The queue drives the work; unfiled findings get lost.

---

## Key Code Locations

| What | Where |
|------|-------|
| SimulatedWorker (acceleration, moveLoops) | `quarkmind-sc2/.../emulated/SimulatedWorker.java` |
| WorkerSimulation (scouting, transit, tick) | `quarkmind-sc2/.../emulated/WorkerSimulation.java` |
| WorkerTask (SCOUTING, TRANSFERRING) | `quarkmind-sc2/.../emulated/WorkerTask.java` |
| MineralPatch (MAX_MINERS=1) | `quarkmind-sc2/.../emulated/MineralPatch.java` |
| EmulatedGame (scouting hooks) | `quarkmind-sc2/.../emulated/EmulatedGame.java` |
| WorkerSimulationTest (saturation curve, scouting, transit tests) | `quarkmind-sc2/.../emulated/WorkerSimulationTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents the full `mvn test -pl quarkmind-sc2` suite from running. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
