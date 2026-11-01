# Targeted Emulator Calibration — Design Spec

**Issue:** casehubio/quarkmind#484
**Parent:** #481 (Epic: EmulatedGame playbook calibration)
**Follow-up from:** #405 (calibrated multiplication assessment)

## Problem

The tracker-driven EmulatedGame achieves 99.8% unit accuracy at 5min across 118 oracle replays (v4.9.3), but the remaining 0.2% gap (16 units out of 6807) is concentrated in Terran production:

| Unit Type | 5min GT | 5min Em | Delta | % of gap |
|-----------|---------|---------|-------|----------|
| Marine | 372 | 365 | -7 | 44% |
| Hellion | 52 | 50 | -2 | 13% |
| Stalker | 51 | 49 | -2 | 13% |
| WidowMine | 28 | 27 | -1 | 6% |
| Immortal | 10 | 9 | -1 | 6% |
| Medivac | 9 | 8 | -1 | 6% |
| SiegeTank | 33 | 32 | -1 | 6% |
| Reaper | 36 | 35 | -1 | 6% |

At 7min, WidowMine degrades to 87.9% (29/33), indicating an accumulating error. The Terran divergences are likely related — Barracks/Factory/Starport production with Reactor/TechLab addons is the most complex production mechanic in SC2.

#405 established that the right path is fixing emulator physics divergences, not post-hoc calibration of extraction heuristics (compensating errors are structural — SCV over-count balances MULE under-count).

## Goal

1. Systematically diagnose and fix Terran emulator production divergences
2. Raise tracker-driven unit accuracy to ≥99.9% at 5min (close ≥9 of 16-unit gap)
3. Expand the complete replay dataset by ingesting SC2EGSet (17,930 tournament replays)
4. Document the divergence → fix → validate cycle as a repeatable process

## Scope

**In scope:**
- General-purpose replay divergence scanner and tracer (replaces bespoke per-unit diagnostic tests)
- All Terran unit divergences: Marine, Hellion, WidowMine, SiegeTank, Medivac, Reaper
- EmulatedGame physics fixes for root causes identified by the tracer
- SC2EGSet download and Python pipeline ingestion
- Path B (targeted playbooks against real SC2) is a follow-up issue, not in scope

**Out of scope:**
- Non-Terran divergences (Stalker, Immortal — small absolute deltas, separate investigation)
- Economy tracking gaps (mineralsCollectionRate 92%+ error — separate problem)
- Java-side SC2EGSet ingestion (IEM10JsonSimulatedGame race parameterisation — follow-up)

## Architecture

### Two-phase diagnosis (D1)

**Phase 1: ReplayDivergenceScanner** (`@Tag("report")`, plain JUnit, package `io.quarkmind.sc2.replay`)

Iterates all 118 oracle replays (Terran players only), runs each through `ReplayValidationHarness.run()`, and collects per-replay per-unit-type deltas at the 1/3/5/7min checkpoints. Outputs a ranked table sorted by absolute delta descending:

```
Replay                     | Player | UnitType   | 5min GT | 5min Em | Delta
3307d878e8...              | 1      | MARINE     | 24      | 19      | -5
a92f1c4b...                | 2      | HELLION    | 8       | 5       | -3
```

Answers: "which replays have the worst divergences for which unit types?"

**Phase 2: ReplayDivergenceTracer** (`@Tag("diagnostic")`, plain JUnit, package `io.quarkmind.sc2.replay`)

Takes a specific replay hash + player ID (identified by the scanner). Runs `ReplayValidationHarness` with an `IntentExecutionListener` registered on `EmulatedGame`. For every tick where the target unit type count changes (in GT or Em), logs:

- Tick number and game loop
- Intent applied (if any): unit type, building tag, building type
- Outcome: trained / rejected (with reason: building not found, building incomplete, queue full, supply blocked)
- Current production state: all buildings of the relevant producer type, their completion status, addon type (Reactor/TechLab/none)
- Running GT vs Em count for the target unit type

Output is a causal timeline: "tick 145: TrainIntent(MARINE, Barracks r-82) → rejected: building incomplete. GT=12, Em=11. Gap opened here."

### IntentExecutionListener (D2)

A lightweight callback interface added to `EmulatedGame`:

```java
public interface IntentExecutionListener {
    void onTrainOutcome(int tick, TrainIntent intent, String buildingTag,
                        boolean success, String rejectionReason);
}
```

- `EmulatedGame` gets a `setIntentExecutionListener(listener)` method
- Fires at every `processTrainIntent()` call site — success or rejection
- Null-safe: no listener registered = no overhead
- Existing tests and harness are unaffected — the listener is opt-in
- The tracer sets the listener on the `EmulatedGame` instance before calling `harness.run()`
- No changes to `ReplayValidationHarness` itself

### Emulator physics fixes

For each Terran unit type the tracer identifies as divergent:
1. Run the tracer on the worst-case replay from the scanner
2. Read the causal timeline to identify the root cause
3. Fix the EmulatedGame physics (building routing, reactor detection, queue management)
4. Re-run `TrackerDrivenAccuracyBaselineTest` to verify improvement
5. Re-run `DualRunCalibrationTest` self-baselines to verify no regression

**Suspected root causes by unit type:**

- **Marine**: Barracks/Reactor routing — Reactor detection lag causing EmulatedGame to miss a train cycle
- **Hellion / WidowMine**: Factory/Reactor state detection — same pattern, accumulating over time for WidowMine
- **SiegeTank / Medivac / Reaper**: Small absolute deltas (1 each), likely building completion timing or edge cases in `TrackerIntentExtractor.lookupBuildingTag()`

Per protocol `calibration-milestones-are-waypoints`: if we reach 99.9% but the tracer reveals deeper systematic issues (e.g., compensating errors), we assess and replan rather than declaring victory.

### SC2EGSet dataset expansion (D3)

**Download:** Full SC2EGSet from Zenodo (17,930 tournament replays, CC BY 4.0). Nested ZIP format (BZip2 outer, DEFLATE inner).

**Ingestion:** Existing Python pipeline `sc2egset_extractor.py` handles the format and both player perspectives. Output to `quarkmind-classifier/data/sc2egset/`. Expands complete dataset from 632 to ~18K replays.

**Limitation:** SC2EGSet replays cannot be used for Java-side EmulatedGame accuracy validation until `IEM10JsonSimulatedGame` is parameterised for non-Protoss perspectives (follow-up work).

## Testing

| Test | Type | Purpose |
|------|------|---------|
| `ReplayDivergenceScanner` | `@Tag("report")`, plain JUnit | Per-replay per-unit-type delta table across oracle replays |
| `ReplayDivergenceTracer` | `@Tag("diagnostic")`, plain JUnit | Per-tick causal timeline for a specific replay + unit type |
| `IntentExecutionListenerTest` | Plain JUnit | Listener fires on success and rejection |
| `TrackerDrivenAccuracyBaselineTest` | `@Tag("report")` (existing) | Regression gate — verify ≥99.9% at 5min after fixes |
| `DualRunCalibrationTest` | Plain JUnit (existing) | Self-baseline regression — playbook calibration unbroken |
| `SC2PlaybookCalibrationTest` | Plain JUnit (existing) | Playbook assertion regression |
| SC2EGSet Python ingest | pytest | Feature extraction on new tournament ZIPs |

No `@QuarkusTest` needed — all new tests are plain JUnit driving EmulatedGame directly.

## Acceptance Criteria

- [ ] ReplayDivergenceScanner produces per-replay per-unit-type delta table for Terran players
- [ ] ReplayDivergenceTracer produces causal timeline for a specific replay + unit type
- [ ] IntentExecutionListener added to EmulatedGame (null-safe, opt-in)
- [ ] At least one Terran emulator divergence root-caused and fixed
- [ ] Tracker-driven unit accuracy ≥99.9% at 5min (up from 99.8%)
- [ ] SC2EGSet downloaded and ingested through Python pipeline
- [ ] Divergence → fix → validate cycle documented as a repeatable process

## Deliverable Order

1. IntentExecutionListener on EmulatedGame (enables D2)
2. ReplayDivergenceScanner (enables triage)
3. ReplayDivergenceTracer (enables root-cause analysis)
4. Run scanner → identify worst Terran divergences
5. Run tracer → diagnose root causes
6. Fix EmulatedGame physics → validate with baseline tests
7. SC2EGSet download + Python ingest (independent, can run in parallel with 4-6)

## References

- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/TrackerDrivenAccuracyBaselineTest.java` — aggregate accuracy report pattern
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/HellionDeficitDiagnosticTest.java` — per-replay diagnostic pattern
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/replay/ReactorStateAtIntentTest.java` — per-tick reactor state tracing
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/ReplayValidationHarness.java` — tick loop orchestrator
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/replay/TrackerIntentExtractor.java` — tracker events → TimedIntent pipeline
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/EmulatedGame.java` — system under test
- `quarkmind-sc2/src/test/java/io/quarkmind/sc2/playbook/DualRunCalibrationTest.java` — playbook calibration infrastructure
- `quarkmind-classifier/src/sc2egset_extractor.py` — SC2EGSet JSON extraction pipeline
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — current accuracy numbers
- `docs/protocols/calibration-milestones-are-waypoints.md` — calibration milestone discipline
- casehubio/quarkmind#405 — prior work establishing "fix emulator, not extraction"
- casehubio/quarkmind#481 — parent epic
