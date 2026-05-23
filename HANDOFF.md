# Handover — 2026-05-23

**Head commit (project):** `7abccfa` — docs: sync DESIGN.md and CLAUDE.md with #152/#153
**Head commit (workspace):** `0335df3` — docs: add blog entry 2026-05-23-mdp02-override-and-cleanup

## What Changed This Session

- **#153 closed** — sqrt→squared distance in `countProbesPerBase`, defensive clone in `setMiningProbesPerBase`, `ReplayValidationHarnessTest` added (zero-nexus, single-nexus, two-nexus, non-probe filtering).
- **#152 closed** — `tick()` now auto-computes `miningProbesPerBase` from `friendly.buildings`/`friendly.units` via one-shot override flag (`miningProbesOverridden`); `countProbesPerBase` extracted from `ReplayValidationHarness` to `EmulatedGame` as canonical public static method. Replay harness compatibility preserved — it calls `setMiningProbesPerBase` before every tick.
- **Test updated** — `zeroProbesYieldsNoMineralGain` reduced to single-tick scope; 690 tests passing.
- **DESIGN.md + CLAUDE.md synced** — test count 682→690, EmulatedGame description updated, `ReplayValidationHarnessTest` added to unit test list.

## Immediate Next Step

Run `work-start` and pick up **#145** (BuildIntent sub-tick timing + boundary test) or **#150** (IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #145 | BuildIntent sub-tick timing + boundary test (offset=17) | M | Med | UnitInit/UnitDone give building calibration directly |
| #150 | IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream | M | Med | Enables multi-game harness validation |
| #152 | EmulatedGame: per-base probe distribution for AI expansion | S | Low | ✅ CLOSED this session |
| #153 | Code review nits: sqrt, defensive copy, zero-nexus test | XS | Low | ✅ CLOSED this session |
| #140 | Acquire Terran .SC2Replay files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Blog entry (today, session 2) | `blog/2026-05-23-mdp02-override-and-cleanup.md` |
| Blog entry (today, session 1) | `blog/2026-05-23-mdp01-two-precision-fixes.md` |
| Design spec | `docs/superpowers/specs/2026-05-23-review-nits-and-probe-dist-design.md` |
