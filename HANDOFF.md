# HANDOFF — quarkmind

## Orientation for Next Session

You are working on EmulatedGame physics accuracy — making the economy simulation match real SC2 tick-for-tick. The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go** when you change constants, find stale values, or discover new physics. It is a living document, not a snapshot.

**Current state:** 12w mineral total is at **0.0%** (exact match with SC2 at tick 304). Per-tick cadence mismatch is 81.6% — 248 of 304 ticks have different per-tick mineral deltas between EM and SC2. The mineral total is right but the delivery timing within each tick is wrong.

**What was tried this session and why it was reverted:**
- Per-patch approach docking formula (`approach = base + scale * (avgDist - patchDist)`) improved cadence to 75.3% and saturation to +2.0%. But it was a compensation constant — the approach time should emerge from simulation physics, not a distance formula. **Reverted.** The per-patch infrastructure remains in the code (MineralPatch.approachDockingLoops, computeApproachDocking) but APPROACH_DISTANCE_SCALE=0 disables it.
- Same-lane collision experiments (stronger speed reduction when workers on the same mineral patch cross paths). Both symmetric and asymmetric variants either broke the mineral total or worsened cadence. The straight-line movement model is the ceiling — workers cross for 1-2 loops, not the sustained blocking SC2 creates through mineral-line corridors.
- Gas entrance queuing (serialised geyser entry). No effect — workers never arrive simultaneously due to walk-cycle stagger.

**What to work on next:**
- **#631 — Edge-to-edge walk targets.** Workers currently walk from town hall center to patch center. SC2 workers walk to the nearest edge of the town hall (deposit) and nearest edge of the patch (mine). This creates naturally distance-dependent cycles (close patches are closer to the town hall edge) and convergence at the deposit point (workers from the same side cluster). This is a real physics improvement that will require recalibrating all phase constants (approach, deposit, stagger) afterward.
- After edge-to-edge: revisit collision physics with the new geometry — workers may now converge at the deposit point, creating genuine sustained collisions.

**Design rules (from CLAUDE.md):**
- Never use compensation constants. If the EM doesn't match SC2, fix the root cause.
- Always use double arithmetic. No integer truncation in the income path.
- Calibrate from controlled SC2 dual-run baselines, not oracle replays.
- Per-tick delivery deltas must be exact multiples of 5 (mineral cargo).

**Key diagnostic tools:**
- `DualRunCalibrationTest#deliveryCadenceDiagnostic` — per-tick EM vs SC2 mineral comparison (run with `-Pdiagnostic`)
- `ApproachDockingSweepTest` — sweep approach, stagger, docking overhead, gas harvest, and entry lock (run with `-Pdiagnostic`)
- `DualRunCalibrationTest` — full 31-test calibration suite (default profile)

---

## Last Session

12w mineral accuracy at **0.0%** (stagger recalibrated 6.0→6.1). Discovered delivery quantization — uniform constants produce discrete totals with no intermediate values. Per-patch formula and collision experiments investigated as cadence fixes, both reverted. Gas entrance queuing ruled out. Physics doc swept. Blog written and revised.

### Accuracy

| Baseline | Mineral Δ | Gas Δ |
|----------|-----------|-------|
| 12w Protoss | **0.0%** | — |
| 12w Terran | **0.0%** | — |
| 16w saturation | +3.4% | — |
| 24w saturation | +3.8% | — |
| 3w gas | +1.3% | -8.0% |
| 6w gas | +1.9% | -1.2% |

| Oracle MAPE (7-min) | Value |
|---------------------|-------|
| mineralsCollectionRate | 26.5% |
| mineralsCurrent | 74.1% |
| Per-tick mismatch (12w) | 81.6% |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active | #578...#640 |

---

## Queue

```
[ ] #631 — Edge-to-edge walk targets (S/Low) — next physics improvement
[ ] #634 — Worker cycle variance gap (S/Low) — blocked by movement model
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High)
[ ] #616 — SC2 morph via ability API (needs SC2)
```

### Key findings

1. **Delivery count is quantized** — uniform phase constants produce discrete totals (683 or 673 deliveries, nothing between). The SC2 target (682) sits in the gap.
2. **Stagger breaks the quantum** — stagger=6.1 shifts phase relationships so exactly 682 deliveries fall in the window. But stagger is chaotically sensitive (±0.1 loops → ±30 mineral swings).
3. **Straight-line movement is the cadence ceiling** — SC2 cycles span 2-17 ticks (σ≈3+), EM spans 4-9 (σ≈0.5). Collision strength can't overcome the lack of path geometry.
4. **Gas model is structurally limited** — no single harvest duration satisfies both 3w and 6w baselines. 8% vespene floor.
5. **Formulas are dead ends** — per-patch approach formula improved numbers but was reverted as a compensation constant. The approach time must emerge from physics.
