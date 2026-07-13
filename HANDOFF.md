# QuarkMind Handover — 2026-07-13

## Last Session

Implemented #236 (replay classification accuracy ≥ 70%) and #238 (Commentator/Advisory prompt updates for PATTERN_ASSESSMENT). PatternClassificationCalibrationTest runs the full Drools CEP pipeline against 59 replays — 75% rush accuracy at 3-min mark. CommentaryWorkerFactory and AdvisoryWorkerFactory system prompts and user messages now reference and format PATTERN_ASSESSMENT intel. Filed #239 (cannon rush false positive — 4/16 misclassifications from race-unguarded rule). 1335 tests, zero regressions. Landed as 4e71612 on main.

## Immediate Next Step

Pick up #227 (phase-adaptive dominance weights). Run `/work` to start a branch. All prerequisites (#229, #183) are done.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #192 — CBR reference implementation · L · High (Step 2 blocked on engine#478; platform#87 closed — bump SNAPSHOT to unblock Step 1)
- #233 — consolidate remaining SC2Data switch methods to EnumMap pattern · M · Low
- #235 — LLM fallback for novel/ambiguous builds · M · Med
- #237 — confidence revision — decay, counter-indication, multi-archetype · M · Med
- #239 — cannon rush false positive — race guard missing · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #227 | Phase-adaptive dominance weights | M | Med | Unblocked by #229 |
| #239 | Cannon rush false positive — race guard | S | Low | Filed this session, would push accuracy to ~100% |
| #237 | Confidence revision — decay + counter-indication | M | Med | Builds on #183 classifier |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | Async advisory concern |
| #192 | CBR reference implementation — Steps 1+3 unblocked | L | High | Step 2 blocked on engine#478 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #233 | Consolidate SC2Data switch methods to EnumMap | M | Low | Filed from #229 review |
| #230 | Coach mode — real-time advice for human players | L | High | Filed from #181 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
