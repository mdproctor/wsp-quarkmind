# Handover — 2026-05-18

**Head commit (project):** `d7fbd54` — protocol(PP-20260518-cdcbb8): issue-refs-must-exist
**Head commit (workspace):** `35c04dd` — docs: add blog entry 2026-05-18 right-fix-wrong-hypothesis

⚡ **Both repos on:** `epic-saturation-mining` (EPIC-CLOSED.md committed — branches retained per user request)

## What Changed This Session

**epic-saturation-mining** — full lifecycle: started, implemented, closed.

- **Implemented** `SC2Data.mineralIncomePerTick(int probeCount)` — three-tier probe saturation model replacing flat `MINERALS_PER_PROBE_PER_TICK`. Array-driven (`MINERAL_TIER_RATES_PER_TICK[]`), configurable via `MINERAL_PATCHES_PER_BASE`. Mineral delta full-replay: 11,564 → 1,243 (~10×).
- **Discovered** unit divergence at tick 36 is NOT caused by minerals — same divergence with both flat and saturation models. Root cause: `trainTimeInTicks` integer outer-tick resolution (264 loops) vs real SC2 Probe train time (268.8 loops = 12.22 ticks). Filed as **#142**.
- **Filed #143** — multi-base mining (single-base assumption in `mineralIncomePerTick` documented in Javadoc)
- **#141 closed** via epic close
- **Protocol PP-20260518-cdcbb8** — issue refs in code comments must have real GitHub issues (`docs/protocols/issue-refs-must-exist.md`)
- **Garden** — GE-0144 revised: added "deleted source leaves orphaned .class, throws NoSuchMethodError" variant
- **Blog** — `2026-05-18-mdp01-right-fix-wrong-hypothesis.md` committed to workspace

## Immediate Next Step

**Delete the epic branches and return to main.** Run `/epic` from workspace — it will offer to delete both `epic-saturation-mining` branches and switch both repos to main.

Then pick next issue:
- **#142** Train-timing precision — `trainTimeInTicks` fractional tick rounding
- **#138** Terran/Zerg `EmulatedGame` mechanics
- **#140** Terran replay files for `AbilityMapping` discovery

## Open Issues

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

| Context | Where |
|---------|-------|
| Protocol index | `docs/protocols/INDEX.md` |
| DESIGN.md | `docs/DESIGN.md` |
| Blog entry | `blog/2026-05-18-mdp01-right-fix-wrong-hypothesis.md` |
| Spec | `docs/superpowers/specs/2026-05-17-saturation-mining-design.md` |
