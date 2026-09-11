## D1: Expansion detection approach

**Choice:** Hybrid — unit heuristic (suspected) + building confirmation (confirmed)
**Alternatives:**
- A: Live building census — simplest, but loses early-game detection from unit heuristic
- B: Temporal eviction — minimal change, but doesn't fix false positives from attacking armies
**Rationale:** Two-tier model gives early coverage (unit heuristic fires before expansion building is visible) and accurate retraction (building confirmation is retractable when the building is destroyed). The unit heuristic remains the early signal; building tracking adds ground truth and the MACRO→ALL_IN reversion capability.
**Trade-offs:** More complex than single-source approaches. Two parallel tracking systems in ScoutingSessionManager. DRL rules need to handle both tiers.
**Sources:** ScoutingSessionManager.java, DroolsScoutingTask.drl, Building.java, #304 findings (expansion heuristic limitations)
**Exploration:** quick
**Status:** captured
