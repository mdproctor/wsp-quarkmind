# Combinatorial Gap Analysis — Detector Coverage Audit

**Issue:** casehubio/quarkmind#413
**Branch:** issue-413-combinatorial-gap-analysis
**Date:** 2026-10-18

## Problem

The detector catalog (`binary_catalog.py`) defines 9 binary detectors across 3 game phases, producing 23 trained ONNX models (detector × matchup). These detectors feed into the Bayesian evidence fusion pipeline (`DetectorOrchestrator` → `SC2NetworkBuilder` → `EvidenceFusionEngine`). But there is no systematic audit of:

- Which `StrategyArchetype` values (58 total) are covered by at least one detector
- Which phase × matchup × category combinations have no detector at all
- Whether detectors that fire simultaneously have adequate co-occurrence training data
- What happens at phase boundaries when early detectors go silent and mid detectors activate

Without this, the CBR meta-learning loop (D9) will discover gaps at runtime — knowing them upfront lets us prioritise training data collection and catalog expansion.

## Scope

**In scope:**
- Static catalog audit: archetype → detector coverage, missing cells, phase transition chains
- Data-driven within-phase co-occurrence from existing `.label.json` sidecars
- Gap classification: rare-strategy vs fixable
- Markdown report with recommendations

**Out of scope:**
- Cross-phase data-driven co-occurrence (requires re-extracting features at multiple timepoints — future issue)
- Training new detectors or modifying the catalog (this issue audits; follow-up issues act)
- Modifying the Bayesian network or DetectorOrchestrator

## Design

### Module

`quarkmind-classifier/src/coverage_gaps.py` — a standalone analysis script, runnable as `python3 -m src.coverage_gaps`.

### Data sources

| Source | How accessed | What it provides |
|--------|-------------|-----------------|
| `StrategyArchetype.java` | Parsed from source via regex | 58 archetypes with race, phase, category |
| `binary_catalog.py` | Imported directly (`DETECTOR_CATALOG`) | 9 detector definitions with phase, matchup, positive archetype lists |
| `data/reconstituted/**/*.label.json` | Scanned from filesystem | Per-replay heuristic archetype labels (matchup, coarseLabel, fineGrainedLabel) |
| `output/detectors/detector_manifest.json` | Read as JSON | Training stats per detector×matchup (accuracy, tp/fp/fn/tn, train counts) |

### Analysis layers

#### Layer 1 — Static catalog audit

**Archetype coverage matrix.** For each of the 58 `StrategyArchetype` entries, determine which detector(s) include it (or its coarse label) as a positive. Output a table:

| Archetype | Race | Phase | Category | Detectors |
|-----------|------|-------|----------|-----------|
| TERRAN_MARINE_RUSH | TERRAN | EARLY | RUSH | early_rush |
| PROTOSS_DT_HARASS | PROTOSS | EARLY | HARASS | (none) |
| ... | | | | |

Archetypes with no detector coverage are highlighted as gaps.

**Missing cells.** Enumerate all phase × matchup × category combinations. For each, check if any detector covers at least one archetype in that cell. Missing cells indicate systematic blind spots (e.g., no HARASS detector for any phase).

**Weak detectors.** Cross-reference with `detector_manifest.json` to flag detectors with accuracy < 0.5 as effectively non-functional. Currently known: `mid_ground_heavy_vs_terran` (20.0%), `late_ground_army_vs_protoss` (16.1%).

#### Layer 2 — Data-driven co-occurrence

For each `.label.json` sidecar across all reconstituted datasets:
1. Read the `fineGrainedLabel` and `matchup` for each perspective
2. Determine which detectors would classify this replay as positive (by checking if the label appears in the detector's `positives` list for that matchup)
3. Accumulate into a co-occurrence matrix: detector × detector, cell = replay count where both fire positive

Within-phase co-occurrence is meaningful (detectors in the same phase can fire simultaneously). Cross-phase co-occurrence provides a weaker signal — it tells us which early strategies co-occur with which mid strategies in the same replay, even though the detectors don't fire at the same time.

**Sparse cell threshold:** <50 replays flags a combination as potentially undertrained.

#### Layer 3 — Gap classification

For each gap identified in Layers 1-2:

- **Rare-strategy gap:** The archetype occurs in <50 total replays across all datasets. No detector would help — the strategy is genuinely rare. Acceptable.
- **Fixable gap:** The archetype occurs in ≥50 replays but no detector covers it. A new detector definition or expanded positive list would close the gap. Actionable.
- **Weak-detector gap:** A detector exists but performs below 50% accuracy. The definition may be wrong (positive archetypes too broad/narrow) or the training data insufficient. Actionable.

#### Phase transition chains

Structural analysis (not data-driven) mapping natural successors:
- RUSH/CHEESE → TIMING (aggressive early → aggressive mid)
- MACRO → COMPOSITION (greedy early → standard mid)
- HARASS → variable (depends on harass success — could transition to any mid strategy)
- TECH → COMPOSITION/TECH (tech investment carries forward)

This is qualitative game knowledge, not computed. The report documents which early→mid transitions the detector catalog can track vs which are blind.

### Output

Markdown report at `docs/benchmarks/detector-coverage-gaps.md` with sections:

1. **Archetype coverage matrix** — table of all 58 archetypes and their detector coverage
2. **Missing cells** — phase × matchup × category with no coverage
3. **Weak detectors** — detectors with accuracy < 50%
4. **Within-phase co-occurrence** — matrix per phase, sparse cells flagged
5. **Cross-phase co-occurrence** — early×mid matrix from replay labels
6. **Phase transition chains** — structural analysis
7. **Gap assessment** — classified as rare / fixable / weak, with recommendations

### Testing

`quarkmind-classifier/tests/test_coverage_gaps.py`:
- `test_parse_archetypes` — validates parsing extracts all 58 archetypes with correct fields
- `test_static_audit_completeness` — all archetypes appear in the coverage matrix
- `test_co_occurrence_matrix_structure` — matrix is square, symmetric, diagonal counts match replay totals

### Runnable

```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.coverage_gaps
```

Add to CLAUDE.md development commands.

## References

- `quarkmind-classifier/src/binary_catalog.py` — detector definitions
- `quarkmind-sc2/.../StrategyArchetype.java` — 58 archetypes
- `quarkmind-sc2/.../ArchetypeCategory.java` — 6 categories
- `quarkmind-sc2/.../DetectorOrchestrator.java` — Bayesian fusion consumer
- `quarkmind-sc2/.../DetectorRegistry.java` — phase/race filtering
- `quarkmind-sc2/.../SC2NetworkBuilder.java` — star-topology BJT builder
- `output/detectors/detector_manifest.json` — training stats
- ADR-0010 — Bayesian fusion architecture decision
