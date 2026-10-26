# Hypothesis Tracker Integration — Design Spec

**Issue:** casehubio/quarkmind#456
**Branch:** issue-456-sc2-replay-scouting-pipeline
**Date:** 2026-10-26

## Overview

Integrate neocortex's `HypothesisTracker` into the SC2 scouting pipeline as a
general-purpose hypothesis tracking tool — replacing the current
`DetectorOrchestrator` Bayesian fusion (tier 2) with a composed network that
tracks both **opponent strategy detection** and **our strategy response
selection**. Priors are learned from existing labelled replay data and
expert-defined matchup knowledge.

The HypothesisTracker is another tool in the detection toolkit alongside Drools
rules and ONNX detectors. It maintains competing hypotheses that evolve as
scouting observations arrive, eliminates dead paths, and drives strategy
selection as the game state changes.

## Architecture

### Current Tier 2 Flow

```
EvidenceMarkers (Drools) ─┐
                           ├─► DetectorOrchestrator ─► BayesNetwork ─► PatternAssessment[]
ONNX detector scores ─────┘    (SC2NetworkBuilder)     (naive Bayes
                                                         star topology)
```

`SC2NetworkBuilder` creates a naive Bayes star: one `Strategy` parent node with
uniform priors and N binary detector children (HIGH/LOW). `DetectorOrchestrator`
wraps this with `EvidenceFusionEngine`, temporal smoothing, hard evidence
thresholding, and sensitivity pruning.

### New Flow

```
Discrete scouting observations ─┐
                                 │
EvidenceMarkers (Drools) ────────┼─► HypothesisTracker ─► NetworkComposition ─► PatternAssessment[]
                                 │   (learned priors,       (opponent detection
ONNX detector scores ────────────┘    path elimination)      + response selection)
```

The `HypothesisTracker` replaces the `DetectorOrchestrator`'s
`BayesNetwork`/`EvidenceFusionEngine` as the fusion layer. It adds:
- **Learned priors** from replay data (not uniform)
- **Discrete scouting observations** as first-class evidence (not just detector
  scores)
- **Path elimination** — hypotheses whose posterior drops below a viability
  threshold are removed, narrowing the field over time
- **Response selection** — a composed network that links opponent detection to
  our strategy selection
- **Expert overrides** — matchup knowledge as prior overrides for sparse data
  scenarios

### What Changes

| Component | Before | After |
|---|---|---|
| `SC2NetworkBuilder` | Naive Bayes star, uniform priors | **Retired** — replaced by `SC2HypothesisNetworkBuilder` |
| `DetectorOrchestrator` | Wraps `EvidenceFusionEngine` + `BayesNetwork` | **Replaced** by `HypothesisTrackerFusion` |
| `CascadingPatternClassifier` tier 2 | Calls `detectorOrchestrator.classify()` | Calls `hypothesisTrackerFusion.classify()` |
| Priors | Uniform `1.0 / N` or hand-tuned | Learned from replay data + expert overrides |
| Observations | ONNX detector scores only | ONNX scores + discrete scouting events + Drools evidence |

### What Stays

- **Tier 1 (Drools rules)** — unchanged. Still produces `EvidenceMarker` lists
  with noisy-OR confidence aggregation.
- **Tier 3 (LLM fallback)** — unchanged. Still triggers when tiers 1 and 2 are
  uncertain.
- **`CascadingPatternClassifier` outer structure** — the three-tier cascade,
  cumulative confidence tracking, transition detection, and `CascadeResult`
  output all remain.
- **`BinaryDetectorRunner`** — still runs ONNX detectors. Their outputs feed
  the HypothesisTracker as observations instead of the old fusion engine.
- **`PatternAssessment` and `StrategyArchetype`** — output format unchanged.

## Component 1: ScoutingObservationExtractor

A new Java class in `io.quarkmind.sc2.replay` that reads game_json (produced by
`ReplayFeatureExtractor`) and extracts discrete scouting observation records.

### Discrete Variables

These variables model what a scouting player observes during a game. Each has a
small set of named states. The variables and states are defined for the
HypothesisTracker's Bayesian network — they represent the information available
to drive hypothesis updates.

| Variable | States | Derivation |
|---|---|---|
| `first_building` | Per-race building names (e.g. `gateway`, `forge`, `hatchery`, `barracks`, `command_center`) | First non-base building in tracker events |
| `gas_timing` | `early` (< 1:30), `normal` (1:30–2:30), `late` (> 2:30), `none` | Loop of first Assimilator/Extractor/Refinery |
| `expansion_timing` | `fast` (< 2:00), `normal` (2:00–3:30), `late` (> 3:30), `none` | Loop of second base |
| `tech_path` | Per-race tech buildings (e.g. `twilight_council`, `stargate`, `robotics_facility`, `factory`, `starport`, `spire`, `hydralisk_den`) | First tech structure |
| `army_composition` | `light` (ling/marine/zealot heavy), `armored` (roach/tank/stalker heavy), `air` (muta/banshee/phoenix heavy), `mixed` | Unit type ratios at checkpoint frames |
| `worker_saturation` | `under` (< 16/base), `saturated` (16–22/base), `over` (> 22/base) | Worker count / base count |
| `aggression_timing` | `early_pressure` (< 3:00), `timing_attack` (3:00–5:00), `macro_play` (> 5:00), `no_aggression` | Frame of first combat engagement or army movement toward opponent |

### Output Format

Records in `io.quarkmind.domain` (plain Java, no framework deps — consumed by
both replay extraction and live game-tick extraction):

```java
public record ScoutingObservation(
    String variable,   // e.g. "first_building"
    String state,      // e.g. "gateway"
    long detectedAtLoop // game loop when this was observable
) {}

public record ScoutingObservationSequence(
    String replayHash,
    int playerId,       // perspective player
    Race playerRace,
    Race opponentRace,
    List<ScoutingObservation> observations, // ordered by detectedAtLoop
    String groundTruthStrategy             // from .label.json
) {}
```

### Extraction Modes

- **Replay mode (batch):** Reads game_json, applies fog-of-war (only what the
  scouting player could see), extracts observations at known checkpoints.
  Produces `ScoutingObservationSequence` per player per replay.
- **Live mode:** Reads from `GameState` during a game tick, produces individual
  `ScoutingObservation` records as new information becomes visible. Used by
  `DroolsScoutingTask` to feed the HypothesisTracker during play.

## Component 2: SC2HypothesisNetworkBuilder

A new Java class in `io.quarkmind.sc2.detector` that builds the
`NetworkComposition` for the HypothesisTracker. Replaces `SC2NetworkBuilder`.

### Network Structure

The composed network has two subnetworks:

**Opponent Detection Network:**
- **Hidden variable:** `OpponentStrategy` — states are `StrategyArchetype`
  label strings filtered by enemy race and game phase (same as current
  `SC2NetworkBuilder` but with learned priors instead of uniform)
- **Observation variables:** One per discrete scouting variable (from
  `ScoutingObservationExtractor`), conditioned on `OpponentStrategy`
- **Detector variables:** One per active ONNX binary detector (HIGH/LOW or
  HIGH/MEDIUM/LOW for learned CPTs), conditioned on `OpponentStrategy` — same
  structure as current `SC2NetworkBuilder` detector children

**Response Selection Network:**
- **Hidden variable:** `OurStrategy` — states are our available strategy
  archetypes for the current game state
- **Conditioning variable:** `OpponentAssessment` — linked from the opponent
  detection network's current leader/distribution
- **Context variables:** `our_economy_state` (behind/even/ahead),
  `our_army_state` (smaller/even/larger), `our_tech_state` (behind/even/ahead)
  — derived from game state

The two networks compose via `NetworkComposition`. The opponent detection
network's posterior flows into the response selection network as a conditioning
input.

### Hypothesis Definitions

Each `StrategyArchetype` for the enemy race becomes a hypothesis in the opponent
tracker. Each of our available strategies becomes a hypothesis in the response
tracker. The priors come from the learned prior pipeline (Component 3).

### Builder API

```java
public class SC2HypothesisNetworkBuilder {

    public static HypothesisTracker buildOpponentTracker(
        Race enemyRace,
        GamePhase phase,
        Map<String, Double> learnedPriors,          // from prior learning pipeline
        List<String> activeDetectorNames,
        Map<String, Map<String, Double>> detectorCpts,
        Map<String, Map<String, double[]>> observationCpts // scouting variable CPTs
    );

    public static HypothesisTracker buildResponseTracker(
        Race ourRace,
        Map<String, Double> responsePriors,          // P(our strategy | opponent strategy)
        Map<String, Double> opponentPosterior        // current opponent assessment
    );
}
```

## Component 3: Prior Learning Pipeline

Mines existing labelled replay data for empirical prior frequencies. Two data
sources:

### Source 1: Replay Strategy Frequencies

From existing `.label.json` sidecars in `quarkmind-classifier/data/reconstituted/`:

```
For each labelled replay:
  - Extract matchup (vs_terran, vs_zerg, vs_protoss)
  - Extract strategy label (fineGrainedLabel)
  - Map classifier label → StrategyArchetype
  - Count frequencies per matchup per game phase
```

Output: `Map<String, Double>` per matchup per phase — the prior distribution.
Fed to `HypothesisPriorLearner.fromOutcomes()`.

### Source 2: Response Win Rates

From replay outcomes — both players' strategies and the game result:

```
For each labelled replay with Win/Loss result:
  - Player A ran strategy X, Player B ran strategy Y
  - Player A won/lost
  - Record: (our_strategy=X, opponent_strategy=Y, outcome=win/loss)
```

Output: `Map<String, Double>` — P(our_strategy | opponent_strategy, win).
These become priors for the response selection tracker.

### Source 3: Expert Overrides

A configuration file (YAML or Java constants) with expert-defined matchup
knowledge:

```yaml
vs_protoss:
  when_opponent: PROTOSS_CANNON_RUSH
    preferred_response: PROTOSS_FAST_EXPAND  # expand behind defense
    confidence_boost: 0.3
  when_opponent: PROTOSS_DT_RUSH
    preferred_response: PROTOSS_MACRO  # observer + defend
    confidence_boost: 0.4
```

Applied as prior overrides via
`HypothesisTrackerBuilder.hypothesis(name).priorWeight(learnedPrior + boost)`.

### Storage

Learned priors are exported as JSON files on the classpath, following the
existing pattern from `LearnedCptLoader`:

```
resources/
  priors/
    opponent-priors-EARLY-vs-terran.json
    opponent-priors-MID-vs-protoss.json
    response-priors-vs-zerg.json
    ...
```

Each file contains a `Map<String, Double>` (archetype label → prior weight).

### CPT Learning for Scouting Variables

The discrete scouting observation variables need CPTs: P(observation_state |
OpponentStrategy). These are learned from labelled replay data:

```
For each labelled replay:
  - Run ScoutingObservationExtractor → discrete observations
  - Pair each observation with the ground-truth strategy label
  - Count co-occurrences → normalize to CPTs
```

Output: `Map<String, Map<String, double[]>>` per matchup — variable name →
archetype → probability distribution over states. Stored as JSON on classpath,
loaded alongside detector CPTs.

## Component 4: HypothesisTrackerFusion

A new Java class in `io.quarkmind.sc2.detector` that replaces
`DetectorOrchestrator`. Same role — fuses evidence from multiple sources into
strategy assessments — but uses `HypothesisTracker` instead of raw
`EvidenceFusionEngine`.

### Public API

```java
public class HypothesisTrackerFusion {

    public HypothesisTrackerFusion(
        DetectorRegistry registry,
        Map<String, Double> learnedPriors,
        Map<String, Map<String, Double>> detectorCpts,
        Map<String, Map<String, double[]>> observationCpts,
        Map<String, BayesNetwork> learnedNetworks,
        int smoothingWindowSize
    );

    // Initialize for a specific phase/race combination
    public void initialize(
        List<String> detectorNames,
        GamePhase phase,
        Race enemyRace
    );

    // Feed discrete scouting observations
    public void observeScouting(ScoutingObservation observation);

    // Classify from all evidence sources
    public List<PatternAssessment> classify(
        Map<String, Float> detectorOutputs,
        List<EvidenceMarker> droolsMarkers,
        long frame
    );

    // Response selection — what should we do?
    public List<PatternAssessment> selectResponse(
        Race ourRace,
        Map<String, Double> gameStateContext  // economy, army, tech state
    );

    // Hypothesis tracking state
    public Map<String, Double> opponentWeights();
    public List<String> viableOpponentHypotheses();
    public String opponentLeader();
    public boolean isOpponentResolved();

    public void reset();
}
```

### Integration with CascadingPatternClassifier

The `CascadingPatternClassifier` constructor changes to accept a
`HypothesisTrackerFusion` instead of a `DetectorOrchestrator`. The tier 2 flow
becomes:

```java
// Before (DetectorOrchestrator):
detectorOrchestrator.initializeNetwork(detectorKeys, phase, enemyRace);
var toEvaluate = detectorOrchestrator.getDetectorsToEvaluate(allKeys);
var scores = detectorRunner.runActiveDetectors(phase, enemyRace, features, toEvaluate);
var assessments = detectorOrchestrator.classify(scores, markers, frame);

// After (HypothesisTrackerFusion):
hypothesisTrackerFusion.initialize(detectorKeys, phase, enemyRace);
// Feed any new discrete observations accumulated this tick
for (ScoutingObservation obs : newObservations) {
    hypothesisTrackerFusion.observeScouting(obs);
}
var scores = detectorRunner.runActiveDetectors(phase, enemyRace, features, detectorKeys);
var assessments = hypothesisTrackerFusion.classify(scores, markers, frame);
```

### DroolsScoutingTask Changes

`DroolsScoutingTask` gains a `LiveScoutingObservationExtractor` that produces
discrete observations from the current `GameState` each tick. These are passed
to the cascade alongside the existing `EvidenceMarker` list and
`StrategyFeatures`.

The `execute()` method adds:

```java
List<ScoutingObservation> newObs = liveExtractor.extract(gameState, frame);
cascadeResult = cascadingClassifier.classify(
    evidence, revisions, features, enemyRace, frame, prevFrame, ctx,
    enemies.size(), newObs);  // new parameter
```

## Testing Strategy

### Unit Tests

- **ScoutingObservationExtractorTest** — extract discrete observations from
  known game_json fixtures. Assert correct variable/state/loop for each
  observation type. Test fog-of-war filtering.
- **SC2HypothesisNetworkBuilderTest** — build networks for each race/phase
  combination. Assert correct hypothesis count, prior weights, variable states.
- **HypothesisTrackerFusionTest** — feed observations and detector outputs,
  assert assessment convergence. Test path elimination. Test reset.
- **PriorLearningExportTest** — extract priors from test label sidecars, assert
  correct frequency distributions per matchup.
- **LiveScoutingObservationExtractorTest** — extract discrete observations from
  GameState snapshots at known game loops.

### Integration Tests

- **HypothesisTrackerFusionIT** (`@QuarkusTest`) — full CDI wiring, verify the
  fusion layer produces assessments through the cascade.
- **CascadingPatternClassifierHypothesisIT** (`@QuarkusTest`) — verify the
  cascade works end-to-end with the new fusion layer, comparing assessment
  quality against the old `DetectorOrchestrator` path.

### Calibration Tests

- **HypothesisTrackerCalibrationTest** — run against AI Arena + IEM10 replay
  datasets, compare accuracy at 3-min and 5-min checkpoints against the
  existing `PatternClassificationCalibrationTest` baseline. The new fusion must
  match or exceed existing accuracy.

## Scope and Phasing

This is a moderate-sized integration. Suggested phasing:

**Phase 1: Data pipeline (prior learning)**
- `ScoutingObservationExtractor` (replay mode)
- Prior frequency extraction from `.label.json` sidecars
- Observation CPT learning from labelled replays
- JSON export of learned priors and CPTs

**Phase 2: Network definition**
- `SC2HypothesisNetworkBuilder` — opponent detection network
- `HypothesisTrackerFusion` — replacement for `DetectorOrchestrator`
- Unit tests against fixtures

**Phase 3: Integration**
- Wire into `CascadingPatternClassifier` (replace tier 2)
- `LiveScoutingObservationExtractor` for game-tick observations
- `DroolsScoutingTask` changes
- Integration and calibration tests

**Phase 4: Response selection**
- Response prior learning from win/loss outcomes
- Response network composition
- Expert override configuration
- Strategy selection integration

## Temporal Smoothing

The current `DetectorOrchestrator` uses `TemporalAssessmentTracker` (sliding
window of 5 posteriors) to smooth noisy detector outputs. The
`HypothesisTracker` has inherent temporal memory — hypothesis weights persist
and update incrementally across observations, so the signal is naturally
smoother than per-tick inference.

Decision: start without explicit smoothing. If calibration tests show the
HypothesisTracker produces noisier assessments than the old path, add a
lightweight smoothing wrapper. The `TemporalAssessmentTracker` class stays
available — it's not deleted, just not wired in by default.

## Trade-offs

- **Replacing tier 2 touches a well-tested path.** Mitigated by calibration
  tests that compare old vs new accuracy. The old path can be kept as a fallback
  behind a config flag during transition.
- **Discrete observation variables lose continuous information.** The
  discretization boundaries (e.g., gas_timing thresholds) are design choices
  that affect sensitivity. Calibrate from replay data, not from intuition.
  Protocol `sc2data-train-times-require-calibration.md` applies.
- **Response selection depends on opponent detection quality.** If opponent
  assessment is wrong, response selection amplifies the error. Path elimination
  and the resolution threshold (`0.95`) provide a gate — don't select a response
  until the opponent assessment is resolved or highly confident.
- **Expert overrides can conflict with learned priors.** The override is additive
  (boost), not a replacement — learned priors provide the base, expert knowledge
  shifts the distribution. If data contradicts the expert, the data wins at
  scale.

## References

- casehubio/neocortex#692 — HypothesisTracker API
- casehubio/neocortex#705 — HypothesisPriorLearner (implemented)
- `HypothesisTracker.java` — builder API, observeHard/observe, path elimination
- `HypothesisPriorLearner.java` — fromOutcomes(), online update()
- `SC2NetworkBuilder.java` — current naive Bayes star (being replaced)
- `DetectorOrchestrator.java` — current fusion layer (being replaced)
- `CascadingPatternClassifier.java` — cascade structure (tier 2 changes)
- `DroolsScoutingTask.java` — game tick integration point
- `BinaryDetectorRunner.java` — ONNX detector execution (stays)
- `StrategyArchetype.java` — 59 archetypes + 3 fallbacks across 3 races
- `PatternAssessment.java` — output format (unchanged)
- `quarkmind-classifier/data/reconstituted/` — labelled replay data source
- `docs/protocols/sc2data-train-times-require-calibration.md` — discretization
  boundaries must be calibrated from data
