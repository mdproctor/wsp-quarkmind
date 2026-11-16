# HANDOFF — quarkmind

## Orientation for Next Session

Base cycle speed fixed (#661). The EM's `stepOneMineralLoop` had zero-cost phase transitions — the `while(!loopConsumed)` loop chained mining→returning and depositing→mining within a single game loop. SC2 processes each transition at a game-loop boundary, consuming 1 loop. This made EM cycles ~10 loops faster at all saturation levels. At 12w/16w, the collision speed factor (0.96) accidentally compensated. At 8w with no collisions, the fast cycle was fully exposed (+17.2%).

Fix: transition loop cost + recalibrated constants (approach 11.5→24.5, deposit 7→12, collision 0.96→1.0). Applied to both mineral and gas workers.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#661:**

| Metric | Before #661 | After #661 |
|--------|-------------|------------|
| 8w minerals | +17.2% | **+4.3%** |
| 12w minerals | +2.7% | **-2.7%** |
| 16w minerals | -1.2% | **-1.7%** |
| 3w gas vespene | +21.0% | **+13.0%** |
| 6w gas vespene | +27.3% | **+20.9%** |
| APPROACH_DOCKING_LOOPS | 11.5 | **24.5** |
| DEPOSIT_DURATION_LOOPS | 7 | **12** |
| COLLISION_SPEED_FACTOR | 0.96 | **1.0** |
| Full test suite | 3365 pass | 3365+ pass |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. The transition loop cost models real SC2 per-loop transition processing.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Next Queue Items

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #656 | Mineral-line pathfinding | M/Med | Tighter walk timing — per-patch path deviation |
| #677 | Gas approach docking calibration test | XS/Low | GAS_APPROACH_DOCKING still at 11.5 — needs 8w-style cadence measurement |
| #676 | Reverse-engineer SC2 rate computation | M/High | 240-loop sliding window; needs tighter physics first |
| #672 | Gas walk arrival-threshold compensation | S/Med | Remaining walk shortening (~9 loops/leg vs SC2) |
| #673 | Dual-run 6w gas surplus two-geyser asymmetry | S/Low | +6.7% surplus at 6w, two geysers |

**Priority path:** #656 → #677 → #676 (rate window)

---

## Rate Algorithm Finding

**Key discovery:** SC2 mineral rates quantise at 28 minerals/min. All reported `mineralsCollectionRate` values satisfy `rate ≡ 27 (mod 28)` after ramp-up. The window approach becomes viable only after tighter physics alignment — now closer after #661.

---

## Branch State

Branch `issue-661-em-base-cycle-speed` — **ready for work-end**.

Key commits:
1. `feat(#661): add 'kill' playbook action + 8w mineral baseline playbook`
2. `feat(#661): record 8w SC2 baseline with worker cadence data`
3. `feat(#661): add CycleDecompositionTest for 8w walk analysis`
4. `wip(#661): diagnostic sweeps for approach/deposit/collision constants`
5. `wip(#661): fix base cycle speed — transition loop cost + recalibrated constants`
6. `fix(#661): add transition loop cost to gas worker phases`
7. `fix(#661): code review fixes`
8. `docs(#661): update physics constants with new calibration`

Issues to close: **#661** (done).

---

## Last Session

**Work completed:**
- #661: Base cycle speed — 8w gap reduced from +17.2% to +4.3%
- Root cause: zero-cost phase transitions in stepOneMineralLoop
- Recorded 8w SC2 baseline with per-tick worker cadence data
- CycleDecompositionTest: measured mining position, deposit distance, walk phases
- 3D parameter sweep to find optimal constants
- Transition loop cost fix applied to mineral and gas workers
- CDI defaults synced with constants (fixed pre-existing protocol violations)
- Gas vespene improved from +21%/+27% to +13%/+21%
- Physics doc updated with new measurements and constants
