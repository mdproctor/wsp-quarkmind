# Detector CPT Learning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #450 — Detector CPTs: learn conditional probabilities from replay data instead of hand-tuning
**Issue group:** #450

**Goal:** Replace the uniform 0.8/0.2 CPTs in the Bayesian detector fusion network with data-driven CPTs learned from 149K labelled replays via `ParameterLearner.fit()`.

**Architecture:** A `@Tag("report")` test runs reconstituted replays through the production Java inference path (EmulatedGame → StrategyFeatures → BinaryDetectorRunner), collects observations at phase checkpoints (3min/5min/7min), calls `ParameterLearner.fit()`, and exports learned networks as `NetworkSpec` JSON resources. At startup, `LearnedCptLoader` loads these resources; `DetectorOrchestrator` uses learned networks when available, falls back to hand-tuned CPTs in dev/test.

**Tech Stack:** Java 21, Quarkus, neocortex `inference-bayes` (ParameterLearner, NetworkSpec, BayesNetworkBuilder), Jackson, ONNX Runtime

## Global Constraints

- All new classes in `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/` unless stated otherwise
- All new tests in `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/` unless stated otherwise
- Unit tests (no `@QuarkusTest`) unless CDI injection is needed
- TDD: write failing test → verify fail → implement → verify pass → commit
- Neocortex prerequisite: `CptSpec` Jackson annotations + `NetworkSpec.fromBayesNetwork()` must be implemented first (separate neocortex issues)

---

## Batch 0: Neocortex Prerequisites

### Task 0: CptSpec Jackson annotations and NetworkSpec.fromBayesNetwork()

**Repo:** `/Users/mdproctor/claude/casehub/neocortex` (cross-repo work)

**Files:**
- Modify: `inference-bayes/src/main/java/io/quarkmind/inference/bayes/CptSpec.java`
- Modify: `inference-bayes/src/main/java/io/quarkmind/inference/bayes/NetworkSpec.java`
- Create: `inference-bayes/src/test/java/io/quarkmind/inference/bayes/NetworkSpecSerializationTest.java`
- Test: `inference-bayes/src/test/java/io/quarkmind/inference/bayes/NetworkSpecSerializationTest.java`

**Interfaces:**
- Produces: `CptSpec` with `@JsonTypeInfo`/`@JsonSubTypes` for Jackson polymorphic deser
- Produces: `NetworkSpec.fromBayesNetwork(BayesNetwork network, String name, String description)` → `NetworkSpec`

- [ ] **Step 1: Write failing test for CptSpec Jackson round-trip**

```java
package io.quarkmind.inference.bayes;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class NetworkSpecSerializationTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void directCptRoundTrips() throws Exception {
        CptSpec original = new CptSpec.DirectCpt(new double[][]{{0.8, 0.2}, {0.3, 0.7}});
        String json = mapper.writeValueAsString(original);
        CptSpec deserialized = mapper.readValue(json, CptSpec.class);
        assertThat(deserialized).isInstanceOf(CptSpec.DirectCpt.class);
        assertThat(((CptSpec.DirectCpt) deserialized).values()).isDeepEqualTo(
            new double[][]{{0.8, 0.2}, {0.3, 0.7}});
    }

    @Test
    void uniformCptRoundTrips() throws Exception {
        CptSpec original = new CptSpec.Uniform();
        String json = mapper.writeValueAsString(original);
        CptSpec deserialized = mapper.readValue(json, CptSpec.class);
        assertThat(deserialized).isInstanceOf(CptSpec.Uniform.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl inference-bayes -Dtest=NetworkSpecSerializationTest -q`
Expected: FAIL — Jackson cannot polymorphically deserialize sealed `CptSpec`

- [ ] **Step 3: Add Jackson annotations to CptSpec**

Add to `CptSpec.java`:
```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = CptSpec.DirectCpt.class, name = "direct"),
    @JsonSubTypes.Type(value = CptSpec.NoisyOr.class, name = "noisy_or"),
    @JsonSubTypes.Type(value = CptSpec.Uniform.class, name = "uniform")
})
public sealed interface CptSpec permits CptSpec.DirectCpt, CptSpec.NoisyOr, CptSpec.Uniform {
```

Add Jackson dependency to `inference-bayes/pom.xml` if not present:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <scope>provided</scope>
</dependency>
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl inference-bayes -Dtest=NetworkSpecSerializationTest -q`
Expected: PASS

- [ ] **Step 5: Write failing test for NetworkSpec.fromBayesNetwork()**

Add to `NetworkSpecSerializationTest.java`:
```java
@Test
void fromBayesNetworkRoundTrips() {
    BayesNetwork original = BayesNetworkBuilder.create()
        .variable("Strategy", "RUSH", "MACRO").prior(0.4, 0.6)
        .variable("detector", "HIGH", "MEDIUM", "LOW")
            .given("Strategy").cpt(0.8, 0.15, 0.05,  0.1, 0.2, 0.7)
        .build();

    NetworkSpec spec = NetworkSpec.fromBayesNetwork(original, "test", "");
    assertThat(spec.variables()).hasSize(2);
    assertThat(spec.edges()).hasSize(1);

    BayesNetwork rebuilt = spec.toBayesNetwork();
    // Compare posteriors — set evidence and check they match
    var engine1 = new io.quarkmind.inference.bayes.EvidenceFusionEngine();
    engine1.initialize(original, "Strategy");
    engine1.setEvidence("detector", new double[]{0.9, 0.05, 0.05});
    engine1.update();
    double[] posterior1 = engine1.marginalize();

    var engine2 = new io.quarkmind.inference.bayes.EvidenceFusionEngine();
    engine2.initialize(rebuilt, "Strategy");
    engine2.setEvidence("detector", new double[]{0.9, 0.05, 0.05});
    engine2.update();
    double[] posterior2 = engine2.marginalize();

    assertThat(posterior2).containsExactly(posterior1, withinPercentage(0.01));
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn test -pl inference-bayes -Dtest=NetworkSpecSerializationTest#fromBayesNetworkRoundTrips -q`
Expected: FAIL — `fromBayesNetwork` does not exist

- [ ] **Step 7: Implement NetworkSpec.fromBayesNetwork()**

Add to `NetworkSpec.java`:
```java
public static NetworkSpec fromBayesNetwork(BayesNetwork network, String name, String description) {
    List<VariableSpec> variables = new ArrayList<>();
    List<EdgeSpec> edges = new ArrayList<>();

    for (var node : network) {
        BayesVariable<?> var = node.getContent();
        String varName = var.getName();
        List<String> outcomes = new ArrayList<>();
        for (Object o : var.getOutcomes()) {
            outcomes.add(o.toString());
        }
        CptSpec cpt = new CptSpec.DirectCpt(var.getProbabilityTable());
        variables.add(new VariableSpec(varName, outcomes, "", SpecStatus.ACCEPTED, cpt));

        for (String parent : var.getGiven()) {
            edges.add(new EdgeSpec(parent, varName, "", SpecStatus.ACCEPTED));
        }
    }

    return new NetworkSpec(name, variables, edges, description);
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn test -pl inference-bayes -Dtest=NetworkSpecSerializationTest -q`
Expected: PASS

- [ ] **Step 9: Write full NetworkSpec JSON round-trip test**

Add to `NetworkSpecSerializationTest.java`:
```java
@Test
void fullNetworkSpecJsonRoundTrip() throws Exception {
    BayesNetwork original = BayesNetworkBuilder.create()
        .variable("Strategy", "RUSH", "MACRO").prior(0.4, 0.6)
        .variable("det1", "HIGH", "MEDIUM", "LOW")
            .given("Strategy").cpt(0.8, 0.15, 0.05,  0.1, 0.2, 0.7)
        .variable("det2", "HIGH", "MEDIUM", "LOW")
            .given("Strategy").cpt(0.2, 0.3, 0.5,  0.7, 0.2, 0.1)
        .build();

    NetworkSpec spec = NetworkSpec.fromBayesNetwork(original, "test-net", "test");
    String json = mapper.writeValueAsString(spec);
    NetworkSpec deserialized = mapper.readValue(json, NetworkSpec.class);
    BayesNetwork rebuilt = deserialized.toBayesNetwork();

    assertThat(rebuilt.size()).isEqualTo(original.size());
}
```

- [ ] **Step 10: Run test and verify pass**

Run: `mvn test -pl inference-bayes -Dtest=NetworkSpecSerializationTest -q`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add inference-bayes/src/
git commit -m "feat: CptSpec Jackson annotations + NetworkSpec.fromBayesNetwork()

Add @JsonTypeInfo/@JsonSubTypes to CptSpec sealed interface for
polymorphic Jackson serialization. Add NetworkSpec.fromBayesNetwork()
static factory for converting learned networks to serializable spec.

Refs casehubio/quarkmind#450"
```

---

## Batch 1: Label Validation and Evidence Mapping

### Task 1: CatalogLabelMapper — validate labels against detector catalog

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/CatalogLabelMapper.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/CatalogLabelMapperTest.java`

**Interfaces:**
- Consumes: `DetectorCatalog.create()` → `DetectorCatalogConfig`
- Produces: `CatalogLabelMapper.isValidLabel(String label, Race enemyRace)` → `boolean`
- Produces: `CatalogLabelMapper.validLabelsForMatchup(Race enemyRace)` → `Set<String>`

- [ ] **Step 1: Write failing test**

```java
package io.quarkmind.sc2.detector;

import io.quarkmind.domain.Race;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CatalogLabelMapperTest {

    @Test
    void rushIsValidForAllRaces() {
        assertThat(CatalogLabelMapper.isValidLabel("RUSH", Race.TERRAN)).isTrue();
        assertThat(CatalogLabelMapper.isValidLabel("RUSH", Race.PROTOSS)).isTrue();
        assertThat(CatalogLabelMapper.isValidLabel("RUSH", Race.ZERG)).isTrue();
    }

    @Test
    void cannonRushIsOnlyValidForProtoss() {
        assertThat(CatalogLabelMapper.isValidLabel("CANNON_RUSH", Race.PROTOSS)).isTrue();
        assertThat(CatalogLabelMapper.isValidLabel("CANNON_RUSH", Race.TERRAN)).isFalse();
        assertThat(CatalogLabelMapper.isValidLabel("CANNON_RUSH", Race.ZERG)).isFalse();
    }

    @Test
    void unknownLabelIsInvalid() {
        assertThat(CatalogLabelMapper.isValidLabel("UNKNOWN", Race.TERRAN)).isFalse();
        assertThat(CatalogLabelMapper.isValidLabel("TERRAN_MARINE_RUSH", Race.TERRAN)).isFalse();
    }

    @Test
    void validLabelsForTerranMatchup() {
        var labels = CatalogLabelMapper.validLabelsForMatchup(Race.TERRAN);
        assertThat(labels).contains("RUSH", "MACRO_ECONOMY", "BIO_TIMING",
                "BANSHEE_HARASS", "AIR_SUPERIORITY", "MECH_PUSH", "TECH_RUSH");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CatalogLabelMapperTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement CatalogLabelMapper**

```java
package io.quarkmind.sc2.detector;

import io.quarkmind.domain.Race;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

public final class CatalogLabelMapper {

    private static final Map<Race, Set<String>> VALID_LABELS;

    static {
        VALID_LABELS = new HashMap<>();
        var config = DetectorCatalog.create();
        for (var def : config.detectors()) {
            for (var entry : def.positiveArchetypes().entrySet()) {
                VALID_LABELS.computeIfAbsent(entry.getKey(), k -> new HashSet<>())
                            .addAll(entry.getValue());
            }
        }
    }

    public static boolean isValidLabel(String label, Race enemyRace) {
        var labels = VALID_LABELS.get(enemyRace);
        return labels != null && labels.contains(label);
    }

    public static Set<String> validLabelsForMatchup(Race enemyRace) {
        return Set.copyOf(VALID_LABELS.getOrDefault(enemyRace, Set.of()));
    }

    private CatalogLabelMapper() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CatalogLabelMapperTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/
git commit -m "feat: CatalogLabelMapper validates labels against detector catalog

Refs #450"
```

### Task 2: Three-bucket evidence mapping in DetectorOrchestrator

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/DetectorOrchestrator.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/DetectorOrchestratorTest.java`

**Interfaces:**
- Consumes: `DetectorOrchestrator.classify(Map<String, Float>, List<EvidenceMarker>, long)` (existing)
- Produces: `initializeNetwork(List<String> detectorNames, GamePhase phase, Race enemyRace)` (new signature)
- Produces: Evidence mapping supports both 2-element (fallback) and 3-element (learned) arrays

- [ ] **Step 1: Write failing test for 3-element evidence mapping**

Add to `DetectorOrchestratorTest.java`:
```java
@Test
void threeOutcomeDetectorProducesAssessments() {
    var priors = Map.of("RUSH", 0.3, "MACRO", 0.7);
    var cpts = Map.<String, Map<String, Double>>of();
    var labelToArchetype = Map.of(
        "RUSH", StrategyArchetype.EARLY_RUSH,
        "MACRO", StrategyArchetype.MACRO_ECONOMY);

    // Build a 3-outcome learned network directly
    var builder = BayesNetworkBuilder.create();
    builder.variable("Strategy", "RUSH", "MACRO").prior(0.3, 0.7);
    builder.variable("det_vs_terran", "HIGH", "MEDIUM", "LOW")
           .given("Strategy").cpt(0.8, 0.15, 0.05,  0.1, 0.2, 0.7);
    var learnedNetwork = builder.build();

    var learnedNetworks = Map.of("EARLY_vs_terran", learnedNetwork);

    var registry = new DetectorRegistry(new DetectorCatalogConfig(List.of(
        new DetectorCatalogConfig.DetectorDef("det", GamePhase.EARLY,
            FeatureTier.COMPOSITION,
            Map.of(Race.TERRAN, List.of("RUSH")), "models/detectors"))));

    var orchestrator = new DetectorOrchestrator(registry, priors, cpts,
        labelToArchetype, 5, 0.95, 0.01, learnedNetworks);

    orchestrator.initializeNetwork(List.of("det_vs_terran"),
        GamePhase.EARLY, Race.TERRAN);

    var result = orchestrator.classify(
        Map.of("det_vs_terran", 0.9f), List.of(), 1000);

    assertThat(result).isNotEmpty();
    assertThat(result.get(0).archetype()).isEqualTo(StrategyArchetype.EARLY_RUSH);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DetectorOrchestratorTest#threeOutcomeDetectorProducesAssessments -q`
Expected: FAIL — constructor doesn't accept `learnedNetworks`, `initializeNetwork` doesn't accept phase/race

- [ ] **Step 3: Implement learned network support in DetectorOrchestrator**

Modify `DetectorOrchestrator.java`:

Add field: `private final Map<String, BayesNetwork> learnedNetworks;`

Add constructor overload:
```java
public DetectorOrchestrator(DetectorRegistry registry,
                            Map<String, Double> priors,
                            Map<String, Map<String, Double>> cpts,
                            Map<String, StrategyArchetype> labelToArchetype,
                            int windowSize,
                            double hardEvidenceThreshold,
                            double sensitivityThreshold,
                            Map<String, BayesNetwork> learnedNetworks) {
    this(registry, priors, cpts, labelToArchetype, windowSize,
         hardEvidenceThreshold, sensitivityThreshold);
    // store learnedNetworks in the new field (can't use this() and assign — use a private init)
}
```

Change `initializeNetwork` signature:
```java
public void initializeNetwork(List<String> newDetectorNames,
                              GamePhase phase, Race enemyRace) {
```

Inside `initializeNetwork`, check for learned network:
```java
String key = phase.name() + "_vs_" + enemyRace.name().toLowerCase();
BayesNetwork learned = learnedNetworks != null ? learnedNetworks.get(key) : null;
if (learned != null) {
    engine.initialize(learned, "Strategy");
    this.outcomeNames = engine.getOutcomeNames();
    this.activeDetectorNames = List.copyOf(newDetectorNames);
    previousDetectorOutputs.clear();
    initialized = true;
    return;
}
// existing build path
```

Change `classify()` evidence mapping to check outcome count:
```java
int detectorOutcomes = outcomeNames.length > 0 ? 3 : 2; // detected from network
// For 3-outcome detectors, use piecewise linear mapping
for (var entry : combined.entrySet()) {
    float p = entry.getValue();
    if (detectorOutcomes == 3) {
        evidence.put(entry.getKey(), toThreeElementLikelihood(p));
    } else {
        // existing 2-element mapping
    }
}
```

Add piecewise linear method:
```java
static double[] toThreeElementLikelihood(float score) {
    if (score >= 0.95) return new double[]{1.0, 0.0, 0.0};
    if (score <= 0.05) return new double[]{0.0, 0.0, 1.0};
    double high, medium, low;
    if (score >= 0.7) {
        double t = (score - 0.7) / 0.3;
        high = 0.5 + 0.5 * t; medium = 0.5 - 0.5 * t; low = 0.0;
    } else if (score >= 0.3) {
        double t = (score - 0.3) / 0.4;
        high = 0.0; medium = 0.5 + 0.5 * Math.sin(Math.PI * (t - 0.5)); low = 1.0 - medium;
        // Simpler: piecewise linear
        if (score >= 0.5) { high = 0.0; medium = 1.0; low = 0.0; }
        else { high = 0.0; medium = 0.5; low = 0.5; }
    } else {
        double t = score / 0.3;
        high = 0.0; medium = 0.5 * t; low = 1.0 - 0.5 * t;
    }
    return new double[]{high, medium, low};
}
```

Actually, use the spec's piecewise linear table directly:
```java
static double[] toThreeElementLikelihood(float score) {
    if (score >= 0.95) return new double[]{1.0, 0.0, 0.0};
    if (score <= 0.05) return new double[]{0.0, 0.0, 1.0};
    double high, medium, low;
    if (score > 0.7) {
        double t = (score - 0.7) / 0.3;
        high = 0.5 * t + 0.5 * (1.0 - t); // lerp: 0.5 at 0.7, 1.0 at 1.0
        high = 0.5 + 0.5 * t;
        medium = 0.5 - 0.5 * t;
        low = 0.0;
    } else if (score > 0.5) {
        double t = (score - 0.5) / 0.2;
        high = 0.0;
        medium = 1.0;
        low = 0.0;
    } else if (score > 0.3) {
        double t = (score - 0.3) / 0.2;
        high = 0.0;
        medium = 0.5 + 0.5 * t;
        low = 0.5 - 0.5 * t;
    } else {
        double t = score / 0.3;
        high = 0.0;
        medium = 0.5 * t;
        low = 1.0 - 0.5 * t;
    }
    return new double[]{high, medium, low};
}
```

- [ ] **Step 4: Update existing initializeNetwork call sites**

In `CascadingPatternClassifier.classify()`, change:
```java
detectorOrchestrator.initializeNetwork(List.copyOf(allKeys));
```
to:
```java
detectorOrchestrator.initializeNetwork(List.copyOf(allKeys), phase, enemyRace);
```

Backward-compat: keep old `initializeNetwork(List<String>)` as a delegate that passes `null` phase/race (uses build fallback).

- [ ] **Step 5: Run all detector tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=DetectorOrchestratorTest,SC2NetworkBuilderTest,CascadingPatternClassifierTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-sc2/src/
git commit -m "feat: DetectorOrchestrator supports learned networks with 3-outcome evidence

Add learned network lookup by phase+race key, piecewise linear
3-element evidence mapping, backward-compatible with 2-outcome
fallback networks.

Refs #450"
```

---

## Batch 2: Loading and Profile-Gated Fallback

### Task 3: LearnedCptResource record and LearnedCptLoader CDI bean

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/LearnedCptResource.java`
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/sc2/detector/LearnedCptLoader.java`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/LearnedCptLoaderTest.java`

**Interfaces:**
- Consumes: `NetworkSpec.toBayesNetwork()` (neocortex)
- Produces: `LearnedCptResource(String matchup, String phase, LearnedFrom learnedFrom, NetworkSpec network)`
- Produces: `LearnedCptLoader.loadFromClasspath()` → `Map<String, BayesNetwork>`

- [ ] **Step 1: Write failing test for LearnedCptResource deserialization**

```java
package io.quarkmind.sc2.detector;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.inference.bayes.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class LearnedCptLoaderTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void learnedCptResourceRoundTrips() throws Exception {
        var network = BayesNetworkBuilder.create()
            .variable("Strategy", "RUSH", "MACRO").prior(0.5, 0.5)
            .variable("det", "HIGH", "MEDIUM", "LOW")
                .given("Strategy").cpt(0.7, 0.2, 0.1,  0.1, 0.3, 0.6)
            .build();

        var spec = NetworkSpec.fromBayesNetwork(network, "test", "");
        var resource = new LearnedCptResource("vs_terran", "EARLY",
            new LearnedCptResource.LearnedFrom(1000, 2000, "abc123", "2026-10-25T12:00:00Z"),
            spec);

        String json = mapper.writeValueAsString(resource);
        var deserialized = mapper.readValue(json, LearnedCptResource.class);

        assertThat(deserialized.matchup()).isEqualTo("vs_terran");
        assertThat(deserialized.phase()).isEqualTo("EARLY");
        assertThat(deserialized.network().variables()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=LearnedCptLoaderTest -q`
Expected: FAIL — classes don't exist

- [ ] **Step 3: Create LearnedCptResource record**

```java
package io.quarkmind.sc2.detector;

import io.quarkmind.inference.bayes.NetworkSpec;

public record LearnedCptResource(
    String matchup,
    String phase,
    LearnedFrom learnedFrom,
    NetworkSpec network
) {
    public record LearnedFrom(
        int replayCount,
        int observationCount,
        String extractorVersion,
        String generatedAt
    ) {}
}
```

- [ ] **Step 4: Create LearnedCptLoader**

```java
package io.quarkmind.sc2.detector;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.Race;
import io.quarkmind.inference.bayes.BayesNetwork;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.io.InputStream;
import java.util.HashMap;
import java.util.Map;

@ApplicationScoped
public class LearnedCptLoader {

    private static final Logger log = Logger.getLogger(LearnedCptLoader.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @ConfigProperty(name = "quarkmind.detector.require-learned-cpts", defaultValue = "false")
    boolean requireLearnedCpts;

    public Map<String, BayesNetwork> loadFromClasspath() {
        Map<String, BayesNetwork> result = new HashMap<>();
        for (GamePhase phase : GamePhase.values()) {
            for (Race race : new Race[]{Race.TERRAN, Race.PROTOSS, Race.ZERG}) {
                String resourcePath = "classifier/learned-cpts-"
                    + phase.name().toLowerCase() + "-vs-" + race.name().toLowerCase() + ".json";
                String key = phase.name() + "_vs_" + race.name().toLowerCase();
                try (InputStream is = getClass().getClassLoader().getResourceAsStream(resourcePath)) {
                    if (is == null) {
                        if (requireLearnedCpts) {
                            throw new IllegalStateException(
                                "Learned CPTs required but missing: " + resourcePath);
                        }
                        log.warnf("Learned CPTs not found: %s — using hand-tuned fallback", resourcePath);
                        continue;
                    }
                    var resource = MAPPER.readValue(is, LearnedCptResource.class);
                    validate(resource, phase, race);
                    result.put(key, resource.network().toBayesNetwork());
                    log.infof("Loaded learned CPTs: %s (%d replays, %d observations)",
                        key, resource.learnedFrom().replayCount(),
                        resource.learnedFrom().observationCount());
                } catch (IllegalStateException e) {
                    throw e;
                } catch (Exception e) {
                    if (requireLearnedCpts) {
                        throw new IllegalStateException("Failed to load " + resourcePath, e);
                    }
                    log.warnf("Failed to load %s: %s — using fallback", resourcePath, e.getMessage());
                }
            }
        }
        return Map.copyOf(result);
    }

    private void validate(LearnedCptResource resource, GamePhase phase, Race race) {
        var spec = resource.network();
        for (var varSpec : spec.variables()) {
            if (varSpec.cptSpec() instanceof io.quarkmind.inference.bayes.CptSpec.DirectCpt direct) {
                for (double[] row : direct.values()) {
                    double sum = 0;
                    for (double v : row) sum += v;
                    if (Math.abs(sum - 1.0) > 0.01) {
                        throw new IllegalStateException(
                            "CPT row sums to " + sum + " for variable " + varSpec.name());
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=LearnedCptLoaderTest -q`
Expected: PASS

- [ ] **Step 6: Add profile config properties**

Add to `quarkmind-sc2/src/main/resources/application.properties`:
```properties
%sc2.quarkmind.detector.require-learned-cpts=true
%emulated.quarkmind.detector.require-learned-cpts=true
%coach.quarkmind.detector.require-learned-cpts=true
```

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/
git commit -m "feat: LearnedCptLoader loads NetworkSpec JSON with profile-gated fallback

Fail fast in sc2/emulated/coach profiles if learned CPTs are missing.
Warn and fall back to hand-tuned defaults in mock/test profiles.

Refs #450"
```

### Task 4: Wire LearnedCptLoader into CascadingPatternClassifier

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/plugin/scouting/CascadingPatternClassifier.java`
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/plugin/scouting/CascadingPatternClassifierTest.java`

**Interfaces:**
- Consumes: `LearnedCptLoader.loadFromClasspath()` → `Map<String, BayesNetwork>`
- Consumes: `DetectorOrchestrator(registry, priors, cpts, labelToArchetype, windowSize, hardEvidenceThreshold, sensitivityThreshold, learnedNetworks)`

- [ ] **Step 1: Write failing test for learned networks passed to orchestrator**

Add to `CascadingPatternClassifierTest.java` a test that constructs a `CascadingPatternClassifier` with learned networks and verifies classification uses them.

- [ ] **Step 2: Modify buildDefaultOrchestrator to accept learned networks**

Change `buildDefaultOrchestrator` to accept `Map<String, BayesNetwork> learnedNetworks` as a parameter and pass it through to the `DetectorOrchestrator` constructor.

- [ ] **Step 3: Add constructor overload that accepts LearnedCptLoader**

Add a CDI-friendly constructor that injects `BinaryDetectorRunner` and `LearnedCptLoader`, calls `loader.loadFromClasspath()`, and passes the result to the orchestrator.

- [ ] **Step 4: Run all CascadingPatternClassifier tests**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CascadingPatternClassifierTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/
git commit -m "feat: wire LearnedCptLoader into CascadingPatternClassifier

Refs #450"
```

---

## Batch 3: CPT Learning Export Pipeline

### Task 5: CptLearningExportTest — batch extraction + learning + export

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/CptLearningExportTest.java`

**Interfaces:**
- Consumes: `BinaryDetectorRunner(Path modelsDir)` (non-CDI constructor)
- Consumes: `ParameterLearner.fit(BayesNetwork, List<Map<String, String>>)`
- Consumes: `NetworkSpec.fromBayesNetwork(BayesNetwork, String, String)`
- Consumes: `CatalogLabelMapper.isValidLabel(String, Race)`
- Produces: JSON files at `quarkmind-sc2/src/main/resources/classifier/learned-cpts-{phase}-vs-{race}.json`
- Produces: Observation JSON files at `quarkmind-sc2/src/test/resources/cpt-observations/`

This is the largest task — the batch pipeline that processes replays, runs detectors, learns CPTs, and exports. It follows the `ReconstitutionExportTest` pattern but with additional ONNX + learning steps.

- [ ] **Step 1: Write test skeleton**

```java
package io.quarkmind.sc2.detector;

import io.quarkmind.domain.GamePhase;
import io.quarkmind.domain.Race;
import io.quarkmind.inference.bayes.*;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

@Tag("report")
class CptLearningExportTest {

    static final int EARLY_FRAME = 4031;
    static final int MID_FRAME   = 6719;
    static final int LATE_FRAME  = 9407;

    @Test
    void learnAndExportCpts() {
        // 1. Discover reconstituted replays + label sidecars
        // 2. For each matchup × phase, collect observations
        // 3. Build structure network, call ParameterLearner.fit()
        // 4. Export as LearnedCptResource JSON
    }
}
```

- [ ] **Step 2: Implement replay discovery and label reading**

Read reconstituted JSON files from `../quarkmind-classifier/data/reconstituted/`, pair with `.label.json` sidecars, filter by non-UNKNOWN labels, group by matchup.

- [ ] **Step 3: Implement feature extraction at checkpoints**

For each replay, tick EmulatedGame to checkpoint frames, extract StrategyFeatures, run BinaryDetectorRunner.

- [ ] **Step 4: Implement observation assembly with 3-bucket discretization**

Discretize detector outputs: <0.3 → "LOW", 0.3–0.7 → "MEDIUM", >0.7 → "HIGH". Log per-detector bucket distributions.

- [ ] **Step 5: Implement ParameterLearner.fit() and export**

Build structure network per matchup × phase, call `ParameterLearner.fit()`, convert to `NetworkSpec`, wrap in `LearnedCptResource`, serialize to JSON.

- [ ] **Step 6: Run the export test**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=CptLearningExportTest -q`
Expected: PASS — JSON files written to `src/main/resources/classifier/`

- [ ] **Step 7: Commit generated resources**

```bash
git add quarkmind-sc2/src/main/resources/classifier/learned-cpts-*.json
git add quarkmind-sc2/src/test/
git commit -m "feat: CptLearningExportTest — learn CPTs from replay data

Batch pipeline processes reconstituted replays through production
inference path, learns CPTs via ParameterLearner.fit(), exports
as NetworkSpec JSON resources (9 files: 3 matchups × 3 phases).

Refs #450"
```

---

## Batch 4: Validation and Cleanup

### Task 6: CptLearningValidationTest — accuracy comparison

**Files:**
- Create: `quarkmind-sc2/src/test/java/io/quarkmind/sc2/detector/CptLearningValidationTest.java`

**Interfaces:**
- Consumes: Learned CPT JSON resources from Task 5
- Consumes: `DetectorOrchestrator` with learned vs hand-tuned networks
- Produces: Accuracy comparison report (stdout)

- [ ] **Step 1: Write validation test**

```java
@Tag("report")
class CptLearningValidationTest {

    @Test
    void learnedCptsOutperformHandTuned() {
        // Load held-out test observations
        // Run classification with learned CPTs → top-1 accuracy
        // Run classification with 0.8/0.2 CPTs → top-1 accuracy
        // Assert learned >= handTuned + 0.02 (2% margin)
    }
}
```

- [ ] **Step 2: Implement comparison logic**

For each held-out observation, set evidence on both learned and hand-tuned networks, compare the MAP strategy to the ground-truth label.

- [ ] **Step 3: Run validation**

Run: `mvn test -pl quarkmind-sc2 -Preport -Dtest=CptLearningValidationTest -q`
Expected: PASS — learned CPTs improve accuracy by ≥2%

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/
git commit -m "feat: CptLearningValidationTest — verify learned CPTs improve accuracy

Compares top-1 classification accuracy of learned vs hand-tuned
0.8/0.2 CPTs on held-out test set. Asserts ≥2% improvement.

Refs #450"
```

### Task 7: Full integration test — existing tests pass with learned CPTs

**Files:**
- No new files — run existing test suite

- [ ] **Step 1: Run full quarkmind-sc2 test suite**

Run: `mvn test -pl quarkmind-sc2 -q`
Expected: PASS — all existing tests still work (fallback path active in test profile)

- [ ] **Step 2: Run CascadingPatternClassifier tests specifically**

Run: `mvn test -pl quarkmind-sc2 -Dtest=CascadingPatternClassifierTest -q`
Expected: PASS

- [ ] **Step 3: Commit if any fixups were needed**

---

## References

- [2026-10-25-detector-cpt-learning-design.md] — design spec this plan implements
- `SC2NetworkBuilder.java` — current network construction
- `CascadingPatternClassifier.java:385-415` — hand-coded CPT assignment
- `DetectorCatalog.java` — 9 detector definitions
- `DetectorOrchestrator.java` — Bayesian fusion consumer
- `BinaryDetectorRunner.java` — production ONNX inference
- `ParameterLearner.java` (neocortex) — MLE batch learning
- `NetworkSpec.java` (neocortex) — persistence format
- `ReconstitutionExportTest.java` — batch replay processing pattern
- GitHub #450 — focal issue
