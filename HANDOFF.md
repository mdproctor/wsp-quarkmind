# HANDOFF — quarkmind

## Orientation for Next Session

**#734, #736, #733 completed** this session. Terrain defaults, rectangular building footprints, enforceWall conversion. #735 (mineral patch orientation) deferred — obstacle size needs empirical calibration before converting shape.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Key changes this session:**

1. **EmulatedGame default terrain (#734):** Constructor now calls `setTerrainGrid(TerrainGrid.emulatedMap())`. All tests get obstacle-aware worker pathfinding without per-site wiring. Fog of war decoupled into separate `fogEnabled` flag (default false). Production code (`EmulatedEngine`, `EmulatedSC2Server`) sets `fogEnabled(true)` explicitly.

2. **Rectangular building footprints (#736):** `SC2Data.buildingFootprint(BuildingType)` returns `int[2]` tile dimensions from SC2 data editor. `enforceWall()` uses `insideFootprint()` rectangular check instead of circular `buildingRadius()`. Static shared arrays for zero hot-path allocation. `buildingRadius()` deprecated.

3. **Mineral patch obstacle calibration finding:** Converting mineral patch obstacles from circular (0.3 radius, ~1 tile) to rectangular (2×1 tiles) caused +4.1% MULE regression. The 0.3-radius value was empirically calibrated for the pathfinder. Converting to rectangular needs its own calibration pass, not a mechanical swap. Reverted mineral patches to circular.

**Accuracy state unchanged** — building footprint change doesn't affect economy (enforceWall affects army, not workers). Mineral patches reverted, so calibration numbers are the same as the previous session.

---

## Completed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #734 | Wire TerrainGrid into all economy tests | Done — constructor default, fog decoupled |
| #736 | SC2Data.buildingFootprint() tile dimensions | Done — static lookup, enforceWall wired, buildingRadius deprecated |
| #733 | Replace circular approximations with rectangular footprints | Done — building collision converted; mineral patches deferred to #735 |

**Deferred:**

| Issue | Title | Why |
|-------|-------|-----|
| #735 | Mineral patch orientation for rectangular footprints | Obstacle SIZE must be calibrated before shape conversion; 2×1 footprint blocks too many tiles |
| #737 | Replace edgePoint circle with rectangle nearest-point | Blocked by #735 (needs orientation) |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #738 | EVOLUTION_CHAMBER footprint verification | XS/Low | May be 3×3 not 2×2 per SC2 data editor |

**Blog entries:** `2026-11-20-mdp01-rectangles-not-radii.md` — building collision switches from circular to rectangular; fog decoupling; mineral patch calibration finding

---

## Queue (open items)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#735** | S/Med | **ACTIVE** — mineral patch orientation (deferred — needs calibration pass) |
| 2 | #737 | S/Med | edgePoint rectangle (blocked by #735) |
| 3 | #731 | M/Med | Army pathfinding around buildings |
| 4 | #732 | M/Med | Army pathfinding around resources |
| 5 | #730 | S/Low | Blog SC2/EM column swap audit |

**Recommendation:** Skip #735/#737 for now and start #731 (army pathfinding) — different subsystem, independent of mineral patch calibration. Or tackle #738 (XS/Low verification) first as a warm-up.
