# Human Feedback Trust Dimensions — Timing Quality and Accuracy

**Issue:** #231
**Branch:** issue-231-human-feedback-trust
**Date:** 2026-09-12
**Depends on:** #181 (commentary infrastructure)

## Problem

Commentary trust scoring has a single auto-evaluated dimension: `response-latency`. Two additional dimensions — timing quality ("was this worth seeing right now?") and accuracy ("was this factually correct?") — require human feedback that the platform can't currently capture. Without these dimensions, trust routing can't distinguish a fast but poorly-timed or factually wrong commentator from a good one.

## Solution

### Per-event feedback via workbench UI (D1, D2)

Each commentary entry in the workbench commentary page gets two button pairs:

| Dimension | Positive | Negative | Trust dimension key |
|-----------|----------|----------|-------------------|
| Timing quality | Good timing | Bad timing | `timing-quality` |
| Accuracy | Accurate | Inaccurate | `accuracy` |

Buttons appear inline on each commentary entry. Clicking a button sends a WebSocket message and disables the pair (one feedback per dimension per event). No feedback = no signal (not counted as positive or negative).

### Commentary feedback pipeline (D3)

Mirrors the existing coaching feedback path:

```
User clicks button in workbench UI
  → WebSocket message: { type: "commentary_feedback", correlationId, dimension, positive }
    → WorkbenchSocket.onMessage() parses and delegates
      → CommentaryFeedbackHandler.recordFeedback(correlationId, dimension, positive)
        → OutcomeRecorder.record(OutcomeRecord) — one record per dimension
```

## Changes Required

### CommentaryFeedbackHandler (new)

**Package:** `io.quarkmind.qa.workbench`
**Annotation:** `@UnlessBuildProfile("prod")` `@ApplicationScoped`

```java
public class CommentaryFeedbackHandler {

    @Inject OutcomeRecorder outcomeRecorder;
    @Inject GameSession gameSession;

    public void recordFeedback(String workerId, String dimension, boolean positive) {
        double score = positive ? 1.0 : 0.0;
        outcomeRecorder.record(new OutcomeRecord(
            workerId,
            gameSession.id(),
            dimension,
            positive ? AttestationVerdict.ENDORSED : AttestationVerdict.CHALLENGED,
            score
        ));
    }
}
```

**Parameters:**
- `workerId` — the commentary worker that produced the event (from `CommentaryCompleted.workerId()`)
- `dimension` — `"timing-quality"` or `"accuracy"`
- `positive` — true for "good timing"/"accurate", false for "bad timing"/"inaccurate"

### WorkbenchSocket

**Updated `onMessage()`** — add `commentary_feedback` handling alongside existing `coaching_response`:

```java
if ("commentary_feedback".equals(type)) {
    String workerId = node.path("workerId").asText(null);
    String dimension = node.path("dimension").asText(null);
    boolean positive = node.path("positive").asBoolean();
    if (workerId != null && dimension != null) {
        feedbackHandler.recordFeedback(workerId, dimension, positive);
    }
}
```

**New injection:** `@Inject CommentaryFeedbackHandler feedbackHandler;`

### WebSocket message format

```json
{
  "type": "commentary_feedback",
  "workerId": "claude:commentary-reactive@v1",
  "dimension": "timing-quality",
  "positive": true
}
```

`workerId` comes from the `CommentaryCompleted` event that produced the commentary entry — already available in the `commentary_snapshot` payload as `workerId`.

### QuarkMindTrustRoutingPolicyProvider

**Updated `buildCommentaryPolicy()`** — add quality floors for the two new dimensions:

```java
private TrustRoutingPolicy buildCommentaryPolicy(int minimumObservations, double latencyFloor) {
    Map<String, Double> qualityFloors = Map.of(
        "response-latency", latencyFloor,
        "timing-quality", 0.3,
        "accuracy", 0.4
    );
    // ... rest unchanged
}
```

`accuracy` has a higher floor (0.4) than `timing-quality` (0.3) — factually wrong commentary is more damaging than poorly timed commentary.

### Workbench UI — commentary page (Lit component)

**File:** `quarkmind-sc2/src/main/webui/workbench/qm-commentary-page.js`

Add feedback buttons to each commentary entry rendered in the list. Each entry already has `workerId`, `gameFrame`, and `commentaryType` from the `commentary_snapshot` payload.

```html
<div class="feedback-buttons">
  <button class="feedback-btn timing good" @click=${() => this._sendFeedback(entry.workerId, 'timing-quality', true)} ?disabled=${entry.timingFeedbackGiven}>👍 Timing</button>
  <button class="feedback-btn timing bad" @click=${() => this._sendFeedback(entry.workerId, 'timing-quality', false)} ?disabled=${entry.timingFeedbackGiven}>👎 Timing</button>
  <button class="feedback-btn accuracy good" @click=${() => this._sendFeedback(entry.workerId, 'accuracy', true)} ?disabled=${entry.accuracyFeedbackGiven}>✓ Accurate</button>
  <button class="feedback-btn accuracy bad" @click=${() => this._sendFeedback(entry.workerId, 'accuracy', false)} ?disabled=${entry.accuracyFeedbackGiven}>✗ Wrong</button>
</div>
```

`_sendFeedback()` sends the WebSocket message and sets the `*FeedbackGiven` flag to disable the pair.

## Testing Strategy

### Unit tests (plain JUnit)

**CommentaryFeedbackHandlerTest:**
- `recordFeedback` with positive=true → OutcomeRecord with ENDORSED verdict, confidence 1.0
- `recordFeedback` with positive=false → OutcomeRecord with CHALLENGED verdict, confidence 0.0
- Both dimensions recorded independently for the same workerId

**QuarkMindTrustRoutingPolicyProviderTest updates:**
- Commentary policy includes `timing-quality` and `accuracy` quality floors
- Existing `response-latency` floor unchanged (regression guard)

### Integration tests (@QuarkusTest)

**WorkbenchSocketIT updates:**
- Send `commentary_feedback` WebSocket message → verify OutcomeRecorder receives the record
- Invalid dimension name → no crash, no record

### Playwright tests (@Tag("browser"))

**WorkbenchRenderTest updates (if applicable):**
- Verify feedback buttons render on commentary entries
- Verify buttons disable after click

## Data Flow

```
Game tick → Commentary LLM → CommentaryCompleted event
  │
  ▼
WorkbenchBroadcaster → commentary_snapshot WebSocket event
  │ (includes workerId, gameFrame, commentaryType)
  ▼
Workbench UI renders commentary with feedback buttons
  │
  ▼ (user clicks)
  │
commentary_feedback WebSocket message
  │ { type, workerId, dimension, positive }
  ▼
WorkbenchSocket.onMessage()
  │
  ▼
CommentaryFeedbackHandler.recordFeedback()
  │
  ▼
OutcomeRecorder.record(OutcomeRecord)
  │ capabilityTag: "timing-quality" or "accuracy"
  │ actorId: workerId
  │ subjectId: gameSessionId
  ▼
Trust scoring → affects future worker routing
```

## Known Limitations

1. **No retroactive feedback.** Feedback applies only to the current session's trust scores. Historical commentary (from previous games) cannot be retroactively rated.

2. **Sparse signal.** Users may not rate most commentary entries — especially during active gameplay. The `minimumObservations` threshold (5 for commentary) handles this — trust routing falls back to default weights until enough feedback accumulates.

3. **Binary scoring.** Each feedback is either 1.0 (positive) or 0.0 (negative). A more granular scale (1-5 stars) would provide richer signal but adds UI complexity. The binary approach matches the coaching pattern and is sufficient for initial trust differentiation.

4. **No cross-session persistence of UI state.** The `*FeedbackGiven` flags are in-memory on the Lit component. If the user refreshes the page, buttons reset (but the OutcomeRecords are already persisted server-side). Double-feedback on the same event is possible after refresh — the trust scoring system handles duplicates gracefully (each record is independent).

## References

- `CoachingAcknowledgmentHandler.java` — coaching feedback pattern (template)
- `WorkbenchSocket.java:41-56` — existing WebSocket message handling
- `QuarkMindTrustRoutingPolicyProvider.java:99-114` — commentary trust policy
- `LlmWorkerLatencyRecorder.java` — OutcomeRecord recording pattern
- `OutcomeRecorder` SPI — trust dimension recording API
- `qm-commentary-page.js` — workbench commentary Lit component
- `2026-07-05-milestone-trust-scoring-design.md` — trust scoring architecture
- `2026-06-10-layer6-trust-weighted-strategy-routing-design.md` — trust-weighted routing
- Protocol `trust-routing-config-flags-required.md` — trust routing config convention
- Issue #181 — commentary infrastructure (prerequisite)
