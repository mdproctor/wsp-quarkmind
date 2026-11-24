# HANDOFF — quarkmind

## Orientation

**Pinned calibration on real TorchesAIE terrain reveals concrete gap.** The EM mines faster than SC2 — especially after expansion. Single-base: -2.3% (EM under-produces slightly). Two-base: +20% (EM over-produces significantly). The methodology is now apples-to-apples: same build order pinned from SC2 baseline at exact ticks, same map, same terrain.

**Start with:** Investigate why the EM over-produces on two bases. The per-tick data shows EM earning 37% more cumulative income by tick 50 (675 vs 925) with the same probe count. After expansion at tick 88, the gap compounds. 2 of the 62 EM probes are from a remaining high-water-mark edge case — fix that, then focus on the per-cycle rate.

**Key data points:**
- Single-base pinned: SC2=3695, EM=3610, **Δ=-2.3%** (36 probes both sides)
- Two-base pinned: SC2=4350, EM=5220, **Δ=+20%** (SC2=60, EM=62 probes)
- Max overshoot: +345 minerals at tick 93 (single-base), +870 at tick 449 (two-base)
- Expansion completes at tick 88 in SC2

---

## What's ready

- **TorchesAIE pathing grid:** 160×208, cached at `terrain/Torches AIE.pathgrid`
- **Pinned calibration method:** `pinnedCalibrate()` replays SC2 build order at exact ticks, compares mineral/vespene only
- **Two baselines recorded:** `cal-single-base-probes.json` (305 ticks) and `cal-two-base-economy.json` (450 ticks)
- **High water mark fix:** Probe oscillation during Protoss building warps no longer causes phantom injections

---

## This Session (2026-11-23/24)

| Fix/Feature | Impact |
|---|---|
| #756 — Terrain grid auto-sizing | Flat grid auto-sized from patch positions |
| #757 — Saturation-aware worker assignment | mineral rate 38.2%→13.7% (oracle) |
| #760 — MapTerrainCache + real terrain | TorchesAIE 160×208 recorded, `calibrate()` uses real coords |
| #760 — Pinned calibration | Replays SC2 build at exact ticks — single-base -2.3%, two-base +20% |
| #761 — Spending leaks | BuildingBornIntent and tracker morphs now deduct costs |
| Standing rules | CLAUDE.md + physics doc: real SC2, controlled replays, real maps |

---

## Branches

Stacked (pipeline bug soredium#617):
- `issue-756-seed-expansion-mineral-patches` (from main)
- `issue-757-worker-transfer-count-varies` (on #756)
- `issue-758-gas-worker-over-assignment` (on #757)
- `issue-761-intent-rejection-diagnostic` (on #758)

---

## Notes

- **Pipeline bug (soredium#617):** Workaround: create branches manually.
- **DroolsScoutingRulesTest/DroolsStrategyL2L3Test:** Pre-existing classpath issues.
- **DivergenceRegressionTest:** Passes.
- **Pinned two-base still has 2 extra probes** (62 vs 60) — high water mark edge case with Protoss building warp oscillation.
