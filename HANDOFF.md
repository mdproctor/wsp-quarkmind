# HANDOFF — quarkmind

## Orientation

**TrackerIntentExtractor loop-0 fix (#751) + stale queue cleanup (#752-#755).**

One real fix applied, four already-completed items cleared from the queue. The queue now has #644 as the next open item after #751.

---

## Changes made (this session)

1. **#751 (S)** — TrackerIntentExtractor: added `if (ue.getLoop() == 0) {continue;}` to skip creating TrainIntents for game-setup spawns (workers/overlords born at loop 0). Previously, these loop-0 UnitBorn events created TrainIntents that flowed through to EmulatedGame, causing duplicate unit creation via `startTraining` (the existing `completionLoop > 0` guard only prevented mineral deduction, not the `startTraining` call guarded by `completionLoop >= 0`). The `unitTagToType` tracking is preserved above the guard so UnitDied events for game-setup units still resolve correctly. Two oracle-guarded tests added.
2. **#752, #753, #754, #755** — Already completed and landed on main from previous sessions. Marked done in the queue to clear stale entries. GitHub issues were already closed.

---

## Current state

| Branch | Status | Commits |
|--------|--------|---------|
| `issue-751-skip-initial-worker-mineral-deduction` | Fix landed on main (91a5dd95), branch stamped closed | 1 real + 1 close stamp |
| `issue-752-per-worker-cycle-variance` | Scaffold only — work already on main | 0 real |
| `issue-753-dynamic-worker-patch-select` | Scaffold only — work already on main | 0 real |
| `issue-754-walk-path-collision` | Scaffold only — work already on main | 0 real |
| `issue-755-regen-standalone-baseline` | Scaffold only — work already on main | 0 real |

All 3450+ tests pass.

---

## Queue state

The workspace `.queue` on main is out of sync — it still shows #751 as active and #752-#755 as open items further down the queue. The project-side queue was updated by `work.py done` but the workspace main wasn't rebased. The landing step failed due to `.queue` conflicts between the stale scaffold branches and main. Next session should reconcile the workspace `.queue` to reflect the done state.

---

## Next in queue

**#644 — Zerg dual-run baseline diverges** (after queue reconciliation).

---

## Notes

- **Landing conflict:** `work end` landing step failed on `.queue` rebase conflict for `issue-755-regen-standalone-baseline`. All actual code changes from #751 are already on main (commit 91a5dd95). The scaffold branches for #752-#755 have no code changes — they exist only because the pipeline required branches to mark items done.
- The stale branches can be deleted safely — they contain no unreplicated work.
