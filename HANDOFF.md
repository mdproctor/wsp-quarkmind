# HANDOFF — quarkmind

## Last Session

12w mineral accuracy at **0.0%** (exact match). Discovered delivery quantization in deterministic simulations. Per-patch approach formula implemented then reverted — it was a compensation constant, not physics. Collision experiments showed the straight-line movement model is the cadence ceiling. Gas entrance queuing investigated and ruled out. Physics doc swept (15 fixes). Blog written.

### What changed

1. **#578 (stagger=6.1)** — Fine sweep found stagger=6.1 produces exactly 682 deliveries matching SC2 at tick 304. Approach-docking sweep revealed delivery count quantization: uniform constants produce discrete totals (683 or 673, nothing between).

2. **#636 (per-patch approach — implemented then reverted)** — Formula `approach = base + scale * (avgDist - patchDist)` improved cadence 81.6%→75.3% but was a compensation constant, not simulation. Reverted to SCALE=0. Per-patch infrastructure retained for physics-driven values.

3. **Collision experiments** — Same-lane collision factor (symmetric 0.75 and asymmetric yielding) both either broke mineral total or worsened cadence. The straight-line movement model is the ceiling — workers cross paths for 1-2 loops, not the sustained blocking that creates SC2's 2-17 tick cadence range.

4. **#639 (gas queuing investigated)** — Entry lock has no effect. The 3w/6w gap is structural.

5. **Physics doc sweep** — 15 staleness/gap/consistency fixes.

6. **Blog** — "The Quantum That Blocked Zero" — needs revision to reflect formula revert.

### Accuracy

| Baseline | Mineral Δ | Gas Δ | Previous |
|----------|-----------|-------|----------|
| 12w Protoss | **0.0%** | — | +0.1% |
| 12w Terran | **0.0%** | — | +0.1% |
| 16w saturation | +3.4% | — | +4.4% |
| 24w saturation | +3.8% | — | +4.0% |
| 3w gas | +1.3% | -8.0% | +1.4% |
| 6w gas | +1.9% | -1.2% | +2.4% |

| Oracle MAPE (7-min) | Value | Previous |
|---------------------|-------|----------|
| mineralsCollectionRate | 26.5% | 30.1% |
| vespeneCollectionRate | 35.0% | 35.2% |
| mineralsCurrent | 74.1% | 77.4% |
| Per-tick mismatch (12w) | 81.6% | 80.9% |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active, ~63 commits | #578...#640 |

---

## Queue

```
[ ] #631 — Edge-to-edge walk targets (S/Low) — workers should walk to patch/town hall edges, not centers; creates natural distance-dependent cycles
[ ] #634 — Worker cycle variance gap (S/Low) — SC2 cycles 2-17 ticks, EM 4-9; blocked by straight-line movement model
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High) — blocked by movement model ceiling
[ ] #616 — SC2 morph via ability API (needs SC2)
```

### Key findings

1. **Delivery count is quantized** — uniform phase constants produce discrete totals. The SC2 target sits between two plateaus that no single constant value can reach.
2. **The formula approach was a dead end** — per-patch approach docking improved numbers but was a compensation constant. The approach time should emerge from simulation physics.
3. **Straight-line movement is the cadence ceiling** — workers cross paths for 1-2 loops. SC2 workers block each other through shared mineral-line corridors for sustained periods. Collision strength changes can't overcome this.
4. **Edge-to-edge walk targets (#631) is the next physics improvement** — workers walking to patch/town hall edges creates naturally distance-dependent cycles and convergence-based collisions.
5. **Gas entrance queuing is ineffective** — workers never collide at geyser entrance.
6. **Docking overhead can't fix saturation without breaking gas** — the constants are coupled.
