# Decisions — Issue #656: Mineral-line pathfinding

## D1: Fix strategy — three parallel leads, not sequential

**Choice:** Design all three leads (per-patch approach, collision tightening, contention tuning) as one spec and iterate through them until σ ≤ 0.70 and 7+ tick ≤ 2%.
**Alternatives:**
- Sequential lead-by-lead — simpler per-iteration but potentially more iterations to converge
- Lead 1 only — ignores collision and contention variance sources
**Rationale:** The three leads are independent changes addressing three distinct variance sources (structural, random, contention). Designing together avoids re-calibration between iterations.
**Trade-offs:** More complex spec, but each lead is self-contained and independently testable.
**Sources:** CycleOutlierDiagnosticTest cause breakdown (37% waiting, 40% mixed, 23% long walk), PatchAssignmentDiagnosticTest (verified balanced distribution)
**Exploration:** quick
**Status:** captured

## D2: Per-patch approach docking formula

**Choice:** Additive distance-compensating formula: `approach[i] = base_approach + (median_E2E - E2E[i]) * loops_per_unit`. Close patches get more approach (compensating for shorter walks), far patches get less. The formula models the documented physical effect that SC2 workers navigate around the mineral line — path deviation is greater for close/central patches and less for far/edge patches.
**Alternatives:**
- Multiplicative ratio: `approach[i] = global * (median_E2E / E2E[i])` — simpler but over-corrects at extreme distances
- Direct calibration from 8w cadence data per worker — most accurate but only available for one map's patch geometry
**Rationale:** Additive formula preserves the correct units (loops), doesn't over-correct, and the compensation magnitude is physically grounded in the walk speed conversion. Not a compensation constant — it decomposes an existing lumped constant into per-patch components modeling a real geometric effect.
**Trade-offs:** Only calibratable against one map (TorchesAIE_v4) until more 8w baselines are recorded. Approximate — doesn't model the actual obstacle geometry, just uses distance as a proxy.
**Sources:** CycleDecompositionTest (per-worker deviation 0.72–1.10), physics constants §3/§15, PatchAssignmentDiagnosticTest (patch positions)
**Exploration:** quick
**Status:** captured

## D3: Collision displacement — reduce lateral push magnitude at low saturation

**Choice:** Scale collision perpendicular displacement by patch contention: at ≤ 2 workers/patch (12w and below), reduce lateral push to 25% of current. At 3+/patch (16w+), keep full displacement. This reduces random walk extension at normal saturation while preserving the contention model at high saturation.
**Alternatives:**
- Remove displacement entirely for ≤ 2/patch — simpler but abrupt transition at saturation boundary
- Cap cumulative displacement per cycle — complex tracking state
**Rationale:** The 40% "MIXED" outliers have 1-tick waits caused by workers arriving slightly late due to collision deflection adding ~0.3–0.5u to walk distance. Reducing (not eliminating) the push at low saturation reduces this noise while keeping the model for high saturation where collision IS the physical mechanism.
**Trade-offs:** Introduces a saturation-dependent scaling factor. Need to verify 16w/24w baselines don't regress.
**Sources:** CycleOutlierDiagnosticTest (40% MIXED cause), WorkerSimulation.resolveAllMineralCollisions(), physics constants §3 collision model
**Exploration:** quick
**Status:** captured

## D4: Contention — no direct change, measure after Leads 1-2

**Choice:** Do not directly modify the contention model (reroute threshold, lockout, approach+mine timing). Measure σ and 7+ tick after Leads 1-2. The "WAITING_AT_PATCH" outliers (37%) may resolve automatically when per-patch approach creates better cycle alignment between workers sharing a patch.
**Alternatives:**
- Reduce reroute threshold from 22 to 11 loops — faster rerouting
- Adaptive lockout based on patch contention level
**Rationale:** The contention-based outliers are a symptom of mismatched cycle timing between workers on the same patch. Per-patch approach should fix the timing. Changing the reroute threshold risks destabilizing 16w+ scenarios where rerouting IS correct.
**Trade-offs:** If Leads 1-2 don't bring σ below target, a third iteration on contention will be needed.
**Sources:** PatchAssignmentDiagnosticTest (rerouting is working, distribution balanced), wait-then-reroute documentation in physics constants §3
**Exploration:** quick
**Status:** captured
