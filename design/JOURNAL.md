# Design Journal — epic-saturation-mining

### 2026-05-18 · §Next Steps

Removed #141 (saturation mining model) from the Deferred list — implemented this epic. Added two new deferred items: #142 (train-timing precision — `trainTimeInTicks` uses integer outer-tick resolution; real SC2 train times are fractional, causing 1-tick-early completions) and #143 (multi-base mining — `mineralIncomePerTick` assumes one Nexus; needs a base-count parameter once expansion is modelled).

### 2026-05-18 · §Domain Model

`SC2Data` gained a three-tier probe saturation function — `mineralIncomePerTick(int probeCount)` — to replace the previous flat `MINERALS_PER_PROBE_PER_TICK` constant. The model is parameterised by `MINERAL_PATCHES_PER_BASE` (tier width) and `MINERAL_TIER_RATES_PER_TICK[]` (one rate per worker-slot per patch), making the saturation curve swappable by changing two constants. The flat constant was removed entirely. The function enforces a precondition on negative input. Single-base assumption is documented in Javadoc with a TODO reference to #143 for multi-base expansion.

### 2026-05-18 · §Mock Infrastructure

`EmulatedGame.tick()` now delegates mineral accumulation to `SC2Data.mineralIncomePerTick(miningProbes)` instead of the flat multiplication. `miningProbes` semantics changed from "probe count × LOOPS_PER_TICK" to actual probe count; `ReplayValidationHarness` no longer multiplies by `LOOPS_PER_TICK` when syncing probe count from GT. The saturation model reduced the 3-minute mineral delta from ~11,564 to ~850 (10× improvement). Unit count divergence (≤ 2 over 3 minutes) was found to be caused by train-timing precision, not mining — `trainTimeInTicks` uses integer outer-tick resolution while real SC2 train times are fractional ticks; filed as #142.
