# HANDOFF — quarkmind

## Orientation

**Apples-to-apples calibration methodology established.** Pinned build order from SC2, real TorchesAIE terrain, recorded building positions. Single-base: -2.3%. Two-base: +80% — EM over-produces minerals at expansion.

**Start with:** Investigate why the EM over-produces at the expansion. The pinned two-base test has 70/70 probes (same build order) but EM accumulates +80% more minerals. The gap is in worker distribution between bases — the EM's `rebalanceMineralWorkersAcrossBases` and least-saturated `onWorkerProduced` behave differently from SC2.

**Key data:**
- Single-base pinned: SC2=3695, EM=3610, **Δ=-2.3%** (physics correct on single base)
- Two-base pinned: SC2=3805, EM=6850, **Δ=+80%** (expansion mining broken)
- Expansion Nexus at (114.5, 53.5) — 11.2u from main at (124.5, 48.5)
- Building positions recorded from SC2 in `*-buildings.json`

---

## What's ready

- **TorchesAIE pathing grid:** 160×208 cached
- **Building position recording:** `PlaybookSC2Adapter` captures every building's position + tick
- **Pinned calibration:** replays SC2 build at exact ticks, same probes, compares mineral balance
- **Snapshot fix:** building workers excluded from unit count (matches SC2 reporting)
- **Baselines:** `cal-single-base-probes.json`, `cal-two-base-economy.json` + `-buildings.json`

---

## This Session (2026-11-23/24)

| Fix/Feature | Impact |
|---|---|
| #756 — Terrain grid auto-sizing | Flat grid auto-sized from patch positions |
| #757 — Saturation-aware worker assignment | mineral rate 38.2%→13.7% (oracle — not primary) |
| #760 — Real terrain calibration | TorchesAIE recorded; pinned calibration; building positions; snapshot fix |
| #761 — Spending leaks | BuildingBornIntent and tracker morphs now deduct costs |
| Standing rules | CLAUDE.md + physics doc: real SC2, controlled replays, real maps |

---

## Notes

- **Branches stacked** (soredium#617): 756→757→758→761
- **DivergenceRegressionTest passes** with all changes
- **Single-base economy is accurate (-2.3%)** — the physics are correct for 1 base
- **Two-base economy gap (+80%)** — the expansion worker distribution is the problem, not mining physics
