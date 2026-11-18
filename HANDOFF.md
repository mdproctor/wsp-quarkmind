# HANDOFF — quarkmind

## Orientation for Next Session

Gas physics significantly improved this session. The 3w rotation-limited gap (-25.8%) was fixed by proportional docking absorption — workers that queue at the geyser entrance absorb their approach docking during the wait. The 6w dual-geyser gap improved from -10.9% to -2.8% by removing the deposit congestion model (SC2's dual-geyser is *more* efficient than 2× single-geyser, not less). Empirical-first principle established: Liquipedia values are reference points, not calibration targets.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 2330 | 2445 | **+4.9%** |
| 12w minerals | 3460 | 3395 | **-1.9%** |
| 16w minerals | 4235 | 4165 | **-1.7%** |
| 24w minerals | 3955 | 3850 | **-2.7%** |
| 3w gas (1 geyser) | 552 | 568 | **+2.9%** |
| 6w gas (run-2) | 1204 | 1108 | **-8.0%** |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** |

**Key findings this session:**
- Gas approach docking overlaps with wait time in the rotation-limited regime. Workers position during the wait; proportional absorption subtracts actual wait from docking. 3w rate: 121→165.6 gas/min (+1.6% vs Liquipedia 163).
- Boolean docking skip had a subtle bug — processing-order artifacts caused spurious skips at real map distances (geyser occupied 84% of time). Proportional fix is correct.
- Deposit congestion model (4.0 loops/worker) was fighting reality — SC2 dual-geyser shows +3.3% bonus vs doubled single-geyser. Removed.
- MULE rate stays at empirical 277.8 (not Liquipedia's 210.9). Oracle replay calibration is authoritative.
- Empirical-first principle added to CLAUDE.md and physics doc.

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #698 | 6w dual-geyser deposit lock investigation | S/Med | Remaining -2.8% gap — deposit lock (8 loops) under investigation |
| #699 | 8w mineral +4.9% over-production gap | S/Med | Largest mineral delta — EM too fast at low saturation |
| #700 | Calibration recordings catalogue | S/Low | No formal index of 24 SC2 recordings |
| #701 | SVG XML validation for blog pipeline | XS/Low | Unescaped < broke Typora rendering |
| #702 | Regenerate standalone baseline post-#695 | XS/Low | Current baseline is pre-proportional-docking |

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #674 | artifact_sweep false-positive gaps | Fixed dest_rel mapping in soredium; plans/snapshots excluded |
| #695 | 3w gas rotation-limited cycle vs Liquipedia | Proportional docking absorption: 121→165.6 gas/min |
| #696 | Update 6w gas baseline to multi-run mean | Baseline updated to run-2 (1204); congestion removed: -10.9%→-2.8% |
| #697 | MULE yield and rate reconciliation vs Liquipedia | Documented +33% delta; empirical-first principle established |

---

## Queue Status

Queue drained: 128/128 items complete. Next work requires filing new issues via `work find` or `work start #N`.

---
