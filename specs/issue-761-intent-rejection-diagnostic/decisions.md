## D1: Constant classification framework

**Choice:** Four-category framework: Data editor, Observable, Emergent, Unknown
**Alternatives:**
- Binary measured/unmeasured — too coarse, doesn't distinguish "should be a constant" from "should emerge from physics"
- Per-constant ad-hoc assessment — no reusable framework
**Rationale:** Each category has a distinct fix path. Data editor values are looked up. Observable values are measured from SC2. Emergent values should be deleted once physics is correct. Unknown values need investigation before anything else.
**Trade-offs:** Some constants may straddle categories (approach animation could be Observable or Emergent depending on what it actually models)
**Sources:** WorkerPhysicsConfig.java constants, sc2-physics-constants.md confidence levels
**Exploration:** quick
**Status:** captured

## D2: Measurement mechanism

**Choice:** Dedicated high-resolution SC2 profiler (short controlled windows at per-loop resolution)
**Alternatives:**
- Extend PlaybookSC2RecorderTest with per-loop stepping — slow (6710 loops for a full game), generates large files, full-game context confounds variable isolation
**Rationale:** Measuring physical constants requires isolating variables. Short controlled experiments (50-100 ticks at a fixed worker count) isolate the variable better than full-game recordings.
**Trade-offs:** May miss interactions that only appear in longer runs; new code path separate from existing recorder
**Sources:** PlaybookSC2RecorderTest.java, SC2 bot API RequestStep documentation
**Exploration:** quick
**Status:** captured

## D3: Phased execution order

**Choice:** Phase 1 (zero contention, 8w) → Phase 2 (contention tiers) → Phase 3 (startup, gas) → Phase 4 (delete/validate Emergent)
**Alternatives:**
- All constants in parallel — wastes measurement effort on constants that may become unnecessary after earlier phases
- Hardest first — most noise, least reliable measurements
**Rationale:** Phase 1 constants are the foundation: mining, approach, deposit, cycle decomposition. With these measured, Phase 2 contention measurements have less noise. Phase 3 edge cases become isolatable. Phase 4 Emergent constants may self-resolve.
**Trade-offs:** Slower overall; Phase 4 constants remain patched until Phases 1-3 complete
**Sources:** sc2-physics-constants.md saturation curve analysis, session sweep data showing non-monotonic results from noisy constants
**Exploration:** quick
**Status:** captured

## D4: Profiler architecture

**Choice:** WorkerPhysicsProfilerTest (@Tag("sc2")) with per-loop stepping, WorkerPhysicsAnalyser for state inference and constant derivation
**Alternatives:**
- EM-side WorkerPhaseListener as measurement standard, spot-checked against SC2 — circular (measuring EM to calibrate EM)
**Rationale:** Direct SC2 observation is the only honest measurement. The EM's internal phase tracking is useful for comparison but cannot be the source of truth.
**Trade-offs:** Requires SC2 client running; per-loop stepping is slow for long windows
**Sources:** ObservationTranslator.java (per-unit data available from SC2 API), WorkerPhaseListener.java (EM-side phase tracking for comparison)
**Exploration:** quick
**Status:** captured

## D5: Validation rule

**Choice:** Never revert a measured constant to a swept value. If measured physics produces worse aggregate accuracy, investigate what other mechanism is compensating.
**Alternatives:**
- Revert to swept if regression detected — preserves accuracy but reinforces the compensation treadmill
**Rationale:** A measured constant IS correct (it's what SC2 does). If aggregate results worsen, another constant was swept to compensate for the wrong value. Reverting hides the real error.
**Trade-offs:** May temporarily worsen accuracy during the transition; requires discipline to investigate rather than revert
**Sources:** Project CLAUDE.md design rules: "Never use compensation constants. If the EM doesn't match SC2, fix the root cause."
**Exploration:** quick
**Status:** captured
