# HANDOFF — quarkmind

## Last Session

Working on #386 — Calibrate EmulatedGame physics from tournament ground truth. Upgrade accuracy investigation complete — 2 major bugs found and fixed. Branch: `issue-386-calibrate-emulated-physics`. Issue #387 paused on the stack (depends on #386).

### Completed This Session

1. **Cosmetic tracker upgrade filter** — `ReplaySimulatedGame.applyUpgrade()` was counting cosmetic UpgradeComplete events (sprays, dances, skins, GameHeartActive — ~5.7 per player at loop 0). These inflated the GT denominator from ~103 real upgrades to 1195 at 5-min. Added filter matching the pattern from `OracleAccuracyBaselineTest`. Upgrade accuracy: 2.8% → 33.0%.

2. **UPGRADE_NAME_MAP key alignment** — `ReplayCommandExtractor.UPGRADE_NAME_MAP` used CmdEvent ability names as keys (`CombatShield`, `ZerglingMovementSpeed`) while `AbilityMapping.dispatch()` produces SC2 internal names (`ShieldWall`, `zerglingmovementspeed`). The case-sensitive `Map.get()` silently dropped all intents for 5 upgrades. Fixed 5 entries. Upgrade accuracy: 33.0% → 58.3%.

3. **Per-upgrade breakdown** — Added per-upgrade-name table to `EmulatedGameAccuracyBaselineTest` output, enabling targeted diagnosis.

### Baseline After This Session (5-minute checkpoint, 118 oracle replays)

| Metric | Before Session | After Session | Target |
|--------|---------------|--------------|--------|
| Units | 76.8% | 76.8% | ≥80% |
| Buildings | 100% | 100% | ≥97% ✓ |
| Upgrades | 2.8% (34/1195) | 58.3% (60/103) | >30% ✓ |
| mineralsCurrent error | 16.5% | 12.8% | — |

### Key Findings

- **Upgrade accuracy was a measurement artifact** — cosmetic tracker events inflated the GT denominator 10x. True accuracy was always higher than reported.
- **5 UPGRADE_NAME_MAP keys mismatched** — `ShieldWall` vs `CombatShield`, `PunisherGrenades` vs `ConcussiveShells`, and 3 lowercase Zerg names (`zerglingmovementspeed`, `zerglingattackspeed`, `overlordspeed`) didn't match what `AbilityMapping.dispatch()` produces.
- **4 UpgradeType values missing from UPGRADE_NAME_MAP** — `HIGH_CAPACITY_BARRELS`, `LIBERATOR_RANGE`, `MEDIVAC_SPEED_BOOST`, `RAVEN_CORVID_REACTOR`. Minor impact (rare upgrades).
- **Remaining 42% gap at 5 min** — partially matched upgrades (Stimpack 20%, ZerglingSpeed 50%, ShieldWall 33%) likely blocked by `handleResearch()` mineral gate — minerals aren't synced in the harness. Also ~43% of CmdEvents are missing from Blizzard API replays per `CmdEventGapDiagnosticTest`.
- **Mineral accuracy improved as side-effect** — 16.5% → 12.8% error because correct upgrade cost deductions now track reality.

## Immediate Next Step

Continue #386 — remaining work:

1. **Batch 2, Task 3:** Calibrate upgrade timing constants from oracle replays — still uses formula-derived values that violate protocol. Won't significantly change accuracy but aligns with calibration mandate.

2. **Batch 4, Task 7:** Fix Queen intent generation (145 GT vs 25 emulated at 5 min). Extraction pipeline under-counts.

3. **Batch 4, Task 8:** Fix morph intent generation (Baneling 0%, Roach 35.4%).

4. **Upgrade accuracy ceiling** — the ~50% match rate for `zerglingmovementspeed` and `Stimpack` may be near the ceiling given Blizzard API CmdEvent capture rate (~43%). Further improvement requires tracker-event-based upgrade extraction rather than CmdEvent-based.

## References

- `specs/issue-386-calibrate-emulated-physics/` — design spec
- `plans/2026-10-11-calibrate-emulated-physics.md` — implementation plan
- `docs/benchmarks/emulated-game-accuracy-baseline.md` — current baseline data (includes per-upgrade breakdown)
- ReplaySimulatedGame.java:43-44 — COSMETIC_UPGRADE_PREFIXES
- ReplaySimulatedGame.java:181 — cosmetic filter in applyUpgrade()
- ReplayCommandExtractor.java:113-145 — fixed UPGRADE_NAME_MAP entries
- EmulatedGameAccuracyBaselineTest.java:106-119 — per-upgrade tracking
- #387 — paused on stack, depends on #386
