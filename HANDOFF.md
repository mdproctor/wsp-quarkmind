# QuarkMind Handover — 2026-07-09

## Last Session

Implemented #229 (SC2Data cost table expansion) — UnitCosts record + exhaustive EnumMap replacing three switch-with-default methods. All 67 UnitType values now have Liquipedia-verified LotV final balance costs. Fixes 81% army undervaluation in MultiFactorDominanceAssessor. Also adapted WorkerFunction.Sync calls to new casehub-worker inputType parameter (pre-existing API change). Adversarial design review (5 rounds, $16.84) caught Colossus supply 4→6, Tempest supply 5→4, HELLBAT transformation form, Zergling batch semantics. Landed as 4cc6de2 on main. 1297 tests, zero regressions.

## Immediate Next Step

Pick up #227 (phase-adaptive dominance weights). Run `/work` to start a branch. #229's accurate cost tables are the prerequisite — now unblocked.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #192 — CBR reference implementation · L · High (Step 2 blocked on engine#478)
- #233 — consolidate remaining SC2Data switch methods to EnumMap pattern · M · Low
- #234 — Zergling batch cost deduction in EmulatedGame · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #227 | Phase-adaptive dominance weights | M | Med | Unblocked by #229 |
| #192 | CBR reference implementation — Steps 1+3 unblocked | L | High | Step 2 blocked on engine#478 |
| #74 | Genericise unit/building definitions | L | Med | Independent |
| #183 | Enemy strategy classifier | M | High | Unblocked by #182 |
| #233 | Consolidate SC2Data switch methods to EnumMap | M | Low | Filed from #229 review |
| #234 | Zergling batch cost deduction fix | S | Low | Filed from #229 review |
| #221 | Learning curve observability | M | Med | CBR epic #214, Tier 1-2 |
| #230 | Coach mode — real-time advice for human players | L | High | Filed from #181 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
