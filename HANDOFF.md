# HANDOFF — quarkmind

## Orientation for Next Session

Gas asymmetry resolved (#689). SC2 empirical data showed single-geyser under-production (-3.6%) and dual-geyser over-production (+2.8%) at the same docking constant. Root cause: the EM's gas cycle was geyser-limited (inside time 44.2 loops created slack absorbing all deposit contention via phase-shift stabilization). SC2's empirical dual-geyser penalty (~6%) comes from TH approach path congestion when >3 gas workers share a base.

Fix: gas deposit congestion model — each deposit extends by `GAS_DEPOSIT_CONGESTION_PER_WORKER` (4.0 loops) × max(0, workers_at_base − 3). Zero effect on single-geyser. Combined with docking recalibration 12.5→11.5 (congestion now models part of the cycle overhead the docking previously absorbed).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 2330 | 2430 | **+4.3%** |
| 12w minerals | 3460 | 3430 | **-0.9%** |
| 16w minerals | 4235 | 4090 | **-3.4%** |
| 3w gas (1 geyser) | 552 | 544 | **-1.4%** |
| 6w gas (2 geysers) | 1012 | 1016 | **+0.4%** |
| Full test suite | — | 3368 pass | — |

**Key changes this session:**
- `GAS_APPROACH_DOCKING_LOOPS` = 11.5 (was 12.5 — reduced, congestion models part of cycle overhead)
- `GAS_DEPOSIT_CONGESTION_PER_WORKER` = 4.0 (new — TH approach congestion for multi-geyser bases)
- CDI default synced: gas-approach-docking 12.5→11.5
- Diagnostic tests: perGeyserIncomeTracking, depositLockContentionTrace, dockingVsRegimeSweep
- Calibration sweep: gasApproachDockingSweep, gasDockingCongestionSweep in DualRunCalibrationTest
- Physics doc updated with congestion model and phase-shift stabilization finding

**Key diagnostic findings (#689):**
- Deposit lock is structurally ineffective at ANY value (0–16 loops) — phase-shift stabilization
- Per-geyser income is perfectly symmetric (50/50 split in dual-geyser)
- 22 transient lock contention events over 6710 loops — negligible
- Geyser-limited regime: slack = 2×inside − non_inside = 6.4 loops absorbs all contention
- Gas worker collision (walk-path model) doesn't work: cross-geyser paths are 1.78u apart (> 1.5u COLLISION_DIAMETER on this map geometry)

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #693 | Gas worker walk collision for close-angle geyser maps | S/Med | Cross-geyser walk paths don't overlap on test map (1.78u > 1.5u); may matter on closer-angle maps |
| #694 | WorkerSimulation static fields lack resetToDefaults() | XS/Low | GAS_DEPOSIT_LOCK_LOOPS and GAS_DEPOSIT_CONGESTION_PER_WORKER restored manually in sweeps |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #689 | 3w/6w gas asymmetry | Done — congestion model, combined error 6.4%→1.8% |

---

## Next Queue Item

| Issue | Title | Scale | Priority |
|-------|-------|-------|----------|
| #686 | (next in queue) | — | — |

---

## Branch State

On main. #689 landed as 7db5b88d. 2 commits unpublished.
