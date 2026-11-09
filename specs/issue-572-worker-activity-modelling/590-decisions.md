## D1: Diagnose then fix — not a structural floor

**Choice:** Write diagnostic to quantify cadence contribution, then implement fix if measurable (even 1%)
**Alternatives:**
- Close as structural floor — unacceptable because even small per-window errors compound into large balance drift over full games, making the simulation useless for training
- Full cadence alignment without diagnostic — wastes time if the contribution is negligible relative to income deficit
**Rationale:** Every source of error matters when the goal is training-quality accuracy over 15-20 minute games. Rate errors compound in balance fields (11.4% → 22.8% from 1-min to 7-min). But we should measure first to confirm the mechanism and quantify the gain.
**Trade-offs:** Diagnostic adds half a day before the fix lands
**Sources:** docs/sc2-physics-constants.md §14 (cumulative drift), EconomyTracker.java:43 (RATE_INTERVAL_TICKS=7), issue #591 (income deficit context)
**Exploration:** quick
**Status:** captured

## D2: Rate window interval as Quarkus config property

**Choice:** Expose the rate window as `starcraft.economy.rate-interval-loops` (default 160)
**Alternatives:**
- Hardcode 160 — simpler code, but breaks the calibration pattern where constants are sweepable
**Rationale:** Consistent with existing pattern (mining-duration-loops, gas-harvest-duration-loops). Enables calibration sweeps to test different windows without recompiling. The diagnostic test itself needs to compare multiple window sizes.
**Trade-offs:** One more config property to maintain; default of 160 is an SC2 engine constant that won't change
**Sources:** WorkerPhysicsConfig.java (existing config pattern), EmulatedGameFeatureExtractor.java:9 (PLAYER_STATS_INTERVAL=160)
**Exploration:** quick
**Status:** captured
