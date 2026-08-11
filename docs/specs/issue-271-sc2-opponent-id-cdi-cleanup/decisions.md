## D1: Opponent identity composition in SC2 mode

**Choice:** Race + difficulty composite for AI opponents, hashed player ID for PvP/bot-vs-bot. Scoped to 1v1 only.
**Alternatives:**
- Protocol player ID always — session-local integer for AI opponents is meaningless across games
- Race-only for AI, ID for PvP — loses difficulty dimension which affects opponent behaviour patterns
**Rationale:** Against AI, race+difficulty is the stable identity (e.g. SHA-256 of "ZERG:VeryHard"). Against human/bot opponents (PARTICIPANT type), the protocol player ID is a real persistent identity worth hashing for privacy. The composite gives CBR meaningful opponent-specific case retrieval in both scenarios.
**Scoping assumption:** QuarkMind operates strictly in 1v1 mode — the single-strategy-per-tick architecture and single-opponent CBR case structure both assume exactly one opponent. Multi-player modes are out of scope.
**Trade-offs:** AI opponent hashes won't distinguish between two AIs of the same race+difficulty in different games (acceptable — they behave identically). Hashing loses debuggability for PvP IDs, but privacy justifies it.
**Exploration:** quick
**Status:** revised (R1-03: added explicit 1v1 scope, clarified hashing rationale)

## D2: Runtime opponent ID injection mechanism

**Choice:** GameStarted event enrichment — enrich the existing GameStarted CDI event with opponent identity data; GameStateTranslator observes it and stores the computed opponent ID in an AtomicReference<String>
**Alternatives:**
- OpponentIdProvider CDI bean — introduces mutable shared state with thread-safety gap between game loop thread and scheduler thread
- Add opponentId to GameState record — pollutes domain model with per-game identity on every tick's state
- CaseFile event at game start — bypasses GameStateTranslator, creates a parallel write path to CaseFile
**Rationale:** The codebase already uses GameStarted CDI events (MomentDetectionTask.onGameStarted(), SC2CbrRetentionObserver.onGameStarted()). Enriching GameStarted with opponent identity follows the established pattern. AtomicReference<String> in GameStateTranslator is thread-safe (game loop thread writes once at start, scheduler thread reads each tick). No new CDI bean needed.
**Trade-offs:** GameStarted event grows in responsibility (carries opponent identity alongside game-start signaling). Acceptable given the event is already per-game-lifecycle.
**Exploration:** quick
**Status:** revised (R1-02: replaced OpponentIdProvider with GameStarted event enrichment for thread-safety and pattern consistency)

## D3: Battle detection mechanism

**Choice:** Army value delta with hysteresis — track army value each tick; when it drops sharply (>15%), open a battle window; when it stabilises (quiescence), close the window and emit BATTLE_ENDED with engagement metadata. Independent of BATTLE_STARTED.
**Alternatives:**
- Build on BATTLE_STARTED — only detects incoming timing attacks (TimingAlert.incoming == true), misses player-initiated attacks, skirmishes, harassment, and non-timing defensive engagements
- Unit count delta stabilisation — requires new unit death event infrastructure
- Combat unit proximity — requires GameState access in MomentDetection
**Rationale:** BATTLE_STARTED fires from TimingAlert (incoming timing attack detection by scouting), not a universal battle detector. Building engagement tracking on it creates systematically biased CBR data — only incoming timing attacks would be tracked. Army value delta detects ALL engagements regardless of who initiated. The ARMY_SHIFT rule already demonstrates delta-detection (30% threshold). A lower threshold (15%) with hysteresis (open window on drop, close on stabilisation) captures engagements while ARMY_SHIFT captures dramatic composition changes.
**Trade-offs:** Needs calibration — the 15% threshold and quiescence window need tuning against replays. But this is a calibration problem, not an architectural one.
**Exploration:** quick
**Status:** revised (R1-01: replaced BATTLE_STARTED-gated approach with independent army value delta detection)

## D4: Engagement tracker location

**Choice:** MomentDetectionTask — extend with battle tracking state (army value delta, battle window FSM, army snapshots)
**Alternatives:**
- New EngagementTracker component — cleaner separation but adds a new CDI bean and subscription
- SC2CbrRetentionObserver — retrospective reconstruction at game end, no live snapshots
- Drools CEP temporal pattern — first-class temporal reasoning, but the existing DRL uses Rule Units not CEP
**Rationale:** MomentDetectionTask already tracks cross-tick state (previousArmyValue, previousPosture, lastSupplyBlockFrame) and has CaseContext access in execute(). The battle window FSM (idle → in_battle → quiescent → idle) fits alongside existing state. Army snapshots at window open/close provide unit trade data.
**Trade-offs:** MomentDetectionTask grows in responsibility. If engagement logic becomes complex, extract to a separate component.
**Depends on:** D3 (army value delta determines the detection logic)
**Exploration:** quick
**Status:** revised (R1-01 dependency: updated to reflect army-value-delta approach)

## D5: Per-engagement data flow to CBR

**Choice:** Typed EngagementOutcome record in BATTLE_ENDED's Map<String,Object> metadata — single entry with typed record
**Alternatives:**
- Raw untyped metadata entries — multiple untyped casts at consumer sites
- New EngagementRecord domain type with separate bus — adds a new type and bus level
- Aggregate CaseFile keys — running totals on CaseFile, loses per-engagement detail
**Rationale:** A typed EngagementOutcome record in the summarisation package provides compile-time safety while preserving the Map<String,Object> container pattern. Consumer code does one typed cast instead of four untyped ones. Precedent: existing moment metadata has 1-2 simple entries; engagement data has 4+ fields including computed ratio — justifies the typed wrapper.
**Trade-offs:** Adds one record type to the summarisation package. Acceptable for type safety.
**Depends on:** D4 (MomentDetectionTask produces the BATTLE_ENDED moment with this metadata)
**Exploration:** quick
**Status:** revised (R1-05: replaced raw Map entries with typed EngagementOutcome record)
