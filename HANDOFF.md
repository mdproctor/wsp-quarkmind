*Updated: neocortex#162 closed — removed from cross-module.*

# QuarkMind Handover — 2026-07-21

## Last Session

Implemented position-based compliance verification (#244) — sealed VerificationPredicate hierarchy replacing flat count-delta model. 4 predicate types (CountDelta, ArmyCentroidMovement, ExpansionPlacement, UnitsNearLocation), LocationReference vocabulary with 7 permits, domain model enriched with MapInfo/ExpansionLocation/NeutralFeature. Design spec adversarially reviewed (4 rounds, 22 issues). 1717 tests green. Landed as 371e51e on main.

## Immediate Next Step

Pick new work from epic #250 backlog. Run `/work` from main. Next child: #245 (LLM-evaluated compliance) or #248 (coaching acknowledgment UI).

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.


## What's Left

- #74 — genericise unit/building definitions · L · Med
- #233 — consolidate remaining SC2Data switch methods · M · Low
- Neutral feature extraction from real SC2 observations — wired but passing null; needs SC2 unit type mapping for watchtowers/destructibles · S · Low
- ExpansionLocationCalibrationTest against IEM10 replays — validates 12.0 clustering radius · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #245 | LLM-evaluated compliance verification | M | Med | New VerificationPredicate permit — infrastructure ready |
| #248 | Coaching acknowledgment UI (DONE/DECLINE) | M | Med | Visualizer overlay work |
| #251 | Coaching personality model | M | Med | Epic; adaptive intra-game disposition |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | CBR infrastructure in place |
| #243 | Full tactic taxonomy (all game phases) | L | High | Epic #252 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
