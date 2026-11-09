## D1: Per-worker cycle time audit + efficiency factors

**Choice:** Both — audit per-worker cycle time AND apply efficiency factors
**Alternatives:**
- Cycle audit only — finds root cause but leaves MAPE on the table during investigation
**Rationale:** The efficiency factors were reverted (#566) because they were pre-sub-tick. The sweep now shows factors need to go above 1.0 (sweep only goes to 1.0, but MAPE monotonically decreases). Both the root cause investigation and the pragmatic fix are needed.
**Trade-offs:** None — the audit informs the factor calibration
**Sources:** MiningEfficiencyCalibrationTest output (all races optimal at 1.0, sweep doesn't go higher), EconomyDivergenceDiagnosticTest (69% under-earning)
**Exploration:** quick
**Status:** captured

## D2: Factor location — RaceModel per-race

**Choice:** Per-race factor in RaceModel
**Alternatives:**
- WorkerPhysicsConfig single value — simpler but forces all races to same factor when diagnostic shows they differ
**Rationale:** Each race has different unmodeled mechanics (Protoss warp interruption, Zerg drone sacrifice, Terran building pause). RaceModel already handles race-specific constants.
**Trade-offs:** Three config values instead of one
**Sources:** EconomyDivergenceDiagnosticTest per-race breakdown (Zerg 785.9 vs Protoss 521.3 mineral income error)
**Exploration:** quick
**Status:** captured

## D3: Calibration method — playbook-first, oracle-validated

**Choice:** Write targeted calibration playbooks against real SC2 for precise per-worker rate measurement, validate against oracle replays
**Alternatives:**
- Oracle-only factor sweep — fast but noisy (confounding variables: building, scouting, combat)
- Oracle + extended sweep to 1.30 — better than oracle-only but still noisy
**Rationale:** Playbooks give controlled, complete data. 12 workers on 1 base with no buildings isolates per-worker rate from all other variables. PlaybookSC2RecorderTest and DualRunCalibrationTest infrastructure already exists.
**Trade-offs:** Requires SC2 to be running for the recording step. The oracle sweep can validate the playbook results without SC2.
**Sources:** SC2PlaybookCalibrationTest, PlaybookSC2RecorderTest, DualRunCalibrationTest (existing infrastructure), user domain expertise
**Exploration:** quick
**Depends on:** D1 (cycle audit confirms what to calibrate), D2 (per-race factors need per-race playbooks)
**Status:** captured
