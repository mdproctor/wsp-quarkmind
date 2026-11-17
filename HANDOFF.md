# HANDOFF — quarkmind

## Orientation for Next Session

Gas walk ARRIVAL_THRESHOLD compensation landed (#672). The EM's gas income was 13-21% too high because gas workers stopped 0.5u short of the actual building edge — the ARRIVAL_THRESHOLD designed for minerals silently shortened gas walks. Fix: offset gas walk targets inward by ARRIVAL_THRESHOLD so the stopping distance lands workers at the true building edge. Combined with approach docking recalibration (11.5 → 15.8 loops), 3w gas is now 0% error.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state:**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals (304 ticks) | 3460 | 3430 | −0.9% |
| 12w minerals | 3955 | 3850 | −2.7% |
| 16w minerals | 4235 | 4090 | −3.4% |
| 3w gas (1 geyser) | 552 | 552 | **0.0%** |
| 6w gas (2 geysers) | 1012 | 1080 | +6.7% |
| Zerg 12w minerals | 2330 | 2430 | +4.3% |
| Full test suite | — | 3367 pass | — |

**Key changes this session:**
- `WorkerPhysicsConfig` now includes `approachEqualizationScale` and `lowSaturationDodgeScale` (#681)
- `gasTarget()`/`gasReturnTarget()` use `GAS_BUILDING_RADIUS - ARRIVAL_THRESHOLD` and `TOWN_HALL_RADIUS - ARRIVAL_THRESHOLD` (#672)
- `GAS_APPROACH_DOCKING_LOOPS` = 15.8 (was 11.5) (#672)
- Sweep tests use `resetToDefaults()` instead of hardcoded values

**Design insight:** The ARRIVAL_THRESHOLD (0.5u) is a movement system constant that affects all workers equally — but its impact depends on walk distance. For minerals, the approach docking constant (24.5 loops) was calibrated from SC2 cadence data that already included the threshold effect. Gas approach docking was calibrated independently against oracle MAPE, where the walk shortening was masked by other errors. Fixing the walk geometry and recalibrating the docking constant separately resolved the gas income error.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #681 | Add equalization/dodge to WorkerPhysicsConfig | Done — both constants wired through CDI |
| #672 | Gas walk ARRIVAL_THRESHOLD compensation | Done — 3w gas 0% error |
| #673 | Dual-run 6w gas surplus asymmetry | Closed — geysers symmetric, not the cause |
| #680 | Eliminate 7+ tick outliers | Closed — deferred to #679, 0.14% negligible |

## Filed This Session

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #682 | 6w dual-geyser +6.7% gas surplus | S/Med | Multi-geyser surplus after walk fix — likely assignment timing |

---

## Next Queue Items

| Issue | Title | Scale | Priority |
|-------|-------|-------|----------|
| #682 | 6w dual-geyser gas surplus | S/Med | Builds on #672 gas walk work |
| #677 | Gas approach docking calibration test | — | Validate 15.8 constant against dual-run |
| #679 | Per-patch walk distance extension | M/Med | Root fix for #680 and cross-map accuracy |
| #676 | Reverse-engineer SC2 rate computation | — | Break ~16% MAPE floor |
| #666 | Forward simulation economy mode | L/High | EM-driven income without GT sync |

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment.

---

## Branch State

On main. All work landed and published.
