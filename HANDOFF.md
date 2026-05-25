# Handover — 2026-05-25

**Head commit (project):** `0c4dd41` — chore: update settings.local.json
**Head commit (workspace):** `c9e5d24` — feat: promote blog from issue-145-build-intent-subtick

## What Changed This Session

- **#145 closed:** `buildTimeInLoops(BuildingType)` calibrated from UnitInit→UnitDone tracker event diffs across 30 AI Arena replays — 34 building types, all three races. Key findings: NEXUS was `default → 40 ticks` (880 loops); actual = 1600 loops (72 ticks), 32-tick error. FACTORY n=32 rejected — all addon completions via toBuildingType aliasing; fixed with addon name filter, n=16 clean structures at 960 loops.
- `handleBuild` now uses sub-tick formula: `gameFrame + (loopOffset + buildTimeInLoops(bt)) / LOOPS_PER_TICK`. `applyIntent(TimedIntent)` passes `ti.loop()`; enemy AI path loopOffset=0.
- `SC2BuildTimeCalibrationTest` new — UnitInit/UnitDone diff, modal assertion, addon filter.
- `buildTimeInTicks` now derives from `buildTimeInLoops / LOOPS_PER_TICK`.
- `docs/DESIGN.md` and `sc2data-train-times-require-calibration.md` updated.
- **#154 created:** addon calibration cleanup (FACTORY/BARRACKS/STARPORT re-calibration, makeTag extraction).
- 2 garden entries: `GE-20260525-a8c35a` (addon contamination gotcha), `GE-20260525-1a1a7f` (UnitInit/UnitDone technique).
- Blog: `2026-05-25-mdp01-two-wrong-build-times.md` published.

## Immediate Next Step

Run `work-start` and pick up **#154** (addon calibration cleanup — quick) or **#150** (`IEM10JsonSimulatedGame`: expose `gameEvents` as `TimedIntent` stream).

## What's Left

- `origin/issue-142-tick-rounding` remote branch — retained under 14-day policy (due ~2026-06-06)
- `backup/pre-squash-main-20260523` — retained until ~2026-06-06

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #154 | Addon calibration: filter toBuildingType aliases in SC2BuildTimeCalibrationTest; makeTag extraction | S | Low | Quick win; unblocks clean FACTORY/BARRACKS/STARPORT values |
| #150 | IEM10JsonSimulatedGame: expose `gameEvents` as `TimedIntent` stream | M | Med | Enables multi-game harness validation |
| #140 | Acquire Terran `.SC2Replay` files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Previous handover (#152/#153) | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-25-mdp01-two-wrong-build-times.md` |
| Addon calibration issue | `gh issue view 154 --repo mdproctor/quarkmind` |
