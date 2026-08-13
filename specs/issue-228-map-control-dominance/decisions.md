## D1: Map control as 5th factor vs replacing bases

**Choice:** Add mapControl as a 5th factor alongside bases, with runtime weight configuration
**Alternatives:**
- Replace bases entirely — simpler (4-field structure unchanged), but loses the absolute base count signal
- Keep bases only — defers the work, territorial coverage remains unmodeled
**Rationale:** Runtime weights give both options without code branching — set bases weight to near-zero to try replacement, or non-zero for both. Try both, see what works. Note: the temporal strategy (current default) allows weights down to 0.0; the Drools strategy enforces MINIMUM_WEIGHT=0.05 floor per factor, so full replacement (bases=0.0) requires the temporal strategy.
**Trade-offs:** Blast radius across DominanceWeights, WeightModifier, MilestoneConfig, all 3 weight strategies, all Drools rules, and all tests. Accepted — user explicitly not concerned about change volume.
**Exploration:** quick
**Status:** captured

## D2: Expansion control definition

**Choice:** Contested/uncontrolled state — expansions with no nearby base are "uncontrolled." Formula: `clamp((ownControlled - enemyControlled) / maxExpectedMapControlDelta)` — consistent with all other factors using a single configurable normalization constant.
**Alternatives:**
- Proximity match with hard threshold only — every expansion either controlled or not, no "uncontrolled" concept, but same practical effect
- Nearest-base assignment — every expansion assigned to closest base regardless of distance, inflates control count unrealistically
- Ratio-based: `(ownControlled - enemyControlled) / totalExpansions` — self-normalizing by map size, but breaks the consistent normalization pattern used by all other factors
**Rationale:** Matches SC2 reality where most expansions are uncontrolled at any given time. Uses the same normalization pattern as economy/army/tech/bases (fixed configurable constant, not variable per map). Expansion locations already available via `GameState.mapInfo().expansions()` (added in #256).
**Trade-offs:** Requires a proximity threshold constant that needs replay calibration. The factor measures expansion-site occupation, not full territorial control (army presence, watchtower control, creep spread) — "mapControl" is the extension point name; richer signals can enrich it later.
**Exploration:** quick
**Status:** captured

## D3: Proximity threshold for "controlled"

**Choice:** 10.0 map units from expansion centroid, with mandatory calibration test against replay datasets
**Alternatives:**
- Use CLUSTER_RADIUS (12.0) — risk adjacent expansion overlap
- Tighter threshold (5.0) — risk missing legitimately placed bases offset from centroid
**Rationale:** 10.0 is comfortably wider than typical building placement offset (~5-8 units) but narrower than CLUSTER_RADIUS (12.0) so adjacent expansions don't overlap. Starting value only — calibration test validates against 59 replays per spatial constants protocol.
**Trade-offs:** Uncalibrated starting value; may need adjustment after calibration test runs.
**Depends on:** D2 (proximity-based control detection)
**Exploration:** quick
**Status:** captured

## D4: Fog-of-war guard

**Choice:** Reuse existing enemyBuildings.isEmpty() guard — return 0.0 when no enemy buildings visible
**Alternatives:**
- Require at least one enemy base building visible (stricter) — but inconsistent with tech/bases factors which use the same enemyBuildings guard
- No additional guard (rely on combined threshold only) — risk false positives when only enemy units visible
**Rationale:** Consistency with established pattern. The combined threshold (minEnemyVisibility) plus per-factor guard already handles the edge cases proven across the other four factors.
**Trade-offs:** Map control has a fog-of-war asymmetry: you always know your own controlled expansions but may miss enemy expansions behind fog, systematically inflating perceived advantage mid-game before scouting reveals enemy bases. This asymmetry also exists in the bases factor (same data source). Bounded by dead zone threshold and milestone confidence. Not unique enough to warrant a different guard pattern.
**Exploration:** quick
**Status:** captured

## D5: Default weight allocation

**Choice:** economy=0.30, army=0.35, tech=0.20, bases=0.05, mapControl=0.10
**Alternatives:**
- Equal split from all factors (each loses ~0.02) — dilutes every factor for marginal map control signal
- Larger mapControl weight (0.15) taken from economy — economy is a strong signal, reducing it hurts early-game accuracy
**Rationale:** Map control and bases are related (both territorial). Taking most of the budget from bases (0.15→0.05) keeps the stronger signals intact. Uncalibrated defaults — reasonable starting position to test from.
**Trade-offs:** Bases factor at 0.05 sits at the Drools MINIMUM_WEIGHT floor — modifiers can only push it up, never down (asymmetric responsiveness). With the temporal strategy (current default), 0.05 is a true minimum. The factors overlap significantly but don't fully subsume each other — a player with 3 orbital commands at natural gets high bases but low mapControl; standard play shows high correlation.
**Depends on:** D1 (5-factor structure enables separate weights)
**Exploration:** quick
**Status:** captured
