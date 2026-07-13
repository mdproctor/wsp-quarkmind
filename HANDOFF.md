# QuarkMind Handover — 2026-07-13

## Last Session

Implemented #234 (Zergling batch cost fix) and #183 (enemy strategy classifier). #234 fixed handleTrain() to multiply mineral/gas costs by trainCount for batch-trained units, including the enemy-path startTraining() fallback (caught by adversarial design review). #183 added a PatternClassificationRuleUnit with 18 Drools evidence rules across 10 archetypes, confidence scoring via 1-∏(1-weight_i), and strategy layer consumption at salience 215. Also fixed pre-existing dependency breakages (OutboundMessage, RoutingResult, TrustRoutingPolicy, ChannelCreateRequest API migrations). 1324 tests, zero regressions. Landed as 746fb27 on main.

## Immediate Next Step

Pick up #227 (phase-adaptive dominance weights). Run `/work` to start a branch. The accurate cost tables (#229) and pattern classification (#183) are both prerequisites — now unblocked.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #192 — CBR reference implementation · L · High (Step 2 blocked on engine#478)
- #233 — consolidate remaining SC2Data switch methods to EnumMap pattern · M · Low
- #235 — LLM fallback for novel/ambiguous builds · M · Med
- #236 — replay classification accuracy ≥ 70% acceptance criterion · S · Low
- #237 — confidence revision — decay, counter-indication, multi-archetype · M · Med
- #238 — Commentator/Coach prompt updates for PATTERN_ASSESSMENT · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #227 | Phase-adaptive dominance weights | M | Med | Unblocked by #229 |
| #237 | Confidence revision — decay + counter-indication | M | Med | Builds on #183 classifier |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | Async advisory concern |
| #192 | CBR reference implementation — Steps 1+3 unblocked | L | High | Step 2 blocked on engine#478 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #183 follow-ons | #236 accuracy criterion, #238 prompt updates | S | Low | Filed from #183 review |
| #233 | Consolidate SC2Data switch methods to EnumMap | M | Low | Filed from #229 review |
| #230 | Coach mode — real-time advice for human players | L | High | Filed from #181 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
