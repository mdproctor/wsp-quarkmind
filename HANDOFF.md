# Handover — 2026-05-22

**Head commit (project):** `b010c67` — docs(claude-md): add Name field for write-blog frontmatter
**Head commit (workspace):** `c239751` — docs: add blog entry 2026-05-22-the-vespene-fix

## What Changed This Session

- **#144, #147, #148 closed and merged to main.** `maxUnitDelta ≤ 2` (was ≤ 15) — gas units now train via `setVespeneForHarness(gtBefore.vespene())`. The 2-unit residual is timing imprecision (±1 tick), not vespene rejection. `economicallyAccurate()` is permanently false — spec was too optimistic; `maxUnitDelta ≤ 2` is the correct assertion.
- **Switch expressions** — both `applyIntent` overloads converted to `Runnable` switch expressions for compile-time sealed-interface exhaustiveness.
- **CLAUDE.md tutorial layer discipline** — Layering Rule, Reference Documents, Design Phase References, Tutorial Structure, Foundation Gates added. LAYER-LOG.md created. Aligned with AML/clinical pattern.
- **CLAUDE.md hygiene** — git workflow (fork model: origin=personal fork, upstream=casehubio), blog routing (no hardcoded username), `**Name:** quarkmind` field added.
- **IntelliJ project path fixed** — `recentProjects.xml`, `trusted-paths.xml`, `.idea/workspace.xml` updated after repo relocation to `casehub/quarkmind`.
- **Orphaned `.meta` on workspace main** — `design/.meta` (branch: issue-148-emulated-fixes) merged with the work branch. Needs cleanup: `git rm design/.meta && git rm design/JOURNAL.md` on workspace main and commit.
- **2 garden entries** — GE-20260522-d6a1c9 (IntelliJ relocation config files), GE-20260522-5ff0b2 (Runnable switch expression pattern).

## Immediate Next Step

Clean up the orphaned `.meta` on workspace main:
```bash
git -C /Users/mdproctor/claude/public/quarkmind rm design/.meta design/JOURNAL.md
git -C /Users/mdproctor/claude/public/quarkmind commit -m "chore: remove orphaned work-branch scaffold from main"
git -C /Users/mdproctor/claude/public/quarkmind push origin main
```
Then run `work-start` and pick up **#143** (multi-base mining in `mineralIncomePerTick`) or **#145** (BuildIntent sub-tick timing + boundary test).

## What's Left

- Orphaned `design/.meta` on workspace main — clean up before next work-start · XS · Low
- `epic-phase-6` workspace branch: 7+ days without commits — check if still active or close it · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #143 | Multi-base mining in `mineralIncomePerTick(int probeCount, int nexusCount)` | M | Med | Single-base assumption documented |
| #145 | BuildIntent sub-tick timing + boundary test for offset=17 | M | Med | UnitInit/UnitDone give building calibration directly |
| #147 | Queued-unit loop imprecision — comment updated; tracking still open | XS | Low | Comment in drainBuildingQueues sharpened; issue tracks the underlying accuracy gap |
| #150 | IEM10JsonSimulatedGame: expose gameEvents as TimedIntent stream | M | Med | Enables multi-game harness validation |
| #140 | Acquire Terran .SC2Replay files for AbilityMapping | M | Med | Data exploration |
| #138 | Terran/Zerg EmulatedGame mechanics | L | High | Substantial new physics |

## References

| Context | Where |
|---------|-------|
| Blog entry | `blog/2026-05-22-mdp02-the-vespene-fix.md` |
| Design spec | `docs/superpowers/specs/2026-05-22-emulatedgame-fixes-design.md` |
| DESIGN.md | `docs/DESIGN.md` |
| LAYER-LOG.md | `LAYER-LOG.md` (project root — harness layer entries, header only) |
| Garden entries | `tools/GE-20260522-d6a1c9.md`, `jvm/GE-20260522-5ff0b2.md` |
