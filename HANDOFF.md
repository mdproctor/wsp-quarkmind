# HANDOFF — quarkmind

## Orientation for Next Session

Gas approach docking recalibrated (#677). After serial access (#682), the docking constant was zeroed — the assumption was that rotation wait would absorb approach overhead. It didn't: gas over-production spiked to +14% (3w) and +20% (6w). Sweep found 12.5 loops minimises combined error. The cycle is now rotation-limited (136 loops actual vs 135 theoretical 3×inside).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 3460 | 3430 | **-0.9%** |
| 12w minerals | 3955 | 3850 | **-2.7%** |
| 16w minerals | 4235 | 4090 | **-3.4%** |
| 3w gas (1 geyser) | 552 | 536 | **-2.9%** |
| 6w gas (2 geysers) | 1012 | 1032 | **+2.0%** |
| Zerg 12w minerals | 2330 | 2430 | **+4.3%** |
| Full test suite | — | 3367 pass | — |

**Key changes this session:**
- `GAS_APPROACH_DOCKING_LOOPS` = 12.5 (was 0 after #682, was 15.8 before #682)
- Sweep test `sweepGasApproachDocking` added to `ApproachDockingSweepTest`
- WorkerSimulationTest 3w/2w ratio assertion updated for serial access (1.02 not 1.5)
- Gas cycle: 136 loops, rotation-limited (100.7% of theoretical 3×inside)
- Physics doc fully synced with current constants and cycle table
- Blog: "The Constant the Model Forgot" — why structural model changes invalidate calibrated constants
- Filed #689: 3w/6w gas asymmetry (-2.9% vs +2.0%)

**Gas cycle measured breakdown (3w, 1 geyser, docking=12.5):**

| Phase | Loops | % of cycle |
|-------|-------|------------|
| Walk to geyser | 34 | 25% |
| Wait at entry | 8 | 6% |
| Inside (harvest+dock) | 45 | 33% |
| Walk to TH | 36 | 26% |
| Deposit | 12 | 9% |
| **Total** | **136** | **100%** |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #684 | Gas worker cycle timing diagnostic test | Done — per-phase measurement, 3 test methods |
| #677 | Gas approach docking recalibration | Done — 12.5 loops, 3w -2.9%, 6w +2.0% |

## Filed This Session

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #689 | 3w/6w gas asymmetry | S/Med | -2.9% vs +2.0% at same docking — assignment timing |

---

## Next Queue Items

| Issue | Title | Scale | Priority |
|-------|-------|-------|----------|
| #679 | Per-patch walk distance extension | M/Med | Root fix for straight-line walk gap — affects minerals (-2.7% to -3.4%) and gas (-2.9%) |
| #676 | Reverse-engineer SC2 rate computation | —/— | Break ~16% rate MAPE floor |
| #666 | Forward simulation economy mode | L/High | EM-driven income without GT sync |

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment.

---

## Branch State

On main. All work landed and published.
