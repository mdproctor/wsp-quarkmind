## D1: Fix approach — measure-then-fix from 8w SC2 cadence data

**Choice:** Record 8w SC2 baseline with worker cadence data, decompose the 3-loop/leg gap into constituent causes (mining target position, deposit position, path deviation, acceleration), then fix each identified cause proportionally.
**Alternatives:**
- Geometric path deviation (Approach A) — models mineral-line obstruction only, skips root cause measurement, risks fixing the wrong thing
- Reduce MINERAL_PATCH_RADIUS from geometry (Approach B) — fixes mining target only, uniform effect, no per-patch variance
- Curved approach trajectories (Approach C) — most physically accurate but over-engineered for the accuracy gain
**Rationale:** The 3-loop/leg gap has 5 possible physical causes, all measurable from cadence data. Fixing without measurement is hypothesis-driven — the 8w cadence baseline the user wants to record is also the diagnostic tool. At 8w (1 worker per patch), every cycle is clean with zero contention noise.
**Trade-offs:** Requires recording an SC2 baseline before implementing the fix (delays implementation by one step). The fix shape depends on what the data shows, so the design has a diagnostic phase before the implementation phase.
**Sources:** WorkerSimulation.java:21-40 (edge-to-edge model), MineralPatch.java:59-64 (computeApproachDocking), sc2-physics-constants.md §3 (SC2 observed cycle), sc2-physics-constants.md §14 (mining cycle breakdown)
**Exploration:** deep-analysis
**Status:** captured

## D2: 8w baseline recording structure

**Choice:** Standard playbook + DebugKillUnit at tick 0. New playbook `cal-mineral-baseline-8w.yaml` kills 4 of 12 starting workers immediately, lets 8 mine for 305 ticks. Record both CalibrationBaseline (economy) and WorkerCadenceBaseline (per-tick positions). Add `recordCalMineralBaseline8w()` to PlaybookSC2RecorderTest and `dualRunCalMineralBaseline8wAgainstSC2()` to DualRunCalibrationTest.
**Alternatives:**
- Extended duration (600+ ticks) — more cycle samples but no additional root cause signal; 305 ticks gives ~50+ full cycles per worker, which is ample
**Rationale:** Matches existing playbook infrastructure. DebugKillUnit is already used in MiningAnimationMeasurementTest and PlaybookSC2Adapter. The cadence data provides mining positions, walk distances, and path deviation per patch.
**Trade-offs:** None significant — standard pattern.
**Sources:** PlaybookSC2RecorderTest.java (recorder pattern), PlaybookSC2Adapter.java:366 (DebugKillUnit usage), MiningAnimationMeasurementTest.java:275 (DebugKillUnit usage)
**Exploration:** quick
**Depends on:** D1 (measure-then-fix approach)
**Status:** captured
