## D1: Analysis approach

**Choice:** Python analysis script in quarkmind-classifier/src/
**Alternatives:**
- Java diagnostic test — uses real Java types but mixes concerns (training pipeline analysis in the game engine module)
- Manual document — fastest but stale immediately, no rerunnable quantification
**Rationale:** The gap analysis is a training pipeline concern — it audits detector coverage and training data completeness. Living alongside binary_catalog.py keeps it maintainable and rerunnable as the catalog evolves.
**Trade-offs:** None significant — the script parses StrategyArchetype.java directly at runtime, so no hardcoded Python mirror is needed and no sync test required.
**Sources:** binary_catalog.py, detector_manifest.json, StrategyArchetype.java
**Exploration:** quick
**Status:** captured

## D2: Analysis granularity

**Choice:** Both coarse (detector-level) and fine-grained (per-StrategyArchetype) coverage
**Alternatives:**
- Coarse detector-level only — misses archetypes invisible to all detectors
- Fine-grained archetype-level only — doesn't address pairwise co-occurrence
**Rationale:** The coarse level answers "do detectors work together?" while the fine-grained level answers "what's completely invisible?" — both are needed for a complete gap assessment.
**Trade-offs:** More analysis code, but the two views share most of the data loading.
**Sources:** StrategyArchetype.java (58 archetypes), binary_catalog.py (9 detectors)
**Exploration:** quick
**Status:** captured

## D3: Data source for co-occurrence

**Choice:** Reconstituted .label.json sidecars + static catalog analysis
**Alternatives:**
- NPZ training splits — no replay IDs, can't correlate across detectors
- Cross-phase feature re-extraction — heavy pipeline, out of scope for S-scale issue
**Rationale:** Label JSONs exist for all reconstituted replays and contain the heuristic archetype classification. Cross-phase transitions analysed structurally from the catalog rather than re-extracting features.
**Trade-offs:** Within-phase co-occurrence only (not cross-phase data-driven). Cross-phase analysis is structural/qualitative. Acceptable for an audit — a future issue can add temporal re-extraction if gaps warrant it.
**Sources:** data/reconstituted/*/\*.label.json, DroolsCoarseLabelExportTest
**Exploration:** quick
**Status:** captured
