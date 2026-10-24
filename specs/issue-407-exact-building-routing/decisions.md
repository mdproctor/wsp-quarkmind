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
