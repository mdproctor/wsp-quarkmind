# Handover — 2026-05-25

**Head commit (project):** `3ea47b9` — chore: update settings.local.json
**Head commit (workspace):** `615f39e` — feat: promote blog from issue-154-s-xs-batch

## What Changed This Session

- **S/XS batch closed:** #136, #154, #134, #135, #139 — all on `issue-154-s-xs-batch`, now merged to main.
- **#154:** `makeTag` extracted to `Sc2ReplayShared` (package-private); FACTORY confirmed empirical at 960 loops (n=16 clean after addon filter). BARRACKS/STARPORT obs counts updated.
- **#134:** `attackingUnits` removed from `PlayerState` entirely. `isAttack` param from `setTarget` also removed (was log-only after field removal). ADR-0003 superseded. DESIGN.md corrected.
- **#135:** Focus-fire convergence test added — two Stalkers directed via AttackIntents every tick converge on ZealotA; auto-engage at same tick count leaves ZealotA alive.
- **#136:** `deadUnitRemovedFromRetreatingSet` comment + `autoAttack_prefersLowerHpAtEqualDistance` snapshot API fixed.
- **#139:** LAYER-LOG.md Layer 1 (conceptual baseline) + Layer 2 (CaseHub blackboard) written. CLAUDE.md tutorial structure table updated with issue links #155–#159 for Layers 3–7.
- Garden: `GE-20260525-56c580` — freeze technique (`moveEnemyUnits` does not remove targets on arrival).
- Blog: `2026-05-25-mdp02-dead-state-lucky-guesses.md` published.
- Peer repo issue: casehubio/parent#72 — quarkmind.md LAYER-LOG pending note stale.

## Immediate Next Step

Run `work-start` and pick up **#150** (`IEM10JsonSimulatedGame`: expose `gameEvents` as `TimedIntent` stream) or **#127** (Phase 5 completion epic review).

## What's Left

- `issue-154-s-xs-batch` workspace/project branch — EPIC-CLOSED.md written, deletion due 2026-06-08
- `origin/issue-142-tick-rounding` remote branch — retained until ~2026-06-06
- `backup/pre-squash-main-20260523` — retained until ~2026-06-06
- casehubio/parent#72 — quarkmind.md doc sync (peer repo, filed, their responsibility)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #150 | IEM10JsonSimulatedGame: expose `gameEvents` as `TimedIntent` stream | M | Med | Enables multi-game harness validation |
| #140 | Acquire Terran `.SC2Replay` files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Epic; may have child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate: qhorus pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate: ledger pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-25-mdp02-dead-state-lucky-guesses.md` |
| Garden entry | `GE-20260525-56c580` (jvm/) |
