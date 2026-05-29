# Handover — 2026-05-29

**Head commit (project):** `5a5c811` — docs(#140,#160,#161): sync DESIGN.md — test inventory + #140 Terran done
**Head commit (workspace):** `6d011b6` — archive(issue-160-remove-mask-tests): move plans to attic

## What Changed This Session

- **#160, #161 closed:** `IEM10CommandExtractorSelectionDeltaTest` (10 tests covering all 4 removeMask
  variants), `isCompleteReturnsFalseByDefault` in `SimulatedGameTest`. `applySelectionDelta` made
  package-private.
- **#140 Terran coverage added:** `AbilityMapping.onSelection` rewritten to handle Scelight binary
  `Delta.getRemoveMask()` variants correctly — ZeroIndices (retain-index), Mask (BitArray, instanceof guard),
  None (additive), OneIndices (descending removal). Fixes silent building-tag corruption in human replays
  (1787–1998 Mask events/game). Terran constants: ABIL_COMMAND_CENTER=155 (SCV), ABIL_BARRACKS=159
  (Marine idx=0, Marauder idx=3). `TerranReplayCommandExtractorTest` (10 assertions, Nothing_4720935
  player 2). `AbilityDiscoveryTest` extended with PvT replays.
- **#162 filed:** SelectionState unification — two independent removeMask implementations (binary +
  IEM10 JSON) diverging. Low priority, tracked.
- **1 garden entry:** GE-20260529-a2681d — Scelight `Delta.getRemoveMask()` heterogeneous payload types.
- Branch `issue-160-remove-mask-tests` closed, deletion due 2026-06-12.

## Immediate Next Step

Start **#138** (Terran/Zerg EmulatedGame mechanics — L, High) as the next substantive piece,
or pick up **#162** (SelectionState unification — XS, Low) as a quick warmup.

## What's Left

- `issue-160-remove-mask-tests` workspace/project branch — EPIC-CLOSED.md written, deletion due 2026-06-12
- `issue-150-iem10-timed-intent-stream` workspace/project branch — EPIC-CLOSED.md, deletion due 2026-06-11
- `issue-154-s-xs-batch` workspace/project branch — EPIC-CLOSED.md, deletion due 2026-06-08
- `origin/issue-142-tick-rounding` remote — retained until ~2026-06-06
- `backup/pre-squash-main-20260523` — retained until ~2026-06-06
- casehubio/parent#83 — quarkmind.md doc sync (peer repo, their responsibility)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #162 | SelectionState unification — shared removeMask logic between binary and JSON extractors | XS | Low | Quick follow-up from this session |
| #140 | Terran train time calibration from replay data | S | Med | Constants added but uncalibrated (default 672) |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Epic; may have child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate: qhorus pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate: ledger pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-29-mdp01-zeroindices-doesnt-mean-zero.md` |
| Garden entry | `~/.hortora/garden/scelight/GE-20260529-a2681d.md` |
