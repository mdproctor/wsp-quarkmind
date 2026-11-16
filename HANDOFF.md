# HANDOFF — quarkmind

## Orientation for Next Session

Gas harvest cycle decomposed (#675). `GAS_HARVEST_DURATION_LOOPS` was doing two jobs: real harvest time (32 loops, measured by SC2 profiler) + walk-shortening compensation (~34 loops). Separating them into `GAS_HARVEST_DURATION_LOOPS = 32` + `GAS_APPROACH_DOCKING_LOOPS = 11.5` reduced gas rate MAPE from 29.9% to 15.3% at 5-min.

The rate algorithm investigation (#676) found SC2 rates are quantised at 28 minerals/min (`rate ≡ 27 mod 28`), implying a 240-loop sliding window — not an EMA. Implementing it raised MAPE (EM delivery timing too loose), so deferred. The rate algorithm floor (~15%) is shared between minerals (16.6%) and gas (15.3%) and blocks further MAPE improvement without first tightening delivery timing.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#675:**

| Metric | Before #675 | After #675 |
|--------|-------------|------------|
| Gas rate MAPE (5-min) aggregate | 29.9% | **15.3%** |
| Gas rate MAPE Protoss | 27.2% | **15.2%** |
| Gas rate MAPE Terran | 24.0% | **13.4%** |
| Gas rate MAPE Zerg | 31.8% | **25.4%** |
| Mineral rate MAPE (5-min) | 16.5% | 16.5% (unchanged) |
| GAS_HARVEST_DURATION_LOOPS | 66 | **32** |
| GAS_APPROACH_DOCKING_LOOPS | (not modelled) | **11.5** |
| MIN_MINERAL_WORKERS_BEFORE_GAS | 8 | **4** |
| Full test suite | 3362 pass | 3362 pass |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. `GAS_HARVEST_DURATION_LOOPS = 32` is the measured real harvest. `GAS_APPROACH_DOCKING_LOOPS = 11.5` is calibrated entry overhead — legitimate, not compensation.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Next Queue Items

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #661 | EM base cycle speed edge-to-edge walks | S/Med | Physics prerequisite for #676 sliding window |
| #656 | Mineral-line pathfinding | M/Med | Tighter walk timing |
| #677 | Gas approach docking calibration test | XS/Low | Validate 11.5 against dual-run, not just oracle MAPE |
| #676 | Reverse-engineer SC2 rate computation | M/High | Needs #661, #656 first — 240-loop window requires tighter physics |
| #672 | Gas walk arrival-threshold compensation | S/Med | Remaining walk shortening (~9 loops/leg vs SC2) |
| #673 | Dual-run 6w gas surplus two-geyser asymmetry | S/Low | +6.7% surplus at 6w, two geysers |

**Priority path:** #661 → #656 → #677 → #676 (rate window)

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. Derivation: `MINERAL_CARGO(5) × LOOPS_PER_MINUTE(1344) / 240 = 28`. SC2 counts discrete deliveries in a 240-loop sliding window, integer-truncated. Not an EMA.

**Why we didn't switch:** Implementing the window raised gas MAPE from 15.3% to 18.8%. The EMA's smoothing was compensating for per-delivery timing errors. The window requires the EM's delivery cadence to match SC2's first. That's #661, #656, #672 work.

---

## Branch State

Branch `issue-675-gas-rate-mape` — **ready for work-end**.

5 commits:
1. `fix(#675): decompose gas harvest into measured harvest + approach docking`
2. `fix(#675): rebalance gas workers after mineral worker sync`
3. `docs(#676): document SC2 rate quantisation at 28 minerals/min`
4. `fix(#675): code review fixes and docs — CDI default, setter methods, physics doc`
5. Blog and queue updates

Issues to close: **#675** (done). #676 remains open (investigation documented, implementation deferred).

---

## Last Session

**Work completed:**
- #675: Gas harvest cycle decomposition — 29.9% → 15.3% MAPE
- #676: Rate algorithm investigation — 240-loop window discovered, deferred pending physics alignment
- #677: Filed (gas approach docking calibration test)
- Code review: CDI default fix (11.0→11.5), added setter/resetter methods
- Physics doc: updated constants table, added MIN_MINERAL_WORKERS_BEFORE_GAS note, rate algorithm quantisation finding
- Blog: "Sixty-Six Was Doing Two Jobs" (2026-11-16-mdp01)
