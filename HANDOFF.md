# HANDOFF — quarkmind

## Last Session

Fixed three replay coordinate bugs (#295): game event target points at half tracker event scale (2x fix), moveOrders() checked targetPoint before targetUnit (wrong branch for mining), UnitOrderTracker walked units through walls (terrain collision added with TDD). Switched commentary LLM from OpenAI to platform AgentProvider (Claude/Vertex). Closed #291 as premature. Discovered pipeline gap (#296): cascade classifier returns empty so all workbench tabs show no data.

## Immediate Next Step

Fix #296 — cascade classifier returns empty assessments. DISPATCH_THRESHOLD (0.3) filters everything because Drools evidence never reaches that level for mid-game PvZ. Options: lower threshold, add mid-game rules, or fallback assessment when enemies visible. Unblocks pattern → strategy → summarisation → moments → commentary.

## References

- Branch: `issue-295-replay-coordinate-fix` (pushed, not PR'd)
- Branch: `issue-294-replay-connector-config` (connector dummy config)
- Issues: #294 (connector), #295 (coordinates — closes), #296 (pipeline gap — open)
- Diagnostic test: `WorkbenchPipelineIT.java` — FAILS at layer 2
- Key files: `AbilityMapping.java`, `UnitOrderTracker.java`, `ReplaySimulatedGame.java`, `ReplayEngine.java`
