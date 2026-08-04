# LLM Fallback for Ambiguous Builds — Design Spec

**Issue:** #235
**Date:** 2026-08-04
**Branch:** `issue-235-llm-fallback-ambiguous-builds`

## Problem

When the Drools CEP pattern classifier (`PatternClassificationRuleUnit`) cannot reach sufficient confidence (< 0.5) for any `StrategyArchetype` after the early detection window, the system has no fallback. Novel, ambiguous, or hybrid builds that don't match hand-authored or data-driven Drools rules go unclassified indefinitely. Downstream consumers (strategy, tactics, advisory) receive no archetype signal and operate blind.

## Solution

A standalone CaseHub capability (`scouting-llm-fallback`) that fires when Drools confidence is low. An LLM receives the chronological unit observation timeline and Drools' current confidence scores, and produces a `PatternAssessment` (archetype + confidence + rationale) that overrides the cumulative confidence map and flows through the existing `publishIntel()` dual-stack path.

## Architecture

### Approach

New CaseHub capability with dedicated worker — follows the existing advisory pattern. DroolsScoutingTask sets a trigger key; CaseHub orchestrates the LLM call via ContextChangeTrigger; the result is read back on the next tick and integrated into the cumulative confidence map.

**Why not inline in DroolsScoutingTask:** Couples scouting to LLM, bypasses CaseHub orchestration, violates plugin seam pattern.
**Why not CDI event observer:** Runs outside CaseHub lifecycle, async timing is fragile, requires mutable state sharing across beans.

### Component Overview

```
DroolsScoutingTask                 QuarkMindCaseHub                    LlmPatternClassifierWorkerFactory
  │                                  │                                   │
  │ all confidences < 0.5            │                                   │
  │ after min game time              │                                   │
  │ cooldown expired                 │                                   │
  │                                  │                                   │
  ├─ set trigger key ──────────────> │                                   │
  │  game.scouting.llm-fallback      │                                   │
  │  .trigger                        ├─ ContextChangeTrigger fires ────> │
  │                                  │                                   │
  │                                  │                                   ├─ build prompt
  │                                  │                                   │  (unit timeline +
  │                                  │                                   │   Drools confidences +
  │                                  │                                   │   race-filtered archetypes)
  │                                  │                                   │
  │                                  │                                   ├─ ChatModel.chat()
  │                                  │                                   │
  │                                  │                                   ├─ parse response
  │                                  │                                   │  (archetype + confidence
  │                                  │                                   │   + rationale)
  │                                  │                                   │
  │                                  │  <── write CaseContext keys ──────┤
  │                                  │      agent.scouting.llm-fallback  │
  │                                  │      .archetype / .confidence     │
  │                                  │      / .rationale                 │
  │                                  │                                   │
  │                                  │  <── fire AdvisoryCompleted ──────┤
  │                                  │      (advisorId =                 │
  │                                  │       llm-classifier:             │
  │                                  │       pattern-fallback)           │
  │                                  │                                   │
  │ [next tick]                      │                                   │
  ├─ read llm-fallback keys          │                                   │
  ├─ override cumulative confidence  │                                   │
  ├─ publishIntel(PatternAssessment) │                                   │
  └─ clear CaseContext key           │                                   │
```

## Section 1: Trigger Detection

**Where:** `DroolsScoutingTask.execute()`, after the pattern classification block.

**Three gates — all must pass:**

1. **Confidence gate:** Every entry in `cumulativeConfidence` has value < configurable threshold (default 0.5). An empty map also passes (no archetypes detected at all).
2. **Time gate:** Current frame exceeds `scouting.llm-fallback.min-game-time-frames` (default 2160 = 3 min at 12 FPS).
3. **Cooldown gate:** `frame - lastLlmFallbackFrame >= cooldownFrames` (default 500 = ~42s).

**Trigger key written to CaseContext:**

Key: `game.scouting.llm-fallback.trigger`

Value: map containing:
- `gameFrame` — monotonic value ensuring `ContextChangeTrigger` fires each time
- `cumulativeConfidences` — current Drools confidence map (archetype name → score), serialised
- `unitTimeline` — `EnemyUnitFirstSeen` records from `ScoutingSessionManager` (unit type + timestamp)
- `enemyRace` — detected enemy race (constrains archetype selection in the prompt)

**New instance state in DroolsScoutingTask:**
- `long lastLlmFallbackFrame = -1` — cooldown tracking
- `double llmFallbackConfidenceThreshold` — loaded from preferences
- `long llmFallbackMinGameTimeFrames` — loaded from preferences
- `long llmFallbackCooldownFrames` — loaded from preferences
- `boolean llmFallbackEnabled` — loaded from preferences
- `String lastProcessedLlmArchetype = null` — prevents re-processing same result

**Reset on frame regression** (game restart): `lastLlmFallbackFrame = -1`, `lastProcessedLlmArchetype = null` — added to the existing reset block.

## Section 2: CaseHub Wiring

**Where:** `QuarkMindCaseHub` — new `wireLlmFallback()` method called from `getDefinition()`.

**New constants:**
```java
static final String CAPABILITY_SCOUTING_LLM_FALLBACK = "scouting-llm-fallback";
static final String LLM_FALLBACK_TRIGGER = "game.scouting.llm-fallback.trigger";
```

**`wireLlmFallback()` method:**
1. Guards on `chatModelInstance.isResolvable()` — returns 0 if no ChatModel.
2. Creates `Capability` named `scouting-llm-fallback`.
3. Creates `Binding` with `ContextChangeTrigger(LLM_FALLBACK_TRIGGER)`.
4. Creates a single `Worker` via `LlmPatternClassifierWorkerFactory.createWorker(chatModel, completionCallback)`.
5. Completion callback fires `AdvisoryCompleted` with `advisorId = "llm-classifier:pattern-fallback"` and `capability = "scouting-llm-fallback"`.

**Reuses existing audit/evaluation infrastructure:**
- `AdvisoryChannelBroker` observes `AdvisoryCompleted` — the LLM classification gets the same audit trail.
- `DeferredAdvisoryEvaluator` observes `AdvisoryCompleted` — outcome scoring works unchanged.
- No new CDI event types needed.

## Section 3: LLM Worker

**Where:** New class `LlmPatternClassifierWorkerFactory` in `io.quarkmind.plugin.scouting`.

**Package rationale:** Classification is scouting's domain. Advisory workers advise on what to do; this worker classifies what the enemy is doing.

**Static factory — plain Java, no CDI:**

```java
public final class LlmPatternClassifierWorkerFactory {
    public static Worker createWorker(ChatModel chatModel, CompletionCallback onCompletion)
}
```

### Prompt Structure

**System prompt:**
```
You are a StarCraft II strategy classifier. Given a sequence of enemy unit
observations and their timestamps, classify the enemy's strategy.

The enemy race is: {race}

Valid archetypes for this race:
{StrategyArchetype values filtered by race, one per line}

A rule-based classifier attempted classification but could not reach
sufficient confidence. Its current scores:
{archetype: score entries from cumulativeConfidences, sorted descending}

Respond with exactly:
ARCHETYPE: <one archetype from the list above>
CONFIDENCE: <0.0 to 1.0>
RATIONALE: <one sentence explaining why>
```

**User message:**
```
Enemy unit observation timeline (chronological):
{timestamp_seconds}s — {unitType}
{timestamp_seconds}s — {unitType}
...

Current game time: {minutes}:{seconds}
Classify this build.
```

### Response Parsing

Uses `extractSection()` (same pattern as `AdvisoryWorkerFactory`):
- `ARCHETYPE` → validated via `StrategyArchetype.valueOf()`. Invalid value → log warning, return empty `WorkerResult` (no classification).
- `CONFIDENCE` → parsed as double. Invalid → default 0.6 (the LLM was asked because Drools failed; a moderate default is reasonable).
- `RATIONALE` → free text, stored as-is.

### Output Keys

```
agent.scouting.llm-fallback.archetype  — StrategyArchetype name
agent.scouting.llm-fallback.confidence — double as string
agent.scouting.llm-fallback.rationale  — explanation text
```

## Section 4: Result Integration

**Where:** `DroolsScoutingTask.execute()`, after the pattern classification block and after the trigger detection.

**Read-on-next-tick pattern:** The LLM worker writes to CaseContext; `DroolsScoutingTask` reads on the next tick. This avoids mutable state sharing between beans.

**Integration logic:**

1. Read `agent.scouting.llm-fallback.archetype` from CaseContext.
2. If present and different from `lastProcessedLlmArchetype`:
   - Parse via `StrategyArchetype.valueOf()`
   - Read confidence from `agent.scouting.llm-fallback.confidence`
   - **Override** cumulative: `cumulativeConfidence.put(archetype, confidence)`
   - Build `PatternAssessment` with the LLM's rationale
   - Publish via `publishIntel(new PatternAssessmentPayload(List.of(assessment)))` — dual-stack path
   - Set `lastProcessedLlmArchetype = archetypeName`
   - Clear the CaseContext keys to prevent re-processing
3. After override, Drools' normal decay (`DECAY_PER_FRAME = 0.99948`) applies on subsequent ticks. If new evidence contradicts the LLM, cumulative logic naturally erodes the score.

**New `requires()` entry:** `QuarkMindCaseFile.LLM_FALLBACK_ARCHETYPE` added to `DroolsScoutingTask.requires()`.

**New CaseFile key constants in `QuarkMindCaseFile`:**
```java
String LLM_FALLBACK_TRIGGER    = "game.scouting.llm-fallback.trigger";
String LLM_FALLBACK_ARCHETYPE  = "agent.scouting.llm-fallback.archetype";
String LLM_FALLBACK_CONFIDENCE = "agent.scouting.llm-fallback.confidence";
String LLM_FALLBACK_RATIONALE  = "agent.scouting.llm-fallback.rationale";
```

## Section 5: Configuration

Loaded in `DroolsScoutingTask.initThresholds()` via `PreferenceProvider`.

| Preference key | Default | Purpose |
|---|---|---|
| `scouting.llm-fallback.enabled` | `true` | Master toggle (effective only when ChatModel is available) |
| `scouting.llm-fallback.confidence-threshold` | `0.5` | All archetypes must be below this for fallback to trigger |
| `scouting.llm-fallback.min-game-time-frames` | `2160` | Earliest frame fallback can fire (3 min at 12 FPS) |
| `scouting.llm-fallback.cooldown-frames` | `500` | Minimum frames between LLM calls (~42s at 12 FPS) |

No new config files. Same `PreferenceProvider` mechanism as existing scouting thresholds. Refreshed on `refreshThresholds()`.

## Section 6: Testing

### Unit tests (plain JUnit)

**`LlmPatternClassifierWorkerFactoryTest`:**
- System prompt includes race-filtered archetypes and Drools confidence scores
- User message has chronological unit timeline with correct formatting
- Valid archetype parsed correctly
- Invalid archetype returns empty result (graceful fallback)
- Missing/malformed confidence defaults to 0.6
- Worker has correct name and capability name

**`DroolsScoutingTaskTest` additions:**
- Trigger fires when all confidences < 0.5 after min game time with cooldown expired
- No trigger when any archetype >= 0.5
- No trigger before min game time
- No trigger during cooldown window
- Reads LLM result from CaseContext and overrides cumulative confidence
- Publishes `PatternAssessmentPayload` after LLM override
- Clears CaseContext key after processing
- LLM-injected confidence decays normally on subsequent ticks
- Reset on frame regression clears LLM state

### Integration test (`@QuarkusTest`)

**`LlmPatternFallbackIT`:**
- Capability wired when ChatModel available
- Capability omitted gracefully when no ChatModel
- Full trigger → worker → result integration → publishIntel cycle
- `AdvisoryCompleted` fires with `advisorId = "llm-classifier:pattern-fallback"`
- Assessment appears in `ScoutingIntelBroker.current(PATTERN_ASSESSMENT)`

## Files Changed

| File | Change |
|---|---|
| `QuarkMindCaseFile.java` | 4 new key constants |
| `DroolsScoutingTask.java` | Trigger detection + result integration + new fields + requires/produces |
| `LlmPatternClassifierWorkerFactory.java` | **New** — worker factory |
| `QuarkMindCaseHub.java` | `wireLlmFallback()` + constants + call from `getDefinition()` |
| `LlmPatternClassifierWorkerFactoryTest.java` | **New** — unit tests |
| `DroolsScoutingTaskTest.java` | Additional test cases |
| `LlmPatternFallbackIT.java` | **New** — integration test |
