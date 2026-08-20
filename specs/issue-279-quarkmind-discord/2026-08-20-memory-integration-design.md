# QuarkMind Chat — Memory Integration (D5 Four-Layer Memory)

**Date:** 2026-08-20
**Status:** Draft
**Issue:** #280
**Depends on:** Phase 1 spec (2026-08-17-quarkmind-discord-design.md)
**Decisions:** D13–D20 in `decisions.md` (+ D21–D24 from review)

## Overview

Wire episodic, semantic, and relationship memory into the ChatAgencyLoop so the chat agent remembers conversations, learns about people, and develops insights over time. The foundation already provides all storage and retrieval primitives — this work is composition and integration, plus cross-repo API additions (importance scoring in neocortex, response handling in quarkmind-core).

## Cross-Repo Work

### casehub-neocortex: updateImportance() on CaseMemoryStore (D17)

Add a method to update a memory's importance score after initial storage:

```java
// CaseMemoryStore (memory-api)
default void updateImportance(String memoryId, String tenantId, double importance) {
    if (importance < 0.0 || importance > 1.0) {
        throw new IllegalArgumentException("importance must be in [0, 1], got " + importance);
    }
    throw new UnsupportedOperationException("updateImportance not supported");
}
```

Range validation at the interface level: importance must be in [0.0, 1.0], matching `MemoryInput`'s existing validation. Primitive `double` is intentional — an update always provides a concrete score; the null/unset case doesn't apply to an update operation.

Implementations:
- **JpaMemoryStore** — `UPDATE memory_entry SET importance = :importance WHERE id = :memoryId AND tenant_id = :tenantId`
- **SqliteMemoryStore** — `UPDATE memory_entry SET importance = :importance WHERE memory_id = :memoryId AND tenant_id = :tenantId`
- **InMemoryMemoryStore** — lookup by ID, replace importance
- **NoOpCaseMemoryStore** — no-op (ignore silently)
- **Mem0CaseMemoryStore** — no-op with warning log (Mem0 REST API does not expose post-hoc importance updates; importance remains null in Mem0-backed memories)
- **GraphitiCaseMemoryStore** — no-op with warning log (standalone REST-backed store implementing `GraphCaseMemoryStore`; Graphiti episodes and facts have no importance field in the REST API)
- **CaseEnrichmentDecorator** — delegation to underlying store:
  ```java
  @Override public void updateImportance(String memoryId, String tenantId, double importance) {
      delegate.updateImportance(memoryId, tenantId, importance);
  }
  ```
  Required because the decorator explicitly overrides every `CaseMemoryStore` method with delegation. Without this override, the decorator inherits the interface default (throw `UnsupportedOperationException`), intercepting all CDI-injected `updateImportance()` calls even when the underlying store supports it.

Idempotent — calling twice with the same score is safe. On failure (memory not found, store unavailable), log a warning and continue — a missing importance score degrades retrieval ranking but does not break the system. Unscored memories default to 1.0 in retrieval re-rankers (`PersonalityWeightedRetrieval` and `MemoryOrder.SALIENCE` treat null importance as 1.0), ensuring unscored memories remain fully visible until async scoring completes.

SETTLED: Importance default is 1.0 for null importance in retrieval re-rankers — degradation-safe for async scoring. D5's original 0.5 choice predated the async scoring pattern; 1.0 is correct because unscored memories should not be penalized before they've had a chance to be scored. (from R1-03)

Contract tests: extend `CaseMemoryStoreContractTest` with `updateImportance_updatesStoredScore()`, `updateImportance_unknownMemory_noError()`, and `updateImportance_outOfRange_throwsIllegalArgument()`.

### quarkmind-core: LlmRequest response handler (D17)

Add a response handler to `LlmRequest` for async LLM operations that need the response:

```java
public record LlmRequest(String prompt, LlmPriority priority,
                          Map<String, Object> metadata,
                          Consumer<String> responseHandler) {
    public LlmRequest(String prompt, LlmPriority priority, Map<String, Object> metadata) {
        this(prompt, priority, metadata, null);
    }
    public LlmRequest {
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }
}
```

When `responseHandler` is non-null, the queue processor invokes it with the LLM response after execution. When null, the request is fire-and-forget (backward compatible). This preserves `LlmRequestQueue`'s prioritization and rate limiting while enabling async response delivery for importance scoring and reflection synthesis.

## quarkmind-chat Module Changes

### ChatMemoryFacade (D13)

A single class in `quarkmind-chat-agent` that composes neocortex stores:

```java
public class ChatMemoryFacade {

    private final CaseMemoryStore memoryStore;
    private final GraphCaseMemoryStore graphStore;  // may be NoOp
    private final boolean graphAvailable;

    // Retrieve memories relevant to the current conversation
    List<Memory> recall(String agentId, String tenantId,
                        String conversationContext,
                        Set<String> participantIds,
                        PersonalityWeights weights, Instant now);

    // Store an observation from this tick
    String ingest(String agentId, String tenantId,
                  String observationText,
                  Map<String, String> sourceRefs,
                  Set<String> participantIds);

    // Update importance after async LLM scoring
    void scoreImportance(String memoryId, String tenantId, double importance);
}
```

**`recall()` flow — two-stage pipeline:**
1. **Stage 1 — Semantic retrieval:** Build query from conversation context: `ExperienceQuery.search(agentId, tenantId, conversationSummary)`. Query `CaseMemoryStore.query()` — returns semantically similar episodic memories ranked by `MemoryOrder.RELEVANCE`.
2. If `graphAvailable`: for each participant, `graphStore.graphQuery(GraphMemoryQuery.forEntity(participantId, chatDomain, tenantId, "what do I know about this person?"))` — returns relationship facts.
3. Merge episodic + relationship results.
4. **Stage 2 — Personality re-ranking:** `PersonalityWeightedRetrieval.reweight(merged, weights, now)` — re-ranks by `recency(τ=168h) × importance(default 1.0 if null) × domainWeight(personality preference)`.
5. Return top N memories (configurable, default 15).

Episodic retrieval is intentionally broad: `ExperienceQuery.search()` queries by agentId + tenantId + semantic similarity, with no participant filtering. Cross-conversation knowledge pollination is valuable — if the agent learned that Bob works on NLP in a previous conversation, it should recall that when talking to Alice about NLP. The relationship layer (`GraphCaseMemoryStore` or attribute-based) handles per-person context; episodic memory is deliberately agent-wide.

**`ingest()` flow:**
1. Build `MemoryInput`: entityId=agentId, domain=`ExperienceEvents.DOMAIN`, text=observationText, importance=null, attributes include source references + participant tags
2. Store via `CaseMemoryStore.store(input)` — returns memoryId
3. Return memoryId for async importance scoring

SETTLED: Use `ExperienceEvents.DOMAIN` ("experience") for episodic memories, not a custom domain. `ExperienceQuery`, `ReflectionService`, and `PersonalityWeightedRetrieval` all operate on this domain — a custom domain fragments the ecosystem and makes stored memories invisible to recall. (from R1-01)

**Graphiti-optional (D20):** Constructor checks if the injected `GraphCaseMemoryStore` is a real implementation or `NoOpCaseMemoryStore`. Sets `graphAvailable` flag. When false, `recall()` skips graph queries — relationship context comes only from participant tags on episodic memories.

### ChatAgencyLoop Changes

The tick flow gains memory integration (marked with **→**):

```
1.  Read perception (existing)
2.  Build conversation delta via ChatPerceptionBridge (existing)
3.  **→ Retrieve memories via ChatMemoryFacade.recall()**
4.  Build user prompt — now includes memories + needs + conversation context
5.  LLM invoke — response now includes action + observation
6.  Parse response — extract intents + observation text
7.  **→ Ingest observation via ChatMemoryFacade.ingest()**
8.  **→ Submit async importance scoring (LlmRequestQueue LOW priority with responseHandler)**
9.  **→ On heartbeat wake: check IdleReflectionTrigger.shouldReflect()**
10. Return intents (existing)
```

**Observation from same LLM call (D18):** The JSON response schema gains an `observation` field:

```json
{
  "action": "SEND|REPLY|REACT|WAIT",
  "channel": "channel-id",
  "text": "message",
  "observation": "Had a conversation with Bob about machine learning. He shared a paper about transformers."
}
```

On WAIT, the observation is still captured — staying quiet is a meaningful experience. The observation is never null (the LLM always describes what it observed).

**Memory in prompt:** Retrieved memories are formatted as a "What I remember" section between needs and conversation context:

```
Needs: SOCIAL=75, CURIOSITY=60, EXPRESSION=40

What I remember:
- [2h ago] Bob and I discussed transformer architectures. He works on NLP at Google.
- [1d ago] Alice mentioned she's starting a new ML course. I recommended Andrew Ng's.
- [3d ago, about Bob] He prefers concise explanations with code examples.

[Conversation context follows]
```

**Reflection orchestration (wired in tick):**

On heartbeat wake (no message activity), after step 9:
1. `ChatAgencyLoop` calls `IdleReflectionTrigger.shouldReflect(consecutiveIdleTicks)`
2. If true: invoke `ReflectionOrchestrator.reflect(agentId, tenantId, lastReflectionTimestamp, maxSourceMemories)`
3. `lastReflectionTimestamp` is a field on `ChatAgencyLoop`, initialized to startup time, updated after each reflection completes
4. After reflection: call `IdleReflectionTrigger.reset()`

The `since` parameter scopes source memories to those created after the last reflection, preventing re-processing of already-reflected experiences.

### Memory Attributes Schema

Each observation stored in CaseMemoryStore carries these attributes:

| Key | Value | Purpose |
|-----|-------|---------|
| `source.channelId` | Discord channel ID | Source reference for verification |
| `source.firstMessageId` | First message ID in tick | Source range start |
| `source.lastMessageId` | Last message ID in tick | Source range end |
| `source.timestamp.start` | ISO-8601 | Temporal range start |
| `source.timestamp.end` | ISO-8601 | Temporal range end |
| `participant.<userId>` | Display name | Tag each participant for attribute-based queries |
| `tick.action` | SEND/REPLY/REACT/WAIT | What the agent did |

### Async Importance Scoring (D17)

After each tick's observation is stored:

1. `ChatAgencyLoop` submits an `LlmRequest` at `LlmPriority.LOW` with the observation text and a `responseHandler` callback
2. Prompt: "Rate the importance of this experience on a scale of 0.0 to 1.0, where 0.0 is mundane and 1.0 is life-changing. Respond with a single number."
3. The `responseHandler` callback: parses the score, calls `ChatMemoryFacade.scoreImportance(memoryId, tenantId, score)`, then calls `IdleReflectionTrigger.accumulate(score)` to feed the reflection trigger
4. On LLM failure or unparseable response, log and continue — the memory remains with importance=null, retrieval treats it as 1.0 (fully visible until scored)

```java
llmQueue.submit(new LlmRequest(prompt, LlmPriority.LOW, metadata, response -> {
    try {
        double score = Double.parseDouble(response.trim());
        facade.scoreImportance(memoryId, tenantId, score);
        reflectionTrigger.accumulate(score);
    } catch (Exception e) {
        log.warn("Importance scoring failed for memory " + memoryId, e);
    }
}));
```

### LLM-based ReflectionSynthesizer (D19)

Implements `io.casehub.neocortex.memory.reflection.ReflectionSynthesizer`:

```java
@Alternative
@Priority(1)
@ApplicationScoped
public class LlmReflectionSynthesizer implements ReflectionSynthesizer {

    private final ChatAgencyLoop.LlmInvoker llmInvoker;

    @Override
    public List<ReflectionEvent> synthesize(String agentId, String tenantId,
                                            List<Memory> sources, int targetLevel) {
        // Format source memories into a summary request
        // Call llmInvoker.invoke() synchronously — reflection runs during idle time
        // Parse response into ReflectionEvent records
        // ReflectionEvents with importance=null use the formula:
        //   Math.min(0.3 + (level * 0.2), 1.0) — level 1 = 0.5, level 3 = 0.9
        // Set explicit importance on ReflectionEvent when the formula default
        // is not appropriate for the specific insight
    }
}
```

**CDI displacement:** `@Alternative @Priority(1)` displaces `NoOpReflectionSynthesizer` (`@DefaultBean @ApplicationScoped` in neocortex). Lives in `quarkmind-chat-agent` (not neocortex) because of the dependency on `ChatAgencyLoop.LlmInvoker` — placing it in neocortex would create a neocortex → quarkmind-chat-agent dependency.

`ReflectionService` orchestrates: queries episodic memories via `ExperienceQuery.forAgent()` → passes to synthesizer → stores reflections via `ReflectionEvents.toMemoryInput()` (domain: `ReflectionEvents.DOMAIN` = "reflection") → fires `ReflectionRecorded` CDI event.

### IdleReflectionTrigger (quarkmind-core)

```java
public class IdleReflectionTrigger {
    private double accumulatedImportance = 0.0;
    private final double threshold;        // configurable, default 3.0
    private final int idleHeartbeats;      // configurable, default 5

    // Called from async importance scoring responseHandler callback
    void accumulate(double importance);

    // Called on heartbeat wake — returns true if reflection should fire
    boolean shouldReflect(int consecutiveIdleTicks);

    // Called after reflection completes — resets accumulator
    void reset();
}
```

Reflection importance is determined by `ReflectionEvents.toMemoryInput()` using the formula `Math.min(0.3 + (level × 0.2), 1.0)`:
- Level 1 (first-pass synthesis): importance 0.5
- Level 2 (meta-reflection): importance 0.7
- Level 3 (deep insight): importance 0.9
- Level 4+: importance 1.0

`LlmReflectionSynthesizer` can set explicit importance on individual `ReflectionEvent` records when the formula default is not appropriate — the formula only applies when `ReflectionEvent.importance()` is null.

## Retrieval Composite Model

Retrieval follows a two-stage pipeline inspired by Park et al. 2023:

**Stage 1 — Semantic retrieval:** `CaseMemoryStore.query()` with `MemoryOrder.RELEVANCE` (via `ExperienceQuery.search()`) returns memories ranked by semantic similarity to the conversation context. This is the relevance factor from Park et al.

**Stage 2 — Personality re-ranking:** `PersonalityWeightedRetrieval.reweight()` re-ranks the retrieved memories by:

```
score(memory) = recencyDecay(memory.createdAt, now)  // exp(-hours / τ), τ=168h
              × importance(memory)                    // 0.0–1.0, default 1.0 if null
              × domainWeight(memory.domain)           // PersonalityWeights per domain
```

`recencyDecay` is exponential decay with time constant τ=168 hours (~1 week). At 168h elapsed, the decay factor is `exp(-1) ≈ 0.37`. `domainWeight` is a per-domain personality preference weight from `PersonalityWeights`, not semantic relevance — relevance is handled in stage 1.

No new retrieval machinery needed — `PersonalityWeightedRetrieval.reweight()` already computes this formula.

When mood state is available (future — requires eidos mood integration), `MoodModulatedRetrieval.reweight()` adds a mood alignment factor using the PAD (pleasure-arousal-dominance) model.

## Episodic Memory Store: CaseMemoryStore — not CbrCaseMemoryStore (D14)

The phase 1 spec D5 named CbrCaseMemoryStore for episodic memory. This was incorrect — CbrCaseMemoryStore is for structured feature-vector cases (SC2GameCbrCase with supply_count, unit_composition). Chat conversations are unstructured text where semantic embedding search is the right retrieval model.

`CaseMemoryStore` with `ExperienceQuery.search()` (MemoryOrder.RELEVANCE) provides semantic retrieval. This is what `ExperienceQuery`, `ReflectionService`, `PersonalityWeightedRetrieval`, and `MoodModulatedRetrieval` already use. No schema definition needed — the store handles embedding generation internally.

## Observation Model (D15, D16, D18)

Memories are observation summaries — what the agent perceived and did — not verbatim message transcripts. The platform (Discord) is the source of truth for raw messages. Observations carry source references (channel ID + message ID range) for verification via `MessageHistory`.

One observation per channel per agency tick. On WAIT (no action taken), the observation still captures what was perceived — the decision not to act is meaningful context for future recall.

The observation is produced by the same LLM call that decides the action — no extra async call for observation generation. The LLM already has the full conversation context; asking it to describe what happened adds ~50 tokens to the response.

## Testing Strategy

- **ChatMemoryFacadeTest** (plain JUnit): mock CaseMemoryStore + GraphCaseMemoryStore. Verify recall() composes queries correctly with `ExperienceQuery.search()`, ingest() builds MemoryInput with `ExperienceEvents.DOMAIN` and correct attributes, scoreImportance() delegates to updateImportance().
- **ChatMemoryFacade graph-fallback test**: verify recall() skips graphQuery when GraphCaseMemoryStore is NoOp, falls back to attribute-based filtering.
- **ChatAgencyLoopTest updates**: verify the loop calls recall() before LLM, ingest() after, submits async importance scoring via LlmRequestQueue with responseHandler, and checks shouldReflect() on heartbeat wakes.
- **LlmReflectionSynthesizerTest**: mock LlmInvoker, verify reflection prompt composition and ReflectionEvent parsing.
- **CaseMemoryStoreContractTest extension** (neocortex): updateImportance happy path + unknown memoryId no-error + out-of-range IllegalArgumentException.
- **End-to-end test**: full tick cycle with in-memory stores — perception → recall (empty initially) → LLM → ingest → verify memory stored → importance scoring callback fires → verify importance updated → second tick → recall (previous observation returned).

## Scope

**In scope for #280:**
- Cross-repo neocortex: `updateImportance()` on CaseMemoryStore + all 7 implementations + CaseEnrichmentDecorator delegation
- Cross-repo quarkmind-core: `LlmRequest.responseHandler` field for async LLM response delivery
- ChatMemoryFacade with recall() + ingest()
- ChatAgencyLoop memory retrieval + ingestion + reflection orchestration integration
- Async importance scoring via LlmRequestQueue with responseHandler callback
- IdleReflectionTrigger in quarkmind-core (was spec'd in phase 1 but not built)
- LlmReflectionSynthesizer (`@Alternative @Priority(1)` in quarkmind-chat-agent) + wiring to ReflectionService
- Observation model with source references
- Graphiti-optional relationship memory

**Out of scope (separate issues):**
- #281: Personality growth pipeline (D6 Layer 3) — reflection → disposition activation → evolution
- Mood-modulated retrieval (requires eidos mood state integration)
- Importance scoring backfill for existing memories (GitHub issue to be filed)
- Custom PersonalityWeights per agent (uses defaults for v1)

## References

- `casehub-neocortex/memory-api` — CaseMemoryStore, GraphCaseMemoryStore, MemoryInput, ExperienceQuery, ExperienceEvents, GraphMemoryQuery, PersonalityWeightedRetrieval, MoodModulatedRetrieval
- `casehub-neocortex/memory` — ReflectionService, ReflectionSynthesizer, ReflectionEvents, NoOpReflectionSynthesizer
- `quarkmind-chat/quarkmind-chat-agent` — ChatAgencyLoop, ChatPerceptionBridge (phase 1)
- `quarkmind-core` — LlmRequestQueue, LlmRequest, LlmPriority, NeedThresholdWake, IdleReflectionTrigger
- Park et al. 2023 "Generative Agents" — composite retrieval model inspiration (recency × importance × relevance)
- GE-20260630-815259 — cross-repo SPI extends CDI displacement (CbrCaseMemoryStore standalone pattern)
- GE-20260811-aa9a26 — casehub-blocks dependency transitively requires neocortex config properties
- GE-20260717-0489d1 — CbrQuery.of() Path scope parameter (API break awareness)
- Phase 1 spec: D5 (four-layer memory architecture), D8 (direct LLM, no CaseEngine)
- decisions.md: D13–D20 (memory integration decisions), D21–D24 (review-surfaced implicit decisions)
