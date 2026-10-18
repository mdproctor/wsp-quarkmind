# Combinatorial Gap Analysis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #413 — Combinatorial gap analysis — detector coverage audit across phase transitions and compound strategies
**Issue group:** #413

**Goal:** Build a rerunnable Python analysis script that audits detector coverage gaps across archetypes, phases, and matchups, producing a markdown report.

**Architecture:** Single Python module `coverage_gaps.py` with three analysis layers: static catalog audit (parsing `StrategyArchetype.java` and importing `DETECTOR_CATALOG`), data-driven co-occurrence (scanning `.label.json` sidecars), and gap classification. Output is a markdown report at `docs/benchmarks/detector-coverage-gaps.md`.

**Tech Stack:** Python 3, no additional dependencies beyond what quarkmind-classifier already has.

## Global Constraints

- All code in `quarkmind-classifier/src/` and `quarkmind-classifier/tests/`
- Follow existing test conventions (see `test_binary_catalog.py`)
- `StrategyArchetype.java` is parsed from source — path is relative to project root: `quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyArchetype.java`
- `.label.json` files are in `data/reconstituted/**/*.label.json` (relative to `quarkmind-classifier/`)
- Manifest is at `output/detectors/detector_manifest.json` (relative to `quarkmind-classifier/`)
- Report output: `docs/benchmarks/detector-coverage-gaps.md` (relative to project root)
- Run command: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.coverage_gaps`

---

## Batch 1: Static catalog audit with archetype parsing

### Task 1: Parse StrategyArchetype.java and build coverage matrix

**Files:**
- Create: `quarkmind-classifier/src/coverage_gaps.py`
- Test: `quarkmind-classifier/tests/test_coverage_gaps.py`

**Interfaces:**
- Consumes: `StrategyArchetype.java` (file on disk), `DETECTOR_CATALOG` from `src.binary_catalog`
- Produces:
  - `parse_archetypes(java_path: Path) -> List[Archetype]` where `Archetype = namedtuple("Archetype", ["name", "race", "phase", "category"])`
  - `build_coverage_matrix(archetypes: List[Archetype], catalog: List[DetectorDef]) -> Dict[str, List[str]]` mapping archetype name → list of detector names that cover it
  - `find_missing_cells(archetypes: List[Archetype], catalog: List[DetectorDef]) -> List[Tuple[str, str, str]]` returning `(phase, matchup, category)` tuples with no detector coverage

- [ ] **Step 1: Write failing test for archetype parsing**

```python
# quarkmind-classifier/tests/test_coverage_gaps.py
"""Tests for detector coverage gap analysis."""
from pathlib import Path
from src.coverage_gaps import parse_archetypes

JAVA_PATH = Path(__file__).parent.parent.parent / (
    "quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyArchetype.java"
)


def test_parse_archetypes_count():
    archetypes = parse_archetypes(JAVA_PATH)
    assert len(archetypes) == 58


def test_parse_archetypes_fields():
    archetypes = parse_archetypes(JAVA_PATH)
    by_name = {a.name: a for a in archetypes}
    marine_rush = by_name["TERRAN_MARINE_RUSH"]
    assert marine_rush.race == "TERRAN"
    assert marine_rush.phase == "EARLY"
    assert marine_rush.category == "RUSH"


def test_parse_archetypes_all_phases():
    archetypes = parse_archetypes(JAVA_PATH)
    phases = {a.phase for a in archetypes}
    assert phases == {"EARLY", "MID", "LATE"}


def test_parse_archetypes_all_races():
    archetypes = parse_archetypes(JAVA_PATH)
    races = {a.race for a in archetypes}
    assert races == {"TERRAN", "ZERG", "PROTOSS"}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'src.coverage_gaps'`

- [ ] **Step 3: Implement parse_archetypes**

```python
# quarkmind-classifier/src/coverage_gaps.py
"""Detector coverage gap analysis — static audit + data-driven co-occurrence.

Audits the detector catalog against StrategyArchetype definitions and
reconstituted replay labels to identify coverage gaps, sparse co-occurrence
cells, and phase transition blind spots.

Usage:
    cd quarkmind-classifier
    PYTHONPATH=. .venv/bin/python3 -m src.coverage_gaps
"""
import json
import re
from collections import namedtuple
from pathlib import Path
from typing import Dict, List, Tuple

from src.binary_catalog import DETECTOR_CATALOG, DetectorDef

Archetype = namedtuple("Archetype", ["name", "race", "phase", "category"])

PROJECT_ROOT = Path(__file__).parent.parent.parent
JAVA_PATH = PROJECT_ROOT / (
    "quarkmind-sc2/src/main/java/io/quarkmind/domain/StrategyArchetype.java"
)

_ENUM_ENTRY = re.compile(
    r"^\s+(\w+)\(Race\.(\w+),\s*GamePhase\.(\w+),\s*ArchetypeCategory\.(\w+)\)"
)


def parse_archetypes(java_path: Path = JAVA_PATH) -> List[Archetype]:
    archetypes = []
    for line in java_path.read_text().splitlines():
        m = _ENUM_ENTRY.match(line)
        if m:
            archetypes.append(Archetype(m.group(1), m.group(2), m.group(3), m.group(4)))
    return archetypes
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v`
Expected: 4 PASSED

- [ ] **Step 5: Write failing tests for coverage matrix and missing cells**

Append to `tests/test_coverage_gaps.py`:

```python
from src.coverage_gaps import build_coverage_matrix, find_missing_cells
from src.binary_catalog import DETECTOR_CATALOG


def test_coverage_matrix_all_archetypes_present():
    archetypes = parse_archetypes(JAVA_PATH)
    matrix = build_coverage_matrix(archetypes, DETECTOR_CATALOG)
    assert len(matrix) == 58
    for a in archetypes:
        assert a.name in matrix


def test_coverage_matrix_known_covered():
    archetypes = parse_archetypes(JAVA_PATH)
    matrix = build_coverage_matrix(archetypes, DETECTOR_CATALOG)
    assert "early_rush" in matrix["TERRAN_MARINE_RUSH"]


def test_coverage_matrix_known_uncovered():
    archetypes = parse_archetypes(JAVA_PATH)
    matrix = build_coverage_matrix(archetypes, DETECTOR_CATALOG)
    assert matrix["TERRAN_DROP_HARASS"] == []


def test_missing_cells_not_empty():
    archetypes = parse_archetypes(JAVA_PATH)
    missing = find_missing_cells(archetypes, DETECTOR_CATALOG)
    assert len(missing) > 0
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v`
Expected: FAIL — `ImportError: cannot import name 'build_coverage_matrix'`

- [ ] **Step 7: Implement build_coverage_matrix and find_missing_cells**

Append to `coverage_gaps.py`:

```python
_COARSE_TO_FINE = {
    "RUSH": ["RUSH", "PROXY"],
    "ROACH_RUSH": ["ROACH_RUSH"],
    "CANNON_RUSH": ["CANNON_RUSH"],
    "MACRO_ECONOMY": ["MACRO"],
    "BANSHEE_HARASS": ["HARASS"],
    "AIR_SUPERIORITY": ["COMPOSITION"],
    "MECH_PUSH": ["TIMING", "COMPOSITION"],
    "BIO_TIMING": ["TIMING"],
    "TECH_RUSH": ["TECH"],
    "DT_RUSH": ["RUSH", "HARASS"],
    "BLINK_STALKER": ["TIMING"],
    "COLOSSUS_PUSH": ["TIMING", "COMPOSITION"],
    "HYDRA_PUSH": ["TIMING", "COMPOSITION"],
    "LING_BANE": ["COMPOSITION"],
    "MUTA_HARASS": ["HARASS"],
}

_RACE_TO_MATCHUP = {"TERRAN": "vs_terran", "ZERG": "vs_zerg", "PROTOSS": "vs_protoss"}


def build_coverage_matrix(
    archetypes: List[Archetype], catalog: List[DetectorDef]
) -> Dict[str, List[str]]:
    matrix: Dict[str, List[str]] = {}
    for arch in archetypes:
        detectors = []
        matchup = _RACE_TO_MATCHUP[arch.race]
        for det in catalog:
            if matchup not in det.positives:
                continue
            for pos_label in det.positives[matchup]:
                cats = _COARSE_TO_FINE.get(pos_label, [])
                if arch.category in cats or arch.name.endswith(pos_label):
                    detectors.append(det.name)
                    break
        matrix[arch.name] = detectors
    return matrix


def find_missing_cells(
    archetypes: List[Archetype], catalog: List[DetectorDef]
) -> List[Tuple[str, str, str]]:
    occupied = set()
    for det in catalog:
        for matchup in det.matchups():
            occupied.add((det.phase, matchup, det.name))

    all_cells = set()
    for arch in archetypes:
        matchup = _RACE_TO_MATCHUP[arch.race]
        all_cells.add((arch.phase.lower(), matchup, arch.category))

    covered_cells = set()
    matrix = build_coverage_matrix(archetypes, catalog)
    for arch in archetypes:
        if matrix[arch.name]:
            matchup = _RACE_TO_MATCHUP[arch.race]
            covered_cells.add((arch.phase.lower(), matchup, arch.category))

    return sorted(all_cells - covered_cells)
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v`
Expected: 8 PASSED

- [ ] **Step 9: Commit**

```bash
git add quarkmind-classifier/src/coverage_gaps.py quarkmind-classifier/tests/test_coverage_gaps.py
git commit -m "feat(classifier): add static catalog audit — archetype parsing and coverage matrix  Refs #413"
```

---

## Batch 2: Data-driven co-occurrence and gap classification

### Task 2: Scan label files and build co-occurrence matrix

**Files:**
- Modify: `quarkmind-classifier/src/coverage_gaps.py`
- Modify: `quarkmind-classifier/tests/test_coverage_gaps.py`

**Interfaces:**
- Consumes: `DETECTOR_CATALOG` from `src.binary_catalog`, `.label.json` files from `data/reconstituted/`
- Produces:
  - `scan_labels(data_dir: Path) -> List[ReplayLabel]` where `ReplayLabel = namedtuple("ReplayLabel", ["replay_hash", "matchup", "fine_label", "dataset"])`
  - `build_cooccurrence(labels: List[ReplayLabel], catalog: List[DetectorDef]) -> Dict[str, Dict[str, int]]` returning detector×detector co-occurrence counts
  - `classify_gaps(archetypes, catalog, labels, manifest) -> List[Gap]` where `Gap = namedtuple("Gap", ["kind", "description", "archetype", "severity"])`

- [ ] **Step 1: Write failing test for label scanning**

Append to `tests/test_coverage_gaps.py`:

```python
from src.coverage_gaps import scan_labels, ReplayLabel

DATA_DIR = Path(__file__).parent.parent / "data" / "reconstituted"


def test_scan_labels_returns_results():
    if not DATA_DIR.exists():
        pytest.skip("reconstituted data not available")
    labels = scan_labels(DATA_DIR)
    assert len(labels) > 0


def test_scan_labels_fields():
    if not DATA_DIR.exists():
        pytest.skip("reconstituted data not available")
    labels = scan_labels(DATA_DIR)
    first = labels[0]
    assert isinstance(first, ReplayLabel)
    assert first.matchup.startswith("vs_")
    assert first.fine_label != ""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py::test_scan_labels_returns_results -v`
Expected: FAIL — `ImportError: cannot import name 'scan_labels'`

- [ ] **Step 3: Implement scan_labels**

Append to `coverage_gaps.py`:

```python
ReplayLabel = namedtuple("ReplayLabel", ["replay_hash", "matchup", "fine_label", "dataset"])


def scan_labels(data_dir: Path) -> List[ReplayLabel]:
    labels = []
    for label_file in sorted(data_dir.rglob("*.label.json")):
        dataset = label_file.parent.name
        with open(label_file) as f:
            data = json.load(f)
        for perspective in data.get("perspectives", {}).values():
            matchup = perspective.get("matchup", "")
            fine = perspective.get("fineGrainedLabel", "")
            if matchup and fine:
                labels.append(ReplayLabel(data["replayHash"], matchup, fine, dataset))
    return labels
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v -k "scan_labels"`
Expected: 2 PASSED (or 2 SKIPPED if data not present)

- [ ] **Step 5: Write failing test for co-occurrence**

Append to `tests/test_coverage_gaps.py`:

```python
from src.coverage_gaps import build_cooccurrence


def test_cooccurrence_matrix_symmetric():
    if not DATA_DIR.exists():
        pytest.skip("reconstituted data not available")
    labels = scan_labels(DATA_DIR)
    cooc = build_cooccurrence(labels, DETECTOR_CATALOG)
    det_names = list(cooc.keys())
    for a in det_names:
        for b in det_names:
            assert cooc[a].get(b, 0) == cooc[b].get(a, 0), f"{a}/{b} asymmetric"


def test_cooccurrence_diagonal_nonnegative():
    if not DATA_DIR.exists():
        pytest.skip("reconstituted data not available")
    labels = scan_labels(DATA_DIR)
    cooc = build_cooccurrence(labels, DETECTOR_CATALOG)
    for det in cooc:
        assert cooc[det].get(det, 0) >= 0
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v -k "cooccurrence"`
Expected: FAIL — `ImportError: cannot import name 'build_cooccurrence'`

- [ ] **Step 7: Implement build_cooccurrence**

Append to `coverage_gaps.py`:

```python
def _label_activates_detector(fine_label: str, matchup: str, det: DetectorDef) -> bool:
    if matchup not in det.positives:
        return False
    return fine_label in det.positives[matchup]


def build_cooccurrence(
    labels: List[ReplayLabel], catalog: List[DetectorDef]
) -> Dict[str, Dict[str, int]]:
    det_names = [d.name for d in catalog]
    cooc: Dict[str, Dict[str, int]] = {d: {d2: 0 for d2 in det_names} for d in det_names}

    for label in labels:
        activated = [
            d for d in catalog
            if _label_activates_detector(label.fine_label, label.matchup, d)
        ]
        for i, a in enumerate(activated):
            for b in activated[i:]:
                cooc[a.name][b.name] += 1
                if a.name != b.name:
                    cooc[b.name][a.name] += 1

    return cooc
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v -k "cooccurrence"`
Expected: 2 PASSED (or 2 SKIPPED)

- [ ] **Step 9: Write failing test for gap classification**

Append to `tests/test_coverage_gaps.py`:

```python
from src.coverage_gaps import classify_gaps, Gap

MANIFEST_PATH = Path(__file__).parent.parent / "output" / "detectors" / "detector_manifest.json"


def test_classify_gaps_finds_uncovered():
    archetypes = parse_archetypes(JAVA_PATH)
    if not DATA_DIR.exists():
        pytest.skip("reconstituted data not available")
    labels = scan_labels(DATA_DIR)
    manifest = json.loads(MANIFEST_PATH.read_text()) if MANIFEST_PATH.exists() else []
    gaps = classify_gaps(archetypes, DETECTOR_CATALOG, labels, manifest)
    assert len(gaps) > 0
    kinds = {g.kind for g in gaps}
    assert "uncovered" in kinds or "weak" in kinds or "rare" in kinds
```

- [ ] **Step 10: Run test to verify it fails**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py::test_classify_gaps_finds_uncovered -v`
Expected: FAIL — `ImportError: cannot import name 'classify_gaps'`

- [ ] **Step 11: Implement classify_gaps**

Append to `coverage_gaps.py`:

```python
Gap = namedtuple("Gap", ["kind", "description", "archetype", "severity"])

SPARSE_THRESHOLD = 50
WEAK_ACCURACY_THRESHOLD = 0.5


def classify_gaps(
    archetypes: List[Archetype],
    catalog: List[DetectorDef],
    labels: List[ReplayLabel],
    manifest: list,
) -> List[Gap]:
    gaps = []
    matrix = build_coverage_matrix(archetypes, catalog)

    label_counts: Dict[str, int] = {}
    for label in labels:
        label_counts[label.fine_label] = label_counts.get(label.fine_label, 0) + 1

    for arch in archetypes:
        detectors = matrix[arch.name]
        if not detectors:
            name_parts = arch.name.split("_", 1)
            coarse_key = name_parts[1] if len(name_parts) > 1 else arch.name
            count = label_counts.get(coarse_key, 0)
            if count < SPARSE_THRESHOLD:
                gaps.append(Gap("rare", f"{arch.name}: no detector, {count} replays", arch.name, "low"))
            else:
                gaps.append(Gap("uncovered", f"{arch.name}: no detector, {count} replays available", arch.name, "high"))

    weak_keys = set()
    for entry in manifest:
        if entry.get("accuracy", 1.0) < WEAK_ACCURACY_THRESHOLD:
            weak_keys.add(entry["key"])
    for key in sorted(weak_keys):
        gaps.append(Gap("weak", f"{key}: accuracy below {WEAK_ACCURACY_THRESHOLD}", key, "medium"))

    return gaps
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m pytest tests/test_coverage_gaps.py -v`
Expected: all PASSED (data-dependent tests SKIP if data unavailable)

- [ ] **Step 13: Commit**

```bash
git add quarkmind-classifier/src/coverage_gaps.py quarkmind-classifier/tests/test_coverage_gaps.py
git commit -m "feat(classifier): add co-occurrence matrix and gap classification  Refs #413"
```

---

## Batch 3: Report generation and CLAUDE.md update

### Task 3: Generate markdown report and wire as __main__

**Files:**
- Modify: `quarkmind-classifier/src/coverage_gaps.py`
- Create: `docs/benchmarks/detector-coverage-gaps.md` (generated)
- Modify: `CLAUDE.md` (add run command)

**Interfaces:**
- Consumes: all functions from Tasks 1-2
- Produces: `generate_report(output_path: Path) -> None` — writes full markdown report; `__main__` block that calls `generate_report`

- [ ] **Step 1: Implement generate_report and __main__**

Append to `coverage_gaps.py`:

```python
PHASE_TRANSITIONS = {
    "RUSH": ["TIMING", "COMPOSITION"],
    "CHEESE": ["TIMING"],
    "MACRO": ["COMPOSITION", "TECH"],
    "HARASS": ["TIMING", "COMPOSITION", "HARASS"],
    "TECH": ["COMPOSITION", "TECH"],
}

CLASSIFIER_DIR = Path(__file__).parent.parent
REPORT_PATH = PROJECT_ROOT / "docs" / "benchmarks" / "detector-coverage-gaps.md"
DATA_DIR = CLASSIFIER_DIR / "data" / "reconstituted"
MANIFEST_PATH = CLASSIFIER_DIR / "output" / "detectors" / "detector_manifest.json"


def generate_report(output_path: Path = REPORT_PATH) -> None:
    archetypes = parse_archetypes()
    matrix = build_coverage_matrix(archetypes, DETECTOR_CATALOG)
    missing = find_missing_cells(archetypes, DETECTOR_CATALOG)

    labels = scan_labels(DATA_DIR) if DATA_DIR.exists() else []
    cooc = build_cooccurrence(labels, DETECTOR_CATALOG) if labels else {}
    manifest = json.loads(MANIFEST_PATH.read_text()) if MANIFEST_PATH.exists() else []
    gaps = classify_gaps(archetypes, DETECTOR_CATALOG, labels, manifest)

    lines = ["# Detector Coverage Gap Analysis\n"]
    lines.append(f"Generated from {len(archetypes)} archetypes, "
                 f"{len(DETECTOR_CATALOG)} detectors, {len(labels)} replay labels.\n")

    # Section 1: Archetype coverage matrix
    lines.append("## 1. Archetype Coverage Matrix\n")
    lines.append("| Archetype | Race | Phase | Category | Detectors |")
    lines.append("|-----------|------|-------|----------|-----------|")
    for arch in archetypes:
        dets = matrix[arch.name]
        det_str = ", ".join(dets) if dets else "**(none)**"
        lines.append(f"| {arch.name} | {arch.race} | {arch.phase} | {arch.category} | {det_str} |")

    uncovered = [a for a in archetypes if not matrix[a.name]]
    lines.append(f"\n**Uncovered archetypes: {len(uncovered)} / {len(archetypes)}**\n")

    # Section 2: Missing cells
    lines.append("## 2. Missing Phase × Matchup × Category Cells\n")
    if missing:
        lines.append("| Phase | Matchup | Category |")
        lines.append("|-------|---------|----------|")
        for phase, matchup, category in missing:
            lines.append(f"| {phase} | {matchup} | {category} |")
    else:
        lines.append("All cells covered.\n")

    # Section 3: Weak detectors
    lines.append("\n## 3. Weak Detectors (accuracy < 50%)\n")
    weak = [e for e in manifest if e.get("accuracy", 1.0) < WEAK_ACCURACY_THRESHOLD]
    if weak:
        lines.append("| Key | Accuracy | Train Pos | Train Neg |")
        lines.append("|-----|----------|-----------|-----------|")
        for e in weak:
            lines.append(f"| {e['key']} | {e['accuracy']:.1%} | {e['train_pos']} | {e['train_neg']} |")
    else:
        lines.append("All detectors above threshold.\n")

    # Section 4: Within-phase co-occurrence
    lines.append("\n## 4. Within-Phase Co-occurrence\n")
    if cooc:
        for phase in ["early", "mid", "late"]:
            phase_dets = [d.name for d in DETECTOR_CATALOG if d.phase == phase]
            if not phase_dets:
                continue
            lines.append(f"### {phase.capitalize()} phase\n")
            header = "| | " + " | ".join(phase_dets) + " |"
            sep = "|---|" + "|".join(["---"] * len(phase_dets)) + "|"
            lines.append(header)
            lines.append(sep)
            for a in phase_dets:
                cells = [str(cooc.get(a, {}).get(b, 0)) for b in phase_dets]
                lines.append(f"| {a} | " + " | ".join(cells) + " |")
            lines.append("")

        # Cross-phase: early × mid
        lines.append("### Cross-phase: Early × Mid\n")
        early_dets = [d.name for d in DETECTOR_CATALOG if d.phase == "early"]
        mid_dets = [d.name for d in DETECTOR_CATALOG if d.phase == "mid"]
        header = "| | " + " | ".join(mid_dets) + " |"
        sep = "|---|" + "|".join(["---"] * len(mid_dets)) + "|"
        lines.append(header)
        lines.append(sep)
        for a in early_dets:
            cells = [str(cooc.get(a, {}).get(b, 0)) for b in mid_dets]
            lines.append(f"| {a} | " + " | ".join(cells) + " |")
        lines.append("")

    # Section 5: Phase transition chains
    lines.append("## 5. Phase Transition Chains\n")
    lines.append("Structural analysis — which early categories naturally transition to mid categories.\n")
    lines.append("| Early Category | → Mid Categories | Detector Coverage |")
    lines.append("|----------------|------------------|-------------------|")
    for early_cat, mid_cats in PHASE_TRANSITIONS.items():
        mid_str = ", ".join(mid_cats)
        early_covered = any(
            d.phase == "early" for d in DETECTOR_CATALOG
            if any(early_cat in str(p) for p in d.positives.values())
        )
        mid_covered = any(
            any(d.phase == "mid" for d in DETECTOR_CATALOG
                if any(mc.lower() in str(p).lower() for p in d.positives.values()))
            for mc in mid_cats
        )
        coverage = "both" if early_covered and mid_covered else "early only" if early_covered else "mid only" if mid_covered else "**none**"
        lines.append(f"| {early_cat} | {mid_str} | {coverage} |")

    # Section 6: Gap assessment
    lines.append("\n## 6. Gap Assessment\n")
    for kind in ["uncovered", "weak", "rare"]:
        kind_gaps = [g for g in gaps if g.kind == kind]
        if kind_gaps:
            label = {"uncovered": "Fixable (no detector, data available)",
                     "weak": "Weak (detector exists, poor accuracy)",
                     "rare": "Rare strategy (acceptable)"}[kind]
            lines.append(f"### {label}\n")
            for g in kind_gaps:
                lines.append(f"- {g.description}")
            lines.append("")

    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_text("\n".join(lines) + "\n")
    print(f"Report written to {output_path}")


if __name__ == "__main__":
    generate_report()
```

- [ ] **Step 2: Run the script to generate the report**

Run: `cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.coverage_gaps`
Expected: `Report written to .../docs/benchmarks/detector-coverage-gaps.md`

- [ ] **Step 3: Verify the generated report**

Read `docs/benchmarks/detector-coverage-gaps.md` and verify:
- All 58 archetypes appear in the coverage matrix
- Missing cells table is populated
- Weak detectors section lists `mid_ground_heavy_vs_terran` and `late_ground_army_vs_protoss`
- Co-occurrence matrices are present and symmetric
- Gap assessment classifies each gap

- [ ] **Step 4: Add run command to CLAUDE.md**

Add under the `## Development Commands` section after the existing classifier commands:

```markdown
**Coverage gap analysis (classifier — requires reconstituted data in data/reconstituted/):**
```bash
cd quarkmind-classifier && PYTHONPATH=. .venv/bin/python3 -m src.coverage_gaps
```
```

- [ ] **Step 5: Commit**

```bash
git add quarkmind-classifier/src/coverage_gaps.py docs/benchmarks/detector-coverage-gaps.md CLAUDE.md
git commit -m "feat(classifier): generate detector coverage gap report  Refs #413"
```

---

## References

- [2026-10-18-combinatorial-gap-analysis-design.md] — design spec this plan implements
- `quarkmind-classifier/src/binary_catalog.py` — detector definitions
- `quarkmind-classifier/tests/test_binary_catalog.py` — existing test pattern
- `quarkmind-sc2/.../StrategyArchetype.java:1-96` — 58 archetypes parsed at runtime
- `quarkmind-sc2/.../DetectorOrchestrator.java` — Bayesian fusion consumer
- `output/detectors/detector_manifest.json` — training stats
- GitHub #413 — focal issue
