# HANDOFF — quarkmind

## Last Session

Working on #386 — Calibrate EmulatedGame physics from tournament ground truth. Phase 1 partially complete. Branch: `issue-386-calibrate-emulated-physics`. Issue #387 paused on the stack (depends on #386).

### Completed

1. **EconomyTracker gross income** — Replaced net balance delta with gross income tracking. `mineralsCollectionRate` now reflects actual minerals mined, not income-minus-spending. `mineralsCurrent` error dropped from 71.7% to 16.5%.

2. **EGG phantom filtering** — Excluded `UnitType.EGG` from accuracy baseline unit counts. Removes ~73 phantom units from emulated count.

3. **Mineral sync A/B test** — Added mineral sync to harness (same as vespene), ran A/B. Result: 5% unit accuracy regression (71.7% with sync vs 76.8% without). Reverted. Key finding: mineral gate is NOT the root cause of 3% upgrade accuracy.

4. **MULE verification** — Pipeline exists end-to-end (AbilityMapping → MuleCalldownIntent → TerranRaceModel.onCalldown → spawn). Tests pass. Still shows 0% at baseline — likely OC energy or intent timing issue.

### Baseline After Phase 1 (5-minute checkpoint, 118 oracle replays)

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| Units | 78.7% | 76.8% | ≥80% |
| Buildings | 100% | 100% | ≥97% ✓ |
| Upgrades | 3.4% | 2.8% | >30% |
| mineralsCurrent error | 71.7% | 16.5% | — |
| collectionRate error | 95.9% | 93.6% | ≤50% MAPE |

### Key Findings

- **Unit accuracy ~2% drop** is from EGG filter removing inflation — the old 78.7% was inflated by phantom EGGs masking under-production.
- **Economy collection rate** still 93.6% wrong. SC2's `mineralsCollectionRate` (208K aggregate) uses a different formula/unit than our gross income delta (13K). Needs reverse-engineering from oracle PlayerStats.
- **Upgrade 2.8%** — mineral rejection disproved by A/B test. Next hypothesis: building tag mismatch in `handleResearch()` or `ResearchIntent` extraction gap.
- **MULE 0%** — calldown pipeline works in unit tests. Harness may not generate MuleCalldownIntents, or OC energy divergence blocks calldowns.
- **Queen 17.2%** — extraction pipeline under-counts (145 GT vs 25 emulated). Extraction issue, not EmulatedGame physics.

## Immediate Next Step

Continue #386 — three investigation threads needed:

1. **Economy collection rate formula** — Extract PlayerStats from oracle replays at multiple checkpoints, compare against EmulatedGame gross income to reverse-engineer what SC2's `mineralsCollectionRate` actually measures.

2. **Upgrade root cause** — Add debug logging to `handleResearch()` during a baseline run to trace why research intents are rejected. Check building tag format (replay extraction tags vs harness-injected tags).

3. **Unit accuracy +3.2pp** — Top targets: Queen extraction (120 units missing), worker shortfall (~20% each race), MULE (69 missing).

## References

- `specs/issue-386-calibrate-emulated-physics/` — design spec and decisions
- `plans/2026-10-11-calibrate-emulated-physics.md` — implementation plan (Batches 1-3 done, 2 remaining)
- `docs/benchmarks/emulated-game-accuracy-baseline.md` — current baseline data
- EconomyTracker.java:49-69 — gross income tracking (new)
- EmulatedGame.java:150-170 — income loop with recordIncome() wiring
- ReplayValidationHarness.java:102 — vespene sync (mineral sync reverted)
- EmulatedGame.java:764-806 — handleResearch() with rejection gates
- #387 — paused on stack, depends on #386
