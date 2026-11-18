# HANDOFF — quarkmind

## Orientation for Next Session

Gas physics investigations complete. The deposit lock has zero effect on 6w dual-geyser throughput — systematically ruled out via parameter sweeps (lock=0 to 16, all produce identical output). The remaining -8.0% gap against a single SC2 run shrinks to -2.8% when measured against the 5-run mean. SC2 is bimodal: 3/5 runs produce 1204 gas, 2/5 produce 1044. The EM's deterministic 1108 sits between modes.

The 8w mineral +4.3% over-production root cause is identified: the uncontested base cycle is ~5.0 ticks vs SC2's ~5.26 ticks — 0.26 ticks/cycle faster, producing 20 extra deliveries over 305 ticks. The fix path (#690, #705) involves adjusting effective mining/deposit radii and worker redistribution after kills.

The reference document is `docs/sc2-physics-constants.md` — **read it first**, use it throughout, and **revise it as you go**.

**Current accuracy state (dual-run, tick 304):**

| Metric | SC2 | EM | Error |
|--------|-----|-----|-------|
| 8w minerals | 2330 | 2430 | **+4.3%** |
| 12w minerals | 3460 | 3430 | **-0.9%** |
| 16w minerals | 4235 | 4090 | **-3.4%** |
| 24w minerals | 3955 | 3850 | **-2.7%** |
| 3w gas (1 geyser) | 552 | 568 | **+2.9%** |
| 6w gas (5-run mean) | 1140 | 1108 | **-2.8%** |

**Standalone baseline (118 oracle replays) vespeneCollectionRate MAPE:**

| Checkpoint | Previous | Current | Delta |
|---|---|---|---|
| 3-min | 35.6% | 33.0% | -2.6pp |
| 5-min | 30.1% | 25.7% | -4.4pp |
| 7-min | 29.7% | 24.9% | -4.8pp |

**Key findings this session:**
- Deposit lock sweep (0–16): zero throughput change at every value. Gas worker cycles naturally space deposits, and phase-shift stabilisation prevents recurring contention.
- Approach docking sweep: combined 3w+6w error stays 10.7–11.4% at every docking value — no single value improves both.
- SC2 bimodal behavior: randomised event processing order causes two distinct gas modes (1044 vs 1204). Multi-run mean is the correct calibration target.
- 8w uncontested cycle: EM per-worker rate 303.8 min/worker vs SC2's 291.3 — 4.3% gap. SC2 8w and 12w per-worker rates are nearly identical (291.3 vs 288.3), but EM shows 6.3% gap due to faster base cycle.
- SVG XML validation added to artifact promotion pipeline (soredium).
- `docs/calibration-catalogue.md` indexes all 24 SC2 recordings, 25 playbooks, and coverage gaps.

**Pre-existing CDI issue:** `@QuarkusTest` tests fail on main due to neocortex `EffectiveCognitiveDefaults` missing bean annotation (#703). Plain JUnit tests unaffected.

---

## Closed This Session

| Issue | Title | Result |
|-------|-------|--------|
| #702 | Regenerate standalone baseline post-#695 | vespeneCollectionRate MAPE: 29.7→24.9% (7min) |
| #701 | SVG XML validation for blog pipeline | Added to soredium artifact_sweep.py + artifact_promote.py |
| #691 | Clean dead APPROACH_EQUALIZATION_SCALE | Removed from 5 files, deleted sweep test |
| #694 | WorkerSimulation.resetToDefaults() | Added; chained from SimulatedWorker.resetToDefaults() |
| #700 | Calibration recordings catalogue | docs/calibration-catalogue.md — 24 recordings, 7 coverage gaps |
| #698 | 6w deposit lock investigation | Deposit lock has zero effect; gap is SC2 bimodal behavior |
| #699 | 8w mineral +4.3% over-production | Root cause: uncontested cycle 4% too fast; fix deferred |

**Filed this session:**

| Issue | Title | Scale | Notes |
|-------|-------|-------|-------|
| #703 | CDI deployment failure — neocortex EffectiveCognitiveDefaults | S/Med | Pre-existing, blocks @QuarkusTest |
| #704 | Fill missing EM deltas in calibration catalogue | XS/Low | 4 recordings with "—" entries |
| #705 | 8w worker redistribution after kill | S/Med | Workers don't redistribute to empty patches |
| #706 | SVG validation test coverage (soredium) | XS/Low | No automated test for new validation |

---

## Queue (4 items, #690 active)

| # | Issue | Scale | What |
|---|-------|-------|------|
| 1 | **#690** | M/Med | **Effective mining/deposit radius correction** — walk targets use observed radii |
| 2 | #704 | XS/Low | Fill missing EM deltas in calibration catalogue |
| 3 | #705 | S/Med | 8w worker redistribution after kill — patches left empty |
| 4 | #706 | XS/Low | SVG validation test coverage (soredium cross-repo) |

**On main** — `work start` to begin next issue.

**Also open but not queued:** #644 (Zerg baseline), #683 (physics config extraction), #703 (CDI failure — dependency issue). Needs triage.

---

## Cross-Repo Changes

**soredium** (commit 672c5fa on main): SVG XML validation added to `artifact_sweep.py` and `artifact_promote.py`. Synced to `~/.claude/skills/work-end/`.

---

## Unpublished

5 commits on quarkmind canonical, not yet pushed. Publish with `canonical_sync.py publish` when ready.
