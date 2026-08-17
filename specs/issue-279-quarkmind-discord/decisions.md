# QuarkMind Discord — Decisions

## D1: Two sub-modules (protocol + agent)

**Choice:** `quarkmind-discord-protocol/` (shared types: DiscordPerception, DiscordIntent) + `quarkmind-discord-agent/` (agency loop, world bridge, Quarkus app). Mirrors QuarkVille's separation pattern.
**Alternatives:**
- Flat single module — simpler, but breaks pattern consistency across quarkmind worlds
**Rationale:** Consistency with QuarkVille's three-module structure (minus the server module Discord doesn't need). If a Discord dashboard or observer tool is added later, it can depend on protocol without pulling in agent dependencies.
**Trade-offs:** Slightly more Maven boilerplate for a module that may never have a third consumer.
**Exploration:** quick
**Status:** captured
