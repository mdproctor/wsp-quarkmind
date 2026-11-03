# HANDOFF — quarkmind

## Last Session (2026-11-03)

Branch `issue-501-map-mineral-patch-init` — implemented map-specific mineral patch initialisation for WorkerSimulation (#501).

### Commits

- `a110bc28` — `feat(#501): map-specific mineral patch initialisation for WorkerSimulation`
  - `MineralPatch` accepts initial amount (was hardcoded 1500). Constants: CLOSE=1500, FAR=750.
  - `WorkerSimulation.onTownHallCompleted` overload accepts `List<Resource>` for replay-driven patches.
  - Default distribution: 4 close (1500) + 4 far (750) per base.
  - `EmulatedGame.registerTownHallWithPatches()` routes all 6 callsites; checks for nearby Resource data first.
  - `ReplayValidationHarness.runInternal()` syncs mineral patch resources from ground truth before reset.
  - 3 new tests in `WorkerSimulationTest`: mixed amounts, explicit patch data, depletion comparison.
  - All 3,264 tests pass including playbook calibration and replay validation.

- `322d3e82` — `wip(content): diary — Every Mineral Patch Is Not the Same`
  - Blog entry explaining why mineral patch calibration matters for game AI economy simulation.
  - Two SVG diagrams: mineral-patch-layout.svg, mineral-data-flow.svg.

### Issues filed

- #502 — Mineral spending reconciliation for mineralsCurrent accuracy (M/Med)
- #503 — Calibrate gas harvesting physics for vespeneCollectionRate accuracy (M/Med)
- #504 — Supply (foodUsed) reconciliation from ground truth (S/Med)
- #505 — Rich mineral patch harvest rate, 7 per trip vs 5 (XS/Low)

All four added as children of epic #481. Epic body updated on GitHub with current accuracy table and all new issues.

## Epic #481 — Calibration State

**22 of 28 closed.** Production accuracy solved. Economy is the remaining frontier.

| Category | 5-min accuracy |
|----------|---------------|
| Units | 100.0% |
| Buildings | 100.0% |
| Upgrades | 100.0% |
| mineralsCollectionRate | 4.9% error |
| vespeneCollectionRate | 35.0% error |
| mineralsCurrent | 396.7% error |
| foodUsed | 35.1% error |

### Open issues (priority order)

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #501 | Map-specific mineral patch init | S/Med | Branch exists, implementation done, needs work-end |
| #505 | Rich mineral patch harvest rate | XS/Low | Small follow-on from #501 — gold patches carry 7 not 5 |
| #502 | Mineral spending reconciliation | M/Med | Biggest numeric gap (396.7%) — harness syncs vespene but not minerals |
| #503 | Gas harvesting physics calibration | M/Med | 35% error — needs per-worker gas trip sim like minerals |
| #504 | foodUsed reconciliation | S/Med | 35% error — supply accounting lags on unit death |
| #402 | Extend BalanceData | L/Med | Phase 1 item, not blocking accuracy |

## Immediate Next Step

Close #501 via work-end. Then #505 (XS — same area, quick follow-on). Then #502 for the biggest accuracy improvement.

## Known Issues

- **Reconstituted data inconsistency**: blizzard_ladder_4.9.3 has mixed calibrated/uncalibrated files from partial export. Needs clean re-export before retraining.
- **blizzard_ladder_4.9.2 reconstituted data deleted** — 928K replays, needs re-export when disk space permits.
- **Ground truth alignment gap**: OnnxClassificationCalibrationTest mid-game accuracy limited because `deriveGroundTruth()` produces archetypes not in model vocabulary.
- **Pre-existing:** `casehub-platform-agent-langchain4j:0.4-SNAPSHOT` POM invalid.
- **Pre-existing test failures:** `test_coarse_label_map_no_hierarchy`, `test_ingest_spawningtool` (5 tests), `test_coverage_gaps` (2 tests).

## References

- `docs/blog/2026-11-03-mdp01-every-mineral-patch-is-not-the-same.md` — blog entry on mineral patch calibration
- `docs/benchmarks/tracker-driven-accuracy-baseline.md` — accuracy numbers
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/MineralPatch.java` — close/far constants
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/emulated/WorkerSimulation.java` — Resource overload
