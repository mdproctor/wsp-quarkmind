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

## D2: DRL posture rule consumption

**Choice:** Confirmed overrides suspected — when building data is available, it takes precedence
**Alternatives:**
- Highest confidence wins — adds confidence scoring complexity to the posture model
- Union (either tier triggers MACRO) — simplest DRL but hardest to revert
**Rationale:** Clean state machine: pre-confirmation phase uses suspected signal (unit heuristic), post-confirmation phase uses building data exclusively. Reversion only fires from the confirmed tier — no oscillation from the suspected tier.
**Trade-offs:** Once confirmed data exists, the unit heuristic is permanently ignored for posture. This is correct (ground truth supersedes proxy) but means the unit heuristic only matters before first building confirmation.
**Sources:** DroolsScoutingTask.drl posture rules, cachedPosture logic in DroolsScoutingTask.java
**Exploration:** quick
**Depends on:** D1 (hybrid approach)
**Status:** captured

## D3: Fog of war handling for confirmed expansions

**Choice:** Sticky until contradicted — confirmed expansion persists until the location is observed without the building
**Alternatives:**
- Volatile (only current observation) — accurate but oscillates with vision changes
- Timed (sticky with decay) — middle ground but adds another arbitrary threshold
**Rationale:** Matches SC2 information model — a player doesn't know the expansion was destroyed until they scout the location again. Prevents oscillation from vision changes while still allowing retraction on confirmed destruction.
**Trade-offs:** If the expansion is destroyed in fog and we never re-scout, MACRO persists indefinitely. But this is correct behavior — intelligence should match what the player can actually observe.
**Sources:** SC2 fog-of-war model, ScoutingSessionManager buffer design
**Exploration:** quick
**Depends on:** D1 (hybrid approach), D2 (confirmed overrides)
**Status:** captured
