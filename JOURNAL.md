# Design Journal — issue-279-quarkmind-discord

## 2026-08-18 — ChatPerceptionBridge placement (deviation from plan)

Plan placed `ChatPerceptionBridge` in quarkmind-core (`io.quarkmind.agency.chat`). During implementation, discovered this creates a circular dependency: the interface references `ChatPerception` from quarkmind-chat-protocol, which depends on quarkmind-core.

**Resolution:** Moved `ChatPerceptionBridge` and `DefaultChatPerceptionBridge` to quarkmind-chat-agent (`io.quarkmind.chat.agent`). The agent module already depends on both core and protocol, so no new dependencies added. `ChatObservationRenderer` stayed in quarkmind-core — it only references `ChatDeltaReport` which is a core type.

**Impact:** D3 (two-tier layering) holds — quarkmind-core remains free of protocol-module dependencies. The bridge is an application-tier integration concern, not a core SPI.

## 2026-08-18 — ChatObservationRenderer per-channel compression

Plan's `renderDelta()` applied ambient compression per-thread. Standalone messages (no `parentRef`) each create their own `ConversationThread` of size 1, defeating the threshold. Fixed to collect all ambient messages per-channel before applying the verbatim threshold. This aligns with the D2 design intent — compression operates at the channel level, not the thread level.
