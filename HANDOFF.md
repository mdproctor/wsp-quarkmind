# HANDOFF — quarkmind

## Orientation for Next Session

#714 (8w per-cycle timing gap) is **closed**. The fix: `LOW_SATURATION_APPROACH_BONUS` (4.5 loops) added to approach docking when base saturation is ≤1 worker per non-depleted patch. The key discovery was that SC2 per-worker cycle time is constant (~116 loops) regardless of saturation.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 8w minerals | 2330 | 2340 | **+0.4%** | Fixed #714 — was +4.3% |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| 12w Zerg minerals | 3415 | 3425 | **+0.3%** | |
| 12w Terran minerals | 3460 | 3450 | **-0.3%** | |
| 16w minerals | 4270 | 4150 | **-2.8%** | |
| 24w minerals | 3955 | 3985 | **+0.8%** | |
| 1w speed (tick 121) | 1370 | 1355 | **-1.1%** | |
| MULE isolation | 3565 | 3686 | **+3.4%** | Fixed #711 — flat rate model |
| Expansion transfer | 2625 | --- | --- | Invalid: instant construction (#713) |
| 3w gas (1 geyser) | 552 | 568 | **+2.9%** | |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** | |

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #714 | 8w per-cycle timing gap at zero contention | Fixed — LOW_SATURATION_APPROACH_BONUS (4.5 loops); 8w +4.3% -> +0.4%, 12w unchanged |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #720 | Wire LOW_SATURATION_APPROACH_BONUS through WorkerPhysicsConfig | XS/Low | Config record integration |
| #721 | Validate low-saturation approach bonus at boundary worker counts | S/Low | Test at 4w, 6w, 10w |

---

## Queue (10 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#706** | XS/Low | **NEXT** — SVG validation test coverage (soredium cross-repo) |
| 2 | #713 | M/Med | skipBuildings construction time delay |
| 3 | #715 | S/Low | Zerg Larva unit lifecycle (training/regen) |
| 4 | #716 | L/High | EM building placement — eliminate skipBuildings |
| 5 | #717 | M/Med | MULE trip-based mining model |
| 6 | #718 | XS/Low | Morph name valueOf warning |
| 7 | #719 | S/Med | WorkerSim phase instrumentation |
| 8 | #720 | XS/Low | Wire LOW_SATURATION_APPROACH_BONUS through WorkerPhysicsConfig |
| 9 | #721 | S/Low | Validate low-saturation approach bonus at boundary worker counts |

**On main** — `work continue` to start #706.

**Recommended next:** #706 (XS/Low SVG validation) for a quick win, or #718 (XS/Low morph warning) for another small fix. #720 and #721 are fast follow-ups to the #714 work if staying in the mining domain.

---

## Cross-Repo Changes

None this session.

---

## Published

Both project and workspace repos are published and up to date with remote.
