# Handover — 2026-05-22

**Head commit (project):** `70656e9` — protocol(PP-20260522-572156): sc2data-train-times-require-calibration
**Head commit (workspace):** `7d31776` — feat: promote blog from issue-149-completesat-rounding

## What Changed This Session

- **QuarkMind moved into CaseHub family** — registered in `casehub/parent` `APPLICATIONS.md` and `docs/repos/quarkmind.md`. Repo physically relocated from `/Users/mdproctor/claude/quarkmind` to `/Users/mdproctor/claude/casehub/quarkmind`. Workspace symlinks (`proj/`, `CLAUDE.md`) and `.claude` project folder updated. **IntelliJ needs manual reopen** from the new path.
- **#149 closed** — `SC2Data.trainTimeInLoops` changed from `double` to `int`. Empirical calibration from 29 AI Arena replays via `SC2TrainTimeCalibrationTest` (range-bounded modal algorithm, two-source cross-validation): PROBE=272, ZEALOT=618, STALKER=698. The community formula `seconds × 22.4` gives wrong floats — SC2 uses integers that don't round consistently. `firstUnitDivergenceTick` moved from 86 to 150.
- **IEM10 incompatible for cross-validation** — 2016 build uses different abilLink IDs (Nexus=167 vs 175 in AI Arena). The 28 other AI Arena replays serve as source C instead.
- **4 garden entries submitted** (jvm/): SC2EGSet userID≠playerID-1 gotcha, SC2 abilLink patch-version gotcha, modal calibration technique, SC2 integer training-time undocumented.
- **Protocol PP-20260522-572156** — `sc2data-train-times-require-calibration`: SC2Data.trainTimeInLoops must be calibrated from replay data, not derived from seconds × 22.4.
- **Blog** `2026-05-22-mdp01-what-the-replay-actually-said.md` — published to mdproctor.github.io.

## Immediate Next Step

Run `work-start` and pick up **#148** (vespene income model — simplest path: sync GT vespene at each tick) or **#143** (multi-base mining in `mineralIncomePerTick`).

## Cross-Module

*None known.*

## What's Left

- `epic-phase-6` workspace branch: no EPIC-CLOSED.md, last commit 7 days ago — check if still active or close it. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #148 | Vespene income model — sync GT vespene or model Assimilator-based gas income | S | Low | Vespene sync is simpler |
| #143 | Multi-base mining in `mineralIncomePerTick(int probeCount, int nexusCount)` | M | Med | Single-base assumption documented |
| #145 | BuildIntent sub-tick timing + boundary test for offset=17 | M | Med | UnitInit/UnitDone give building calibration directly — see #145 comment |
| #147 | Switch exhaustiveness nits + queued-unit loop imprecision | XS | Low | Batched from code review |
| #144 | SC2DataTest: cover trainTimeInLoops default branch | XS | Low | |
| #150 | IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream | M | Med | Enables multi-game harness validation |
| #140 | Acquire Terran .SC2Replay files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Calibration spec | `docs/superpowers/specs/2026-05-21-completesat-rounding-design.md` |
| DESIGN.md | `docs/DESIGN.md` |
| Blog entry | `blog/2026-05-22-mdp01-what-the-replay-actually-said.md` |
| Garden entries | `jvm/GE-20260522-{f0d1ec,eccbde,a575c3,27d097}.md` |
| Protocol | `docs/protocols/sc2data-train-times-require-calibration.md` |
