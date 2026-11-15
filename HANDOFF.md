# HANDOFF — quarkmind

## Orientation for Next Session

Gas rate structural floor addressed (#665). Edge-to-edge gas walks, SC2 API measurement, per-loop cycle decomposition. Gas MAPE 34.4% → 29.9%. Remaining gap is data pipeline: TrackerIntentExtractor misses 14% of expansion gas buildings (#671). The physics model is calibrated (dual-run ±7%).

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current state — post-#665:**

| Baseline | SC2 | EM | Gap |
|----------|-----|-----|-----|
| 12w Protoss | 3460 | 3540 | **+2.3%** |
| 16w saturation | 4235 | 4185 | **−1.2%** |
| 24w saturation | 3955 | 4025 | **+1.8%** |
| 3w gas minerals | 3885 | 3925 | +1.0% |
| 3w gas vespene | 552 | 552 | **0% exact** |
| 6w gas minerals | 3375 | 3495 | +3.6% |
| 6w gas vespene | 1012 | 1080 | +6.7% |

**Oracle MAPE improvements (5-min, 118 replays):**

| Metric | Before | After |
|--------|--------|-------|
| Gas rate MAPE | 34.4% | **29.9%** |
| Mineral rate MAPE | 16.9% | **15.8%** |
| Per-worker gas ratio | 0.71 | **0.77** |

**Design rules (from CLAUDE.md):**
- Never use compensation constants. Mining efficiency factor = 1.0.
- Constants that represent real animation wait times are legitimate.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.

---

## Last Session

**Branch:** `issue-665-gas-rate-distance-dependent` (7 commits, not yet merged)

**What was done:**
- Measured gas building radius (1.69) and town hall radius (2.75) from SC2 API via `GasCycleProfilerTest`
- Added edge-to-edge gas walks: `GAS_BUILDING_RADIUS=1.75`, `gasTarget()`/`gasReturnTarget()` helpers
- Corrected `TOWN_HALL_RADIUS` from 2.5 to 2.75 per SC2 API
- Per-loop gas cycle decomposition: harvest inside geyser = exactly 32 loops (range=32-32)
- Recalibrated `GAS_HARVEST_DURATION_LOOPS` from 26 to 66 (absorbs arrival-threshold walk compensation)
- Gas worker count fidelity diagnostic: EM misses 14% of gas buildings at 5-min (27% at 7-min)
- Blog entry: "Where the Gas Workers Walk" (`docs/blog/2026-11-15-mdp01-where-the-gas-workers-walk.md`)

**Key finding:** The `GAS_HARVEST_DURATION_LOOPS=66` is a composite: 32 loops real harvest + 34 loops compensating for the 0.5u `ARRIVAL_THRESHOLD` shortening gas walks. Fixing this properly requires either a gas-specific arrival model or removing edge subtraction for gas walks (#672).

**Issues filed this session:**
- #668 Gas approach docking (closed — no separate phase, absorbed into walk)
- #669 Gas worker count fidelity (closed — 14% building miss found)
- #670 Town hall radius correction (closed — fixed 2.5→2.75)
- #671 TrackerIntentExtractor misses 14% gas buildings (open — **next priority**)
- #672 Gas walk arrival-threshold compensation (open — physics fidelity)
- #673 Dual-run 6w gas surplus (open — two-geyser asymmetry)

---

## Queue State

**Epic:** #607 (Standalone economy accuracy)
**Active:** #671 (TrackerIntentExtractor gas building coverage)

Open queue items:
1. **#671** — TrackerIntentExtractor misses 14% of gas buildings (M/Med) ← next
2. **#672** — Gas walk arrival-threshold compensation (S/Med)
3. **#673** — Dual-run 6w gas surplus (XS/Low)
4. #656 — Mineral-line pathfinding (open)
5. #661 — EM base cycle speed edge-to-edge walks (open)
6. #666 — Forward simulation economy mode (open)

---

## Resumption Notes

- Branch `issue-665-gas-rate-distance-dependent` has 7 commits ready for `work end`
- All 209 calibration tests pass on the branch
- The branch includes a diagnostic enhancement from a fork agent (`f7226e72`) — verify it compiles cleanly before merging
- Start next session with `work end` to land #665, then `work next` to pick up #671
- For #671: trace which `BuildingType`/`abilLink` events `TrackerIntentExtractor` uses for gas buildings, cross-reference with tracker events at the worst 5-min delta replays
