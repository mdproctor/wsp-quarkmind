# Handover — 2026-06-02

**Head commit (project):** `e0cfba9` — adr: 0008 RaceModel.canProduce read-only enforcement and MULE calldown intent extraction
**Head commit (workspace):** `4732e0a` — archive(issue-165-racemodel-canproduce-readonly): move plan to attic

## What Changed This Session

- **#165 closed:** `RaceModel.canProduce()` is now structurally read-only via `PlayerStateView` interface. MULE calldown extracted from `TrainIntent` into `MuleCalldownIntent` (sealed `Intent` subtype). `ProductionDecision { PROCEED, BLOCKED }` replaces `ProductionResult { PROCEED, HANDLED, BLOCKED }`. New `RaceModel.onCalldown()` default method handles ability-use spawns. `TerranRaceModel.canProduce()` is a one-liner; MULE logic moved to `onCalldown()`. 843 tests passing. 7 squashed commits on project main (16 → 6 on branch, then rebased).
- **ADR-0008** filed: `docs/adr/0008-racemodel-canproduce-readonly-and-mule-intent-extraction.md`
- **Protocol PP-20260601-5fa812** updated: `ProductionDecision` + `PlayerStateView` + `MuleCalldownIntent` added to public type examples.
- **DESIGN.md** synced: RaceModel, TerranRaceModel, and EmulatedGame rows updated.
- **Blog:** `2026-06-02-mdp01-canproduce-readonly.md` (published).

## Immediate Next Step

Run `/work` to start #127 (Phase 5 — EmulatedGame accuracy gaps).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Main thread; may spawn child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-06-02-mdp01-canproduce-readonly.md` |
| ADR | `docs/adr/0008-racemodel-canproduce-readonly-and-mule-intent-extraction.md` (project repo) |
| Spec | `docs/superpowers/specs/2026-06-01-racemodel-canproduce-readonly-design.md` (project repo) |
