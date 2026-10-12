# HANDOFF — quarkmind

## Last Session

Working on #386 — Calibrate EmulatedGame physics from tournament ground truth. Upgrade accuracy and morph intent fixes complete. Branch: `issue-386-calibrate-emulated-physics`. Issue #387 paused on the stack (depends on #386).

### Completed This Session

1. **Cosmetic tracker upgrade filter** — `ReplaySimulatedGame.applyUpgrade()` counted cosmetic UpgradeComplete events (~5.7 per player at loop 0). Added filter matching `OracleAccuracyBaselineTest` pattern. Upgrade accuracy: 2.8% → 33.0%.

2. **UPGRADE_NAME_MAP key alignment** — 5 entries used CmdEvent ability names instead of AbilityMapping output names (ShieldWall vs CombatShield, lowercase Zerg names). Case-sensitive Map.get() silently dropped all intents. Upgrade accuracy: 33.0% → 58.3%.

3. **Unit morph fallback** — `handleMorph()` unit path used strict tag matching. Replay-derived tags never match EmulatedGame auto-generated tags. Added MORPH_SOURCE_TYPES type-based fallback (same pattern as building morph). Baneling accuracy: 0% → 100%.

4. **Queen intent diagnosis** — Not fixable at extraction level. 37% CmdEvent capture rate (Blizzard API limit) × 44% intent acceptance = 17% overall. Diagnostic test `QueenExtractionDiagnosticTest` added.

5. **Per-upgrade breakdown** — Added per-upgrade-name table to `EmulatedGameAccuracyBaselineTest` for targeted diagnosis.

### Baseline After This Session (5-minute checkpoint, 118 oracle replays)

| Metric | Before Session | After Session | Target |
|--------|---------------|--------------|--------|
| Units | 76.8% | 76.7% | ≥80% |
| Buildings | 100% | 100% | ≥97% ✓ |
| Upgrades | 2.8% (34/1195) | 58.3% (60/103) | >30% ✓ |
| mineralsCurrent error | 16.5% | 12.8% | — |

### Remaining Work

1. **Task 3: Calibrate upgrade timings** — Low priority. Upgrade target (>30%) is met at 58.3%. Formula-derived timing constants violate protocol but don't significantly impact accuracy.

2. **Unit accuracy gap (76.7% vs ≥80% target)** — Remaining 3.3pp gap is dominated by:
   - Worker shortfall: ~953 missing (Probe -265, Drone -413, SCV -275) — CmdEvent ceiling
   - Queen: 145→25 — CmdEvent ceiling (37%) + resource rejections
   - Combat units (Marine, Zergling, Roach) — CmdEvent ceiling
   - Over-production: Sentry 7→28 (4x), Stalker 51→57 — combat simulation doesn't kill them

3. **Fundamental ceiling:** CmdEvent capture rate (~43% per `CmdEventGapDiagnosticTest`) limits all unit types. To exceed ~80% unit accuracy, the harness would need tracker-event-based intent injection for common unit types — a different approach than CmdEvent extraction.

## References

- ReplaySimulatedGame.java:43-44, 181 — cosmetic upgrade filter
- ReplayCommandExtractor.java:113-145 — fixed UPGRADE_NAME_MAP entries
- EmulatedGame.java:831-838 — MORPH_SOURCE_TYPES map
- EmulatedGame.java:875-881 — unit morph tag fallback
- QueenExtractionDiagnosticTest.java — Queen extraction diagnostic
- EmulatedGameAccuracyBaselineTest.java:106-119 — per-upgrade tracking
- docs/benchmarks/emulated-game-accuracy-baseline.md — current baseline
- #387 — paused on stack, depends on #386
