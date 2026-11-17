# HANDOFF — quarkmind

## Orientation for Next Session

Gas worker cycle diagnostic landed (#684). Analytical gas cycle formulas gave inconsistent results (17-46 loops depending on method). The diagnostic test instruments each gas worker through a full cycle at tick resolution, measuring walk-to-geyser, wait-at-entry, inside-geyser, walk-to-TH, and deposit phases separately. Key finding: the cycle is walk-limited (60% of time), not harvest-limited. Rate is 45.6 gas/min/worker vs SC2 target 44.3 — the 3% over-production traces to straight-line walks lacking SC2 path deviation.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state:**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals (304 ticks) | 3460 | 3430 | −0.9% |
| 12w minerals | 3955 | 3850 | −2.7% |
| 16w minerals | 4235 | 4090 | −3.4% |
| 3w gas (1 geyser) | 552 | 552 | **0.0%** |
| 6w gas (2 geysers) | 1012 | 1080 | +6.7% |
| Gas rate/worker | 44.3 | 45.6 | +3% |
| Full test suite | — | 3367 pass | — |

**Key changes this session:**
- `GasWorkerCycleDiagnosticTest` — new diagnostic: per-phase gas cycle timing (3w and 6w configurations)
- Physics doc Section 4: harvest 31.7, approach 0 (serial access absorbed docking)
- Physics doc Section 14: gas cycle breakdown updated from measured values (116 loops/cycle)
- Blog: "Stop Guessing, Start Counting" — measurement vs analytical formulas

**Gas cycle measured breakdown (3w, 1 geyser):**

| Phase | Loops | % of cycle |
|-------|-------|------------|
| Walk to geyser | 34 | 29% |
| Wait at entry | 2 | 2% |
| Inside geyser | 32 | 28% |
| Walk to TH | 36 | 31% |
| Deposit | 12 | 10% |
| **Total** | **116** | **100%** |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #684 | Gas worker cycle timing diagnostic test | Done — 3 test methods, physics doc synced |

---

## Next Queue Items

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #682 | 6w dual-geyser gas surplus | S/Med | +6.7% error — serial access model (#682 commit on main) |
| #677 | Gas approach docking calibration test | —/— | Post-#682 context: approach=0, 3% over from walk distance |
| #679 | Per-patch walk distance extension | M/Med | Root fix for straight-line walk gap, covers gas 3% over |
| #676 | Reverse-engineer SC2 rate computation | —/— | Break ~16% MAPE floor |
| #666 | Forward simulation economy mode | L/High | EM-driven income without GT sync |

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment.

---

## Branch State

On main. All work landed and published.
