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
  scouting player could see), extracts observations at the game loop when they
  become visible to the scouting player's units. `detectedAtLoop` records when
  the player first had line-of-sight, not when the event actually occurred.
  Produces `ScoutingObservationSequence` per player per replay.
- **Live mode:** Reads from `GameState` during a game tick, produces individual
  `ScoutingObservation` records as new information becomes visible. Used by
  `DroolsScoutingTask` to feed the HypothesisTracker during play.

### Fog-of-War Design Rationale

Fog-of-war extraction is intentional, not a limitation. The CPTs model
P(observation_state | strategy) — the conditional probability of what a player
sees given the opponent's strategy, at the moment of observation. Variables that
have not yet been scouted simply have no evidence set on the tracker, which is
the correct Bayesian treatment: prior weights hold until evidence arrives.

This matches live prediction semantics: in a real game, observations arrive when
scouted, not when the event occurs. Training CPTs from full-information replays
would create a mismatch where the model expects observations earlier than the
live system can provide them. The CPT values themselves are independent of
observation timing — P(first_building = gateway | rush) is a structural property
of the strategy, regardless of when the scout reaches the enemy base.

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
- **Output variable:** `OurStrategy` — states are our available strategy
  archetypes, conditioned on `OpponentAssessment` + all context variables
- **Conditioning variable:** `OpponentAssessment` — linked from the opponent
  detection network's current leader/distribution
- **Context variables:** `our_economy_state` (behind/even/ahead),
  `our_army_state` (smaller/even/larger), `our_tech_state` (behind/even/ahead)
  — root nodes with uniform priors, set as hard evidence from `GameState`

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
        Map<String, Double> responsePriors,
        Map<String, Double> opponentPosterior,
        Map<String, double[]> contextCpts            // economy/army/tech state CPTs
    );
}
```

### Construction Walkthrough

The `buildOpponentTracker()` factory encapsulates the following multi-step
construction. This walkthrough documents what the factory does internally,
following the `HypothesisTrackerScoutingExample` pattern from neocortex.

**Step 1: Build intel JunctionTree**

```java
var intelBuilder = BayesNetworkBuilder.create("intel");

// Root variable — strategy archetypes for this race/phase
String[] archetypes = filteredArchetypes(enemyRace, phase);
double[] priorArray = /* from learnedPriors, indexed by archetypes */;
intelBuilder.variable("enemy_strategy", archetypes).prior(priorArray);

// Observation variables — one per scouting variable
for (var obsEntry : observationCpts.entrySet()) {
    String varName = obsEntry.getKey();  // e.g. "first_building"
    String[] states = obsEntry.getValue().keySet().toArray(String[]::new);
    double[] flatCpt = /* flatten observationCpts[varName] by archetype order */;
    intelBuilder.variable(varName, states).given("enemy_strategy").cpt(flatCpt);
}

// Detector variables — one per active ONNX detector
for (String detName : activeDetectorNames) {
    Map<String, Double> cpt = detectorCpts.get(detName);
    double[] flatCpt = /* build [pHigh, 1-pHigh] per archetype, clamped to [0.01, 0.99] */;
    intelBuilder.variable(detName, "HIGH", "LOW").given("enemy_strategy").cpt(flatCpt);
}

// Drools evidence variable — for EvidenceMarker integration
String[] markerStates = archetypes; // one state per archetype
double[] uniformPrior = /* uniform 1/N */;
intelBuilder.variable("drools_evidence", markerStates)
    .given("enemy_strategy").cpt(/* identity-like CPT: P(evidence=X | strategy=X) = high */);

JunctionTree intelTree = intelBuilder.buildJunctionTree();
```

**Step 2: Build response JunctionTree**

Context variables are root nodes set as hard evidence at inference time — they
represent our game state, not the opponent's strategy. `our_strategy` is
conditioned on all four parents so that context influences the recommendation:

```
enemy_strategy ──────┐
our_economy_state ───┤
our_army_state ──────┼──► our_strategy
our_tech_state ──────┘
```

```java
var responseBuilder = BayesNetworkBuilder.create("response");
responseBuilder.variable("enemy_strategy", archetypes).prior(priorArray);
// Context variables — root nodes with uniform priors, set as hard evidence
responseBuilder.variable("our_economy_state", "behind", "even", "ahead")
    .prior(1.0/3, 1.0/3, 1.0/3);
responseBuilder.variable("our_army_state", "smaller", "even", "larger")
    .prior(1.0/3, 1.0/3, 1.0/3);
responseBuilder.variable("our_tech_state", "behind", "even", "ahead")
    .prior(1.0/3, 1.0/3, 1.0/3);
// Response conditioned on enemy strategy AND all context variables
String[] ourStrategies = /* our available strategies */;
responseBuilder.variable("our_strategy", ourStrategies)
    .given("enemy_strategy", "our_economy_state", "our_army_state", "our_tech_state")
    .cpt(/* P(our_strategy | enemy_strategy, economy, army, tech) */);
JunctionTree responseTree = responseBuilder.buildJunctionTree();
```

The joint CPT has `|enemy_strategies| × 3 × 3 × 3 = 27 × |enemy_strategies|`
rows. For 20 enemy archetypes this is 540 rows × |our_strategies| values.
Initial CPTs are seeded from the response prior learning (Source 2) with uniform
context dimensions, then refined through replay analysis that pairs game state
snapshots with strategy outcomes.

**Step 3: Compose via CompositionBuilder**

```java
NetworkComposition composition = CompositionBuilder.create("sc2-scouting")
    .slot("intel", intelTree)
    .slot("response", responseTree)
    .link("intel", "enemy_strategy").to("response", "enemy_strategy")
    .build();
```

Slot names: `"intel"` (opponent detection), `"response"` (strategy selection).
The link transfers the `enemy_strategy` posterior from intel to response via
`TransferMode.POSTERIOR` (default).

**Step 4: Build HypothesisTracker**

```java
var builder = HypothesisTracker.builder(composition);
for (var archetype : filteredArchetypes(enemyRace, phase)) {
    double prior = learnedPriors.getOrDefault(archetype, 1.0 / N);
    double[] override = new double[archetypes.length]; // concentrate on this archetype
    // Set override[i] = 0.85 for this archetype, 0.05/(N-1) for others
    builder.hypothesis(archetype)
        .priorWeight(prior)
        .priorOverride("intel", "enemy_strategy", override);
}
builder.eliminationThreshold(0.01)
       .resolutionThreshold(0.95);
return builder.build();
```

Each hypothesis concentrates probability on its archetype in the
`enemy_strategy` variable via `priorOverride()`, then the tracker evolves
weights as observations arrive. The `priorWeight` reflects the learned
frequency of this archetype in the matchup.

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
  - when_opponent: PROTOSS_CANNON_RUSH
    preferred_response: PROTOSS_FAST_EXPAND
    prior_multiplier: 3.0
  - when_opponent: PROTOSS_DT_RUSH
    preferred_response: PROTOSS_MACRO
    prior_multiplier: 4.0
```

Applied as multiplicative prior adjustments: the learned prior for the
preferred response is multiplied by `prior_multiplier`, then the full response
prior distribution is renormalized. This preserves the relative scale of the
learned distribution — a 3x multiplier has the same proportional effect whether
the base prior is 0.05 or 0.40.

Combination rule: `adjustedPrior[i] = learnedPrior[i] * multiplier[i]`, then
normalize so all priors sum to 1.0. When no expert override applies,
`multiplier = 1.0` (identity).

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
  - Count co-occurrences
  - Apply Laplace smoothing (add-1) before normalizing to CPTs
  - Clamp all CPT entries to [0.01, 0.99] floor/ceiling
```

Laplace smoothing prevents zero-probability CPT entries. Without it, a single
zero P(observation | strategy) entry causes immediate hard elimination in the
`HypothesisTracker` — `if (likelihood == 0.0) → ELIMINATED`. With 62 strategy
archetypes and sparse replay data, zero entries are guaranteed for rare
strategy-observation combinations. The 0.01 floor matches the existing
`SC2NetworkBuilder` clamping pattern (`Math.max(0.01, Math.min(0.99, pHigh))`).

For prior learning, use `HypothesisPriorLearner(allHypotheses, smoothing=1.0)`
which already supports Laplace smoothing via its constructor parameter.

Output: `Map<String, Map<String, double[]>>` per matchup — variable name →
archetype → probability distribution over states. Stored as JSON on classpath,
loaded alongside detector CPTs.

### Data Pipeline: Source → Classpath

The prior learning pipeline runs as a standalone build step, not at runtime:

```
quarkmind-classifier/data/reconstituted/*.label.json  (source data)
    │
    ▼  ScoutingObservationExtractor (batch mode) + PriorLearner
    │
quarkmind-sc2/src/main/resources/priors/              (classpath resources)
    opponent-priors-EARLY-vs-terran.json
    opponent-priors-MID-vs-protoss.json
    observation-cpts-EARLY-vs-terran.json
    response-priors-vs-zerg.json
    ...
```

The pipeline reads `.label.json` sidecars from `quarkmind-classifier` (a sibling
project in the same workspace), extracts observations and strategy labels, learns
priors and CPTs, and writes JSON files to `quarkmind-sc2/src/main/resources/priors/`.
These are committed to the repository as resources, following the existing pattern
from `LearnedCptLoader` which loads `classifier/learned-cpts-*.json` from the
classpath. The pipeline is invoked manually (or via a Maven exec plugin) when new
replay data is available — not on every build.

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
        PhaseResolver phaseResolver
    );

    // Initialize for a specific phase/race combination
    public void initialize(
        List<String> detectorNames,
        GamePhase phase,
        Race enemyRace
    );

    // Feed discrete scouting observations (one-time, uses observeHard)
    public void observeScouting(ScoutingObservation observation);

    // Set per-tick evidence (replaces without weight multiplication)
    public void setDetectorEvidence(Map<String, Float> detectorOutputs);
    public void setDroolsEvidence(List<EvidenceMarker> droolsMarkers);

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

    // Diagnostic APIs — equivalents of DetectorOrchestrator methods
    public StrategyArchetype getMapStrategy();
    public BayesStats getStats();
    public Map<String, Double> getSensitivities();
    public Map<String, Object> getMapAssignment();
    public String getExplanationSummary();
    public Set<String> getDetectorsToEvaluate(Set<String> available);

    public void reset();
}
```

### Evidence Model — Dual-Path Architecture

The `HypothesisTracker.observe()` API cumulatively **multiplies** hypothesis
weights by the computed likelihood on each call:

```java
// HypothesisTracker.observe():
currentWeights.put(name, currentWeights.get(name) * likelihood);
comp.setEvidence(slotName, variableName, distribution);
comp.update();
```

This is correct for one-time sequential observations (a scout sees a gateway —
that information arrives once and permanently updates beliefs). It is **wrong**
for per-tick evidence (detector scores that refresh every game tick) — calling
`observe()` K times with unchanged evidence multiplies the weight K times,
exponentially converging on the best-fitting hypothesis regardless of whether
new information arrived.

The fusion layer therefore separates two evidence paths:

**Path 1: One-time scouting observations → `tracker.observeHard()`**

Called once per observation event. Multiplies hypothesis weights by the
observation likelihood, may eliminate hypotheses. Drives hypothesis narrowing.

```java
// In observeScouting():
tracker.observeHard("intel", observation.variable(), observation.state());
```

Scouting observations are categorical facts (`first_building = "gateway"`) so
`observeHard()` is used, not `observe()`.

**Path 2: Per-tick evidence → `tracker.setPerTickEvidence()`**

Called every tick with current detector scores and Drools markers. **Replaces**
evidence on each hypothesis's `CompositionInstance` via `setEvidence()` +
`update()` — without multiplying hypothesis weights. The junction tree
propagation reflects current evidence, but weight evolution is driven only by
scouting observations.

```java
// In classify():
// Detector outputs — per-tick replacement
for (var entry : detectorOutputs.entrySet()) {
    double[] distribution = toThreeElementLikelihood(entry.getValue());
    tracker.setPerTickEvidence("intel", entry.getKey(), distribution);
}

// Drools markers — per-tick replacement
if (!droolsMarkers.isEmpty()) {
    double[] likelihood = new double[archetypes.length];
    Arrays.fill(likelihood, 0.1);
    for (EvidenceMarker marker : droolsMarkers) {
        int idx = archetypeIndex(marker.archetype());
        likelihood[idx] = Math.max(likelihood[idx], marker.weight());
    }
    tracker.setPerTickEvidence("intel", "drools_evidence", likelihood);
} else {
    tracker.clearPerTickEvidence("intel", "drools_evidence");
}
```

**`setPerTickEvidence()`** is a new method on `HypothesisTracker` (see
§Required Neocortex API Additions). It delegates to each viable hypothesis's
`CompositionInstance.setEvidence()` + `update()` without touching weights:

```java
// New HypothesisTracker method:
public void setPerTickEvidence(String slotName, String variableName, double[] distribution) {
    for (var entry : compositions.entrySet()) {
        if (statuses.get(entry.getKey()) == HypothesisStatus.VIABLE) {
            entry.getValue().setEvidence(slotName, variableName, distribution);
            entry.getValue().update();
        }
    }
}
```

**Assessment derivation:** The output `List<PatternAssessment>` is built from
the hypothesis weights (which reflect only scouting observation updates) combined
with the compositions' current posteriors (which reflect all evidence including
per-tick detector/Drools state). The assessment confidence for each archetype
comes from the tracker's normalized hypothesis weight — which is only updated
when scouting observations arrive.

### Detector Output Conversion

ONNX detector outputs are continuous floats (0.0–1.0). The existing
`DetectorOrchestrator.toThreeElementLikelihood()` conversion is reused: each
float score maps to a `[pHigh, pMedium, pLow]` array for 3-outcome detectors,
or `[p, 1-p]` for binary detectors. The conversion logic is promoted to a
shared utility.

For binary detectors (the current default), the hard evidence threshold still
applies: scores ≥ 0.95 become `[1.0, 0.0]`, scores ≤ 0.05 become `[0.0, 1.0]`,
and intermediate values become soft evidence `[p, 1-p]`.

### EvidenceMarker Integration

Drools `EvidenceMarker` records are integrated via a dedicated `drools_evidence`
variable in the intel network (see Construction Walkthrough, Step 1). Each
marker's archetype maps to a state of this variable, and the marker weight
becomes the likelihood for that state with a 0.1 baseline — mirroring the
existing `DetectorOrchestrator.computeMarkerLikelihood()` semantics. The
dedicated variable gives the Bayesian network cleaner separation between Drools
rule evidence and ONNX detector evidence.

Evidence is set via the per-tick path (`setPerTickEvidence`) since Drools
markers may change or disappear between ticks.

### Phase Transition Semantics

When the game transitions between phases (EARLY → MID → LATE), the tracker is
reinitialized:

1. Capture terminal weights via `tracker.snapshot()`
2. Build a new `HypothesisTracker` for the new phase (different archetypes,
   different detectors, potentially different network structure)
3. Seed the new tracker's priors from the terminal weights of the previous phase:
   archetypes that exist in both phases carry forward their terminal weight;
   new-phase-only archetypes receive the learned default prior

```java
// In initialize() when phase changes:
TrackerSnapshot terminal = opponentTracker.snapshot();
opponentTracker = SC2HypothesisNetworkBuilder.buildOpponentTracker(
    enemyRace, newPhase, seedPriors(learnedPriors, terminal), ...);
```

This preserves accumulated intelligence across phases without maintaining stale
network structure. The `initialize()` method detects phase changes by comparing
the new phase against the current phase (not just the detector set), mirroring
the existing `DetectorOrchestrator` guard:
`if (initialized && currentPhase == newPhase && currentSorted.equals(newSorted)) return;`

### Diagnostic API Mapping

| `DetectorOrchestrator` method | `HypothesisTrackerFusion` equivalent | Source |
|---|---|---|
| `getMapStrategy()` | `getMapStrategy()` → `labelToArchetype.get(tracker.leader())` | `HypothesisTracker.leader()` |
| `getStats()` | `getStats()` → delegates to `CompositionInstance.getStats("intel")` | `BayesInstance.getStats()` via composition |
| `getSensitivities()` | `getSensitivities()` → delegates to `CompositionInstance.sensitivity()` | Requires new `sensitivity()` method on `CompositionInstance` (see §Required Neocortex API Additions) |
| `getMapAssignment()` | `getMapAssignment()` → `Map.of("Strategy", tracker.leader())` + per-slot marginals | Derived from tracker weights and slot marginals |
| `getExplanationSummary()` | `getExplanationSummary()` → built from `TrackerSnapshot`: top 5 viable hypotheses with weights and eliminated hypotheses | Different format: hypothesis weights rather than variable influence traces |
| `getDetectorsToEvaluate()` | `getDetectorsToEvaluate()` → hypothesis-based pruning (see below) | `HypothesisTracker.viable()` |

**Sensitivity pruning replacement:** The `HypothesisTracker` enables a
different, potentially more effective pruning mechanism: hypothesis-based
pruning. Once hypotheses are eliminated, detectors that only differentiate
between eliminated hypotheses can be skipped. If only 3 of 62 hypotheses remain
viable, only detectors whose CPTs distinguish between those 3 need to run.

```java
public Set<String> getDetectorsToEvaluate(Set<String> available) {
    List<String> viable = tracker.viable();
    if (viable.size() <= 1) return Set.of(); // resolved — no detectors needed
    Set<String> needed = new HashSet<>();
    for (String det : available) {
        if (distinguishesViableHypotheses(det, viable)) {
            needed.add(det);
        }
    }
    return needed.isEmpty() ? available : needed;
}

// A detector distinguishes viable hypotheses if its CPT values differ
// meaningfully across the viable set — i.e. the detector would produce
// different evidence distributions under different viable hypotheses.
private boolean distinguishesViableHypotheses(String detName, List<String> viable) {
    double minP = Double.MAX_VALUE, maxP = -Double.MAX_VALUE;
    for (String archetype : viable) {
        double pHigh = detectorCpts.get(detName).getOrDefault(archetype, 0.5);
        minP = Math.min(minP, pHigh);
        maxP = Math.max(maxP, pHigh);
    }
    return (maxP - minP) >= 0.1; // 10% CPT variance threshold
}
```

The 10% threshold means a detector is skipped only when its CPT values are
nearly identical across all viable hypotheses — it cannot distinguish between
them. This is conservative: in practice, most detectors are designed to
differentiate specific archetype groups.

### Learned Networks Parameter

The `learnedNetworks` parameter (`Map<String, BayesNetwork>`, keyed by
`PHASE_vs_race`) provides pre-trained networks from `LearnedCptLoader`. When a
learned network exists for the current phase/race combination, the
`HypothesisTrackerFusion` uses it as the intel slot's `JunctionTree` instead of
the generated one from `SC2HypothesisNetworkBuilder`. The learned network's
variable structure must match the expected slot structure (same `enemy_strategy`
outcomes, same detector and observation variables).

```java
// In initialize():
String key = phase.name() + "_vs_" + enemyRace.name().toLowerCase();
BayesNetwork learned = learnedNetworks.get(key);
if (learned != null) {
    JunctionTree intelTree = new JunctionTreeBuilder(learned).build();
    // Use learned tree for intel slot instead of generated
}
```

### Concurrency

`DroolsScoutingTask.execute()` runs per game tick at 22.4 fps.
`HypothesisTracker.observe()` performs Bayesian updating: marginalization,
weight multiplication, normalization, and potential elimination. The
`CompositionInstance.update()` propagates through the junction tree.

For the expected network sizes (≤62 hypotheses, ≤15 variables per slot), each
`observe()` call is O(hypotheses × variable_states) — microsecond-scale per
observation. The `CompositionInstance.update()` includes incremental dirty-slot
tracking (`slotsSkipped` counter in `CompositionStats`), so unchanged slots
are not re-propagated. This is well within the ~45ms per-tick budget.

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
var toEvaluate = hypothesisTrackerFusion.getDetectorsToEvaluate(allKeys);
var scores = detectorRunner.runActiveDetectors(phase, enemyRace, features, toEvaluate);
var assessments = hypothesisTrackerFusion.classify(scores, markers, frame);
```

The `classify()` signature on `CascadingPatternClassifier` is **unchanged** —
scouting observations are not threaded through the cascade. Instead,
`CascadingPatternClassifier` exposes a delegation method:

```java
public void observeScouting(ScoutingObservation observation) {
    if (hypothesisTrackerFusion != null) {
        hypothesisTrackerFusion.observeScouting(observation);
    }
}
```

This preserves tier separation: the cascade routes between tiers based on
confidence without needing to know about scouting observation records. The
fusion layer has already accumulated observations when the cascade invokes
tier 2.

The `detectorStats()` method is updated to use the `HypothesisTrackerFusion`
diagnostic APIs, maintaining the `DetectorStatsPublished` CDI event format.

### DroolsScoutingTask Changes

`DroolsScoutingTask` gains a `LiveScoutingObservationExtractor` that produces
discrete observations from the current `GameState` each tick. Observations are
fed directly to the cascade's `observeScouting()` method before classification:

```java
List<ScoutingObservation> newObs = liveExtractor.extract(gameState, frame);
for (ScoutingObservation obs : newObs) {
    cascadingClassifier.observeScouting(obs);
}
cascadeResult = cascadingClassifier.classify(
    evidence, revisions, features, enemyRace, frame, prevFrame, ctx,
    enemies.size());  // signature unchanged
```

### Phase Resolution

The `HypothesisTrackerFusion` constructor accepts a `PhaseResolver` instance
and uses it consistently. The static `CascadingPatternClassifier.resolvePhase(frame)`
is replaced with a call to the injected `PhaseResolver`:

```java
// Before: GamePhase phase = resolvePhase(frame);  // static time thresholds
// After:  GamePhase phase = phaseResolver.resolve(gameState);  // injected resolver
```

This eliminates the dual-track inconsistency where `DroolsScoutingTask` uses
`phaseResolver.resolve(gameState)` but `CascadingPatternClassifier` uses
`resolvePhase(frame)`. Both now use the same `PhaseResolver` instance, ensuring
the phase used for learned prior selection matches the phase set on the
`CaseContext`. `CascadingPatternClassifier.resolvePhase()` is retained as a
package-private fallback for tests that don't inject a `PhaseResolver`.

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
  datasets, compare accuracy at 3-min and 5-min checkpoints against the full
  cascade with the current `DetectorOrchestrator` tier 2 baseline. Both old
  (DetectorOrchestrator) and new (HypothesisTrackerFusion) paths run the same
  inputs and measure assessment accuracy. The new fusion must match or exceed
  existing accuracy. Note: `PatternClassificationCalibrationTest` exercises
  only the Drools-tier static methods (`computeAllConfidences`,
  `mergeCumulative`, `applyRevisions`, `allAssessments`) — it does not invoke
  `CascadingPatternClassifier.classify()` or exercise tier 2. The new test
  must exercise the full cascade with both tier 2 implementations
  (old `DetectorOrchestrator` and new `HypothesisTrackerFusion`) side-by-side
  for direct comparison.

## Scope and Phasing

This spec is a comprehensive design document covering the full integration
vision. Issue #456 scopes to Phase 1 only (replay parsing, observation
extraction, strategy labelling, dataset generation). Subsequent phases are
tracked under separate issues:

| Phase | Issue | Scope |
|---|---|---|
| Phase 1: Data pipeline | casehubio/quarkmind#456 | Replay extraction, prior learning, CPT export |
| Phase 2: Network definition | To be filed | `SC2HypothesisNetworkBuilder`, `HypothesisTrackerFusion` |
| Phase 3: Live integration | To be filed | Cascade wiring, `LiveScoutingObservationExtractor`, `DroolsScoutingTask` |
| Phase 4: Response selection | To be filed | Response priors, response network, expert overrides |

**Phase 1: Data pipeline (prior learning)** — casehubio/quarkmind#456
- `ScoutingObservationExtractor` (replay mode)
- Prior frequency extraction from `.label.json` sidecars
- Observation CPT learning from labelled replays (with Laplace smoothing)
- JSON export of learned priors and CPTs

**Phase 2: Network definition**
- `SC2HypothesisNetworkBuilder` — opponent detection network
- `HypothesisTrackerFusion` — replacement for `DetectorOrchestrator`
- Unit tests against fixtures

**Phase 3: Live integration**
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
  Protocol `sc2data-train-times-require-calibration.md` applies — to be created
  as part of Phase 1, covering discretization boundary calibration methodology.
- **Response selection depends on opponent detection quality.** If opponent
  assessment is wrong, response selection amplifies the error. Path elimination
  and the resolution threshold (`0.95`) provide a gate — don't select a response
  until the opponent assessment is resolved or highly confident.
- **Expert overrides can conflict with learned priors.** The override is
  multiplicative, not a replacement — learned priors provide the base, expert
  knowledge scales specific entries in the distribution. At scale, the learned
  data dominates because the multiplicative adjustment is applied once at
  construction time while the data informs the base prior.

## Required Neocortex API Additions

This design requires two additions to `casehub-neocortex-inference-bayes`:

### 1. `HypothesisTracker.setPerTickEvidence()` / `clearPerTickEvidence()`

Sets evidence on all viable hypotheses' `CompositionInstance` objects without
multiplying hypothesis weights. Required for per-tick detector and Drools
evidence that refreshes every game tick.

```java
public void setPerTickEvidence(String slotName, String variableName, double[] distribution) {
    for (var entry : compositions.entrySet()) {
        if (statuses.get(entry.getKey()) == HypothesisStatus.VIABLE) {
            entry.getValue().setEvidence(slotName, variableName, distribution);
            entry.getValue().update();
        }
    }
}

public void clearPerTickEvidence(String slotName, String variableName) {
    for (var entry : compositions.entrySet()) {
        if (statuses.get(entry.getKey()) == HypothesisStatus.VIABLE) {
            entry.getValue().unsetEvidence(slotName, variableName);
            entry.getValue().update();
        }
    }
}
```

Semantics: `setPerTickEvidence()` replaces evidence without weight
multiplication. `observe()` / `observeHard()` remain the weight-updating path
for one-time observations. This preserves the tracker's core contract while
supporting the per-tick evidence replacement pattern that `DetectorOrchestrator`
uses via `EvidenceFusionEngine.setAllEvidence()`.

### 2. `CompositionInstance.sensitivity()`

Delegates sensitivity analysis to the slot's underlying `BayesInstance`:

```java
public double sensitivity(String slotName, String evidenceVariable, String targetVariable) {
    return getInstance(slotName).sensitivity(evidenceVariable, targetVariable);
}
```

Required for `HypothesisTrackerFusion.getSensitivities()` which provides the
`DetectorStatsPublished` CDI event with per-variable sensitivity values.
`BayesInstance.sensitivity()` already exists (line 834) — this is a
pass-through delegation.

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
