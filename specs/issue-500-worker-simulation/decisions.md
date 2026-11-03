# Decisions — #500 Individual Worker Simulation

## D1: Mineral patch model

**Choice:** Positional patches with depletion — each patch has a position and mineral amount (uniform 1500 start), depletes naturally as workers mine
**Alternatives:**
- Anonymous slots — 8 slots per base, no positions, no depletion. Simpler but loses close-patch vs far-patch income difference
- Positional, no depletion — patches have positions but don't deplete. Middle ground but unrealistic long-game
**Rationale:** Positional patches enable UI visualisation of worker paths and patch state. Depletion is a natural consequence of tracking amounts. Map-specific starting amounts deferred to #501
**Trade-offs:** More state per patch (position + amount) vs anonymous slot counting
**Sources:** SC2Data.java (existing tiered model), issue #500, saturation-mining-design.md
**Exploration:** quick
**Status:** captured

## D2: Worker simulation cycle fidelity

**Choice:** Tick-granular — worker has position, velocity, and current task. Advances each tick. Walk time depends on distance to target. Income from deposit events, not formulas
**Alternatives:**
- Fixed-duration cycle — constant N ticks per mining trip. Loses distance-based income variance
- Distance-bucketed — near/far buckets with different cycle times. Approximation of tick-granular
**Rationale:** The whole point is "no formulas" — income emerges from simulation. Worker movement is event-driven: arrive at patch → mine, finish mining → walk to town hall, arrive at town hall → deposit and turn around. Also enables UI improvements showing worker paths
**Trade-offs:** More computation per tick (advancing N workers) but workers move in straight lines — cheap
**Sources:** Issue #500 approach section, user clarification ("no algorithm matching, all event-driven and task-driven")
**Exploration:** quick
**Status:** captured

## D3: Gas harvesting model

**Choice:** Full cycle — workers enter gas building, harvest for N ticks, exit, walk to town hall, deposit, walk back. 3 worker slots per building. Same fidelity as mineral model
**Alternatives:**
- Fixed rate per worker — constant income per assigned gas worker. Simple but inconsistent with mineral model
- Simplified rotation — track workers inside building (max 1), others wait. No walk time
**Rationale:** Consistent fidelity across both resource types. The vespene error is the primary target — full cycle simulation gives the most accurate gas income
**Trade-offs:** Gas cycle is slightly more complex (building interior as implicit queue) but consistent with mineral approach
**Sources:** Issue #500 acceptance criteria (vespeneCollectionRate < 15%), SC2Data.gasIncomePerTick
**Exploration:** quick
**Status:** captured

## D4: GT harness worker assignment

**Choice:** GT workers mapped to resources — count GT workers near each mineral patch and gas building, set simulated worker assignments to match these resource-level counts
**Alternatives:**
- GT-to-simulated-worker matching — match each GT worker to a simulated worker by proximity. More precise but requires cross-tick identity tracking
**Rationale:** Simpler — no need to track individual GT worker identity. The harness just needs to know how many workers are at each resource, not which specific simulated worker maps to which GT worker
**Trade-offs:** Loses individual worker identity correlation but that's unnecessary for income accuracy
**Sources:** ReplayValidationHarness.java (current setMiningProbesPerBase), issue #500
**Exploration:** quick
**Status:** captured

## D5: Class design — replace WorkerAssignment

**Choice:** Replace WorkerAssignment entirely with WorkerSimulation — single class owns worker state, position, task, movement, resource tracking, and income computation
**Alternatives:**
- New class alongside — keep WorkerAssignment as read-only view, add WorkerSimulation driving it. More classes, unclear benefit since all counting logic becomes redundant
**Rationale:** WorkerAssignment (160 lines) is pure counting logic that becomes redundant when workers have individual state. mineralWorkersPerBase() and gasWorkersForBuilding() become derived queries on WorkerSimulation state
**Trade-offs:** Larger single class but cleaner than maintaining two overlapping abstractions
**Sources:** WorkerAssignment.java, EmulatedGame.tick() L141-194
**Exploration:** quick
**Status:** captured

## D6: Integration approach

**Choice:** WorkerSimulation as self-contained tick component — EmulatedGame.tick() calls workerSimulation.tick() which returns total mineral/gas income for the tick
**Alternatives:**
- Worker state on Unit — extend Unit record with task/target/cargo. Requires making Unit mutable or parallel state map
- Physics engine entities — workers use EmulatedGame's A* pathfinding. Heavyweight for straight-line paths
**Rationale:** Workers move in straight lines (resource ↔ town hall), no obstacles. Unit is an immutable record — adding mutable state would break its contract. Self-contained component is independently testable
**Trade-offs:** Workers don't participate in combat movement system (acceptable — mining workers don't pathfind around obstacles)
**Sources:** EmulatedGame.java, PlayerState.java (Unit is a record)
**Exploration:** quick
**Status:** captured
