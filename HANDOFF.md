# HANDOFF — quarkmind

## Orientation for Next Session

#717 (MULE trip-based mining) is **closed**. The fix: replaced the flat-rate MULE income model (277.8 min/min spread uniformly per tick) with trip-based mining through WorkerSimulation — MULEs walk to patches, mine 25 minerals, walk back, deposit, repeat until expiry. MULE isolation delta improved from +3.4% to +1.4%.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304–305):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 8w minerals | 2330 | 2340 | **+0.4%** | |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| 12w Zerg minerals | 3415 | 3425 | **+0.3%** | |
| 12w Terran minerals | 3460 | 3450 | **-0.3%** | |
| 16w minerals | — | — | diverges tick 62 | PROBE 17 vs 15 (#723) |
| 24w minerals | — | — | diverges tick 62 | PROBE 17 vs 15 (#723) |
| 1w speed (tick 121) | 1370 | 1355 | **-1.1%** | |
| MULE isolation (tick 305) | 3565 | 3615 | **+1.4%** | trip-based, asserted ≤2% |
| Expansion transfer | — | — | diverges tick 101 | tolerance 5 passes |
| 3w gas (1 geyser) | — | — | diverges tick 63 | PROBE 17 vs 15 (#723) |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** | passes |

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #717 | MULE trip-based mining model | Fixed — MULEs in separate `mules` map in WorkerSimulation; cargo=25, mining overhead=20 loops; delta +1.4% |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #724 | MULE worker snapshots not visible | XS/Low | workerSnapshots() doesn't include MULEs |
| #725 | Remove dead muleIncomePerTick | XS/Low | Flat rate method no longer used in production |
| #726 | MULE cargo calibration — verify 25 vs 30 | S/Low | Empirical verification needed |

**Blog written:** `2026-11-18-mdp05-mules-dont-get-a-flat-rate.md` — trip-based MULE mining replacing flat rate, connects to "Simulate the Mechanism" philosophy.

---

## Queue (9 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#718** | XS/Low | **NEXT** — Morph name valueOf warning |
| 2 | #719 | S/Med | WorkerSim phase instrumentation |
| 3 | #720 | XS/Low | Wire LOW_SATURATION_APPROACH_BONUS through WorkerPhysicsConfig |
| 4 | #721 | S/Low | Validate low-saturation approach bonus at boundary worker counts |
| 5 | #722 | XS/Low | Diagnostic tests debugCreateBuilding bypass |
| 6 | #723 | S/Med | Expansion-transfer SC2 divergence tick 62 |
| 7 | #724 | XS/Low | MULE worker snapshots not visible |
| 8 | #725 | XS/Low | Remove dead muleIncomePerTick |
| 9 | #726 | S/Low | MULE cargo calibration verify 25 vs 30 |

**On main** — `work continue` to start #718.

**Recommended next:** #718 (XS/Low morph warning) for a quick fix, or #725 (XS/Low dead code cleanup) since it's directly related to the MULE work just landed. #723 (S/Med tick-62 divergence) is the next calibration-domain item if staying in physics work.

---

## Unpublished

Both project (1 commit) and workspace (5 commits) repos have unpublished commits on main. Run `canonical_sync.py publish` to push.
