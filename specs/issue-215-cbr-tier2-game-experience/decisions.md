# CBR Tier 2 — Design Decisions

## D1: Case model — single enriched case

**Choice:** Enrich the existing `SC2GameCbrCase` (cbrType `sc2-strategy`) with all Tier 2 fields from all six domains (moments, tactics, economics, scouting, opponent, observability).
**Alternatives:**
- Separate case types per domain (sc2-tactics, sc2-economics, sc2-scouting) — cleaner separation but requires cross-type correlation at retrieval time
- Hybrid (enriched game case + separate advisory + separate tactical) — partial separation, more complexity
**Rationale:** One game is one case. A single query returns all enrichment data. Neocortex handles sparse features — old cases with missing Tier 2 fields get neutral similarity scores. Avoids cross-type join complexity.
**Trade-offs:** Larger feature schema per case. Schema grows as new domains are added. But games are bounded in size and the case base is small (< 1000 games).
**Exploration:** quick
**Status:** captured

## D2: Data capture — bus subscriber injection

**Choice:** `SC2CbrRetentionObserver` subscribes to `SummarisationLifecycle` buses (momentBus, phaseBus, arcBus) and accumulates L2/L3/L4 data during the game. At game close, accumulated data is available alongside CaseFile snapshot keys.
**Alternatives:**
- Write enrichment data to CaseFile keys (bloats CaseFile with serialised lists)
- Post-close enrichment step via CaseEnrichmentStep SPI (two-phase store, more complex)
**Rationale:** Same pattern `MultiFactorDominanceAssessor` already uses — proven, no new infrastructure. Observer has everything it needs at close time without CaseFile bloat.
**Trade-offs:** Observer becomes stateful (accumulates data between game start and close). Must clear on `GameStarted` event. But this is exactly what `AdvisoryInvocationCounter` already does.
**Exploration:** quick
**Status:** captured

## D3: Opponent ID — top-level feature

**Choice:** Add `opponent_id` as a categorical feature on the `sc2-strategy` case. Use `CbrQuery.withFilter("opponent_id", Contains("bot-xyz"))` for per-opponent retrieval. Hash player IDs in SC2 mode for privacy.
**Alternatives:**
- Separate opponent registry with case cross-references — richer model but over-engineered for current needs
**Rationale:** Uses existing infrastructure directly. CbrQuery filtering is purpose-built for this. Opponent ID is just another feature — no new abstractions needed.
**Trade-offs:** No aggregation across opponents built-in (pattern detection like "this opponent always opens roach rush" requires application-level logic over retrieved cases). Acceptable — #221 QA endpoints can provide this.
**Exploration:** quick
**Status:** captured

## D4: Observability — query-time computation

**Choice:** QA endpoints compute win rate, strategy evolution, and retrieval stats on demand from `CbrCaseMemoryStore.scan()`. No pre-computed aggregates.
**Alternatives:**
- Pre-computed aggregator bean (faster response but mutable state, staleness)
- Hybrid with caching (best response time but adds caching complexity)
**Rationale:** Case base is small (< 1000 games). Scan + compute is fast enough. Matches existing QA endpoint pattern (`@UnlessBuildProfile("prod")`). No mutable state to manage.
**Trade-offs:** Response time grows linearly with case base size. Acceptable for dev/test QA endpoints — not production-facing.
**Exploration:** quick
**Status:** captured

## D5: Data granularity — aggregate summary metrics for all domains

**Choice:** Store aggregate summary metrics (not full temporal data) for tactics, economics, and scouting. Consistent approach across all enrichment domains.
**Alternatives:**
- Full engagement list as StructListVal for tactics — richer queries but similarity scoring for ObjectList/StructList is less proven
- Summary + one NumberListVal trajectory per domain — DTW matching on economic curves but adds complexity
- Full temporal data per issue spec — maximum fidelity but inconsistent across domains
**Rationale:** Aggregate metrics keep similarity scoring straightforward using well-tested numeric and categorical feature types. Individual event data (L2 moments, engagement details) is available via the L2 moment list in the case for context display — it just doesn't drive similarity scoring. Consistent granularity across domains makes the schema predictable.
**Trade-offs:** Cannot query by temporal pattern ("find games where economy spiked at minute 5") — that's Tier 3 (#222) territory. Aggregate metrics lose ordering information. Acceptable — Tier 2 is about structured enrichment, not temporal matching.
**Depends on:** D1 (single enriched case)
**Exploration:** quick
**Status:** captured

## D6: Schema evolution — extend existing schema

**Choice:** Add new `FeatureField` entries to the existing `sc2-strategy` schema in `SC2CbrSchemaRegistrar`. Old cases with missing features get neutral similarity scores (neocortex handles sparse features natively).
**Alternatives:**
- New sc2-strategy-v2 schema — cleaner boundary but requires migration
- Versioned schema with auto-migration — most robust but over-engineered for an empty-start case base
**Rationale:** Schema extension is the simplest path. No migration needed. Sparse feature handling is built into the similarity scorer. The case base is effectively empty (starts fresh with each deployment cycle).
**Trade-offs:** Old cases in a persistent store would have missing features — scored neutrally, which is correct behaviour (no information = no signal). Schema grows monotonically.
**Depends on:** D1 (single enriched case)
**Exploration:** quick
**Status:** captured
