# QuarkMind Handover — 2026-07-13

## Last Session

Implemented #227 (phase-adaptive dominance weights). DominanceWeightStrategy SPI with two implementations — TemporalDominanceWeightStrategy (anchor-point interpolation from gameFrame) and SituationalDominanceWeightStrategy (temporal baseline + phase modifiers). AnchorInterpolator shared helper. MultiFactorDominanceAssessor migrated to strategy delegation with phase bus subscription. Design review (4 rounds, 18 issues, all resolved). 1368 tests, zero regressions. Filed #240 (Drools-based weight strategy). Landed as 7e0a529 on main.

## Immediate Next Step

Pick up #239 (cannon rush false positive — race guard missing). Quick fix, would push pattern classifier accuracy toward ~100%. Run `/work` to start a branch.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #192 — CBR reference implementation · L · High (Step 2 blocked on engine#478)
- #233 — consolidate remaining SC2Data switch methods to EnumMap pattern · M · Low
- #235 — LLM fallback for novel/ambiguous builds · M · Med
- #237 — confidence revision — decay, counter-indication, multi-archetype · M · Med
- #239 — cannon rush false positive — race guard missing · S · Low
- #240 — Drools-based DominanceWeightStrategy · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #239 | Cannon rush false positive — race guard | S | Low | Would push accuracy to ~100% |
| #237 | Confidence revision — decay + counter-indication | M | Med | Builds on #183 classifier |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | Async advisory concern |
| #240 | Drools-based DominanceWeightStrategy | M | Med | SPI seam ready from #227 |
| #192 | CBR reference implementation — Steps 1+3 unblocked | L | High | Step 2 blocked on engine#478 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #233 | Consolidate SC2Data switch methods to EnumMap | M | Low | Filed from #229 review |
| #230 | Coach mode — real-time advice for human players | L | High | Filed from #181 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
