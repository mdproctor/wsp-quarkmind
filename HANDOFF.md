# QuarkMind Handover — 2026-07-25

## Last Session

Branch `issue-243-tactic-taxonomy` — completed Tasks 1-5 of 8 for #243 (strategy taxonomy). Three rename waves landed (EnemyArchetype → StrategyArchetype, EnemyPatternAssessment → PatternAssessment, summarisation.GamePhase → TacticalPosture). YAML taxonomy with StrategyTaxonomy loader created. Two rounds of upstream API breakage fixed. Blocked by #262 (ReactiveAgentIdentityVerificationService CDI bean removed — breaks all @QuarkusTest execution).

## Immediate Next Step

Fix #262 first — `mvn test` blocked. Then continue with Task 6 (SignatureFact + SignatureFactBuilder + generic Drools rules). Plan at `plans/2026-07-23-strategy-taxonomy.md`, spec at `specs/issue-243-tactic-taxonomy/2026-07-23-strategy-taxonomy-design.md`.

## Blocked by

- **casehub-platform** — #262 ReactiveAgentIdentityVerificationService CDI bean removed; blocks all @QuarkusTest integration tests · S · Low

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- Task 6: Detection pipeline (SignatureFact, SignatureFactBuilder, generic DRL rules) · M · Med
- Task 7: Phase resolution + coaching integration (TimeBasedPhaseResolver, GAME_PHASE key, prompt enrichment) · M · Med
- Task 8: New archetypes + multi-window calibration + follow-up issues · L · Med
- #74 — genericise unit/building definitions · L · Med
- Garden push pending — GE-20260630-91be72 revision committed locally, push to GitHub failed (auth)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #245 | LLM-evaluated compliance verification | M | Med | New VerificationPredicate permit |
| #248 | Coaching acknowledgment UI (DONE/DECLINE) | M | Med | Visualizer overlay |
| #251 | Coaching personality model | M | Med | Epic; adaptive disposition |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | CBR infrastructure in place |
