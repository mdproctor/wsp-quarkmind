# HANDOFF — quarkmind

## Last Session

SC2 playbook calibration — recorded 8 baselines against real SC2, diagnosed adapter infrastructure bugs, established per-worker physics accuracy. Filed #596-#599 for remaining calibration work. The critical blocker for 0% error is #596 (EM implementation gap).

### What was done

- **SC2 adapter fixes (3 bugs fixed, 2 remaining):**
  1. **(Fixed)** Build commands at hardcoded `(30, 30)` → always rejected by SC2. Switched to debug API `DebugCreateUnit` for supply/tech buildings — bypasses placement validation entirely.
  2. **(Fixed, untested)** Assimilator geyser-targeting commands created but never dispatched — `onStep` build path called `translateStep` but ignored the return value. Fix committed (af5ca46b).
  3. **(Fixed, untested)** Morph (`CC→OC`) and calldown (`MULE`) actions unimplemented. Added `translateMorph` (targets source building with morph ability) and `translateCalldown` (targets nearest mineral patch with `EFFECT_CALL_DOWN_MULE`).
  4. **(Remaining — #597)** SC2 does NOT auto-assign workers to completed Assimilators. Adapter needs explicit worker-to-gas assignment action.
  5. **(Remaining — #598)** Expansion playbook needs rally commands or debug-spawned workers at the expansion location.

- **8 SC2 baselines recorded** (5 working, 3 broken):
  - **Working mineral baselines:** `cal-mineral-baseline-12w` (Protoss), `cal-mineral-baseline-terran`, `cal-mineral-baseline-zerg` — 12 workers, pure mining, ~5 min each
  - **Working saturation:** `cal-mineral-saturated-16w` — 12→23 workers via continuous training + debug Pylon
  - **Working validation:** `cal-worker-speed-single` — 12w with asserts, redundant with 12w baseline
  - **Broken gas:** `cal-gas-baseline-3w`, `cal-gas-dual-6w` — 0 vespene (Assimilator dispatch bug + no worker-to-gas assignment)
  - **Broken MULE:** `cal-mule-isolation` — identical to Terran baseline (morph/calldown not dispatched)

- **DualRunCalibrationTest extended:**
  - `selfBaselineCalWorkerSpeedSingle` — self-baseline for worker-speed playbook
  - `dualRunCalMineralBaseline12wAgainstSC2` — loads SC2 baseline, runs EM, prints per-tick mineral delta

- **4 new issues filed:** #596 (EM implementation gap), #597 (gas adapter), #598 (MULE/expansion adapter), #599 (saturation contention model)

---

## Current Accuracy State

### What works — per-worker mineral physics

SC2 playbook calibration with 12 workers (no buildings, pure mining) on TorchesAIE_v4:

| Race | SC2 per-worker rate | EM at real distances | Delta |
|---|---|---|---|
| Protoss | 57.9 minerals/min | ~57.8 (weighted 4×6.6u + 4×7.6u) | **<1%** |
| Terran | 57.2 minerals/min | ~57.8 | **~1%** |
| Zerg | 57.2 minerals/min | ~57.8 | **~1%** |

**Conclusion:** Per-worker mineral physics is correct at real map distances. Mining-duration=12, gas-harvest=22, acceleration=3.5 constants are calibrated.

### What doesn't work — EM implementation gap (#596, BLOCKER)

The dual-run comparison (`dualRunCalMineralBaseline12wAgainstSC2`) shows EM producing **3275 minerals vs SC2's 3460 at tick 304** — a cumulative shortfall of 185 minerals (5.3%). The EM's internal per-worker rate is 54.0/min/worker at 6.0u, 15% below the physics formula's predicted 63.4.

**This is the critical blocker.** Until we know WHERE the EM loses loops in its mining cycle, adjusting constants is futile. A single-worker trace diagnostic (print every loop transition) will reveal the source — startup delay, tick-boundary waste, assignment overhead, or cycle mechanics.

### What doesn't work — saturation (#599)

| Workers/patch | SC2 efficiency | EM efficiency | Gap |
|---|---|---|---|
| 1.5 (12 workers) | 100% (base) | ~95% | — |
| 2.9 (23 workers) | **84%** | **~90%** | EM 6% too lenient |

3rd workers per patch add proportionally too much income. The contention model needs steeper penalties.

### What's not yet measured

| Category | Status | Blocker |
|---|---|---|
| Gas per-worker rate | No data (0 vespene in baselines) | #597 — Assimilator dispatch + worker assignment |
| MULE income per calldown | No data | #598 — morph + calldown dispatch |
| Expansion transfer delay | No data | #598 — rally commands or debug workers |
| 24w saturation (3/patch) | No data (supply-capped at 15) | Needs re-recording with fixed adapter |

### Standalone EmulatedGameBaselineTest — 7-min checkpoint (oracle replays)

| Field | MAPE | Status |
|---|---|---|
| mineralsCollectionRate | 22.1% | Worker allocation gap, not physics |
| vespeneCollectionRate | 29.1% | Gas physics not yet SC2-calibrated |
| mineralsCurrent | 67.5% | Cumulative drift from rate error |
| vespeneCurrent | 65.4% | Cumulative drift |
| foodMade/Used | 3.1%/3.7% | At floor |
| workersActiveCount | 2.5% | At floor |

---

## Issue Map

### Critical path to 0% mineral accuracy

```
#596 — EM implementation gap diagnosis (BLOCKER)
  ↓ (fix per-worker base rate first)
#599 — Saturation contention model
  ↓ (then fix multi-worker behavior)
#597 — Gas adapter fix (parallel with above)
#598 — MULE/expansion adapter (parallel with above)
```

### Open issues by priority

| # | Title | Scale | Complexity | Blocked by | Status |
|---|---|---|---|---|---|
| **596** | EM implementation efficiency gap | M | Med | — | **NEXT — blocker for 0% accuracy** |
| **599** | Saturation contention model | L | High | #596 | New — needs base rate fix first |
| **597** | SC2 adapter gas dispatch | S | Med | — | Fix committed, untested |
| **598** | SC2 adapter MULE/expansion | M | High | — | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | #596 | May be resolved by #596 |
| 593 | SC2 rate algorithm RE | M | High | — | Deprioritised — playbook calibration more direct |
| 594 | Sub-tick rate tracking | S | Med | — | Deprioritised |
| 595 | Gas rate regression | S | Med | #597 | Needs gas data first |

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 591 | Worker income direction analysis | Factors reverted to 1.0, physics proven correct |

---

## What to do next

### Priority 1 — #596: Single-worker trace diagnostic

Write a test that traces ONE worker's mining cycle loop-by-loop for 100 ticks:
```
loop 0: IDLE → assigned to patch 3
loop 0: start walk to (6.0, 0.0), distance=6.0
loop 47: arrive at patch, start mining
loop 59: mining complete, 5 minerals harvested
loop 59: start walk home to (0, 0)
loop 106: arrive at base, deposit 5 minerals → total 55
loop 106: start walk to patch 3...
```

Compare actual cycle length to the formula. The gap between 54.0 observed and 63.4 predicted must come from one of:
1. **Startup delay** — workers don't all start at loop 0
2. **Tick-boundary waste** — loops lost at state transitions despite sub-tick chaining
3. **Assignment overhead** — finding a free patch takes loops
4. **MineralPatch contention** — with 12 workers on 8 patches, some wait

### Priority 2 — #597: Gas calibration

1. Verify Assimilator dispatch fix works (run `recordCalGasBaseline3w` once)
2. If Assimilator appears but vespene=0, add explicit worker-to-gas assignment
3. Extract per-worker gas rate, calibrate gas-harvest-duration-loops

### Priority 3 — #599: Saturation model

After #596 is fixed, compare EM saturation against the 16w baseline tick-by-tick. Steepen contention penalties until EM matches SC2's 84% efficiency at 23 workers.

---

## Key Code Locations

| What | Where |
|------|-------|
| PlaybookSC2Adapter (debug build, geyser targeting, morph, calldown) | `quarkmind-sc2/.../real/PlaybookSC2Adapter.java` |
| ResolvedCommand (targetUnitTag for geyser targeting) | `quarkmind-sc2/.../real/ResolvedCommand.java` |
| QuarkusSC2Transport.sendActions (targetUnitTag dispatch) | `quarkmind-sc2/.../real/QuarkusSC2Transport.java:285` |
| DualRunCalibrationTest (SC2 comparison + self-baselines) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |
| PlaybookSC2RecorderTest (10 SC2 recording scenarios) | `quarkmind-sc2/.../real/PlaybookSC2RecorderTest.java` |
| SC2 calibration baselines (8 JSON files) | `quarkmind-sc2/src/test/resources/calibration/cal-*.json` |
| Calibration playbooks (10 YAML files) | `quarkmind-sc2/src/test/resources/playbooks/cal-*.yaml` |
| MineralPatch.PATCH_DISTANCE (6.0f uniform) | `quarkmind-sc2/.../emulated/MineralPatch.java:11` |
| EmulatedGame.tick() (efficiency factor wiring) | `quarkmind-sc2/.../emulated/EmulatedGame.java:208-209` |
| SC2 physics constants reference | `docs/sc2-physics-constants.md` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name: `-Dtest=ClassName`. Not related to Epic #534.
- **Terran/Zerg unit type mapping** — SCVs and Drones show as `UNKNOWN` in SC2 baselines. `ObservationTranslator` doesn't map all ocraft unit types. Mineral data is correct (unit types don't affect mineral counts).
