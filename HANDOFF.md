# HANDOFF — quarkmind

## Last Session

Calibration roadmap issues #600 and #601. Per-race mining efficiency factors calibrated. Sub-tick training resolution implemented. 16w gap root-caused to Probe arrival timing, not per-worker rate.

### What was done

- **#600 — Per-race mining efficiency factors (CLOSED):**
  1. SC2 dual-run sweep: factor 0.984 matches Terran/Zerg baselines exactly (3415 at tick 304)
  2. `TerranRaceModel.miningEfficiencyFactor()` → 0.984
  3. `ZergRaceModel.miningEfficiencyFactor()` → 0.984
  4. Added `perRaceMineralAccuracy` tests (±0.5% assertion per race)
  5. Added `miningEfficiencyFactorSweep` diagnostic

- **#601 — Sub-tick training resolution (CLOSED):**
  1. `PendingCompletion.completesAtLoop` replaces `completesAtTick` (loop precision)
  2. `fireCompletions(gameFrame * LOOPS_PER_TICK)` — loop-level comparison
  3. All production (train/build/research/morph) + Chrono Boost use loop scheduling
  4. Probe cadence corrected: [13, 12, 13, 12, 12] (was [12, 12, 13, 12, 12])
  5. Protoss factor refined to 0.997 (12w now 0.0% delta, was +0.3%)
  6. 11 existing tests updated for loop-precision timing

### Key finding — 16w gap root cause

The 16w saturation gap (6.6%, was 6.9%) is NOT from per-worker rate:
- **Pure 16w mining:** EM produces exactly 16/12 = 1.333× of 12w (perfect ratio, no contention at 2w/patch)
- **Steady-state rate (ticks 150-304):** EM and SC2 produce at identical rates (both ~18.3 minerals/tick at 23 workers)
- **The entire gap is from the ramp-up phase (ticks 0-150).** SC2 trains 23 Probes (not 16 — continuous to supply cap). The EM reaches Probe milestones slightly earlier, accumulating 285 extra worker-tick income

The fix is NOT a physics constant change — it requires matching the EM's Probe arrival timing to SC2's. Investigation needed: per-tick Probe count comparison between EM and SC2 baseline.

### Calibrated results

| Metric | EM | SC2 | Delta |
|--------|-----|-----|-------|
| 12w Protoss (tick 304) | 3460 | 3460 | **0.0%** |
| 12w Terran (tick 304) | 3415 | 3415 | **0.0%** |
| 12w Zerg (tick 304) | 3415 | 3415 | **0.0%** |
| 16w dual-run (tick 304) | 4539 | 4260 | +6.6% |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-600-per-race-mining-efficiency` | Complete, not merged | #600 |
| `issue-601-sub-tick-training` | Complete, not merged | #601 |

---

## Issue Map

### Closed this session
| # | Title | Resolution |
|---|---|---|
| 600 | Terran/Zerg mining efficiency | Factor 0.984 — exact SC2 match |
| 601 | Sub-tick training resolution | Loop-precision PendingCompletion. Protoss factor 0.997 |

### Open — calibration roadmap

| # | Gap | Current | Target | Blocked by | Notes |
|---|-----|---------|--------|------------|-------|
| 602 | Geyser position recording | +8.1% gas | ±3% | — | Needs SC2 |
| 603 | Worker initialization delay | ±30 minerals | ±5 | 605 | |
| 604 | 24w/MULE baseline re-record | diverge | pass | 601 ✅ | Unblocked |
| 605 | Mineral patch position recording | approx 7.0u | exact | — | Needs SC2 |
| 606 | Gas harvest contention (6w) | unknown | ±5% | 602 | |

### Open — other

| # | Title | Scale | Complexity | Status |
|---|---|---|---|---|
| 597 | SC2 adapter gas dispatch | S | Med | EM gas rate 108% of SC2 |
| 598 | SC2 adapter MULE/expansion | M | High | Morph/calldown implemented, untested |
| 592 | Overhead absorption limit | S | Med | May be resolved by calibration |

Recommended order: 604 (unblocked by #601) → 605+602 (need SC2) → 603 → 606

---

## Key Code Locations

| What | Where |
|------|-------|
| PhysicsState (loop-precision PendingCompletion) | `quarkmind-sc2/.../emulated/PhysicsState.java` |
| EmulatedGame (fireCompletions, startTraining) | `quarkmind-sc2/.../emulated/EmulatedGame.java` |
| ProtossRaceModel (factor 0.997) | `quarkmind-sc2/.../emulated/ProtossRaceModel.java` |
| TerranRaceModel (factor 0.984) | `quarkmind-sc2/.../emulated/TerranRaceModel.java` |
| ZergRaceModel (factor 0.984) | `quarkmind-sc2/.../emulated/ZergRaceModel.java` |
| MiningCycleDiagnosticTest (sweeps) | `quarkmind-sc2/.../emulated/MiningCycleDiagnosticTest.java` |
| DualRunCalibrationTest (SC2 comparisons) | `quarkmind-sc2/.../playbook/DualRunCalibrationTest.java` |

---

## Pre-existing Issues

- **DroolsScoutingRulesTest class loading failure** — prevents full `mvn test -pl quarkmind-sc2` suite. Run tests by name.
