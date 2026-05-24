# Handover — 2026-05-24

**Head commit (project):** `15e9178` — feat(#152): auto-compute miningProbesPerBase in tick() with one-shot override
**Head commit (workspace):** `cc16f24` — docs: add 2026-05-15 entry to blog INDEX.md

## What Changed This Session

- **git-squash (first run):** 620 → 452 commits on `main`. filter-repo stripped `docs/_posts/` (11 commits pruned); 154 squash-classified commits absorbed. Force-pushed to `mdproctor/quarkmind`.
- **Backup branch retained:** `backup/pre-squash-main-20260523` — 14-day retention.
- **quarkmind.md updated** in `casehubio/parent` — reflected post-Phase 6 state (690 tests, per-base mining, vespene sync, sub-tick timing, training queues, auto-engage, visualiser). Pushed to both forks.
- **Branch verification:** all 7 closed issue branches confirmed — 0 unique commits missing from squashed main. Workspace artifacts promoted:
  - `blog/2026-05-15-mdp01-two-engines-same-replay.md` (was missing from workspace main)
  - `plans/` — 3 plan files promoted (issue-142, issue-143, issue-153-152)
  - `design/JOURNAL.md` — consolidated journals from issues 142, 143, 146, 149, 153-152
- **Published:** 2 blog entries to `mdproctor.github.io/_notes/` (2026-05-15, 2026-05-23-mdp02)
- **`origin/issue-142-tick-rounding`** remote branch still exists — not deleted per retention policy

## Immediate Next Step

Run `work-start` and pick up **#145** (BuildIntent sub-tick timing + boundary test) or **#150** (`IEM10JsonSimulatedGame`: expose `gameEvents` as `TimedIntent` stream).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #145 | BuildIntent sub-tick timing + boundary test (offset=17) | M | Med | UnitInit/UnitDone give building calibration directly |
| #150 | IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream | M | Med | Enables multi-game harness validation |
| #140 | Acquire Terran .SC2Replay files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Previous handover (issues #152/#153) | `git show HEAD~2:HANDOFF.md` |
| Blog entries (today) | `blog/2026-05-15-mdp01-two-engines-same-replay.md` (promoted), `blog/2026-05-23-mdp02-override-and-cleanup.md` |
| Consolidated design journals | `design/JOURNAL.md` |
