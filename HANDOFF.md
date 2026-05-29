# Handover — 2026-05-28

**Head commit (project):** `c148910` — protocol: replay tag prefix per source + extractor separation
**Head commit (workspace):** `9746808` — docs: mark issue-150 closed

## What Changed This Session

- **#150 closed:** `IEM10CommandExtractor` added — extracts `List<TimedIntent>` from SC2EGSet
  JSON `gameEvents` using IEM10 2016 patch constants (patch-specific abilLink table derived
  from narrow-window modal analysis across all 30 games).
- `ReplayValidationHarness` generalised: new `run(SimulatedGame, List<TimedIntent>, int)`
  overload; existing binary-replay overload delegates. `assertInitialStateMatch` allows ±1
  unit tolerance for SC2EGSet loop-0 UnitBorn quirk.
- 30-game IEM10 baseline established: PvP 9.6 / PvT 22.6 / PvZ 40.1 mean ΔUnits.
- 2 new protocols: `replay-tag-prefix-per-source`, `extractor-separate-from-simulated-game`.
- 3 garden entries: GE-20260528-3b9ccb (r- prefix gotcha), -6ebb38 (±1 unit at loop 0),
  -f89f62 (training command filter `data: {None: null}`).
- Issues filed: #160 (removeMask unit tests), #161 (CLAUDE.md + SimulatedGame test),
  casehubio/parent#83 (quarkmind.md doc sync).
- History squashed (14→7), pushed to origin, work-end complete.

## Immediate Next Step

Pick up **#160** (removeMask unit tests — XS, ~30 min) to clear follow-up debt,
or start **#140** (Terran `.SC2Replay` acquisition for AbilityMapping) as the next
substantive piece.

## What's Left

- `issue-150-iem10-timed-intent-stream` workspace/project branch — EPIC-CLOSED.md written, deletion due 2026-06-11
- `issue-154-s-xs-batch` workspace/project branch — EPIC-CLOSED.md, deletion due 2026-06-08
- `origin/issue-142-tick-rounding` remote — retained until ~2026-06-06
- `backup/pre-squash-main-20260523` — retained until ~2026-06-06
- casehubio/parent#83 — quarkmind.md doc sync (peer repo, filed, their responsibility)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #160 | removeMask unit tests for IEM10CommandExtractor | XS | Low | Quick follow-up from code review |
| #161 | CLAUDE.md test inventory + SimulatedGame.isComplete unit test | XS | Low | Quick follow-up |
| #140 | Acquire Terran `.SC2Replay` files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Epic; may have child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate: qhorus pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate: ledger pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-28-mdp01-three-ways-to-extract-nothing.md` |
| IEM10 spec | `docs/superpowers/specs/2026-05-27-iem10-timed-intent-stream-design.md` |
