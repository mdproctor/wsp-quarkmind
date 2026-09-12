## D1: Feedback granularity

**Choice:** Per-event — each commentary entry gets feedback buttons
**Alternatives:**
- Per-session aggregate — simpler UI but loses per-event attribution
- Both (per-event + session-end summary) — most data but most UI complexity
**Rationale:** Per-event feedback enables attribution to specific workers/triggers. Follows the coaching pattern (per-advice accept/decline). Required for meaningful trust dimension scoring.
**Trade-offs:** More UI elements per commentary entry. User may ignore buttons if commentary is frequent.
**Sources:** CoachingAcknowledgmentHandler.java (per-event coaching pattern), WorkbenchSocket.java
**Exploration:** quick
**Status:** captured

## D2: Feedback signals

**Choice:** Two separate button pairs — Good Timing / Bad Timing + Accurate / Inaccurate
**Alternatives:**
- Single thumbs up/down — conflates timing and accuracy dimensions
- Three buttons (Skip/Wrong/Good) — implicit positive, less clear dimension mapping
**Rationale:** Maps 1:1 to the two trust dimensions. User can mark timing bad but content accurate (or vice versa). Clean attribution per dimension.
**Trade-offs:** Four buttons per commentary entry — more visual weight than a single thumbs up/down.
**Sources:** Issue #231 (proposed dimensions), QuarkMindTrustRoutingPolicyProvider.java (quality floors)
**Exploration:** quick
**Depends on:** D1 (per-event granularity)
**Status:** captured

## D3: Pipeline architecture

**Choice:** Mirror coaching pattern — dedicated CommentaryFeedbackHandler + commentary_feedback WebSocket message type
**Alternatives:**
- Inline in WorkbenchSocket — less code but mixes concerns, harder to test
- Generic feedback handler — over-engineered for two dimensions (YAGNI)
**Rationale:** Follows the established CoachingAcknowledgmentHandler pattern. Handler is a separate CDI bean, testable in isolation. WebSocket just parses and delegates.
**Trade-offs:** One additional class (CommentaryFeedbackHandler), but minimal — pattern well-established.
**Sources:** CoachingAcknowledgmentHandler.java, WorkbenchSocket.java, OutcomeRecorder SPI
**Exploration:** quick
**Depends on:** D1 (per-event), D2 (two dimensions)
**Status:** captured
