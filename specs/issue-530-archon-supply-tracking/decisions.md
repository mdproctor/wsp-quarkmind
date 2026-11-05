## D1: Sequencing — #533 diagnostic before #523 building lifecycle

**Choice:** #533 first — diagnostic reveals overcount sources, then #523 design uses findings
**Alternatives:**
- Design both together — faster if overcount sources are predictable, but designing #523 on hypotheses is risky for L/High work
- #533 only this session — conservative but may be too slow given both issues are under the same epic
**Rationale:** #533's findings directly inform #523's spending accounting. Designing the building lifecycle without knowing what's actually wrong risks building the wrong fix.
**Trade-offs:** Sequential means #523 design waits, but the diagnostic is S/Med and fast to implement.
**Sources:** standalone-emulated-game-baseline.md (5-min MAPE 23.9%), issue #533, issue #523
**Exploration:** quick
**Status:** captured

## D2: Diagnostic instrumentation — per-source counters in EconomyTracker

**Choice:** Add package-private per-source counters in EconomyTracker that break down technology spending by source (build, research, morph, destruction) and by building type
**Alternatives:**
- External reconstruction from building/upgrade lists — no code changes but can't detect internal double-counting paths
- Full event trace log — maximum visibility but memory overhead for 236 player runs
**Rationale:** Per-source counters give direct visibility into internal accounting. Package-private means zero production impact. They answer "which building types and sources contribute how much" directly.
**Trade-offs:** Adds ~30 lines to EconomyTracker. External reconstruction will be added as a cross-check (zero-cost complement).
**Sources:** EconomyTracker.java, StandaloneEmulatedGameBaselineTest.java, issue #533
**Exploration:** quick
**Status:** captured
**Depends on:** D1
