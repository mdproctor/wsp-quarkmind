# QuarkMind Handover — 2026-07-14

## Last Session

Closed #239 (cannon rush false positive) and #237 (confidence revision) on a single branch. UnitType and EnemyArchetype now carry Race via constructor parameter — DRL rules gate race-specific archetypes on observed units. PatternClassifier gains frame-based decay (60s half-life), multiplicative counter-indication (expansion/tech/prediction-window), and multi-archetype publishing. Design-reviewed (5 rounds, 13 issues). 1547 tests, zero regressions, 100% calibration accuracy. Landed as 9e66ce8 on main.

## Immediate Next Step

Pick up #240 (Drools-based DominanceWeightStrategy). The SPI seam from #227 is ready; `allAssessments()` from #237 provides the multi-archetype input. Run `/work` to start a branch.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #192 — CBR reference implementation · L · High (Step 2 blocked on engine#478)
- #233 — consolidate remaining SC2Data switch methods to EnumMap pattern · M · Low
- #235 — LLM fallback for novel/ambiguous builds · M · Med
- #240 — Drools-based DominanceWeightStrategy · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | Drools-based DominanceWeightStrategy | M | Med | SPI seam ready from #227, allAssessments available |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | Can reason over multi-archetype assessments now |
| #192 | CBR reference implementation — Steps 1+3 unblocked | L | High | Step 2 blocked on engine#478 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #233 | Consolidate SC2Data switch methods to EnumMap | M | Low | Filed from #229 review |
| #230 | Coach mode — real-time advice for human players | L | High | Filed from #181 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
