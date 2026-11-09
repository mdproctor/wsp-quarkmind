## D1: Acceleration model — per-loop physics simulation

**Choice:** Per-loop movement simulation. Each of the 22 game loops within a tick, update worker speed (accelerate at 2.5 u/s², decelerate near target, clamp to max 3.94) and move by `currentSpeed * loopDuration`. Trip time emerges from the simulation. No computed arrival times, no formulas.
**Alternatives:**
- Analytical arrival time — compute trip time from d=½at² formula upfront. Violates simulation-over-formulas principle. Formulas have repeatedly failed to achieve 100% accuracy.
- Per-tick simulation — one movement update per 22-loop tick. Too coarse for short-distance worker trips; deceleration zone is overshot in a single tick.
**Rationale:** SC2 runs movement at per-loop granularity internally. Matching this granularity produces accurate trip times without approximation. A week of trying formulas proved they always miss something — simulation is the only path to 100%.
**Trade-offs:** 22x more movement updates per tick per worker. At 24 workers this is ~528 updates per tick — negligible.
**Sources:** SC2Mapster wiki (Data/Units — workers have acceleration 2.5, deceleration defaults to acceleration), SimulatedWorker.moveToward (current constant-speed implementation), memory: feedback-simulation-over-formulas
**Exploration:** quick
**Status:** captured

## D2: Implementation placement — SimulatedWorker.moveToward

**Choice:** Add `currentSpeed` field to `SimulatedWorker`. `moveToward` runs a sub-loop of 22 iterations (one per game loop), updating speed and position each iteration. Callers unchanged.
**Alternatives:**
- WorkerSimulation.tick — couples physics to the tick loop
- Separate MovementModel class — adds indirection for a single-concern change
**Rationale:** `moveToward` is the natural home for movement physics. The sub-loop is internal to the method — callers don't need to know about per-loop granularity.
**Trade-offs:** None significant.
**Sources:** SimulatedWorker.java:49-63, SC2Data.LOOPS_PER_TICK (22)
**Exploration:** quick
**Status:** captured
