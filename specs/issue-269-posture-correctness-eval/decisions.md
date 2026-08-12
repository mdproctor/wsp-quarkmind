## D1: Ground truth source for posture correctness

**Choice:** Latest PatternAssessment from CaseFile snapshot at game close, compared against the initial archetype (first routing, never overwritten)
**Alternatives:**
- Compare routed archetype vs broker.current() — routed archetype is overwritten on pivot, so comparison collapses to 1.0 after any pivot (review R1-02)
- Composition-derived reverse-classifier — most accurate but requires new inference logic
**Rationale:** Uses data the system already produces. Measures whether the scouting system's early read converged with its final read — the exact signal CBR retrieval needs. Capturing the initial archetype separately avoids the pivot-collapse bug.
**Trade-offs:** Still measures convergence, not absolute correctness. A consistently wrong scouting system would show high convergence. Feature naming reflects this — `scouting_convergence` not `posture_correct`.
**Revision:** Review R1-01, R1-02 — renamed features to reflect convergence semantics; added STRATEGY_INITIAL_ARCHETYPE key for first-routing capture.
**Exploration:** quick
**Status:** revised

## D2: Scouting accuracy scoring model

**Choice:** Phase-aware tiered scoring — exact archetype match = 1.0, same category within same GamePhase = 0.5, cross-phase transition from EARLY to MID/LATE = 0.5 (not penalised), different category within same phase = 0.0.
**Alternatives:**
- Phase-unaware tiered — penalises normal game progression as scouting failure (review R1-04)
- Binary — exact match or nothing, loses "close enough" signal
- Confidence-weighted — conflates two independent signals
**Rationale:** Cross-phase transitions (e.g., early RUSH detected correctly → opponent transitions to MID COMPOSITION) are normal game progression, not scouting errors. The evaluator checks: if phases differ, the early detection was phase-appropriate and shouldn't be penalised.
**Trade-offs:** 0.5 for cross-phase transitions treats all transitions as equally benign — a RUSH that transitions to MACRO is different from HARASS that transitions to COMPOSITION. Acceptable for v1.
**Revision:** Review R1-04 — added phase-awareness to prevent normal game evolution being scored as scouting failure.
**Exploration:** quick
**Status:** revised

## D3: Evaluation logic placement

**Choice:** Separate `PostureCorrectnessEvaluator` in `agent/` package — pure function taking initial archetype + latest assessments, returning a result record. SC2CbrRetentionObserver calls it and adds features.
**Alternatives:**
- In `agent/cbr/` — archetype comparison is domain judgment, not CBR mechanics (review R1-07)
- Inline in SC2CbrRetentionObserver — harder to test independently
**Rationale:** Follows the pattern of `MultiFactorDominanceAssessor` (lives in `agent/` despite being called from CBR observer). Pure function with no CDI dependencies — trivially unit-testable.
**Trade-offs:** None significant.
**Revision:** Review R1-07 — moved from `agent/cbr/` to `agent/`.
**Depends on:** D2 (scoring model determines what the evaluator computes)
**Exploration:** quick
**Status:** revised

## D4: Access to final PatternAssessment

**Choice:** Write latest assessment to a CaseFile key (`SCOUTING_FINAL_ASSESSMENT`) from the scouting task on each tick. SC2CbrRetentionObserver reads from `event.caseFileSnapshot()` like all other data.
**Alternatives:**
- Inject ScoutingIntelBroker directly — breaks the snapshot-only data flow convention, creates timing risk with GameStarted clearing the broker before onOutcome completes (review R1-03)
- Subscribe to PatternAssessmentPublished events — more wiring than needed
**Rationale:** Consistent with the snapshot-only pattern used by every other data source in onOutcome(). SC2AdvisoryCbrRetentionObserver uses the same approach. Avoids timing dependency on broker lifecycle.
**Trade-offs:** Requires a scouting task change to write the new CaseFile key. Small — one line per tick.
**Revision:** Review R1-03 — switched from broker injection to CaseFile snapshot for data flow consistency.
**Depends on:** D1 (ground truth is the latest assessment via snapshot)
**Exploration:** quick
**Status:** revised

## D5: Multi-assessment list handling

**Choice:** Compare initial archetype against `.getFirst()` (highest confidence) from the `PatternAssessmentPayload` list.
**Alternatives:**
- Any-match (lenient) — if any assessment in the list matches, score 1.0. Too generous — inflates convergence scores.
**Rationale:** Consistent with how `SC2StrategyRouterTask` selects the archetype for routing — it uses `.getFirst()`. Comparing against the same selection method is apples-to-apples.
**Trade-offs:** Ignores secondary assessments that might show partial correctness. Acceptable — the primary assessment is the one that drives behaviour.
**Exploration:** quick
**Status:** captured

## D6: Feature naming

**Choice:** `scouting_convergence` (double, 0.0–1.0) and `assessment_stable` (boolean, convergence >= 0.5) instead of the original `scouting_accuracy` and `posture_correct`.
**Alternatives:**
- Keep original names (`posture_correct`, `scouting_accuracy`) — misleading, implies absolute correctness when we're measuring internal consistency (review R1-01)
**Rationale:** Names should reflect what the metric measures. This is convergence (did the early read match the late read), not correctness (was scouting right about reality). Every downstream consumer of these features should understand what they're getting.
**Trade-offs:** Departs from the issue's original naming. Issue #269 can document the rationale for the rename.
**Exploration:** quick
**Status:** captured
