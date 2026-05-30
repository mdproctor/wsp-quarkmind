# Handover — 2026-05-30

**Head commit (project):** `61f9b6d` — chore: update settings.local.json permissions (on top of adr: 0007)
**Head commit (workspace):** `1b4c953` — feat: promote blog from issue-138-terran-zerg-mechanics

## What Changed This Session

- **#138 closed:** `RaceModel` plugin seam introduced — `EmulatedGame` fully race-agnostic. Three SC2 implementations: `ProtossRaceModel`, `TerranRaceModel` (MULE calldown + expiry), `ZergRaceModel` (larva/Egg/Queen inject). 828 tests passing. `countProbesPerBase` → `countWorkersPerBase`. `UnitType.EGG` added. SC2Data extended with all Terran/Zerg unit stats.
- **#163 closed:** `AbilityMappingSelectionDeltaTest` — 14 unit tests for `onSelection` variants using synthetic Scelight events (`baseBuild=16561` required to skip old-format compatibility block).
- **#140 closed:** already done in prior session; closed on GitHub this session.
- **#164 filed:** `PlayerState` public API needed before external race plugins can work (#74 prerequisite).
- **ADR-0007:** RaceModel plugin seam decision recorded.
- **2 garden entries:** GE-20260530-f75780 (baseBuild < 16561 corruption), GE-20260530-f5b1d9 (Delta.getRemoveMask returns Pair directly).

## Immediate Next Step

Pick up #127 (Phase 5 completion — EmulatedGame accuracy gaps) or #131 (Visualizer deferred work — 4 sub-tasks, M scale). Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #131 | Visualizer deferred — probe overlap, HTML mineral display, geyser sprite, time-based UI tests | M | Low | 4 discrete sub-tasks |
| #127 | Phase 5 completion — EmulatedGame accuracy gaps | L | High | Epic; may have child issues |
| #155 | Layer 3: casehub-qhorus inter-plugin messaging | L | High | Foundation gate pending |
| #156 | Layer 4: casehub-ledger audit trail | L | High | Foundation gate pending |

## References

| Context | Where |
|---------|-------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog entry (this session) | `blog/2026-05-30-mdp01-three-races-one-seam.md` |
| ADR-0007 | `docs/adr/0007-racemodel-plugin-seam.md` (project repo) |
| Garden entries | `~/.hortora/garden/scelight/GE-20260530-f75780.md`, `GE-20260530-f5b1d9.md` |
