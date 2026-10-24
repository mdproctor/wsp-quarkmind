## D1: Correlation mechanism — pre-indexed CmdEvent map

**Choice:** Pre-index game events into a `commandLoop → selectedBuildingTag` map, then look up building tags when processing tracker events
**Alternatives:**
- Interleaved single-pass — elegant but tracker/game events have different loop semantics (completion vs initiation), making stream ordering complex
- Reuse AbilityMapping directly — couples to full AbilityMapping pipeline (morph multiplication, abilLink dispatch) when we only need selection tracking; also CmdEvent-driven intent counts differ from tracker ground truth
**Rationale:** Keeps TrackerIntentExtractor's existing tracker-event-driven structure (correct intent counts from ground truth) and just enriches with building tag lookup. Index-building step is self-contained and testable.
**Trade-offs:** Two passes over data instead of one. Acceptable — game event arrays are already in memory.
**Sources:** TrackerIntentExtractor.java, AbilityMapping.java:779-783, TrackerEventFeatureExtractor.java:130-232
**Exploration:** quick
**Status:** captured

## D2: Selection tracking approach — lightweight inline

**Choice:** Private helper method in TrackerIntentExtractor with `TreeMap<Integer, String>` (loop → building tag). Inline the ~40 lines of SelectionDelta processing from AbilityMapping.onSelection().
**Alternatives:**
- Extract SelectionUnitLinkTracker to top-level class — tracks unitLinks not decoded tags; would need extending for our use case
- Use AbilityMapping for selection only — 900+ line class with unnecessary state; selectionSnapshotForTest() is a test-only API
**Rationale:** We need exactly one thing: the decoded building tag from selection.first() at each CmdEvent loop. TreeMap.floorEntry() gives efficient ±window lookup. Minimal, self-contained, testable.
**Trade-offs:** Duplicates ~40 lines of SelectionDelta processing logic from AbilityMapping. Acceptable — the alternative is coupling to a 900-line class for 40 lines of logic.
**Depends on:** D1 (pre-indexed map structure)
**Sources:** AbilityMapping.java:391-465, SelectionState.java, GameEventStream.java:34-36
**Exploration:** quick
**Status:** captured

## D3: Index key structure — loop-only

**Choice:** `TreeMap<Integer, String>` keyed by CmdEvent loop only, most-recent-wins for same-loop events
**Alternatives:**
- Composite (loop, abilLink) key — more precise but unnecessary; selection state doesn't change between same-loop CmdEvents
**Rationale:** Selection state is set by SelectionDeltaEvents which precede all CmdEvents at that loop. Same-loop CmdEvents share the same selection, making abilLink disambiguation unnecessary.
**Trade-offs:** None significant — simpler key is also correct.
**Depends on:** D1 (pre-indexed map structure), D2 (selection tracking)
**Sources:** SelectionDeltaEvent ordering semantics in TrackerEventFeatureExtractor.java:165-168
**Exploration:** quick
**Status:** captured
