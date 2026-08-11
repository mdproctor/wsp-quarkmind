# SC2 Opponent Identity & Per-Engagement Outcome Tracking

**Branch:** `issue-271-sc2-opponent-id-cdi-cleanup`
**Issues:** #271 (SC2 opponent identity), #270 (per-engagement outcome tracking)
**Scope:** 1v1 mode only

---

## 1. SC2 Opponent Identity (#271)

### Problem

`GameStateTranslator` gets `opponentId` from a static `@ConfigProperty("quarkmind.opponent.id")` at CDI boot time. Per-profile config covers mock ("mock-opponent"), emulated ("PROTOSS_4GATE"), and test ("mock-opponent") modes. SC2 mode has no config — falls back to "unknown". The real opponent identity is available at runtime from the SC2 protocol's `ResponseGameInfo.getPlayersInfo()`.

### Design

#### Identity Composition

- **AI opponent** (PlayerType.COMPUTER): `SHA-256("RACE:DIFFICULTY")` — e.g. `SHA-256("ZERG:VeryHard")`
- **Human/bot opponent** (PlayerType.PARTICIPANT): `SHA-256(playerId)` — hashed for privacy

The non-bot `PlayerInfo` entry (where `playerType != PARTICIPANT` for AI, or the other PARTICIPANT for PvP) provides race, difficulty, and player ID.

#### GameStarted Event Enrichment

Add opponent identity fields to the `GameStarted` record:

```java
public record GameStarted(
    // existing fields...
    Race opponentRace,
    SC2Api.PlayerType opponentType,
    SC2Api.Difficulty opponentDifficulty  // nullable for PARTICIPANT opponents
)
```

`QuarkusSC2Transport.gameLoop()` already parses `ResponseGameInfo` and fires `GameStarted`. It extracts opponent identity from `PlayerInfo` entries.

Mock/emulated/test engines fire `GameStarted` with synthetic identity matching current config values. The `@ConfigProperty("quarkmind.opponent.id")` is removed from `GameStateTranslator`.

#### GameStateTranslator Lifecycle Observer

```java
@ApplicationScoped
public class GameStateTranslator {
    private final AtomicReference<String> opponentId = new AtomicReference<>("unknown");

    void onGameStarted(@Observes GameStarted event) {
        opponentId.set(computeOpponentId(event));
    }

    static String computeOpponentId(GameStarted event) {
        if (event.opponentType() == PlayerType.COMPUTER) {
            return sha256(event.opponentRace() + ":" + event.opponentDifficulty());
        }
        return sha256(String.valueOf(event.opponentPlayerId()));
    }

    public Map<String, Object> toMap(GameState state) {
        // ... existing code ...
        data.put(QuarkMindCaseFile.OPPONENT_ID, opponentId.get());
        // ...
    }
}
```

Thread-safe: game loop thread writes `AtomicReference` once at start, scheduler thread reads each tick.

---

## 2. Per-Engagement Outcome Tracking (#270)

### Problem

`BATTLE_STARTED` fires from `TimingAlert.incoming == true` — incoming timing attack detection by scouting. It does not detect player-initiated attacks, skirmishes, harassment, or non-timing defensive engagements. `BATTLE_ENDED` exists in `GameMomentType` but nothing produces it. No per-engagement unit loss tracking exists. CBR has aggregate `battleCount` but no per-engagement outcomes.

### Design

#### Army Value Delta FSM

MomentDetectionTask gains a 3-state FSM for engagement detection, independent of `BATTLE_STARTED`:

```
IDLE ──[army value drops >15%]──► IN_BATTLE
IN_BATTLE ──[army value stabilises <5% change]──► QUIESCENT
QUIESCENT ──[224 frames of stability]──► IDLE (emit BATTLE_ENDED)
QUIESCENT ──[army value drops again]──► IN_BATTLE (battle resumed)
```

State is tracked via new fields in `MomentDetectionTask`:

```java
private enum BattleState { IDLE, IN_BATTLE, QUIESCENT }
private BattleState battleState = BattleState.IDLE;
private long quiescenceStartFrame = -1;
private int battleStartArmyValue = 0;
private List<Unit> battleStartOwnUnits = List.of();
private List<Unit> battleStartEnemyUnits = List.of();
private long battleStartFrame = -1;
```

Detection uses the `ArmySize` scouting intel already consumed by MomentDetectionTask, combined with `GameState` from CaseContext for unit snapshots. The existing `previousArmyValue` field provides the baseline for delta computation.

`BATTLE_STARTED` continues to fire from `TimingAlert` for scouting-level alerting (advisory/coaching triggers). Engagement tracking uses the army value delta channel — the two are independent.

#### Army Snapshots

At IDLE→IN_BATTLE: snapshot own and enemy units from `CaseContext` (`QuarkMindCaseFile.ARMY`, `ENEMY_UNITS`). Compute total resource value via `SC2Data.mineralCost()` + `SC2Data.gasCost()`.

At QUIESCENT→IDLE (BATTLE_ENDED): snapshot again. Delta gives per-engagement losses.

#### EngagementOutcome Record

```java
package io.quarkmind.plugin.summarisation;

public record EngagementOutcome(
    long startFrame,
    long endFrame,
    int ownUnitsLost,
    int enemyUnitsLost,
    int ownValueLost,
    int enemyValueLost,
    double unitTradeRatio,
    Outcome outcome
) {
    public enum Outcome { WON, LOST, EVEN }
}
```

- `unitTradeRatio`: `enemyValueLost / ownValueLost` (>1 = favorable; `Double.MAX_VALUE` if `ownValueLost == 0`)
- Outcome: `WON` if `enemyValueLost > ownValueLost * 1.2`. `LOST` if `ownValueLost > enemyValueLost * 1.2`. `EVEN` otherwise.

Carried in BATTLE_ENDED metadata: `Map.of("engagement", engagementOutcome)`.

#### CBR Enrichment

`SC2CbrRetentionObserver` extracts `EngagementOutcome` from BATTLE_ENDED moments at retention time. Three new fields in `EnrichedGameData`:

| Field | Type | Source |
|-------|------|--------|
| `engagementsWon` | int | Count of WON outcomes |
| `engagementsLost` | int | Count of LOST outcomes |
| `unitTradeRatio` | double | Aggregate: total enemy value lost / total own value lost |

These become CBR case features in `SC2GameCbrCase.buildForGameEnriched()`.

---

## 3. Testing

### #271 Tests

- **`GameStateTranslatorTest`** — fire `GameStarted` with opponent identity, assert `AtomicReference` is read by `toMap()`. Test AI identity (race+difficulty hash) and PvP identity (player ID hash) separately.
- **`QuarkusSC2TransportTest`** — verify `GameStarted` event carries opponent identity fields from `ResponseGameInfo`.
- Existing CBR tests (`SC2CbrRetentionObserverTest`, `SC2CbrRetentionIT`) — unaffected (write `OPPONENT_ID` directly to CaseFile).

### #270 Tests

- **`MomentDetectionTaskTest`** — battle FSM transitions:
  - IDLE→IN_BATTLE on >15% army value drop
  - IN_BATTLE→QUIESCENT on stabilisation
  - QUIESCENT→IDLE after 224 frames → assert BATTLE_ENDED with `EngagementOutcome`
  - QUIESCENT→IN_BATTLE on resumed fighting (no premature BATTLE_ENDED)
  - Multiple sequential engagements in one game
- **`EngagementOutcomeTest`** — outcome determination (WON/LOST/EVEN thresholds, edge cases: zero losses on one side)
- **`SC2CbrRetentionObserverTest`** — extend to verify `engagementsWon`, `engagementsLost`, `unitTradeRatio` extraction from BATTLE_ENDED moments
- **`SC2GameCbrCaseTest`** — verify new features appear in CBR case

### Calibration (follow-up)

The 15% battle-start threshold and 224-frame quiescence window need tuning against replay data. Add a `@Tag("benchmark")` calibration test that runs replays and reports engagement detection accuracy — similar to `ScoutingCalibrationTest`.

---

## 4. Files Modified

### #271
| File | Change |
|------|--------|
| `GameStarted.java` | Add opponentRace, opponentType, opponentDifficulty fields |
| `GameStateTranslator.java` | Remove @ConfigProperty, add @Observes GameStarted, AtomicReference |
| `QuarkusSC2Transport.java` | Extract opponent identity from ResponseGameInfo into GameStarted |
| `MockStartupBean.java` / emulated engines | Fire GameStarted with synthetic identity |
| `application.properties` | Remove `quarkmind.opponent.id` per-profile entries |
| `GameStateTranslatorTest.java` | Update to fire GameStarted |
| `QuarkusSC2TransportTest.java` | Verify GameStarted opponent fields |

### #270
| File | Change |
|------|--------|
| `EngagementOutcome.java` | New record in summarisation package |
| `MomentDetectionTask.java` | Battle FSM state, army snapshots, BATTLE_ENDED emission |
| `MomentDetectionTask.drl` | No change (BATTLE_STARTED stays for advisory triggers) |
| `EnrichedGameData.java` | Add engagementsWon, engagementsLost, unitTradeRatio |
| `SC2CbrRetentionObserver.java` | Extract EngagementOutcome from BATTLE_ENDED moments |
| `SC2GameCbrCase.java` | Add engagement features |
| `MomentDetectionTaskTest.java` | New/extended — battle FSM tests |
| `EngagementOutcomeTest.java` | New — outcome determination tests |
| `SC2CbrRetentionObserverTest.java` | Extend with engagement data |
| `SC2GameCbrCaseTest.java` | Extend with engagement features |
