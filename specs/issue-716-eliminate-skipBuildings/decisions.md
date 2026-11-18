# Decisions — #716 Eliminate skipBuildings

## D1: Route builds through applyIntent instead of debugCreateBuilding

**Choice:** Replace the `skipBuildings` branch in `DualRunCalibrationTest.calibrate()` with `game.applyIntent(BuildIntent.of(...))` — the same path used by `SC2DeliveryHandler.doBuild()` and all production code.
**Alternatives:**
- Add construction delay to skipBuildings (#713) — compensation hack layered on a workaround, doesn't model physics
- Implement terrain-aware placement with valid build grids — over-engineered for calibration; exact positions don't affect economy
**Rationale:** `EmulatedGame.handleBuild()` already implements the full building lifecycle: resource deduction, construction time via PendingCompletion, supply on completion, town hall registration, gas building worker assignment. The only reason `skipBuildings` exists is to avoid needing a valid position — but calibration doesn't need exact SC2 positions, just reasonable ones.
**Trade-offs:** Building positions won't match SC2 exactly (Pylons at offset positions vs SC2 player's chosen spots). This doesn't affect economy calibration — supply timing is what matters.
**Sources:** EmulatedGame.java:763 (handleBuild), SC2DeliveryHandler.java:88 (doBuild), DualRunCalibrationTest.java:836 (skipBuildings branch)
**Exploration:** quick
**Status:** captured

## D2: Position resolution strategy

**Choice:** Compute positions from baseline data already available in CalibrationBaseline:
- Gas buildings → geyser positions from `baseline.geyserPositions()` (already used in calibrate)
- Town halls → fixed offset from base position (e.g., natural expansion at +20 units)
- Supply/other → fixed offset from base position (doesn't affect economy)
**Alternatives:**
- Add expansion positions to CalibrationBaseline — requires re-recording all baselines
- Parse SC2 map geometry for placement — massive scope, no calibration benefit
**Rationale:** Economy calibration accuracy depends on construction TIME (supply timing), not construction POSITION. Workers don't walk to building sites in calibration playbooks. Gas building positions matter because `workerSimulation.onGasBuildingCompleted()` uses the position for worker assignment, but geyser positions are already in the baseline.
**Trade-offs:** Town hall expansion positions are approximated. Worker transfer timing may differ slightly from SC2 — acceptable for calibration.
**Sources:** CalibrationBaseline.java (geyserPositions, basePosition fields), DualRunCalibrationTest.java:843 (existing geyser position handling)
**Exploration:** quick
**Depends on:** D1
**Status:** captured
