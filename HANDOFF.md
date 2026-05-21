# Handover — 2026-05-21

**Head commit (project):** `9682f5c` — docs(issue-146-building-cost-mineral-timing): apply design journal
**Head commit (workspace):** `63a0bad` — feat: promote blog from issue-146-building-cost-mineral-timing

## What Changed This Session

- **#146 investigated and closed** — building cost deduction wired into harness made divergence worse (firstUnitDivergenceTick moved from 86→49, maxUnitDelta 2→48). Root cause: GT PlayerStats events fire every ~10 ticks, so GT mineral readings are stale at building-injection time; deducting cost against EM's continuous balance creates artificial debt that blocks legitimate trains.
- **Correct diagnosis established** — two independent divergence causes: (1) `completesAt` in `startTraining` rounds 1 tick early for specific loop offsets (timing formula gap, not mineral); (2) no vespene income model (Stalker/Immortal trains rejected). Filed #148 for vespene gap.
- **`injectReplayBuildingWithCost(Building)` added to EmulatedGame** — clean API with 4 unit tests; allows negative mineral balance (debt); not wired into harness. `docs/DESIGN.md` updated with two-method API and corrected Next Steps (#142 precision + #148 vespene).
- **`ReplayValidationTest` docstring corrected** — two-cause breakdown replaces single-cause mineral narrative.
- **Garden**: GE-20260521-981f62 submitted (SC2 PlayerStats staleness — stale GT mineral readings).
- **Blog**: 2026-05-21-mdp02-the-fix-that-made-things-worse.md — both session entries published to mdproctor.github.io.

## Immediate Next Step

Pick up **#143** (multi-base mining in `mineralIncomePerTick`) or **#142** (investigate the `completesAt` rounding — why does it produce tick 86 instead of tick 87 for that specific loop offset?). Run `work-start` first.

## Cross-Module

*None known.*

## What's Left

- `epic-phase-6` workspace branch: no EPIC-CLOSED.md, last commit 7 days ago. Check if still active or close it. · XS · Low
- `epic-saturation-mining` EPIC-CLOSED.md on workspace main (not the branch) — known gap from previous session. Branches retained until 2026-06-03. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #142 | Investigate `completesAt` rounding — why 1-tick-early for specific loop offset at tick 86 | S | Med | Timing formula fix; may need replay data inspection |
| #143 | Multi-base mining in `mineralIncomePerTick(int probeCount, int nexusCount)` | M | Med | Single-base assumption documented; design needed |
| #148 | Vespene income model — sync GT vespene or model Assimilator-based gas income in harness | S | Low | Vespene sync is simpler; SC2Data.vespeneCost(BuildingType) doesn't exist yet |
| #140 | Terran replay files for `AbilityMapping` discovery | M | Med | Data exploration |
| #138 | Terran/Zerg `EmulatedGame` mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Spec | `docs/superpowers/specs/2026-05-21-building-cost-mineral-timing-design.md` |
| DESIGN.md | `docs/DESIGN.md` |
| Blog entry | `blog/2026-05-21-mdp02-the-fix-that-made-things-worse.md` |
| Garden entry | `jvm/GE-20260521-981f62.md` (SC2 PlayerStats staleness) |
