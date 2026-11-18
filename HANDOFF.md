# HANDOFF — quarkmind

## Orientation for Next Session

#716 (eliminate skipBuildings) is **closed**. The fix: replaced the `debugCreateBuilding()` bypass in `DualRunCalibrationTest.calibrate()` with `game.applyIntent(BuildIntent.of(...))`, routing all calibration builds through the existing `EmulatedGame.handleBuild()` physics pipeline. Buildings now get real construction time via `PendingCompletion`, resource deduction, and supply granted only on completion. The `skipBuildings` and `deductBuildingCosts` parameters were removed entirely.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 8w minerals | 2330 | 2340 | **+0.4%** | |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| 12w Zerg minerals | 3415 | 3425 | **+0.3%** | |
| 12w Terran minerals | 3460 | 3450 | **-0.3%** | |
| 16w minerals | — | — | diverges tick 62 | PROBE 17 vs 15 (#723) |
| 24w minerals | — | — | diverges tick 62 | PROBE 17 vs 15 (#723) |
| 1w speed (tick 121) | 1370 | 1355 | **-1.1%** | |
| MULE isolation | — | — | diverges tick 62 | SCV 17 vs 15 (#723) |
| Expansion transfer | — | — | diverges tick 101 | tolerance 5 passes |
| 3w gas (1 geyser) | — | — | diverges tick 63 | PROBE 17 vs 15 (#723) |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** | passes |

**Post-#716 divergence pattern:** Tests that build structures now diverge at tick 62-63: the EM produces 15 probes/SCVs while SC2 has 17. The delta of 2 is within tolerance for most tests. The root cause is construction time delay — the first Pylon/Depot takes ~18 ticks to build, capping supply at 15 until completion. See #723 for investigation.

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #706 | SVG validation test coverage (soredium cross-repo) | Tests added in soredium — 8 tests covering validate_svg(), promote_gaps, copromote SVG paths |
| #713 | skipBuildings construction time delay | Closed as superseded by #716 — fake delay is a compensation hack |
| #716 | EM building placement — eliminate skipBuildings | Fixed — applyIntent replaces debugCreateBuilding; net -19 lines |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #722 | Diagnostic tests still use debugCreateBuilding bypass | XS/Low | ExpansionTransferDiagnosticTest playbook loop |
| #723 | Expansion-transfer SC2 divergence at tick 62 | S/Med | 2-probe gap from construction time; affects 16w, 24w, gas, MULE |

**Blog revised:** `2026-11-18-mdp02-when-your-tests-pass-but-your-numbers-lie.md` — added "Eliminating the bypass" section covering the #716 resolution and the workaround gravity lesson.

---

## Queue (8 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#715** | S/Low | **NEXT** — Zerg Larva unit lifecycle (training/regen) |
| 2 | #717 | M/Med | MULE trip-based mining model |
| 3 | #718 | XS/Low | Morph name valueOf warning |
| 4 | #719 | S/Med | WorkerSim phase instrumentation |
| 5 | #720 | XS/Low | Wire LOW_SATURATION_APPROACH_BONUS through WorkerPhysicsConfig |
| 6 | #721 | S/Low | Validate low-saturation approach bonus at boundary worker counts |
| 7 | #722 | XS/Low | Diagnostic tests debugCreateBuilding bypass |
| 8 | #723 | S/Med | Expansion-transfer SC2 divergence tick 62 |

**On main** — `work continue` to start #715.

**Queue note:** The queue's active marker was misaligned this session (#715 marked active but #716 was worked). The ceremony closed #715's queue slot but #715 (Zerg Larva) was NOT worked — it remains open on GitHub. Next session should verify queue state before starting.

**Recommended next:** #718 (XS/Low morph warning) for a quick fix, or #715 (S/Low Zerg Larva) to continue the physics calibration sequence. #723 (S/Med tick-62 divergence) is a follow-up to #716 if staying in the building calibration domain.

---

## Cross-Repo Changes

| Repo | Change | Commit |
|------|--------|--------|
| soredium | Added `tests/test_svg_validation.py` — 8 SVG validation tests | bcf95c4 |

---

## Unpublished

Both project (1 commit) and workspace (4 commits) repos have unpublished commits on main. Run `canonical_sync.py publish` to push.
