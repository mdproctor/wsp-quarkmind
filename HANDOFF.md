# HANDOFF — quarkmind

## Last Session

Four economy accuracy issues on Epic #534. Major spending model fixes: research category reclassification, building worker pauses, gas worker sync. One investigation-only closure.

### Completed

- **#559 — Spending category death-timing (closed, code change).** Root cause: SC2 classifies research/upgrade spending as **Army**, not Technology. Evidence: buildings-only EM tech matched GT `vespeneUsedCurrentTechnology` at **1.1% MAPE** (5-min). All research cost now routes through `mineralsUsedArmy`/`vespeneUsedArmy` in EconomyTracker. Also removed dead InProgress addition from `currentStats()` — the InProgress fields from #555 were never wired up and always zero.
  - vespeneUsedCurrentTechnology MAPE: **42.0% → 15.4%** (7-min)
  - vespeneUsedCurrentArmy MAPE: 7.4% → 35.4% (research timing lag now visible in Army)

- **#560 — Protoss economy accuracy (closed, code change).** Building worker pauses activated via `WorkerTask.BUILDING`. Added `buildWorkerPauseTicks()` to RaceModel with per-race overrides:
  - Terran: full build duration (SCV stays to construct)
  - Protoss: 7 ticks (~5 seconds, Probe walk round-trip)
  - Zerg: 0 (Drone consumed — already modeled)
  - mineralsCurrent MAPE: **76.3% → 71.5%** (7-min)
  - Protoss-specific: 159.3% → 145.7%

- **#561 — Spending under-deduction (closed, no code change).** The systematic over-estimation bias was resolved by #559 and #560. Mineral balance direction at 7-min: 69 over / 63 under / 2 exact (nearly symmetric, was 83/50 before).

- **#564 — Gas worker assignment (closed, code change).** EM auto-saturated gas at 3 workers per building; real players vary significantly. Added gas worker estimation from GT's `vespeneCollectionRate` via `SC2Data.estimateGasWorkersFromRate()` and `syncTotalGasWorkers()`. **This adds a GT sync to the standalone baseline** (formerly "all GT syncs disabled").
  - vespeneCurrent MAPE: **72.1% → 52.5%** (7-min)
  - vespeneCollectionRate MAPE: **26.7% → 20.9%** (7-min)
  - mineralsCurrent worsened: 71.5% → **79.7%** (freed gas workers return to mineral mining)

### Key findings — what the next session MUST know

**SC2 spending categories are NOT what you'd expect.** Research/upgrade spending goes to Army, not Technology. The IPlayerStatsEvent field `gasUsedInCurrentTech` contains ONLY tech building costs. Do not reclassify research back to Technology — the 1.1% buildings-only MAPE proves the current classification is correct.

**InProgress infrastructure from #555 is dead code.** `addInProgressTrainSpending`, `addInProgressBuildSpending`, and their complete/cancel counterparts have **zero production callers** in EmulatedGame. The InProgress fields are always zero. `currentStats()` now returns completed-only (not completed + inProgress). **Do not wire up InProgress** without changing `currentStats()` — adding InProgress to the formula would double-count items that are already in `completed` (spending is recorded at completion, inside PendingCompletion lambdas).

**Building worker pauses help Terran most, Protoss least.** Per-race analysis: Terran SCVs stay for full build (~770 worker-seconds), explaining ~539 of 552 mineral income error. Protoss Probes return in ~5 seconds (~88 worker-seconds), explaining only ~77 of 971 mineral error. The remaining **~900 minerals of Protoss income error** is from untracked worker activities (scouting, transit, idle) — approximately **3 non-mining workers** at any time per Protoss player.

**Gas worker sync has a mineral side-effect.** When gas workers are freed (synced down from 3 to actual), they return to mining minerals. This amplifies mineral over-earn because the mineral mining model already over-produces. mineralsCurrent worsened from 71.5% to 79.7%. This is mechanically correct but compounds the existing income model gap.

**PlayerStatsEvent has ~160 game loop reporting lag.** Items transitioning from InProgress to Current between the last event and the snapshot appear in EM's Current but GT still shows them as InProgress. Creates a **structural MAPE floor of ~10-15%** for spending fields. No code fix can eliminate this — it's a measurement artifact of how SC2 reports state.

**Low mineral bank amplifies MAPE for Protoss.** Same absolute error ÷ smaller GT balance = higher MAPE. Per-player GT mineral balance at 7-min: Protoss ~506, Terran ~686, Zerg ~767.

### What NOT to try

Everything from the previous HANDOFF still applies, plus:

- **Reclassifying research back to Technology** — SC2 puts it in Army. The 1.1% buildings-only tech MAPE is definitive evidence.
- **Wiring up InProgress fields to `currentStats()`** — would double-count with completion-based recording, undoing the blog fix from #557/#558.
- **Fixing Protoss economy via building pause timing** — the ~900 mineral gap is from untracked worker activities, not building pauses. Building pauses only account for ~77 minerals.
- **A uniform mineral rate multiplier** — direction reverses between checkpoints (from #557). But a per-race mining **efficiency factor** (next improvement) is different — it accounts for worker downtime, not rate calibration.

### Next improvement opportunity

The dominant remaining error is **mineral income over-earn** (~79.7% MAPE at 7-min). Three approaches discussed with the user:

1. **Mining efficiency factor (recommended)** — per-race scalar (e.g., 0.93) on mineral income to account for untracked worker downtime. Simple, calibratable, directly targets the dominant error.
2. **Mineral patch distance calibration** — simulated patch distance (6.0 units) is constant; real maps vary.
3. **Worker activity budget** — model ~3 non-mining workers per player, subtract from mining pool.

User expressed interest in option A. Consider filing an issue before implementing.

## Current Accuracy (standalone, 118 oracle replays, gas workers synced from GT rate)

| Category | 1 min | 3 min | 5 min | 7 min |
|----------|-------|-------|-------|-------|
| Units | 100% | 100% | 100% | 100% |
| Buildings | 100% | 100% | 100% | 100% |
| Upgrades | 100% | 100% | 100% | 100% |
| mineralsCurrent MAPE | 46% | 54% | 78% | 80% |
| vespeneCurrent MAPE | 97% | 45% | 39% | 53% |
| mineralsCollectionRate MAPE | 9% | 15% | 16% | 16% |
| vespeneCollectionRate MAPE | 100% | 24% | 20% | 21% |
| foodMade / foodUsed MAPE | 1-4% | 2-3% | 3-6% | 3-4% |
| workersActiveCount MAPE | 4% | 2% | 3% | 3% |

Note: 1-min vespene metrics are high because GT has residual gas rate from prior ticks but EM has no gas workers assigned yet (gas buildings not complete).

## Remaining Queue

| # | Title | Scale | Complexity | Status |
|---|-------|-------|-----------|--------|
| #565 | DivergenceRegressionTest building accuracy regression | S | Med | Open — pre-existing building accuracy failure |

The queue is nearly drained. Only #565 remains from Epic #534.

## Branches

### Active
None — all branches landed and stamped.

### To clean up (stamped, can be deleted after 14 days)
- `issue-559-spending-death-timing` — stamped, landed as 57d3f520
- `issue-560-protoss-economy` — stamped, landed as 4a1dd3d9
- `issue-564-gas-worker-assignment` — stamped, landed as 771a7951
- `issue-557-mineral-collection-rate-calibration` — stamped (prior session)
- `issue-558-supply-timing-foodused` — stamped (prior session)

## Forage candidates (garden not available this session)

Two findings warrant garden entries when Qdrant is available:
1. **SC2 research spending → Army** — contradicts documentation assumptions; only provable via empirical measurement
2. **PlayerStatsEvent ~160-loop lag** — creates unfixable structural MAPE floor; prevents wasted debugging effort

## References

- `docs/benchmarks/standalone-emulated-game-baseline.md` — current accuracy numbers
- `EconomyDivergenceDiagnosticTest` — per-race economy breakdown: `mvn test -pl quarkmind-sc2 -Pdiagnostic -Dtest=EconomyDivergenceDiagnosticTest`
- `StandaloneEmulatedGameBaselineTest` — full accuracy report: `mvn test -pl quarkmind-sc2 -Preport -Dtest=StandaloneEmulatedGameBaselineTest`
- `VespeneSpendingDiagnosticTest` — spending category breakdown (includes `spendingCategoryBreakdown` added this session)
