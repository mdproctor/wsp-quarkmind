# HANDOFF — quarkmind

## Current Work (2026-11-06)

Branch `issue-534-emulated-game-100-accuracy` — Epic #534: EmulatedGame 100% accuracy.

### Background — What was achieved in Epic #516

Epic #516 (Standalone EmulatedGame) eliminated all 12 GT syncs from EmulatedGame. The emulator now runs standalone — it produces its own mineral income, gas income, spending tracking, supply, and worker counts without being told the answers by ground truth.

The approach was simulation-first: individual workers walk to individual mineral patches, mine, carry cargo back, deposit. Gas workers walk to geysers, enter, harvest, return. No formulas drive the production model — the income emerges from the physics. Formulas exist only as regression checks.

**Key changes from #516:**
- **Multi-occupant mineral patches** (`MAX_MINERS=2`): the old single-occupant model forced workers to busy-wait at shared patches, cutting throughput in half. Real SC2 workers interleave — two per patch is optimal. This doubled mining throughput.
- **Mining duration calibration** (46→16 game loops): with multi-occupant, the cycle time per trip was recalibrated. 16 workers now produce 846 minerals/min (SC2 target ~930, 91%).
- **Starting unit cost fix**: tracker-driven units born at loop 0 (starting workers) are free — no mineral deduction. Initial minerals set to 50.
- **Gas building destruction**: `onGasBuildingDestroyed()` removes gas slots and returns workers to minerals.
- **Worker lifecycle**: stopped consuming Drone tags in TrackerIntentExtractor — UnitDied events now flow naturally for Drone→building morphs.
- **Spending tracking**: records at order time for builds, completion time for training/research. Death subtraction for units and buildings.

**What worked well:**
- Simulation-based income: physically correct, scales with map geometry
- Diagnostic tests before fixes: `VespeneSpendingDiagnosticTest`, `GasIncomeDiagnosticTest`, `WorkerSimulationTest#mineralIncomeRate_16workers_matchesSC2` — measure first, fix second
- TrackerIntentExtractor as the replay-to-intent pipeline — handles all unit/building lifecycle events
- The `StandaloneEmulatedGameBaselineTest` as the single validation gate (118 oracle replays, 4 checkpoints)

**What didn't work (tried and reverted):**
- Formula-based gas income for balance: overcollects early, causes mineral starvation for Zerg (Hatchery regression to 64%). The formula is correct as a reference but wrong as the production model.
- Per-tick gas slot rebalancing: pulls workers from minerals to gas every tick, starves mineral income. Too aggressive.
- Worker-to-gas on produce: routes new workers to undersaturated gas slots instead of minerals. Marginal improvement (25.3%→24.9%) but causes Hatchery regression.
- Reclassifying building spending categories: 7 hypotheses tested (#533) — defensive→army, production→army, Zerg production, combinations. Every one increased MAPE. Classification is already correct.

### Current Accuracy (post-#516, pre-#534)

| Field | 1-min | 3-min | 5-min | 7-min |
|-------|-------|-------|-------|-------|
| Units | 100% | 98.5% | 99.0% | 99.2% |
| Buildings | 100% | 96.0% | 94.9% | 94.9% |
| Upgrades | 100% | 100% | 100% | 100% |
| foodMade | 1.0% | 1.5% | 2.5% | 3.1% |
| foodUsed | 4.1% | 3.4% | 6.0% | 4.0% |
| workersActiveCount | 3.8% | 3.3% | 3.8% | 3.2% |
| mineralsCollectionRate | 21.5% | 11.9% | 22.8% | 29.3% |
| vespeneCollectionRate | 237% | 26.1% | 26.1% | 38.6% |
| mineralsCurrent | 55% | 58% | 86% | 101% |
| vespeneCurrent | 245% | 47% | 68% | 105% |
| mineralsUsedCurrentArmy | 0% | 5.2% | 9.9% | 7.9% |
| mineralsUsedCurrentEconomy | 19.5% | 16.7% | 13.8% | 12.2% |
| mineralsUsedCurrentTechnology | 0% | 46% | 39% | 37% |
| vespeneUsedCurrentArmy | 0% | 4.5% | 19.7% | 10.9% |
| vespeneUsedCurrentEconomy | 0% | 0% | 100% | 100% |
| vespeneUsedCurrentTechnology | 0% | 140% | 65% | 57% |

### Queue — Epic #534

Ordered by recommended execution:

| # | Title | Scale | Blocked by |
|---|-------|-------|------------|
| #535 | Building accuracy — CC/Hatchery expansion undercount | M | — |
| #536 | Technology spending — defer recording to completion | M | — |
| #537 | Rate reporting — simulated income instead of formula | S | — |
| #540 | Economy spending undercount (13-20% MAPE) | S | — |
| #544 | Vespene economy spending — always zero in EM | XS | — |
| #541 | Army vespene spending — death timing variance | S | — |
| #538 | Mineral balance — per-replay map geometry variance | M | #537 |
| #539 | Vespene balance — income deficit compounds | M | #537, #536 |
| #542 | SCV unit count undercount (4%) | S | #535 |
| #543 | foodUsed overcount (4-6%) | XS | #542 |

### Key Files

| File | What it does |
|------|-------------|
| `quarkmind-sc2/.../emulated/EmulatedGame.java` | The emulator — tick loop, intent handling, economy |
| `quarkmind-sc2/.../emulated/WorkerSimulation.java` | Per-worker mineral/gas trip simulation |
| `quarkmind-sc2/.../emulated/MineralPatch.java` | Patch model — MAX_MINERS=2, depletion |
| `quarkmind-sc2/.../emulated/SimulatedWorker.java` | Worker state — MINING_DURATION_LOOPS=16 |
| `quarkmind-sc2/.../emulated/GasSlot.java` | Gas geyser model — MAX_WORKERS=3 |
| `quarkmind-sc2/.../emulated/EconomyTracker.java` | Spending classification + rate reporting |
| `quarkmind-sc2/.../replay/TrackerIntentExtractor.java` | Replay→intent pipeline |
| `quarkmind-sc2/.../replay/StandaloneEmulatedGameBaselineTest.java` | Validation gate |
| `docs/benchmarks/standalone-emulated-game-baseline.md` | Current accuracy report |

### Next Session Recommendation

Start with **#535** (CC/Hatchery building accuracy). It's the most visible gap (CC at 15.7% accuracy at 7-min) and unblocks #542 (SCV count). The approach: trace which expansion BuildIntents TrackerIntentExtractor produces vs how many appear in GT. Likely cause: expansions come via UnitInit (BuildIntent) which deducts minerals and may be rejected, while starting bases come via UnitBorn (BuildingBornIntent) which always succeeds.

## Known Issues

- **DroolsDominanceWeightStrategyTest** — pre-existing ERROR (not from this work)
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy
- `docs/blog/2026-11-06-mdp01-the-workers-were-right-the-mine-was-wrong.md` — mining throughput blog
- `docs/blog/2026-11-05-mdp02-the-floor-you-cant-fix.md` — spending structural floor
- `docs/blog/2026-11-03-mdp01-every-mineral-patch-is-not-the-same.md` — mineral patch differentiation
- Epic #516 (closed) — predecessor epic, all GT syncs eliminated
- Epic #534 — current epic, 100% accuracy target
