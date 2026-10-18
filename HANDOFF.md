# HANDOFF — quarkmind

## Last Session (2026-10-18)

Branch: `issue-413-combinatorial-gap-analysis` — **scaffolded, not started**
Previous: `issue-411-onnx-detector-library` — **closed**, landed as `83cce71f` on main

### What Was Done

**#411 closed — ONNX detector library complete (all 8 tasks, 5 batches).**

Replaced the multi-class ONNX cascade tier with Bayesian evidence fusion. BJT extracted into quarkmind-core (32 files), DetectorOrchestrator wires binary detector outputs + Drools markers through a star-topology Bayesian network. ADR-0010 recorded. Code reviewed (0 CRITICAL), branch audited (all 4 dimensions clean), squashed to 7 commits.

**#418 filed** — follow-up to upgrade BJT to incremental version (dirty subtree tracking, fast retraction) from `/Users/mdproctor/projects/drools` main branch. Current BJT uses per-tick `engine.reset()`.

### What's Next — #413

**Combinatorial gap analysis — detector coverage audit across phase transitions and compound strategies.**

The detector catalog (`binary_catalog.py`) has 9 DetectorDefs across 3 phases. #413 audits:
- Which phase×matchup×archetype combinations have no detector coverage
- Phase transition blind spots (early→mid, mid→late)
- Compound strategies that cross detector boundaries
- Whether the 6 ArchetypeCategories (RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION) are adequately covered per matchup

**Start with brainstorming** — this is analysis work, not implementation.

### Key Context

**Dependency issue (ongoing):** `casehub-blocks-ui-npm:0.4-SNAPSHOT` missing from remote repos. Workaround: `-Denforcer.skip=true -o`. Also `langchain4j` BOM resolution broken after online `mvn install` — use offline mode for all Maven commands.

**quarkmind-core must be installed before quarkmind-sc2 compiles:** `mvn install -pl quarkmind-core -DskipTests -Denforcer.skip=true -o`

**Key files for #413:**
- `quarkmind-classifier/src/binary_catalog.py` — detector definitions (9 detectors, phases, positives per matchup)
- `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/DetectorCatalogConfig.java` — Java mirror of catalog
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyArchetype.java` — 58 archetypes across 3 races, 3 phases, 6 categories
- `quarkmind-sc2/src/main/java/io/quarkmind/domain/ArchetypeCategory.java` — RUSH, TIMING, HARASS, MACRO, TECH, COMPOSITION

---

## Paused Work

| Branch | Issue | State | Resume with |
|--------|-------|-------|-------------|
| `issue-388-train-onnx-reconstituted-ladder` | #388 | Perspectives format fixed, trained on 148K | `work resume` |
