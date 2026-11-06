# HANDOFF — quarkmind

## Current Work (2026-11-06)

Branch `issue-534-emulated-game-100-accuracy` — Epic #534: EmulatedGame 100% accuracy.

### Completed This Session

**#535 — Building accuracy: CC/Hatchery expansion undercount** (CLOSED)

Root cause: `ReplaySimulatedGame` (ground truth) did not handle `ID_UNIT_TYPE_CHANGE` tracker events. When buildings morphed (CC→OC, Hatchery→Lair), GT kept the original building type forever — inflating CC/Hatchery counts. EM was correct all along; the measurement was wrong.

Fix: Added `applyUnitTypeChange()` to `ReplaySimulatedGame` and `replaceBuildingType()`/`replaceUnitType()` to `SimulatedGame`.

Results:
- CC 7-min: 15.7% → 100%
- Hatchery 7-min: 64.0% → 100%
- Nexus: unchanged at 100% (confirms fix is morph-specific)
- New gaps now visible: OC (84.9-97.9%), Lair (85.4%) — morph completion timing in EM

### Current Accuracy (post-#535)

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| Units | 100% | 98.5% | 98.5% | 98.0% |
| Buildings | 100% | 98.8% | ~97% | 94.7% |
| Upgrades | 100% | 100% | 100% | 100% |
| CC/Hatch/Nexus | 100% | 100% | 100% | 100% |
| OC | — | 84.9% | 95.6% | 97.9% |
| Lair | — | — | — | 85.4% |

### Queue — Epic #534

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #535 | Building accuracy — CC/Hatchery expansion undercount | M | DONE |
| #536 | Technology spending — defer recording to completion | M | Next |
| #537 | Rate reporting — simulated income instead of formula | S | — |
| #540 | Economy spending undercount (13-20% MAPE) | S | — |
| #544 | Vespene economy spending — always zero in EM | XS | — |
| #541 | Army vespene spending — death timing variance | S | — |
| #538 | Mineral balance — per-replay map geometry variance | M | Blocked by #537 |
| #539 | Vespene balance — income deficit compounds | M | Blocked by #537, #536 |
| #542 | SCV unit count undercount (4%) | S | Blocked by #535 (now unblocked) |
| #543 | foodUsed overcount (4-6%) | XS | Blocked by #542 |

### Key Files

| File | What it does |
|------|-------------|
| `quarkmind-sc2/.../mock/ReplaySimulatedGame.java` | GT — now handles UnitTypeChange |
| `quarkmind-sc2/.../mock/SimulatedGame.java` | Base class — new replaceBuildingType/replaceUnitType methods |
| `quarkmind-sc2/.../emulated/EmulatedGame.java` | The emulator — tick loop, intent handling, economy |
| `quarkmind-sc2/.../emulated/WorkerSimulation.java` | Per-worker mineral/gas trip simulation |
| `quarkmind-sc2/.../emulated/EconomyTracker.java` | Spending classification + rate reporting |
| `quarkmind-sc2/.../replay/TrackerIntentExtractor.java` | Replay→intent pipeline |
| `quarkmind-sc2/.../replay/StandaloneEmulatedGameBaselineTest.java` | Validation gate |
| `quarkmind-sc2/.../replay/BuildingExpansionDiagnosticTest.java` | NEW: building diagnostic |
| `docs/benchmarks/standalone-emulated-game-baseline.md` | Updated accuracy report |

### Next Session Recommendation

Continue with **#536** (Technology spending — defer to completion) or **#542** (SCV unit count undercount, now unblocked by #535). #542 is smaller (S) and builds on the understanding from #535; #536 is independent and affects economy MAPE directly.

## Known Issues

- **DroolsDominanceWeightStrategyTest** — pre-existing ERROR (not from this work)
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
- Epic #516 (closed) — predecessor epic, all GT syncs eliminated
- Epic #534 — current epic, 100% accuracy target
