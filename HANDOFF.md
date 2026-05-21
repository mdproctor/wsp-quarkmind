# Handover — 2026-05-21

**Head commit (project):** `ee2bdc1` — docs: add SC2DataTest to unit test list in CLAUDE.md
**Head commit (workspace):** `e499f52` — docs: add blog entry 2026-05-21 two-fixes-finally-together

## What Changed This Session

- **epic-saturation-mining rebased and merged** — 4 orphaned commits (saturation model, design journal, protocol, spec) rebased onto main in both repos. Conflicts in `SC2Data.java` (timing constants + saturation tier rates), `ReplayValidationHarness.java` (probe count multiplier), `ReplayValidationTest.java` (Javadoc merge), `docs/DESIGN.md` (three locations). All resolved. 672 tests pass.
- **Issues #130 and #141 closed** — saturation mining model complete; building-cost divergence (#146) is the remaining gap.
- **Workspace cleanup** — orphaned `design/.meta` removed from workspace main; `EPIC-CLOSED.md` relocated to `design/` (note: landed on workspace main rather than epic branch — minor convention gap, no action needed).
- **CLAUDE.md** — `SC2DataTest` added to unit test list.
- **Garden** — GE-20260521-b6a1a7 submitted: `git merge --ff-only` brings branch-only workflow markers to main.

## Immediate Next Step

Pick up **#146** (building-cost mineral timing — the remaining divergence cause after both #141 and #142) or **#143** (multi-base mining in `mineralIncomePerTick`). Either is ready to start. Run `work-start` first.

## Cross-Module

*None known.*

## What's Left

- `EPIC-CLOSED.md` for saturation-mining is on workspace main, not the epic branch. Hygiene scan won't find it there. Low priority — branch is closed and issues are shut. · XS · Low
- `epic-saturation-mining` branches retained; scheduled deletion 2026-06-03 per `design/EPIC-CLOSED.md`.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #146 | Building-cost mineral timing — deduct building costs in harness to eliminate remaining divergence | M | Med | Direct follow-on from #141/#142; firstUnitDivergenceTick target: -1 |
| #143 | Multi-base mining in `mineralIncomePerTick(int probeCount, int nexusCount)` | M | Med | Single-base assumption documented; design needed |
| #140 | Terran replay files for `AbilityMapping` discovery | M | Med | Data exploration |
| #138 | Terran/Zerg `EmulatedGame` mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Protocol index | `docs/protocols/INDEX.md` |
| DESIGN.md | `docs/DESIGN.md` |
| Blog entry | `blog/2026-05-21-mdp01-two-fixes-finally-together.md` |
| Garden entry | `tools/GE-20260521-b6a1a7.md` (ff-only merge brings branch markers to main) |
