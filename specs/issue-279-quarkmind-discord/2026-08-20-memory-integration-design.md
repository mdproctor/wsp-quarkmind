# QuarkMind Chat — Memory Integration (D5 Four-Layer Memory)

**Date:** 2026-08-20
**Status:** Draft
**Issue:** #280
**Depends on:** Phase 1 spec (2026-08-17-quarkmind-discord-design.md)
**Decisions:** D13–D20 in `decisions.md` (+ D21–D24 from review)

## Overview

Wire episodic, semantic, and relationship memory into the ChatAgencyLoop so the chat agent remembers conversations, learns about people, and develops insights over time. The foundation already provides all storage and retrieval primitives — this work is composition and integration, plus one cross-repo API addition (importance scoring in neocortex).

## Cross-Repo Work: casehub-neocortex

### updateImportance() on CaseMemoryStore (D17)

Add a method to update a memory's importance score after initial storage:

```java
// CaseMemoryStore (memory-api)
default void updateImportance(String memoryId, String tenantId, double importance) {
    throw new UnsupportedOperationException("updateImportance not supported");
}
```

Implementations:
- **JpaMemoryStore** — `UPDATE memory_entry SET importance = :importance WHERE id = :memoryId AND tenant_id = :tenantId`
- **InMemoryMemoryStore** — lookup by ID, replace importance
- **NoOpCaseMemoryStore** — no-op (ignore silently)
- **GraphitiCaseMemoryStore** — delegate to underlying store if backed by JPA; no-op if Graphiti-only

Idempotent — calling twice with the same score is safe. On failure (memory not found, store unavailable), log a warning and continue — a missing importance score degrades retrieval ranking but does not break the system. Unscored memories default to 0.5 in retrieval re-rankers.

Contract tests: extend `CaseMemoryStoreContractTest` with `updateImportance_updatesStoredScore()` and `updateImportance_unknownMemory_noError()`.

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

**`recall()` flow:**
1. Build a semantic query from conversation context: `ExperienceQuery.search(agentId, tenantId, conversationSummary)`
2. Query `CaseMemoryStore.query()` — returns semantically similar episodic memories
3. If `graphAvailable`: for each participant, `graphStore.graphQuery(GraphMemoryQuery.forEntity(participantId, chatDomain, tenantId, "what do I know about this person?"))` — returns relationship facts
4. Merge episodic + relationship results
5. Re-rank with `PersonalityWeightedRetrieval.reweight(merged, weights, now)` — applies recency decay (168h half-life) × importance × personality domain weights
6. Return top N memories (configurable, default 15)

**`ingest()` flow:**
1. Build `MemoryInput`: entityId=agentId, domain=`MemoryDomain.of("chat-experience")`, text=observationText, importance=null, attributes include source references + participant tags
2. Store via `CaseMemoryStore.store(input)` — returns memoryId
3. Return memoryId for async importance scoring

**Graphiti-optional (D20):** Constructor checks if the injected `GraphCaseMemoryStore` is a real implementation or `NoOpCaseMemoryStore`. Sets `graphAvailable` flag. When false, `recall()` skips graph queries — relationship context comes only from participant tags on episodic memories.

### ChatAgencyLoop Changes

The tick flow gains three new steps (marked with **→**):

```
1.  Read perception (existing)
2.  Build conversation delta via ChatPerceptionBridge (existing)
3.  **→ Retrieve memories via ChatMemoryFacade.recall()**
4.  Build user prompt — now includes memories + needs + conversation context
5.  LLM invoke — response now includes action + observation
6.  Parse response — extract intents + observation text
7.  **→ Ingest observation via ChatMemoryFacade.ingest()**
8.  **→ Submit async importance scoring (LlmRequestQueue LOW priority)**
9.  Return intents (existing)
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

1. ChatAgencyLoop submits an `LlmRequest` at `LlmPriority.LOW` with the observation text
2. Prompt: "Rate the importance of this experience on a scale of 0.0 to 1.0, where 0.0 is mundane and 1.0 is life-changing. Respond with a single number."
3. On response, parse the score and call `ChatMemoryFacade.scoreImportance(memoryId, tenantId, score)`
4. On LLM failure, log and continue — the memory remains with importance=null, retrieval treats it as 0.5

### LLM-based ReflectionSynthesizer (D19)

Implements `io.casehub.neocortex.memory.reflection.ReflectionSynthesizer`:

```java
public class LlmReflectionSynthesizer implements ReflectionSynthesizer {

    private final ChatAgencyLoop.LlmInvoker llmInvoker;

    @Override
    public List<ReflectionEvent> synthesize(String agentId, String tenantId,
                                            List<Memory> sources, int targetLevel) {
        // Format source memories into a summary request
        // Submit LLM call: "Given these experiences, what patterns or insights
        //   do you notice? Produce 1-3 generalized reflections."
        // Parse response into ReflectionEvent records
    }
}
```

Triggered by `IdleReflectionTrigger` (quarkmind-core, **built in this phase** — was spec'd in phase 1 but not implemented). The trigger tracks accumulated importance across stored observations. When accumulated importance exceeds a configurable threshold and the agent is idle (no message wake in the last N heartbeats), it fires a reflection event. `ReflectionService` orchestrates: queries episodic memories → passes to synthesizer → stores reflections back as high-importance memories → fires `ReflectionRecorded` CDI event.

### IdleReflectionTrigger (quarkmind-core)

```java
public class IdleReflectionTrigger {
    private double accumulatedImportance = 0.0;
    private final double threshold;        // configurable, default 3.0
    private final int idleHeartbeats;      // configurable, default 5

    // Called after each importance score arrives
    void accumulate(double importance);

    // Called on heartbeat wake — returns true if reflection should fire
    boolean shouldReflect(int consecutiveIdleTicks);

    // Called after reflection completes — resets accumulator
    void reset();
}
```

Reflections are stored via `CaseMemoryStore.store()` with domain `MemoryDomain.of("chat-reflection")` and importance pre-set (reflections are inherently high-importance: 0.8–1.0).

## Retrieval Composite Model

The retrieval model follows the Park et al. composite (recency × importance × relevance), using existing neocortex re-rankers:

```
score(memory) = recencyDecay(memory.createdAt, now)
              × importance(memory)           // 0.0–1.0, default 0.5 if null
              × domainWeight(memory.domain)  // PersonalityWeights per domain
```

`PersonalityWeightedRetrieval.reweight()` already computes this exact formula with exponential recency decay (half-life 168 hours = 1 week). No new retrieval machinery needed.

When mood state is available (future — requires eidos mood integration), `MoodModulatedRetrieval.reweight()` adds a mood alignment factor using the PAD (pleasure-arousal-dominance) model.

## Episodic Memory Store: CaseMemoryStore — not CbrCaseMemoryStore (D14)

The phase 1 spec D5 named CbrCaseMemoryStore for episodic memory. This was incorrect — CbrCaseMemoryStore is for structured feature-vector cases (SC2GameCbrCase with supply_count, unit_composition). Chat conversations are unstructured text where semantic embedding search is the right retrieval model.

`CaseMemoryStore` with `ExperienceQuery.search()` (MemoryOrder.RELEVANCE) provides semantic retrieval. This is what `ExperienceQuery`, `ReflectionService`, `PersonalityWeightedRetrieval`, and `MoodModulatedRetrieval` already use. No schema definition needed — the store handles embedding generation internally.

## Observation Model (D15, D16, D18)

Memories are observation summaries — what the agent perceived and did — not verbatim message transcripts. The platform (Discord) is the source of truth for raw messages. Observations carry source references (channel ID + message ID range) for verification via `MessageHistory`.

One observation per channel per agency tick. On WAIT (no action taken), the observation still captures what was perceived — the decision not to act is meaningful context for future recall.

The observation is produced by the same LLM call that decides the action — no extra async call for observation generation. The LLM already has the full conversation context; asking it to describe what happened adds ~50 tokens to the response.

## Testing Strategy

- **ChatMemoryFacadeTest** (plain JUnit): mock CaseMemoryStore + GraphCaseMemoryStore. Verify recall() composes queries correctly, ingest() builds MemoryInput with correct attributes, scoreImportance() delegates to updateImportance().
- **ChatMemoryFacade graph-fallback test**: verify recall() skips graphQuery when GraphCaseMemoryStore is NoOp, falls back to attribute-based filtering.
- **ChatAgencyLoopTest updates**: verify the loop calls recall() before LLM, ingest() after, and submits async importance scoring.
- **LlmReflectionSynthesizerTest**: mock LlmInvoker, verify reflection prompt composition and ReflectionEvent parsing.
- **CaseMemoryStoreContractTest extension** (neocortex): updateImportance happy path + unknown memoryId no-error.
- **End-to-end test**: full tick cycle with in-memory stores — perception → recall (empty initially) → LLM → ingest → verify memory stored → second tick → recall (previous observation returned).

## Scope

**In scope for #280:**
- Cross-repo: `updateImportance()` on CaseMemoryStore + implementations (neocortex)
- ChatMemoryFacade with recall() + ingest()
- ChatAgencyLoop memory retrieval + ingestion integration
- Async importance scoring via LlmRequestQueue
- IdleReflectionTrigger in quarkmind-core (was spec'd in phase 1 but not built)
- LlmReflectionSynthesizer + wiring to ReflectionService
- Observation model with source references
- Graphiti-optional relationship memory

**Out of scope (separate issues):**
- #281: Personality growth pipeline (D6 Layer 3) — reflection → disposition activation → evolution
- Mood-modulated retrieval (requires eidos mood state integration)
- Importance scoring backfill for existing memories
- Custom PersonalityWeights per agent (uses defaults for v1)

## References

- `casehub-neocortex/memory-api` — CaseMemoryStore, GraphCaseMemoryStore, MemoryInput, ExperienceQuery, GraphMemoryQuery, PersonalityWeightedRetrieval, MoodModulatedRetrieval
- `casehub-neocortex/memory` — ReflectionService, ReflectionSynthesizer, TemporalDecayCbrCaseMemoryStore
- `quarkmind-chat/quarkmind-chat-agent` — ChatAgencyLoop, ChatPerceptionBridge (phase 1)
- `quarkmind-core` — LlmRequestQueue, LlmPriority, NeedThresholdWake, IdleReflectionTrigger
- Park et al. 2023 "Generative Agents" — composite retrieval model (recency × importance × relevance)
- GE-20260630-815259 — cross-repo SPI extends CDI displacement (CbrCaseMemoryStore standalone pattern)
- GE-20260811-aa9a26 — casehub-blocks dependency transitively requires neocortex config properties
- GE-20260717-0489d1 — CbrQuery.of() Path scope parameter (API break awareness)
- Phase 1 spec: D5 (four-layer memory architecture), D8 (direct LLM, no CaseEngine)
- decisions.md: D13–D20 (memory integration decisions), D21–D24 (review-surfaced implicit decisions)
