# QuarkMind Handover — 2026-07-20

## Last Session

Implemented coach mode (#230) end-to-end: 14 new files in `plugin/coaching/`, game loop integration, plugin gating, trust routing, integration tests. Discovered CDI `@ApplicationScoped` proxy gotcha with shared `ConcurrentHashMap` — garden entry submitted (GE-20260719-f5ccc9). Landed as fcc4f1f on main. #253 (integration tests) folded into the same branch and closed.

## Immediate Next Step

Pick new work. Run `/work` from main.

## Cross-Module

**engine#648** (AttestingOutcomeRecorder) — prerequisite for activating milestone evaluation. QuarkMind infrastructure ready; activates automatically when the SPI lands.

**neocortex#162** (InMemoryCbrCaseMemoryStore.retrieveSimilar() empty) — CBR retention IT cannot verify end-to-end store+retrieve. Unit tests pass with mocked store.

## What's Left

- #74 — genericise unit/building definitions · L · Med
- #233 — consolidate remaining SC2Data switch methods · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #235 | LLM fallback for novel/ambiguous builds | M | Med | CBR infrastructure in place |
| #243 | Full tactic taxonomy (all game phases) | L | High | Epic #252; external data sources identified |
| #250 | Coaching compliance evolution | L | High | Epic; position-based + LLM-evaluated verification |
| #251 | Coaching personality model | M | Med | Epic; adaptive intra-game disposition |
| #231 | Human feedback trust dimensions | M | Med | Requires visualizer UI |
