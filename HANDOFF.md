# HANDOFF — quarkmind

## Orientation for Next Session

Gas calibration validated and documented. Multi-run SC2 recording revealed bimodal vespene output — single recordings are unreliable baselines. Liquipedia reference values reconciled against EM constants; 1w/2w gas rates match within 5% at close distance, 3w has a structural -26% gap from rotation-limited cycle timing. Physics doc fully audited and corrected — no stale values remain.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 2330 | 2445 | **+4.9%** |
| 12w minerals | 3460 | 3395 | **-1.9%** |
| 16w minerals | 4235 | 4165 | **-1.7%** |
| 3w gas (1 geyser) | 552 | 544 | **-1.4%** |
| 6w gas (2 geysers) | 1012 | 1016 | **+0.4%** |
| 6w gas (5-run mean) | 1140 | 1016 | **-10.9%** |

**Key findings this session:**
- SC2 6w gas is bimodal: {1044, 1204, 1204, 1044, 1204}. Mean=1140, CV=6.88%. Original single-run baseline (1012) was an outlier below both modes.
- Liquipedia gas rates match EM at 5u distance for 1w (-4.9%) and 2w (-5.1%). 3w rate is -25.8% at every distance — structural rotation-limited gap (3 × inside_time = 129.6 loops vs Liquipedia's implied 99 loops).
- Gas accuracy already within 2% of SC2 baselines (post-#689), no further calibration needed.
- Physics doc Liquipedia reconciliation complete — every value mapped to EM constant with delta and rationale.
- Physics doc fully audited: 4 stale values corrected (approach docking 12.5→11.5, gas cycle table, config block, dual-run gaps).

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #695 | 3w gas rotation-limited cycle vs Liquipedia | S/Med | EM's 3×43.2=129.6 loop cycle vs Liquipedia's implied 99 — investigate if approach docking is too high for rotation regime |
| #696 | Update 6w gas baseline to multi-run mean | S/Low | Replace single-run 1012 baseline with 5-run mean 1140 as the calibration target |
| #697 | MULE yield and rate reconciliation vs Liquipedia | S/Low | EM MULE yield +33%, rate multiplier +17-33% vs Liquipedia |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #686 | Remove dead resolveAllCollisions method | No-op — all dead code already removed in prior sessions |
| #688 | Re-record 6w gas baseline with multiple runs | 5 SC2 games recorded, bimodal distribution discovered |
| #685 | Liquipedia gas rate calibration tests | LiquipediaGasRateCalibrationTest (5 tests), structural 3w gap documented |
| #687 | Reconcile physics doc against Liquipedia reference | Full reconciliation: Gas, Minerals, MULE sections with deltas |
| #678 | Gas vespene cycle cadence measurement | Already resolved by #689, stale doc table corrected |

---

## Next Queue Item

| Issue | Title | Scale | Priority |
|-------|-------|-------|----------|
| #674 | artifact_sweep false-positive gaps | — | Active in queue |
| #695 | 3w gas rotation-limited cycle vs Liquipedia | S/Med | Filed this session |
| #696 | Update 6w gas baseline to multi-run mean | S/Low | Filed this session |
| #697 | MULE yield and rate reconciliation vs Liquipedia | S/Low | Filed this session |

---

## Branch State

On main. All work landed and published (1cf7637). 0 commits unpublished.

---

## Blog

"The Recording That Lied" — diary entry covering the bimodal gas discovery, Liquipedia gap, and calibration methodology lessons. In workspace blog directory, not yet promoted.
