# Decisions — #386 Calibrate EmulatedGame Physics

## D1: Scope — units and economy in parallel

**Choice:** Fix both unit accuracy and economy MAPE in the same issue, working both tracks in parallel
**Alternatives:**
- Unit accuracy first, economy later — lower risk but slower overall progress
- Economy first — harder target, delays quick wins on units
**Rationale:** Both are acceptance criteria; the economy fix is a single measurement bug (high impact, low risk), and unit fixes are independent of it
**Trade-offs:** Larger scope means more to validate per baseline run
**Sources:** EmulatedGameAccuracyBaselineTest baseline data, EconomyTracker source code
**Exploration:** quick
**Status:** captured

## D2: Upgrade accuracy — in scope

**Choice:** Include upgrade accuracy as an explicit target alongside units and economy
**Alternatives:**
- Skip upgrades — not in original acceptance criteria, defer to follow-up
**Rationale:** Upgrade completion logic exists but timing constants are uncalibrated (protocol violation); small fix with potentially large accuracy gain
**Trade-offs:** Adds calibration work for ~80 upgrade types
**Sources:** SC2Data.java upgrade timing constants, PP-20260522-572156 (train time calibration protocol)
**Exploration:** quick
**Status:** captured

## D3: Approach — staged (Phase 1 physics, Phase 2 extraction)

**Choice:** Phase 1: fix EmulatedGame internals (economy tracker, EGG filtering, upgrade calibration, MULE). Re-measure baselines. Phase 2: fix replay extraction pipeline (Queen intents, morph intents) only if unit accuracy still <80%.
**Alternatives:**
- End-to-end in one pass — mixes two problem domains (physics vs extraction)
- EmulatedGame only — risks not hitting ≥80% unit accuracy on this issue
**Rationale:** Clean separation of concerns with measurable progress gates; Phase 1 fixes are all within EmulatedGame, Phase 2 is extraction pipeline
**Trade-offs:** May need two rounds of baseline measurement; Phase 2 may not be needed if EGG filtering alone pushes units over 80%
**Sources:** Fork investigation: Queen is extraction (17.9%), economy is measurement bug (24× gap), upgrades are uncalibrated
**Exploration:** quick
**Status:** captured

## D4: Economy root cause — measurement bug, not physics

**Choice:** Fix EconomyTracker to track gross mineral/gas income (minerals mined) instead of net balance delta (income minus spending). Investigate SC2 PlayerStats formula from oracle replays.
**Alternatives:**
- Tune the income model constants — wrong fix; the income model is fine, the measurement is wrong
**Rationale:** SC2's `mineralsCollectionRate` = gross collection; EmulatedGame's rate = delta of mineral balance over 7 ticks. Heavy spending makes net rate ~0 while gross collection continues. This explains the 24× gap entirely.
**Trade-offs:** Need to verify against oracle replays that gross income tracking produces MAPE ≤50%
**Sources:** EconomyTracker source code analysis, oracle replay PlayerStats data
**Exploration:** quick
**Status:** captured
