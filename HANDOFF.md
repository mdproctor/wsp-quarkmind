# HANDOFF — quarkmind

## Orientation for Next Session

Approach docking decomposed (#679). The 24.5-loop approach docking constant previously absorbed two distinct phenomena: approach animation (fixed docking overhead) and outbound path deviation (workers navigate around the mineral line). These are now separate constants: `APPROACH_ANIMATION_LOOPS` (20.0, constant per base) and per-patch `outboundPathDelay` (0–5 loops, from E2E distance). Total per-patch `approachDockingLoops` is preserved exactly — pure refactoring.

Also fixed two CDI default drift violations (PP-20261116-5f16ed): gas-approach-docking CDI was 15.8 but constant was 12.5; gas-harvest-duration CDI was 32.0 but constant was 31.7.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 2330 | 2445 | **+4.9%** |
| 12w minerals | 3460 | 3395 | **-1.9%** |
| 16w minerals | 4235 | 4165 | **-1.7%** |
| 3w gas (1 geyser) | 552 | 536 | **-2.9%** |
| 6w gas (2 geysers) | 1012 | 1032 | **+2.0%** |
| Full test suite | — | 3368 pass | — |

**Key changes this session:**
- `APPROACH_ANIMATION_LOOPS` = 20.0 (new constant — animation-only)
- `PATH_DEVIATION_BASE` = 4.5, `PATH_DEVIATION_SCALE` = 0.4 (new compile-time constants)
- `outboundPathDelay` per-patch field on `MineralPatch`
- Calibration sweep test `sweepDecomposedApproachConstants` in `ApproachDockingSweepTest`
- Correctness fix: `max(0)` applied to total (animation + rawPathDelay) not to pathDelay alone
- CDI defaults synced: gas-approach-docking 15.8→12.5, gas-harvest-duration 32.0→31.7
- Physics doc drift fixed: §1b EM σ 0.90→0.57, §19 stale values removed
- Blog: "Two Walks, One Constant" — walk asymmetry and approach docking decomposition
- Filed #690 (effective radii), #691 (dead APPROACH_EQUALIZATION_SCALE cleanup), #692 (angular position predictor)

**Walk asymmetry evidence (8w cadence, CycleDecompositionTest):**

| Measure | Outbound | Return |
|---------|----------|--------|
| Walk distance range | 2.69–4.29u | 1.47–2.54u |
| Walk/return ratio | 1.4–1.9x | — |
| Effective radius (SC2) | 1.215u | 3.755u |
| Effective radius (EM) | 1.0u | 2.75u |

The return walk being shorter is primarily an effective radius effect (workers stop further from buildings), deferred to #690.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #679 | Per-patch walk distance extension | Done — decomposed approach docking into animation + path delay |

## Filed This Session

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #690 | Effective mining/deposit radius correction | M/Med | Workers stop at 1.215u (not 1.0u) and 3.755u (not 2.75u) |
| #691 | Clean up dead APPROACH_EQUALIZATION_SCALE | XS/Low | Superseded by PATH_DEVIATION_SCALE but still wired |
| #692 | Improve outbound path delay predictor | M/Med | Add angular position to E2E distance model (R²=0.104 is weak) |

---

## Next Queue Items

| Issue | Title | Scale | Priority |
|-------|-------|-------|----------|
| #666 | Forward simulation economy mode | L/High | EM-driven income without GT sync |
| #676 | Reverse-engineer SC2 rate computation | —/— | Break ~16% rate MAPE floor |

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment.

---

## Branch State

On main. All work landed and published.
