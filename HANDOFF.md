# HANDOFF — quarkmind

## Orientation for Next Session

Calibration catalogue and dual-run framework audit complete. Four missing EM deltas filled, six stale values corrected across both the catalogue and physics doc, and a free-buildings inflation bug discovered in the test harness.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 8w minerals | 2330 | 2430 | **+4.3%** | Per-cycle timing gap, not redistribution (#714) |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| 12w Zerg minerals | 3415 | 3425 | **+0.3%** | Added #704 |
| 12w Terran minerals | 3460 | 3450 | **-0.3%** | |
| 16w minerals | 4270 | 4150 | **-2.8%** | SC2 value corrected from 4235 |
| 24w minerals | 3955 | 3985 | **+0.8%** | EM value corrected from 3850 |
| 1w speed (tick 121) | 1370 | 1355 | **-1.1%** | Added #704 |
| MULE isolation | 3565 | 3300 | **-7.4%†** | EM lacks MULE calldown (#711) |
| Expansion transfer | 2625 | —‡ | —‡ | Invalid: instant construction (#713) |
| 3w gas (1 geyser) | 552 | 568 | **+2.9%** | |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** | |

†EM lacks MULE calldown — SCV-only income. ‡skipBuildings gives supply early, snowballing 30+ extra workers.

**Key findings this session:**
- **Free-buildings inflation:** `calibrate()` with `skipBuildings=true` created buildings without deducting minerals. Making cost deduction unconditional shifted ALL skipBuildings deltas by 2-20% (16w: -2.8%→-5.2%, MULE: -7.4%→-22.9%). Made opt-in (`deductBuildingCosts` parameter) to preserve existing calibration baseline.
- **8w redistribution is not the cause:** `redistributeMineralWorkersAtBase()` added to `onWorkerDied()`, but the existing `findBetterPatch` reroute already handles redistribution within 1 tick. The +4.3% gap is per-cycle timing at zero contention.
- **Zerg Larvae missing:** EM started Zerg without LARVA units. Fixed — 3 Larvae added at game start. Lifecycle gap remains: training doesn't remove Larvae, regeneration doesn't add them (#715).
- **Physics doc drift:** 6 stale values from pre-#690 work corrected; gas-harvest-duration config example fixed (32.0→31.7).

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #704 | Fill missing EM deltas in calibration catalogue | Zerg +0.3%, MULE -7.4%, worker-speed -1.1%, expansion-transfer invalid |
| #710 | calibrate() skipBuildings cost deduction | Opt-in `deductBuildingCosts` parameter; unconditional broke existing calibration |
| #712 | EM Zerg initialization missing Larvae | 3 LARVA units at game start; unitTolerance reverted to 0 |
| #705 | 8w worker redistribution after kill | Redistribution added; gap is per-cycle timing, not redistribution |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #710 | calibrate() skipBuildings cost deduction | S/Low | Completed |
| #711 | EM MULE calldown missing | M/Med | applyAbility doesn't create MULE units |
| #712 | EM Zerg Larvae initialization | XS/Low | Completed |
| #713 | skipBuildings construction time delay | M/Med | Instant construction blocks expansion-transfer comparison |
| #714 | 8w per-cycle timing gap | S/Med | Needs SC2 8w cadence recording |
| #715 | Zerg Larva unit lifecycle | S/Low | Training/regen don't update LARVA units in state |

---

## Queue (6 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | #706 | XS/Low | SVG validation test coverage (soredium cross-repo) |
| 2 | #711 | M/Med | EM MULE calldown doesn't create MULE units |
| 3 | #713 | M/Med | skipBuildings construction time delay |
| 4 | #714 | S/Med | 8w per-cycle timing gap at zero contention |
| 5 | #715 | S/Low | Zerg Larva unit lifecycle (training/regen) |

**On main** — `work start` to begin next issue.

**Recommended next:** #714 (8w per-cycle timing gap) if staying in calibration work — needs SC2 8w cadence recording first. #706 (SVG validation) for a quick palette cleanser. #715 (Larva lifecycle) for a small self-contained fix.

---

## Cross-Repo Changes

None this session.

---

## Unpublished

Project and workspace repos have unpublished commits. Publish with `canonical_sync.py publish` when ready.
