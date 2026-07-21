# QuarkMind Handover — 2026-07-21

## Last Session

Closed two trailing items from #244's position compliance work. ObservationTranslator now extracts neutral units (watchtowers, destructibles, geysers, minerals) from real SC2 observations instead of passing `List.of()`. IEM10JsonSimulatedGame populates minerals/geysers for neutral units, matching ReplaySimulatedGame. ExpansionLocationCalibrationTest validates CLUSTER_RADIUS=12.0 across 59 replays (12-20 expansions per map). Also fixed upstream Uni retirement compilation errors. Landed as 284a8c9 on main.

## Immediate Next Step

Pick new work from epic #250 backlog. Run `/work` from main. Next child: #245 (LLM-evaluated compliance) or #248 (coaching acknowledgment UI).

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #233 — consolidate remaining SC2Data switch methods · M · Low
- MapInfo construction in SC2BotAgent.onStep() — extractNeutralFeatures() and resource extraction now available; needs first-frame caching lifecycle · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #245 | LLM-evaluated compliance verification | M | Med | New VerificationPredicate permit — infrastructure ready |
| #248 | Coaching acknowledgment UI (DONE/DECLINE) | M | Med | Visualizer overlay work |
| #251 | Coaching personality model | M | Med | Epic; adaptive intra-game disposition |
| #235 | LLM fallback for novel/ambiguous builds | M | Med | CBR infrastructure in place |
| #243 | Full tactic taxonomy (all game phases) | L | High | Epic #252 |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
