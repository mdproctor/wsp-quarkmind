# HANDOFF — quarkmind

## Last Session (2026-10-18)

Branch: `issue-413-combinatorial-gap-analysis` — **closed**, landed as `3b3a1afa` on main
Previous: `issue-411-onnx-detector-library` — **closed**, landed as `83cce71f` on main

### What Was Done

**#413 closed — Detector coverage gap analysis complete.**

Built a rerunnable Python analysis script (`quarkmind-classifier/src/coverage_gaps.py`) that audits binary detector coverage against 68 StrategyArchetype entries. Parses `StrategyArchetype.java` directly at runtime (no hardcoded Python mirror). Scans 297K replay labels from reconstituted data for co-occurrence. Classifies gaps into 4 kinds: weak-detector, uncovered (fixable), labelling-pipeline gap, and rare-strategy.

**Key findings:**
- 34% archetype coverage (23/68 archetypes have at least one detector)
- 11 missing phase×matchup×category cells (HARASS has zero coverage everywhere; all late-phase COMPOSITION and TECH are uncovered)
- 2 weak detectors: `mid_ground_heavy_vs_terran` (20%), `late_ground_army_vs_protoss` (16%)
- 45 labelling gaps — the heuristic labeller produces only 16 coarse labels, making most fine-grained archetypes invisible to the entire detection pipeline
- Co-occurrence matrices are diagonal — each replay label activates at most one detector, so no pairwise co-occurrence exists in current data
- **Bottleneck is the labelling pipeline, not detector count** — expanding from 16 to ~30 coarse labels would unlock coverage for the majority of uncovered archetypes

Report: `docs/benchmarks/detector-coverage-gaps.md`
13 pytest tests, CLAUDE.md updated with run command.

Code reviewed (0 CRITICAL), branch audited (all 4 dimensions clean), squashed to 1 commit.

### Key Context

**Dependency issue (ongoing):** `casehub-blocks-ui-npm:0.4-SNAPSHOT` missing from remote repos. Workaround: `-Denforcer.skip=true -o`. Also `langchain4j` BOM resolution broken after online `mvn install` — use offline mode for all Maven commands.

**quarkmind-core must be installed before quarkmind-sc2 compiles:** `mvn install -pl quarkmind-core -DskipTests -Denforcer.skip=true -o`

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
