# Handover — 2026-05-29

**Head commit (project):** `ffa5f60` — docs: update CLAUDE.md — add SelectionStateTest
**Head commit (workspace):** `a80d474` — archive(issue-162-selection-state-unification): move plans to attic

## What Changed This Session

- **#162 closed:** `SelectionState` extracted to `io.quarkmind.sc2` — both
  `AbilityMapping` and `IEM10CommandExtractor` now delegate to it. All 5 variants
  handled via `removeAt`, `removeIf(IntPredicate)`, `truncateTo`, `retainOnly`,
  `addTag`. Full test suite (25 tests in `SelectionStateTest`).
- **#163 filed:** `AbilityMappingTest` has no unit tests for `onSelection()` delta
  variants — binary path only exercised via real replay files.
- **1 garden entry:** GE-20260529-90ed14 — `SelectionDeltaEvent.getDelta()` always
  non-null in Scelight (null guard in `AbilityMapping` is unreachable).

## Immediate Next Step

Start **#140** (Terran train time calibration) via `/work` — run
`mvn test -Dtest=SC2TrainTimeCalibrationTest -q` first to confirm current
Terran defaults, then calibrate from `Nothing_4720935.SC2Replay`.

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #163 | AbilityMappingSelectionDeltaTest — unit coverage for onSelection variants | S | Low | Template: IEM10CommandExtractorSelectionDeltaTest |
| #140 | Terran train time calibration from replay data | S | Med | Constants added; SC2TrainTimeCalibrationTest is the tool |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Epic; may have child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate: qhorus pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate: ledger pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-29-mdp02-two-extractors-one-state.md` |
| Garden entry | `~/.hortora/garden/scelight/GE-20260529-90ed14.md` |
