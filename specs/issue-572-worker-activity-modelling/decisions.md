## D1: Overall approach — validate saturation + worker activity lifecycle

**Choice:** Approach A — validate existing saturation model against SC2 income curves, then add worker activity lifecycle (SCOUTING, TRANSFERRING states) with event hooks that work in both replay and real-game modes.
**Alternatives:**
- Approach B (Effective miner count) — simpler but is a formula disguised as physics; violates simulation-over-formulas principle
- Approach C (Full trip-cycle micro-simulation) — most accurate but massive scope, overlaps #574/#575
**Rationale:** Pure simulation approach. Validates the existing MAX_MINERS=2 contention model before building on it. Worker activity states are independent of mining duration constants, so they compose well with #574/#575 when those land. Both replay and real-game modes addressed from the start.
**Trade-offs:** Needs empirical data (playbook or community curves) to validate the saturation model. If the saturation curve turns out to be wrong, fixing it may require changes to the mining loop that overlap with #575.
**Sources:** WorkerSimulation.java (MAX_MINERS=2, tryStartMining, assignToNearestPatch), MineralPatch.java, GasSlot.java, issue #572, HANDOFF.md accuracy state, memory: feedback-simulation-over-formulas
**Exploration:** quick
**Status:** captured

## D2: Saturation reference data — published community data + playbook validation

**Choice:** Use published SC2 community data as the reference curve for validating the simulation's contention model. Workers 1-16 (2/patch) earn ~40-57 min/min each; workers 17-24 (3rd/patch) earn ~20-22 min/min each — a 35-40% efficiency drop. Close patches may only effectively support 2 workers. If the simulation's curve doesn't match, use playbook-controlled SC2 scenarios for precise measurement.
**Alternatives:**
- Replay-derived only — replay data has missing commands (Blizzard API ~43% CmdEvent coverage); oracle replays have complete tracker events but worker-level income is not directly observable from tracker data
- Community data only without playbook fallback — published numbers vary by source and game speed; playbook gives ground truth
**Rationale:** Community data is well-established and immediately available. The saturation curve shape (2→3 worker efficiency drop) is consistent across all sources. Playbook validation provides precise numbers when the shape alone isn't sufficient.
**Trade-offs:** Community data may not reflect the exact patch version we're simulating. Playbook validation requires SC2 running.
**Sources:** GameReplays.org mineral mechanics guide, TL.net worker saturation thread, Liquipedia Mining Minerals, HANDOFF.md published physics constants
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Transit modelling — physical walk time between bases

**Choice:** Model actual walk time for base transfers. Worker enters TRANSFERRING state, walks from old base to new base position using existing movement system (`moveToward`), then starts mining at the new base. Workers produce no income during transit.
**Alternatives:**
- Instant reassignment with delay — simpler but not physically accurate (worker position doesn't move)
- Skip transit — keep instant reassignment; transit is a smaller contributor than saturation/scouting
**Rationale:** Pure physics — distance / worker speed = transit time. Uses the existing movement infrastructure. Workers in transit are genuinely unavailable for mining, which is the real-game behavior. The `transferMineralWorkers` method already knows both base positions.
**Trade-offs:** Adds a new WorkerTask state (TRANSFERRING) that must be handled in the tick loop. Minor complexity increase in an already-well-structured system.
**Sources:** WorkerSimulation.java:transferMineralWorkers (lines 78-96), SimulatedWorker.moveToward
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D4: Scout detection — game-phase hooks, caller decides

**Choice:** EmulatedGame exposes `onWorkerScouting(tag, returnAtLoop)` and `onWorkerReturnFromScout(tag)` as event hooks. The caller decides when to scout — replay mode detects scouts from GT unit positions (worker far from any base), real-game/playbook mode uses supply/timing thresholds from the strategy layer. #572 builds the infrastructure; specific strategy-layer scout triggers are out of scope.
**Alternatives:**
- Built-in heuristic — EmulatedGame itself decides when to scout at specific supply counts. Bakes strategy into physics, wrong layer.
- Defer real-game scouting — only model in replay mode. Limits the value for playbook/real-game accuracy.
**Rationale:** Separation of concerns — the worker simulation models the EFFECT of scouting (worker off minerals), not the DECISION to scout. Same pattern as `pauseWorkerForBuilding`: the building system calls it, the worker simulation handles the mechanics. Hooks work in both modes without coupling physics to strategy.
**Trade-offs:** Real-game mode won't automatically detect scouts — a strategy-layer caller must invoke the hooks. Until that caller exists, standalone mode won't capture scouting. But the infrastructure is ready.
**Sources:** WorkerSimulation.java:pauseWorkerForBuilding (existing pattern), issue #572 approach options
**Exploration:** quick
**Depends on:** D1
**Status:** captured
