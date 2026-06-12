# Handover — 2026-06-12

**Head commit (project):** `0aee4ba` — chore: allow WebFetch for raw.githubusercontent.com
**Head commit (workspace):** see below after commit

## What Changed This Session

Architecture exploration session — no QuarkMind implementation changes. Audited CaseHub usage in QuarkMind: `createAndSolve()` is a thin plugin dispatch loop; `StrategyTrustRouter`/`StrategySelector`/`StrategyTrustObserver` (228 lines) are application-layer reimplementations of what `casehub-engine` should own. CBR framing: trust routing is degenerate CBR — Retain and Reuse only, Retrieve and Revise missing. Wrote `docs/CBR-CAPABILITY.md` in casehub-parent; updated PLATFORM.md and AGENTIC-HARNESS-GUIDE.md. Filed 11 issues across 7 repos with casehubio/parent#227 as parent epic. Garden entry GE-20260612-bd3b4d submitted (degenerate CBR diagnostic technique).

## Immediate Next Step

`/work #159` — Layer 7: comparison baseline vs naive AI. Or: drop Quarkus Drools CDI injection to remove GE-0053 classloader constraint (half-day prep, enables richer rules). Ask at session start.

## Cross-Module

**casehub-poc issues filed (retiring codebase — workarounds only):**
- `mdproctor/casehub#9` — consolidate `canActivate()` as single activation gate
- `mdproctor/casehub#10` — add `dependsOn` task ordering mechanism

**casehub-engine issues filed:**
- `casehubio/engine#476` — ImplementationRoutingStrategy SPI (Reuse) — replaces StrategyTrustRouter when shipped
- `casehubio/engine#477` — CaseOutcomeObserver SPI (Retain hook)
- `casehubio/engine#478` — CaseRetriever integration at plan creation (Retrieve bridge)

**CBR epic + foundation issues:**
- `casehubio/parent#227` — parent epic, four implementation waves
- `casehubio/neural-text#20` — CaseRetriever contract
- `casehubio/platform#87` — CbrCaseEntry type (Wave 1 — start here)
- `casehubio/ledger#136` — TrustGateService batch scoring (Wave 1 — start here)

**Application issues:**
- `casehubio/quarkmind#192` — QuarkMind CBR reference implementation (Wave 4)
- `casehubio/aml#61`, `casehubio/clinical#78`, `casehubio/devtown#75` — domain CBR

## What's Left

- Remove redundant `entryCriteria()` re-check from `DroolsStrategyTask`/`DroolsTacticsTask` `canActivate()` — blocked on `mdproctor/casehub#9` · XS · Low
- Remove `ENEMY_ARMY_SIZE` ordering hack — blocked on `mdproctor/casehub#10` · XS · Low
- Delete `StrategyTrustRouter`/`StrategySelector`/`StrategyTrustObserver` — blocked on `casehubio/engine#476` · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #159 | Layer 7: comparison vs naive game AI | M | Med | Unblocked |
| #192 | QuarkMind CBR reference implementation | L | High | Blocked on engine#476, #477, #478 (Wave 4) |
| #180 | LLM advisory team on scouting channel | L | High | Depends on Qhorus channel |
| #181–183 | Commentator/Coach LLM + hierarchy + enemy classifier | L | High | Cascade dependency |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| CBR architecture | `casehub-parent/docs/CBR-CAPABILITY.md` |
| CBR epic | `casehubio/parent#227` (four implementation waves) |
| Diary entry | `blog/2026-06-12-mdp02-trust-routing-is-degenerate-cbr.md` |
| ARC42STORIES.MD | L1–L6 ✅, L7 pending (#159) |
