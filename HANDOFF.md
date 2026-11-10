# HANDOFF — quarkmind

## Last Session

12w mineral accuracy at **0.0%** (exact match). Per-patch approach docking implemented (#636) — per-tick cadence mismatch reduced from 81.6% to 75.3%. Saturation accuracy significantly improved. Gas entrance queuing investigated and ruled out (#639).

### What changed

1. **#578 (stagger recalibration)** — Fine sweep found that stagger=6.1→7.5 (with per-patch approach) produces exactly 682 deliveries matching SC2. Approach-docking sweep revealed delivery count quantization: uniform constants produce discrete totals (683 or 673, no intermediate values). Chaotic sensitivity: ±0.1 stagger loops → ±30 mineral swings.

2. **#636 (per-patch approach docking)** — Approach time now varies per mineral patch based on distance from town hall. Close patches get MORE approach (modeling mineral-line contention), far patches get LESS. Formula: `approach = BASE + SCALE * (avgDist - patchDist)`, SCALE=2.5. This desynchronizes worker cycles: different patches have genuinely different total cycle times, breaking delivery quantization.

3. **#639 (gas queuing investigated)** — Added configurable entry lock to GasSlot (ENTRY_LOCK_DURATION). 2D sweep showed entry lock has NO effect — workers never collide. The 3w/6w gap is from per-geyser-distance rate interaction, not entrance contention.

4. **PlaybookRunner stop_at** — Added `stop_at: N supply` parameter to halt continuous training at a supply cap.

### Accuracy

| Baseline | Mineral Δ | Gas Δ | Previous |
|----------|-----------|-------|----------|
| 12w Protoss | **0.0%** | — | +0.1% |
| 12w Terran | **0.0%** | — | +0.1% |
| 16w saturation | **+2.0%** | — | +4.4% |
| 24w saturation | **+3.0%** | — | +4.0% |
| 3w gas | **-0.1%** | -8.0% | +1.4% |
| 6w gas | **+0.9%** | -1.2% | +2.4% |

| Oracle MAPE (7-min) | Value | Previous |
|---------------------|-------|----------|
| mineralsCollectionRate | 27.4% | 30.1% |
| vespeneCollectionRate | 34.9% | 35.2% |
| mineralsCurrent | 72.7% | 77.4% |
| Per-tick mismatch (12w) | **75.3%** | 80.9% |

---

## Branches

| Branch | Status | Issues |
|--------|--------|--------|
| `issue-578-mineral-rate-acceleration-regression` | Active, 57 commits | #578...#640 |

---

## Queue

```
[ ] #634 — Worker cycle variance gap (S/Low) — SC2 cycles 2-17 ticks, EM 4-9; per-trip randomness needed
[ ] #631 — Edge-to-edge targets not wired (S/Low)
[ ] #633 — Per-race mineral income gap
[ ] #623 — Fix EM delivery timing (M/High) — partially addressed by #636
[ ] #616 — SC2 morph via ability API (needs SC2)
```

### Key findings

1. **Delivery count is quantized** — uniform phase constants produce discrete totals. Per-patch approach docking breaks this quantization by giving workers genuinely different cycle times.
2. **Gas entrance queuing is ineffective** — workers never collide at geyser entrance due to walk cycle stagger.
3. **SC2 cycle variance is extreme** — avg 4.4 ticks (min 2, max 17, σ≈3+). EM is 5.2 ticks (min 4, max 9, σ≈0.5). Per-trip variance modeling needed for further reduction.
4. **Per-patch approach docking is physically motivated** — close patches have more mineral-line contention → more approach overhead. Far patches have less.
5. **16w/24w saturation gaps are physics, not worker count** — both SC2 and EM train 23 Probes. Per-patch approach improved 16w from +4.4% to +2.0%.
