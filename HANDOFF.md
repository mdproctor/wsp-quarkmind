# HANDOFF — quarkmind

## Orientation

**Two-base worker distribution fixed.** Root cause was `onWorkerProduced()` routing all workers to `leastSaturatedBaseIndex()`, producing [32,32] distribution vs SC2's all-at-main pattern. Fix: workers mine at training Nexus, auto-transfer disabled in pinned calibration. Two-base: +80% → -21.3%. Remaining gap is high-saturation contention model.

**Start with:** The -21.3% gap at 64 workers on 8 patches (8 workers/patch). The EM's contention model underestimates income at high saturation. Need SC2 per-base data to verify the actual distribution (recording in progress), then investigate the contention model.

**Key data:**
- Single-base pinned: SC2=3695, EM=3510, **Δ=-5.0%** at 36w
- Two-base pinned: SC2=3805, EM=2995, **Δ=-21.3%** at [64, 0]
- Previous two-base: +80% with [32, 32] — fixed by routing workers to training Nexus
- 8w: +0.4%, 12w: -0.3%, 16w: -2.8% — contention model accuracy degrades above 2 workers/patch

---

## What's ready

- **Worker distribution fix:** `injectWorkerForPinnedCalibration()` uses `onWorkerProducedAtBase()`, `autoTransferEnabled` flag on WorkerSimulation
- **Per-base tracking:** `CalibrationSnapshot.workersPerBase` field, `PlaybookSC2Adapter.extractWorkersPerBase()` captures SC2 `assignedHarvesters`
- **Per-base injection:** `injectWorkerAtBase(Point2d)` spawns at specific Nexus, `pinnedCalibrate()` uses per-base data when available
- **Per-base output:** pinned calibration prints `SC2 w/base` and `EM w/base` columns
- **SC2 baseline recording:** in progress — re-recording two-base economy with per-base worker data
- **TorchesAIE pathing grid:** 160×208 cached
- **Baselines:** `cal-single-base-probes.json`, `cal-two-base-economy.json` + `-buildings.json`

---

## This Session (2026-11-24)

| Fix/Feature | Impact |
|---|---|
| #762 — Worker distribution fix | Two-base +80% → -21.3%; workers mine at training Nexus |
| Per-base tracking | CalibrationSnapshot.workersPerBase, SC2 assignedHarvesters capture |
| Auto-transfer control | `setAutoTransferEnabled(false)` for pinned calibration |
| SC2 baseline re-recording | In progress — will include per-base worker counts |

---

## Notes

- **50/50 DualRunCalibrationTest passes** — no regressions
- **CalibrationBaselineTest passes** — 7/7 including new workersPerBase round-trip tests
- **SC2 playbook trains all probes at first Nexus** (`translateTrain` uses `findFirst()`) — no auto-transfer to expansion in debug API
- **High-saturation contention** is the remaining bottleneck — 8w/patch produces -21% less income in EM vs SC2
