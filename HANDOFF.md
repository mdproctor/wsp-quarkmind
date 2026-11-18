# HANDOFF — quarkmind

## Orientation for Next Session

MULE calldown fix landed (#711). The 8w per-cycle timing investigation (#714) is in progress on branch `issue-714-8w-per-cycle-timing` with diagnostic test and physics doc updates.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error | Notes |
|--------|-----|-----|-------|-------|
| 8w minerals | 2330 | 2430 | **+4.3%** | Per-cycle timing gap, not redistribution (#714) |
| 12w minerals | 3460 | 3450 | **-0.3%** | |
| 12w Zerg minerals | 3415 | 3425 | **+0.3%** | |
| 12w Terran minerals | 3460 | 3450 | **-0.3%** | |
| 16w minerals | 4270 | 4150 | **-2.8%** | |
| 24w minerals | 3955 | 3985 | **+0.8%** | |
| 1w speed (tick 121) | 1370 | 1355 | **-1.1%** | |
| MULE isolation | 3565 | 3686 | **+3.4%** | Fixed #711 — was -7.4% (zero MULE income) |
| Expansion transfer | 2625 | --- | --- | Invalid: instant construction (#713) |
| 3w gas (1 geyser) | 552 | 568 | **+2.9%** | |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** | |

**Key findings this session:**
- **MULE calldown root cause:** Naming convention mismatch — playbook YAML uses UPPER_SNAKE_CASE (`ORBITAL_COMMAND`) but `MORPH_BUILDING_TARGETS` maps CamelCase (`OrbitalCommand`). Morph silently dropped, no OC created, all calldowns failed. Fixed with `BuildingType.valueOf()` fallback in `EmulatedGame.handleMorph()` and `SC2DeliveryHandler.doMorph()`.
- **MULE income model:** Flat rate (277.8 min/min) over-produces by 3.4% vs SC2's trip-based mining. Trip-based model filed as #717.
- **8w timing investigation:** EM cycle = 5.12 ticks (112.6 loops) vs SC2 = 5.25 ticks (115.5 loops). Gap is 2.9 loops per cycle. SC2 cadence shows two clusters: close (~4.8t, 3 workers) and far (~5.6t, 5 workers). Analytical walk model doesn't match actual EM by ~10% — per-phase instrumentation needed (#719).
- **skipBuildings is a workaround:** #713 proposes adding fake construction time. Real fix is #716 (EM building placement at valid positions, eliminate skipBuildings entirely).
- **Physics doc drift:** 3 stale entries fixed — MULE calibration table, MULE footnote, §18 per-patch approach docking formula.

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #711 | EM MULE calldown missing | Fixed morph naming mismatch; MULE delta -7.4% -> +3.4% |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #716 | EM building placement — eliminate skipBuildings | L/High | Real fix for #713's workaround |
| #717 | MULE trip-based mining model | M/Med | Replace flat income rate with trip-based mining |
| #718 | Morph name valueOf warning | XS/Low | Log warning on valueOf fallback |
| #719 | WorkerSim phase instrumentation | S/Med | Per-worker phase timing for #714 investigation |

---

## Active Work

**Branch:** `issue-714-8w-per-cycle-timing` (2 commits)
- `wip: add 8w cycle timing diagnostic test`
- `docs(#711): fix stale MULE and approach docking entries in physics doc`

The diagnostic test (`WorkerSimulationTest#measureCycleTimes8w`) runs the 8w calibration scenario and measures EM per-worker delivery cycles. Next step: add per-phase timing instrumentation (#719) to identify which phase is too short.

---

## Queue (9 open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#714** | M/Med | **ACTIVE** — 8w per-cycle timing gap at zero contention |
| 2 | #706 | XS/Low | SVG validation test coverage (soredium cross-repo) |
| 3 | #713 | M/Med | skipBuildings construction time delay (consider closing if #716 supersedes) |
| 4 | #715 | S/Low | Zerg Larva unit lifecycle (training/regen) |
| 5 | #716 | L/High | EM building placement — eliminate skipBuildings |
| 6 | #717 | M/Med | MULE trip-based mining model |
| 7 | #718 | XS/Low | Morph name valueOf warning |
| 8 | #719 | S/Med | WorkerSim phase instrumentation (prerequisite for #714) |

**On branch `issue-714-8w-per-cycle-timing`** — `work continue` to resume.

**Recommended next:** Continue #714 by first doing #719 (phase instrumentation) to identify the gap source. #715 or #718 for quick palette cleansers.

---

## Cross-Repo Changes

None this session.

---

## Unpublished

Project and workspace repos have unpublished commits. Publish with `canonical_sync.py publish` when ready.
