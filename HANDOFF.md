# Handover — 2026-05-23

**Head commit (project):** `d53088a` — docs: update test count 629 → 682
**Head commit (workspace):** `00f8057` — archive(issue-143-multi-base-mining): move plans to attic

## What Changed This Session

- **#147 closed** — `drainBuildingQueues` now propagates the previous unit's absolute completion loop to the next queued unit via `buildingCompletionAtLoop` map on `PlayerState`. Fixes ±1 tick imprecision for queued training. Code review caught a stale-entry leak for non-queued buildings; fixed with conditional cleanup in the completion callback.
- **#143 closed** — `EmulatedGame.miningProbes` (scalar) replaced with `miningProbesPerBase` (int[]). `tick()` sums per-base income. `ReplayValidationHarness.countProbesPerBase()` assigns probes to nearest complete Nexus by Euclidean distance. `SC2Data.mineralIncomePerTick` unchanged — already per-base.
- **#152 created** — deferred: AI expansion mining when EmulatedGame standalone supports building a second Nexus.
- **#153 created** — batched code review minors: sqrt optimization, defensive array copy, zero-nexus test.
- **Branch hygiene** — wrote EPIC-CLOSED.md to `epic-saturation-mining` (#141); cleaned stale `design/EPIC-CLOSED.md` from workspace main. Branches #142/#146/#149 already had markers.
- **DESIGN.md synced** — updated SC2Data, EmulatedGame, PlayerState descriptions; marked #143/#148 as resolved; test count 629→682.

## Immediate Next Step

Run `work-start` and pick up **#145** (BuildIntent sub-tick timing + boundary test) or **#150** (IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #145 | BuildIntent sub-tick timing + boundary test (offset=17) | M | Med | UnitInit/UnitDone give building calibration directly |
| #150 | IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream | M | Med | Enables multi-game harness validation |
| #152 | EmulatedGame: per-base probe distribution for AI expansion | S | Low | Deferred from #143; blocked on AI expansion capability |
| #153 | Code review nits: sqrt, defensive copy, zero-nexus test | XS | Low | Batched minors from #143/#147 review |
| #140 | Acquire Terran .SC2Replay files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Blog entry | `blog/2026-05-23-mdp01-two-precision-fixes.md` |
| Design spec | `docs/superpowers/specs/2026-05-22-multi-base-mining-design.md` |
| Plan (archived) | `plans/attic/issue-143-multi-base-mining/2026-05-22-multi-base-mining.md` |
