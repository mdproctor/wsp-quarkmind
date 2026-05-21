# Design Journal — issue-146-building-cost-mineral-timing

### 2026-05-21 · §Mock Infrastructure

`EmulatedGame` gains `injectReplayBuildingWithCost(Building)` alongside the existing free `injectReplayBuilding(Building)`. The two methods represent distinct injection semantics: free injection is correct for buildings gifted at game start (the initial Nexus, which no player purchases); cost-deducting injection is the right model for buildings ordered during the game. Minerals may go negative — this is intentional, representing a short-lived mineral debt repaid through income. Clamping at 0 was evaluated and rejected: it masks the debt, causes EM to recover too quickly, and produces worse unit-count divergence than the unfixed baseline.

The harness routing between the two paths was designed but not wired in. Investigation showed that deducting building costs from EM's continuous mineral balance against stale GT PlayerStats readings creates artificial debt that blocks trains EM should not miss, reversing the direction of the divergence. The API is ready for future use when the harness mineral model is more precisely aligned with the real player's per-loop balance.

### 2026-05-21 · §Testing Strategy

Investigation into the residual unit-count divergence in `ReplayValidationHarness` clarified that `firstUnitDivergenceTick = 86` has two independent causes, neither of which is purely mineral-timing. (1) A completion-time formula gap in `startTraining`: `completesAt` rounds to 86 while SC2 places the unit's appearance at tick 87 for the specific loop offset of that train command. This is independent of mineral balance — deducting building costs does not affect it. (2) EmulatedGame has no vespene income model; Stalker and Immortal train commands (50 and 100 gas respectively) are always rejected, causing the harness to fall 1 unit behind GT each time the real player trains a gas unit. Filed as #148.

The validation test docstring was updated to document both causes explicitly so future contributors do not re-investigate the mineral path as a fix for the timing gap.
