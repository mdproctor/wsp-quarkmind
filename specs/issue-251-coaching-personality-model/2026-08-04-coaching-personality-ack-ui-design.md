# Coaching Personality Model & Acknowledgment UI

**Issues:** #251 (epic), #246, #247, #248
**Branch:** `issue-251-coaching-personality-model`
**Date:** 2026-08-04

## Scope

Four issues, two decoupled concerns:

- **Personality model** (#246, #247): expand from two fixed coaching personalities to a 2-axis disposition model with urgency-driven intra-game switching
- **Acknowledgment UI** (#248): add Accept/Dismiss buttons to the workbench coaching page, wired to qhorus commitment lifecycle via direct MessageService dispatch

Both systems are independent — the personality model works from auto-compliance and urgency rules; the acknowledgment UI writes to the audit trail but does not influence personality selection. A trust learning bridge (per-personality outcome aggregation → selection policy) is future work, not part of this epic.

**Data model preparation:** `OpenCommitment` gains an `agentId` field so that future trust learning has personality attribution from day one.

## Build Order (Bottom-Up)

1. Data model enrichment (foundation for everything else)
2. 2-axis disposition model (#247)
3. Urgency-aware prompt switching (#246)
4. Acknowledgment UI (#248)

---

## 1. Data Model Enrichment

### OpenCommitment — add personality attribution

Current: `OpenCommitment(String correlationId, CoachingAdvice advice, long issuedAtFrame)`

New: `OpenCommitment(String correlationId, String agentId, CoachingAdvice advice, long issuedAtFrame)`

The `agentId` is available in `CoachingCompleted.workerId()` but currently dropped when `CoachingChannelBroker.onCoachingCompleted()` creates the commitment. Pass it through.

**Downstream:**
- `CoachingEffectivenessTrustRecorder.record()` — add `agentId` parameter: `record(String correlationId, String agentId, String outcome, CoachingAdvice advice)`. Still a logging stub, now logs which personality produced the outcome.
- `CoachingComplianceEvaluator.evaluate()` — pass `commitment.agentId()` to recorder.
- `CoachingComplianceEvaluator.withdrawAll()` — also calls `recorder.record()`: pass `commitment.agentId()` here too.

### CoachingComplianceResolved — add correlationId

Current: `CoachingComplianceResolved(long gameFrame, CoachingDomain domain, String status)`

New: `CoachingComplianceResolved(long gameFrame, CoachingDomain domain, String status, String correlationId)`

The UI currently matches compliance updates by `(gameFrame, domain)` which is fragile — two advice items for the same domain at the same frame would collide. Adding `correlationId` gives the UI a precise handle. `WorkbenchEnricher.onCoachingCompliance()` passes `correlationId` through to `CoachingCompliancePayload`, and visualizer.js matches by `correlationId` instead of `(gameFrame, domain)`.

### CoachingAdvicePublished — add correlationId

Current: `CoachingAdvicePublished(CoachingAdvice advice, CoachingUrgencyTier urgencyTier, long gameFrame)`

New: `CoachingAdvicePublished(CoachingAdvice advice, CoachingUrgencyTier urgencyTier, long gameFrame, String correlationId)`

The correlationId is assigned in `CoachingChannelBroker.onCoachingCompleted()` before the event fires. Explicit pass-through chain: `CoachingChannelBroker` fires `CoachingAdvicePublished(correlationId)` → `WorkbenchEnricher.onCoachingAdvice()` reads `event.correlationId()` and passes to `CoachingPayload(correlationId)` → `WorkbenchBroadcaster.broadcast()` serializes to JSON → visualizer.js stores `correlationId` per coaching item for acknowledgment responses.

### CoachingPayload — add correlationId

Current: `CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame)`

New: `CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame, String correlationId)`

---

## 2. Two-Axis Disposition Model (#247)

### Axis mapping to Eidos foundation

| Coaching axis | Eidos `DispositionAxis` | Values | Controls |
|---|---|---|---|
| Urgency response | `RISK_APPETITE` | `bold` / `cautious` | How aggressively the coach reacts to game pressure |
| Directness | `SOCIAL_ORIENTATION` | `assertive` / `collaborative` | Whether the coach tells or asks |

No new foundation types needed.

### Four derived quadrants

| | Assertive (tell) | Collaborative (ask) |
|---|---|---|
| **Bold** | **Commander** — "Build 3 Stalkers now and move to their natural." | **Rally** — "Your base is under pressure — what could you warp in to defend?" |
| **Cautious** | **Instructor** — "You should start +1 weapons from the Forge." | **Mentor** — "What tech path do you think would counter those Roaches?" |

### CoachingStyle enum (new, derived at runtime)

```java
public enum CoachingStyle {
    COMMANDER,   // bold + assertive
    RALLY,       // bold + collaborative
    INSTRUCTOR,  // cautious + assertive
    MENTOR;      // cautious + collaborative

    public static CoachingStyle resolve(AgentDisposition disposition,
                                         CoachingUrgencyTier tier) {
        boolean bold = tier == CRISIS
                    || tier == STRATEGIC
                    || (disposition != null
                        && "bold".equals(disposition.primaryTerm(RISK_APPETITE)));
        boolean assertive = tier == CRISIS
                         || (disposition != null
                             && "assertive".equals(disposition.primaryTerm(SOCIAL_ORIENTATION)));

        if (bold && assertive) return COMMANDER;
        if (bold)              return RALLY;
        if (assertive)         return INSTRUCTOR;
        return MENTOR;  // default when disposition is null or axes unset
    }
}
```

### Override rules (from #246)

- **CRISIS** → force both axes: bold + assertive → Commander. No questions when the base is burning.
- **STRATEGIC** → force RISK_APPETITE to bold, keep configured SOCIAL_ORIENTATION. A Socratic coach stays Socratic during military moments but with more urgency.
- **ECONOMIC** → use agent's configured defaults for both axes.

Example for a Mentor agent (cautious + collaborative):

| Urgency tier | Effective style | Behaviour |
|---|---|---|
| CRISIS | Commander | Direct orders, imperative voice |
| STRATEGIC | Rally | Urgent but still asks guiding questions |
| ECONOMIC | Mentor | Calm, discovery-oriented questions |

A Commander agent (bold + assertive) stays Commander in all tiers.

### CoachingDispositionTerm — unchanged

DIRECTIVE and SOCRATIC remain as vocabulary terms for registered agent identities. Two registered agents suffice — the 4 quadrants are **not** 4 agents. They are runtime-derived styles computed from `AgentDescriptor.disposition()` + current urgency tier. A single agent traverses different quadrants as urgency changes within a game.

An agent registered as "directive" → `RISK_APPETITE=bold, SOCIAL_ORIENTATION=assertive` in its descriptor YAML. Default quadrant: Commander. Under ECONOMIC tier: stays Commander.
An agent registered as "socratic" → `RISK_APPETITE=cautious, SOCIAL_ORIENTATION=collaborative`. Default quadrant: Mentor. Under CRISIS: becomes Commander. Under STRATEGIC: becomes Rally.

---

## 3. Urgency-Aware Prompt Switching (#246)

### buildSystemPrompt signature change

Current: `buildSystemPrompt(AgentDescriptor descriptor, boolean crisisOverride)`

New: `buildSystemPrompt(AgentDescriptor descriptor, CoachingUrgencyTier tier)`

Internally calls `CoachingStyle.resolve(descriptor.disposition(), tier)` and selects the matching prompt template.

### Prompt templates (one per quadrant)

Each quadrant gets a distinct style instruction and example in the system prompt. The style block replaces the current binary directive/Socratic section in `buildSystemPrompt()`.

### CoachingSessionSelector — minimal change

The selector still caches one agent per game. The `select(tier)` parameter remains accepted but unused — personality switching happens at the prompt level in `buildSystemPrompt()`, not at the agent selection level.

### executeCoaching — pass tier instead of boolean

Current: `boolean isCrisis = isCrisisTrigger(input);` → `buildSystemPrompt(descriptor, isCrisis)`

New: `CoachingUrgencyTier tier = resolveUrgencyTier(input);` → `buildSystemPrompt(descriptor, tier)`

The `resolveUrgencyTier()` method already exists and returns the correct tier from the input map.

---

## 4. Acknowledgment UI (#248)

### Architecture

```
visualizer.js  ──WebSocket──>  WorkbenchSocket  ──>  CoachingAcknowledgmentHandler
                                (@OnTextMessage)           │
                                                           ├─> MessageService.dispatch(DONE/DECLINE)
                                                           │   (qhorus audit trail, correct correlationId)
                                                           ├─> CoachingComplianceEvaluator.resolveHuman()
                                                           │   (removes commitment, records outcome)
                                                           └─> WorkbenchBroadcaster.broadcast()
                                                               (coaching_compliance event → UI update)
```

### CoachingAcknowledgmentHandler (new CDI bean)

```java
// Package: io.quarkmind.qa.workbench (workbench concern, not core coaching)
@UnlessBuildProfile("prod")
@ApplicationScoped
public class CoachingAcknowledgmentHandler {
    // Injected: MessageService, CoachingChannelBroker, CoachingComplianceEvaluator,
    //           WorkbenchBroadcaster

    public boolean acknowledge(String correlationId, boolean accepted) {
        // 1. Resolve commitment FIRST (human-wins override)
        //    - Check commitment exists before any side effects
        //    - DONE → records ENDORSED via trust recorder
        //    - DECLINE → records CHALLENGED via trust recorder
        //    - Returns false if correlationId not found (already resolved by auto-compliance or superseded)
        //    - If false: return immediately — no dispatch, no broadcast
        //
        // 2. Dispatch response via MessageService (bypasses receiveHumanMessage per GE-20260517-5879a9)
        //    - type: accepted ? MessageType.DONE : MessageType.DECLINE
        //    - correlationId: the advice's correlationId
        //    - inReplyTo: correlationId (per GE-20260530-1a7e84)
        //    - actorType: ActorType.HUMAN
        //    - channelId: broker.channelId()
        //    - Per GE-20260605-73c9d6: DECLINE produces CommitmentState.DECLINED
        //
        // 3. Broadcast coaching_compliance event to workbench UI
        //
        // Order matters: resolve first ensures the commitment is claimed before
        // MessageService dispatch (which could fail). If dispatch fails after
        // resolution, the compliance outcome is still recorded — the audit trail
        // gap is acceptable for a dev-only tool.
    }
}
```

### CoachingComplianceEvaluator.resolveHuman() (new method)

```java
public boolean resolveHuman(String correlationId, boolean accepted) {
    // Iterate commitments (max 4 entries — one per CoachingDomain)
    // Find entry where commitment.correlationId().equals(correlationId)
    // Use ConcurrentHashMap.remove(key, value) for atomic removal — prevents
    // race with evaluate() where both threads find the same commitment
    // Record: accepted ? "ENDORSED" : "CHALLENGED" (with agentId)
    // Fire CoachingComplianceResolved with (commitment.issuedAtFrame(), domain, status, correlationId)
    // Returns false if not found or if remove(key, value) lost the CAS race
}
```

### WorkbenchSocket — add @OnTextMessage

```java
@OnTextMessage
public void onMessage(String message, WebSocketConnection connection) {
    // Parse JSON: {type: "coaching_response", correlationId: "...", response: "DONE"|"DECLINE"}
    // Validate: type must be "coaching_response", correlationId must be non-null/non-blank,
    //           response must be "DONE" or "DECLINE". Ignore malformed messages (log.debug).
    // Delegate to CoachingAcknowledgmentHandler.acknowledge()
    // Future message types can be added via the type discriminator without changing the handler.
}
```

### Visualizer.js coaching page changes

- Each advice item renders ✓ Accept and ✗ Dismiss buttons while status is "⏳ Pending"
- Clicking sends `{type: "coaching_response", correlationId: "...", response: "DONE"|"DECLINE"}` via the existing WebSocket connection
- After click or auto-compliance resolution (whichever comes first), buttons disappear and the status badge shows (✅ Endorsed / ❌ Challenged / ⏸ Neutral)
- Race handling: `coaching_compliance` event from server always wins — both human-initiated and auto-initiated compliance events use the same rendering path
- Supersession handling: when `CoachingChannelBroker` replaces a commitment for the same domain (new advice supersedes old), broadcast a `coaching_compliance` event for the old correlationId with status `SUPERSEDED`. The UI hides buttons for the superseded advice. Without this, buttons for old advice remain visible but clicking them returns false (commitment gone)

### Qhorus garden gotcha checklist

| Garden entry | How addressed |
|---|---|
| GE-20260517-5879a9: `receiveHumanMessage()` drops correlationId | Bypass entirely — use `MessageService.dispatch()` directly |
| GE-20260530-1a7e84: response messages require inReplyTo + correlationId | Set both fields on `MessageDispatch.builder()` |
| GE-20260726-3c0da2: COMMAND requires explicit correlationId | Already handled — broker assigns UUID at dispatch time |
| GE-20260605-73c9d6: DECLINE → CommitmentState.DECLINED not CANCELLED | Use `MessageType.DECLINE` (correct speech act) |
| GE-20260529-e32a4d: MessageDispatch requires actorType | Set `actorType(ActorType.HUMAN)` for human responses |

---

## Testing Strategy

### Unit tests (plain JUnit)

| Test | Verifies |
|---|---|
| `CoachingStyleTest` (new) | `resolve()` across 12 combinations: 4 quadrants × 3 urgency tiers. Crisis forces Commander, strategic forces bold, economic uses defaults. |
| `CoachingWorkerFactoryTest` (update) | `buildSystemPrompt(descriptor, tier)` — each quadrant produces correct style instructions. Replace boolean crisisOverride tests with CoachingUrgencyTier tests. |
| `CoachingComplianceEvaluatorTest` (update) | `resolveHuman()`: DONE → ENDORSED + removal, DECLINE → CHALLENGED + removal, unknown correlationId → false. CAS race with `evaluate()` → loser returns false. Verify agentId passed to recorder. Verify `withdrawAll()` passes agentId. |
| `CoachingAcknowledgmentHandlerTest` (new) | DONE dispatches MessageType.DONE with correct correlationId/inReplyTo/actorType, records ENDORSED, removes commitment. DECLINE dispatches DECLINE, records CHALLENGED. Race returns false — no dispatch or broadcast. Resolve-before-dispatch ordering verified. |
| `CoachingChannelBrokerTest` (update) | agentId from CoachingCompleted.workerId() flows into OpenCommitment. correlationId flows into CoachingAdvicePublished. Supersession: replacing commitment for same domain broadcasts SUPERSEDED for old correlationId. |
| `WorkbenchEventTest` (update) | CoachingPayload serialization includes correlationId. CoachingCompliancePayload includes correlationId. |

### Integration tests (@QuarkusTest)

| Test | Verifies |
|---|---|
| `CoachingIntegrationIT` (update) | Full cycle: trigger → worker → channel dispatch with agentId → compliance with attribution. |
| `WorkbenchSocketIT` (update) | Bidirectional: client sends coaching_response, receives coaching_compliance. |

### Playwright render tests (@Tag("browser"))

| Test | Verifies |
|---|---|
| `WorkbenchRenderTest` (update) | Coaching page renders Accept/Dismiss buttons on pending advice. Click Accept → Endorsed badge. Click Dismiss → Challenged badge. Auto-compliance hides buttons. |

### Existing tests — signature updates only

- `CoachingSessionSelectorTest` — no changes
- `CoachingTriggerBuilderTest` — no changes
- `CoachingDispositionTermTest` — no changes

---

## Design Decisions

| Decision | Rationale |
|---|---|
| Decoupled personality model and acknowledgment UI | Neither needs the other to function. Trust learning bridge is future work — the trust recorder is a stub, OpenCommitment didn't track personality. |
| Data model prep (agentId on OpenCommitment) | One field addition enables future per-personality learning without backfill. |
| Prompt-level override, not agent-level switch | Simpler — one agent per session, prompt adapts per urgency tier. Crisis override already works this way. |
| Reuse existing Eidos axes | RISK_APPETITE and SOCIAL_ORIENTATION map naturally to urgency response and directness. No foundation changes needed. |
| Direct MessageService dispatch for human responses | Bypasses receiveHumanMessage() which drops correlationId (GE-20260517-5879a9). Uses proven qhorus APIs. |
| Human-wins override | If the human responded, their intent is ground truth. DONE → ENDORSED, DECLINE → CHALLENGED, commitment removed from auto-evaluator. |
| CoachingAcknowledgmentHandler as separate CDI bean | Separates domain logic from WebSocket transport. Testable without a WebSocket. @UnlessBuildProfile("prod") — QA/dev only. |

## Garden Entries Referenced

- GE-20260517-5879a9 — receiveHumanMessage() correlationId=null
- GE-20260726-3c0da2 — COMMAND requires explicit correlationId
- GE-20260530-1a7e84 — response messages require inReplyTo + correlationId
- GE-20260605-73c9d6 — CommitmentState.DECLINED not CANCELLED
- GE-20260529-e32a4d — MessageDispatch requires actorType
- GE-20260803-c02ab3 — DispositionConfig rejects dominantFunction/auxiliaryFunction (relevant if agent YAML uses these fields)
