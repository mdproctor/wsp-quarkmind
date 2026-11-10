# HANDOFF — quarkmind

## Last Session

12w mineral accuracy reached **0.0%** (exact match). Stagger recalibrated 6.0→6.1 loops. Gas entrance queuing investigated and ruled out — the 3w/6w gap is structural. Approach-docking sweep revealed delivery count quantization.

### What changed

1. **#578 (stagger=6.1)** — Fine sweep found stagger=6.1 produces exactly 682 deliveries matching SC2's 682 at tick 304. The approach-docking sweep showed no single value can hit 3460 — totals jump from 3465 (approach≤7.0) to 3415 (approach≥7.1), a 10-delivery quantum. Stagger shift breaks the phase relationship differently. Chaotic sensitivity: ±0.1 loops → ±30 mineral swings.

2. **#639 (gas queuing investigated)** — Added configurable entry lock to GasSlot (ENTRY_LOCK_DURATION). 2D sweep (lock 0-20 × harvest 14-28) showed entry lock has NO effect — workers never collide at the geyser. The 3w/6w gap is from per-geyser-distance rate interaction: 3w optimal at harvest=16, 6w at harvest=26. No single value satisfies both. harvest=26 remains best compromise.

3. **PlaybookRunner stop_at** — Added `stop_at: N supply` parameter to halt continuous training at a supply cap. Infrastructure for future playbooks.

4. **Standalone baseline refreshed** — mineralsCollectionRate 7-min MAPE: 30.1%→26.5% (3.6pp improvement from stagger change).

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
| vespeneCollectionRate | 35.0% |
| mineralsCurrent | 74.1% |
| Per-tick mismatch (12w) | 248/304 (81.6%) |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active, 51 commits | #578...#640 |

---

## Queue

```
[ ] #634 — Worker cycle variance gap (S/Low)
[ ] #636 — Variable approach docking (S/Med)
[ ] #631 — Edge-to-edge targets not wired (S/Low)
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High)
[ ] #616 — SC2 morph via ability API (needs SC2)
```

### Key findings

1. **Delivery count is quantized** — uniform phase constants produce discrete totals (683 or 673, nothing between). Stagger shifts the phase relationship but doesn't change steady-state cadence.
2. **Gas entrance queuing is ineffective** — workers never collide at geyser entrance due to walk cycle stagger. The 3w/6w gap is per-geyser-distance rate interaction, not contention.
3. **SC2 cycle variance is extreme** — avg 4.4 ticks (min 2, max 17, σ≈3+). EM is 5.2 ticks (min 4, max 9, σ≈0.5). Only 6% of ticks match. The EM needs per-trip variance to reduce per-tick mismatch.
4. **16w/24w saturation gaps are physics, not worker count** — both SC2 and EM train 23 Probes; the gap is from faster EM cycles.
