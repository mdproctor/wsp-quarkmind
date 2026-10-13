# HANDOFF — quarkmind

## Last Session (2026-10-13/14)

Closed #410 (re-label reconstituted replays). Brainstormed ONNX detector library architecture → filed #411 (L/High). Updated #388 with honest evaluation criteria. Audited all replay training data.

### What Shipped (#410)

- `relabel_reconstituted.py` — re-labels reconstituted replays using Python building-timing rules. 100% label coverage (was ~20%).
- `train_coarse.py` — coarse classification experiment (5 classes). Confirmed 403 tournament replays produce majority-class voting — data diversity is the bottleneck.
- `reconstitution_pipeline.py` fix — correct API call signature.
- `model.py` fix — handles 4-feature maps without crashing.

### Key Discovery

**148K Blizzard ladder replays have tracker events.** ~90% of `blizzard_ladder_4.9.3` has good tracker events (10+ per replay). Label distribution is diverse — all 16 fine-grained labels represented. This is the training dataset we need.

**Gate to #388:** `reconstitution_pipeline.py` reads labels in flat format (`coarseLabel` at root). Blizzard labels use `perspectives` format (nested per player). Pipeline silently skips all 148K replays. Format fix unlocks them.

### Replay Data State

**See `notes/replay-data-state.md`** — full audit of all datasets, tracker event coverage, label distribution, format gaps, reconstitution state. Updated 2026-10-14.

### Design Work (#411)

9 design decisions captured in `specs/issue-410-relabel-python-rules/decisions.md`:
- Hybrid decomposition — phase gates which detectors run, each detector is situation-specific
- YAML playbook orchestration (epic #600 primitives) for detector lifecycle
- Pluggable selector + aggregator SPIs with CBR meta-learning loop
- Open decisions: D7 (binary vs multi-class), D8 (CascadingPatternClassifier disposition)

### EmulatedGame Accuracy

99.8% unit accuracy at 5-min checkpoint against 118 oracle replays (tracker-event-driven). Landed as `4c8009fa`.

---

## Next Session

### Immediate: Unblock #388

1. Fix `perspectives` label format in `reconstitution_pipeline.py` — the gate to 148K diverse replays
2. Run pipeline on full `data/reconstituted/` (exclude 4.10.1)
3. Train and evaluate honestly:
   - Does ONNX learn real class boundaries, or just majority-class voting?
   - Does ONNX add signal over what Drools heuristic rules already provide?
   - If not, the features from stripped replays don't carry enough signal — stop

### Strategic approach to data

One more pass to get data quality as high as practical. Then train and use what we have. If ONNX doesn't add signal over Drools rules, either invest in SC2 observation pipeline or accept Drools + LLM as the classification tier. Don't train on garbage — if the data can't support learning, stop.

### Open Issues

| # | Title | Scale | Status |
|---|-------|-------|--------|
| 388 | Train ONNX on reconstituted data | M | Next — gate: perspectives format fix |
| 411 | ONNX detector library architecture | L | After #388 — only if models are worth using |
| 407 | Exact replay reconstruction via building routing | M | Deprioritised — replay fidelity only |
| 408 | Promote situational awareness loop to generic core | M | Depends on #409 |
| 409 | Bidirectional alignment: SC2 ↔ quarkmind-core | L | Architectural, separate workstream |
