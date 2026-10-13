# Replay Data State — Audit 2026-10-14

Comprehensive state of all replay training data for the ONNX strategy classifier.

## Reconstituted Datasets

| Dataset | Source | Replays | Tracker Events | Patch | Labels | Label Format | Status |
|---------|--------|--------:|---------------|-------|--------|-------------|--------|
| blizzard_ladder_4.9.3 | Blizzard API | 148,638 | ~90% good (10+ events) | 4.9.3 (75025) | 100% | `perspectives` (2-player nested) | **Gate: pipeline format fix needed** |
| blizzard_ladder_4.10.1 | Blizzard API | 2,837 | 100% good | 4.10.1 (75800) | 100% | `perspectives` (2-player nested) | **Excluded** — poor building extraction per CLAUDE.md |
| asus_rog_2020 | Tournament pack | 107 | 100% (full tracker) | 82457 | 100%, 0% UNKNOWN | flat | Ready for training |
| dreamhack_dallas_2025 | Tournament pack | 64 | 100% (full tracker) | 93333 | 100%, 0% UNKNOWN | flat | Ready for training |
| ewc_2025 | Tournament pack | 113 | 100% (full tracker) | 94137 | 100%, 0% UNKNOWN | flat | Ready for training |
| hsc_2025 | Tournament pack | 50 | 100% (full tracker) | 94137 | 100%, 0% UNKNOWN | flat | Ready for training |
| iem_pyeongchang_2018 | Tournament pack | 62 | 100% (full tracker) | 60321 | 100%, 0% UNKNOWN | flat | Ready for training |
| fel_cracow_2025 | Tournament pack | 7 | 100% (full tracker) | 94137 | 100%, 0% UNKNOWN | flat | Ready for training (tiny) |
| hsc_xxviii_2026 | Tournament pack | 0 | — | — | — | — | Not reconstituted — 88 raw .SC2Replay exist |
| hsc_xxix_2026 | Tournament pack | 0 | — | — | — | — | Not reconstituted — 54 raw .SC2Replay exist |

**Totals:** 151,878 reconstituted replays. 403 tournament replays ready. 148,638 Blizzard ladder replays blocked on format fix.

## Label Distribution (Blizzard 4.9.3 — sampled ~1000 perspectives)

All 16 fine-grained labels represented with real diversity:

| Label | Count | Notes |
|-------|------:|-------|
| BIO_TIMING | 204 | Good volume |
| MACRO_ECONOMY | 165 | Good volume |
| RUSH | 151 | Good volume |
| PROXY | 121 | Good volume |
| ROACH_RUSH | 112 | Good volume |
| CANNON_RUSH | 60 | Moderate |
| LING_BANE | 56 | Moderate |
| BANSHEE_HARASS | 41 | Moderate |
| MECH_PUSH | 38 | Moderate |
| AIR_SUPERIORITY | 18 | Low but present |
| BLINK_STALKER | 12 | Low |
| HYDRA_PUSH | 8 | Low |
| DT_RUSH | 5 | Very low |
| MUTA_HARASS | 5 | Very low |
| COLOSSUS_PUSH | 3 | Very low |
| TECH_RUSH | 3 | Very low |

At 148K replays, even "very low" categories should have hundreds of examples in the full dataset.

## The Format Gap

`reconstitution_pipeline.py` reads `.label.json` files via `HybridLabeller.label_from_sidecar()`, which expects flat format:
```json
{"coarseLabel": "AGGRESSIVE", "fineGrainedLabel": "RUSH", "confidence": 1.0}
```

Blizzard ladder labels (written by `relabel_reconstituted.py`) use a `perspectives` format:
```json
{"perspectives": [{"playerId": 1, "race": "Terran", "opponentRace": "Zerg", "fineGrainedLabel": "BIO_TIMING", "coarseLabel": "GROUND"}]}
```

**Fix required:** Update `reconstitution_pipeline.py` (or `HybridLabeller`) to read the `perspectives` format. Each replay produces two training perspectives (one per player).

## Not-Yet-Reconstituted Data

| Source | Raw Replays | Location | Worth reconstituting? |
|--------|------------|----------|----------------------|
| HSC XXVII (2025) | 61 | replays/ | Yes — tournament diversity, small effort |
| HSC XXVIII (2026) | 88 | replays/ | Yes — tournament diversity, small effort |
| HSC XXIX (2026) | 54 | replays/ | Yes — latest patch, small effort |
| sc2egset (Zenodo) | 75 dirs | data/sc2egset/ | Maybe — pre-processed JSON, different ingest |
| spawningtool | varies | data/spawningtool/ | Low priority — older data, separate pipeline |

## How We Got Here

1. **Blizzard API download** — 148K+ ladder replays downloaded via Blizzard dev API key. These are real ladder games at all skill levels.
2. **Tournament packs** — downloaded from Spawning Tool replay packs. Pro tournament games.
3. **Reconstitution** — `ReconstitutionExportTest` (Java) extracts tracker events from replays, writes JSON to `data/reconstituted/`.
4. **Labelling** — `relabel_reconstituted.py` (Python, #410) applies building-timing rules from `rules.py`. Achieves 100% label coverage.
5. **Training pipeline** — `reconstitution_pipeline.py` reads reconstituted JSON + labels, builds feature vectors, splits into train/val/test .npz files.

## EmulatedGame Accuracy

- **99.8% unit accuracy** at 5-minute checkpoint against 118 oracle replays (tracker-event-driven)
- Landed as commit `4c8009fa` on main
- Remaining 0.2% is edge cases, not systematic error
- Accuracy is measured against replays WITH tracker events — the same category as the Blizzard ladder data

## What to Run With

**Immediate (unblocks #388):** Fix the `perspectives` label format in the pipeline. Then run on all 148K + 403 tournament replays. This is the "one more pass" — fix the format gap, train, evaluate honestly.

**Low-effort bonus:** Reconstitute HSC XXVIII + XXIX (142 replays). Small increment but latest-patch tournament diversity.

**Not worth pursuing now:** sc2egset and spawningtool data — different ingest paths, older data, marginal value given 148K Blizzard replays.
