# Handover — 2026-06-12

*Updated: #189, #190, casehubio/parent#224 closed — removed from backlog.*

**Head commit (project):** `adb2df6` — protocol: gametick-must-not-write-gameactive
**Head commit (workspace):** `fabd7c9` — mark branch issue-184-cleanup-and-layer6-trust closed

## What Changed This Session

Closed #184, #186, #187, #188 (transport / code review minor fixes). Closed #158: Layer 6 trust-weighted strategy routing — `StrategyTrustRouter`, `StrategySelector`, `StrategyTrustObserver`, `GameOutcomeRecorder`, `EnemyPostureClassifiedEvent`, two new `StrategyTask` implementations (`EarlyPressureStrategyTask`, `EconomicExpansionStrategyTask`). `LedgerLifecycleAdapter` removed — was silently capping `decisionCount` at 1. Three L6 protocols added. ARC42STORIES.MD updated to ✅ L6 complete. Since last handover: closed #189 (win/loss outcome detection from real SC2 wired to L6 trust routing), #190 (ARC42STORIES.MD + BOOTSTRAP phaseScore sync), casehubio/parent#224 (docs sync). New protocol: `gametick-must-not-write-gameactive`.

## Immediate Next Step

`/work #159` — Layer 7: comparison baseline vs naive AI and commercial frameworks.

## Cross-Module

**casehub-poc issues filed this session:**
- `mdproctor/casehub#9` — consolidate `canActivate()` as single activation gate (removes redundant entryCriteria re-check from QuarkMind overrides once fixed)
- `mdproctor/casehub#10` — add `dependsOn` task ordering mechanism (removes `ENEMY_ARMY_SIZE` ordering hack from `DroolsStrategyTask.entryCriteria()` once fixed)

**casehub-engine issue filed this session:**
- `casehubio/engine#476` — `ImplementationRoutingStrategy` SPI — trust-based routing among competing `TaskDefinition` implementations; replaces `StrategyTrustRouter` + `StrategySelector` + `StrategyTrustObserver` in QuarkMind (~300 lines) once shipped

**casehub-parent docs updated this session:**
- `docs/CBR-CAPABILITY.md` — new doc defining CBR as a platform capability (Retrieve/Reuse/Revise/Retain), per-repo responsibilities, component map, QuarkMind as reference implementation
- `docs/PLATFORM.md` — CBR and `ImplementationRoutingStrategy` entries in capability table; boundary rules
- `docs/AGENTIC-HARNESS-GUIDE.md` — CBR section added so all harness apps see it

**CBR epic + all child issues filed:**
- casehubio/parent#227 — Epic tracking all CBR work
- casehubio/engine#476 — ImplementationRoutingStrategy SPI (Reuse)
- casehubio/engine#477 — CaseOutcomeObserver SPI (Retain hook)
- casehubio/engine#478 — CaseRetriever integration at plan creation (Retrieve bridge)
- casehubio/neural-text#20 — CaseRetriever contract (Retrieve)
- casehubio/platform#87 — CbrCaseEntry structured type (Retain schema)
- casehubio/ledger#136 — TrustGateService batch scoring (Reuse)
- casehubio/quarkmind#192 — QuarkMind CBR reference implementation
- casehubio/aml#61 — CBR over AML investigation patterns
- casehubio/clinical#78 — CBR over adverse event history
- casehubio/devtown#75 — CBR over PR review patterns

## What's Left

- casehub-poc fix (InMemoryCaseFile null) — low urgency · S · Med
- Remove redundant `entryCriteria()` re-check from `DroolsStrategyTask` / `DroolsTacticsTask` `canActivate()` — blocked on `mdproctor/casehub#9` · XS · Low
- Remove `ENEMY_ARMY_SIZE` ordering hack from `DroolsStrategyTask.entryCriteria()` — blocked on `mdproctor/casehub#10` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #159 | Layer 7: comparison vs naive game AI | M | Med | Unblocked |
| #180 | LLM advisory team on scouting channel | L | High | Depends on Qhorus channel |
| #181–183 | Commentator/Coach LLM + hierarchical summarisation + enemy classifier | L | High | Cascade dependency |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| L6 spec | `docs/superpowers/specs/2026-06-10-layer6-trust-weighted-strategy-routing-design.md` |
| L6 diary entry | `blog/2026-06-10-mdp01-layer6-trust-routing.md` |
| ARC42STORIES.MD | L1–L6 ✅, L7 pending (#159) |
