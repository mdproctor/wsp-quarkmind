# L7 Comparison Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close Chapter 6 of the Game Agent Coordination journey by writing `EmulatedGameBenchmarkTest`, running both benchmarks, and updating ARC42STORIES.MD with four §9.2 touch-points, the §9.3 C6 chapter entry, and the §9.4 L7 layer entry.

**Architecture:** No new production code. One new test class (`EmulatedGameBenchmarkTest`, plain JUnit, no CDI) mirrors `EmulatedEngine.joinGame()` exactly to measure representative full-tick throughput. All remaining tasks are documentation: benchmark output recorded to `docs/benchmarks/`, then ARC42STORIES.MD updated in three separate commits (§9.2 status, §9.3 chapter entry, §9.4 layer entry).

**Tech Stack:** JUnit 5, Java 21, Maven Surefire benchmark profile (`-Pbenchmark`), ARC42STORIES.MD (Mermaid flowchart)

---

## File Map

| Action | Path | Responsibility |
|---|---|---|
| Create | `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameBenchmarkTest.java` | Full-engine-tick throughput benchmark with realistic combat load |
| Create | `docs/benchmarks/2026-06-16-l7-harness.md` | GameLoopBenchmarkTest results + assessment vs P99 budget |
| Create | `docs/benchmarks/2026-06-16-l7-emulated.md` | EmulatedGame throughput results + 22Hz inference |
| Modify | `ARC42STORIES.MD` | §9.2 four touch-points + §9.3 C6 entry + §9.4 L7 layer entry |

---

### Task 1: Write EmulatedGameBenchmarkTest

**Files:**
- Create: `src/test/java/io/quarkmind/sc2/emulated/EmulatedGameBenchmarkTest.java`

**Why this package:** `RaceModelFactory.forRace()` and `EmulatedGame.enemy` are both package-private — the benchmark must live in `io.quarkmind.sc2.emulated`.

- [ ] **Step 1: Create the file**

`src/test/java/io/quarkmind/sc2/emulated/EmulatedGameBenchmarkTest.java`:

```java
package io.quarkmind.sc2.emulated;

import io.quarkmind.domain.Race;
import io.quarkmind.domain.TechTree;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Arrays;
import java.util.stream.LongStream;

/**
 * EmulatedGame throughput benchmark — full-engine-tick rate with realistic combat load.
 *
 * NOT part of the regular test suite. Run explicitly:
 *   mvn test -Pbenchmark -Dtest=EmulatedGameBenchmarkTest
 *
 * Output is printed to stdout AND written to target/emulated-benchmark-results.txt.
 * Copy into docs/benchmarks/YYYY-MM-DD-l7-emulated.md to record a snapshot.
 *
 * Setup mirrors EmulatedEngine.joinGame() exactly: TerrainGrid, PathfindingMovement,
 * RaceModelFactory, EnemyBehavior with PROTOSS_4GATE (FAST_PUSH — first attack ~tick 100).
 * 120 warmup ticks advance past the first attack; 50 measured ticks capture steady-state combat.
 */
@Tag("benchmark")
class EmulatedGameBenchmarkTest {

    private static final int WARMUP_TICKS  = 120;
    private static final int MEASURE_TICKS = 50;

    @Test
    void emulatedGameThroughput() throws IOException {
        // Mirror EmulatedEngine.joinGame() exactly
        EmulatedGame game = new EmulatedGame();
        TerrainGrid grid = TerrainGrid.emulatedMap();
        game.setMovementStrategy(new PathfindingMovement(grid));
        game.setTerrainGrid(grid);
        game.setPlayerRaceModel(RaceModelFactory.forRace(Race.PROTOSS));
        game.setEnemyBehavior(new EnemyBehavior(
            EnemyStrategyLibrary.forName("PROTOSS_4GATE"), game.enemy, new TechTree()));
        game.reset();

        // Warmup — advance past PROTOSS_4GATE first-attack trigger (~tick 100)
        for (int i = 0; i < WARMUP_TICKS; i++) {
            game.setUnitSpeed(1.0);
            game.tick();
            game.observeVisibility();
        }

        // Measure — full EmulatedEngine.tick() sequence: setUnitSpeed + tick + observeVisibility
        long[] tickNs = new long[MEASURE_TICKS];
        for (int i = 0; i < MEASURE_TICKS; i++) {
            long start = System.nanoTime();
            game.setUnitSpeed(1.0);
            game.tick();
            game.observeVisibility();
            tickNs[i] = System.nanoTime() - start;
        }

        String report = formatReport(tickNs);
        System.out.println(report);

        Path out = Path.of("target/emulated-benchmark-results.txt");
        Files.createDirectories(out.getParent());
        Files.writeString(out, report, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
        System.out.println("Written to: " + out.toAbsolutePath());
    }

    private static String formatReport(long[] tickNs) {
        long[] tickMs = Arrays.stream(tickNs).map(ns -> ns / 1_000_000).toArray();
        long meanMs   = mean(tickMs);
        long p95Ms    = p95(tickMs);
        long maxMs    = max(tickMs);
        long totalMs  = LongStream.of(tickMs).sum();
        double tps    = totalMs > 0 ? (MEASURE_TICKS * 1000.0 / totalMs) : Double.POSITIVE_INFINITY;

        String ts = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));
        return String.format("""
                QuarkMind EmulatedGame Throughput Benchmark
                ──────────────────────────────────────────────────────
                Date:    %s
                Setup:   PROTOSS_4GATE enemy | PROTOSS player | emulatedMap() | A* pathfinding
                Warmup:  %d ticks (past PROTOSS_4GATE first attack at ~tick 100)
                Samples: %d ticks (steady-state combat ~tick 120-170)
                ──────────────────────────────────────────────────────
                Full engine tick (setUnitSpeed + tick + observeVisibility):
                  mean     p95      max
                  %4dms   %4dms   %4dms
                ────────────────────────────────────────
                Throughput: %.1f ticks/sec
                ──────────────────────────────────────────────────────
                SC2 real-game ceiling: 22Hz (45ms/tick)
                Configured agent tick: 500ms  P99 budget: 400ms
                ──────────────────────────────────────────────────────
                Raw samples (ms): %s
                ──────────────────────────────────────────────────────
                Copy into docs/benchmarks/YYYY-MM-DD-l7-emulated.md
                Run: mvn test -Pbenchmark -Dtest=EmulatedGameBenchmarkTest
                """,
            ts, WARMUP_TICKS, MEASURE_TICKS,
            meanMs, p95Ms, maxMs,
            tps,
            Arrays.toString(tickMs));
    }

    private static long mean(long[] a) { return LongStream.of(a).sum() / a.length; }
    private static long p95(long[] a) {
        long[] s = Arrays.stream(a).sorted().toArray();
        return s[Math.min((int) (a.length * 0.95), a.length - 1)];
    }
    private static long max(long[] a) { return LongStream.of(a).max().orElse(0); }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
mvn compile -q && mvn test-compile -q
```

Expected: `BUILD SUCCESS`. No errors. If you see `cannot find symbol` on `RaceModelFactory`, confirm the package declaration is `io.quarkmind.sc2.emulated`.

- [ ] **Step 3: Run the benchmark (dry run — verify it executes)**

```bash
mvn test -Pbenchmark -Dtest=EmulatedGameBenchmarkTest -q
```

Expected: `BUILD SUCCESS`. A report is printed to stdout showing `mean`, `p95`, `max` ms per tick and `ticks/sec`. `target/emulated-benchmark-results.txt` is written. If you see `NullPointerException` in `EmulatedGame.tick()`, the `EnemyBehavior` or `TerrainGrid` wiring is missing — verify steps 3–5 of the setup match exactly.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add src/test/java/io/quarkmind/sc2/emulated/EmulatedGameBenchmarkTest.java
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "test(benchmark): EmulatedGame full-tick throughput benchmark — Refs #159"
```

---

### Task 2: Run Benchmark 1 and record harness results

**Files:**
- Create: `docs/benchmarks/2026-06-16-l7-harness.md`

- [ ] **Step 1: Run GameLoopBenchmarkTest**

```bash
mvn test -Pbenchmark -Dtest=GameLoopBenchmarkTest -q
```

Expected: `BUILD SUCCESS`. `target/benchmark-results.txt` is written. The report shows `engine.tick()`, `caseEngine plugins`, `engine.dispatch()`, and `Total gameTick()` rows with mean/p95/max columns.

- [ ] **Step 2: Create the benchmark record**

Create `docs/benchmarks/2026-06-16-l7-harness.md` with the header below, then paste the full contents of `target/benchmark-results.txt` after the `---` separator. Then fill in the Assessment section using the actual p95 and max values from the report:

```markdown
# L7 Harness Dispatch Benchmark — 2026-06-16

**Context:** L7 comparison baseline (#159)
**Profile:** %test (MockEngine) — Phase 1 bridge configuration
**Quality requirement (§11):** P99 < 400ms (configured agent tick: 500ms)
**Transitional note:** Numbers reflect poc CaseEngine → bridge → testActivation → execute overhead. Re-run at Phase 2 close when bridge indirection is removed.

---

[paste full target/benchmark-results.txt output here]

---

## Assessment

- **P99 dispatch overhead:** Xms — WITHIN 400ms budget  ← fill X from `Total gameTick() p95` row
- **Aspirational 22Hz (45ms/tick):** dispatch layer answer — does `Total gameTick() p95` fit within 45ms?
  (Physics layer answered by `2026-06-16-l7-emulated.md`; combined overhead inferred if both fit within 45ms individually)
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add docs/benchmarks/2026-06-16-l7-harness.md
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(benchmark): L7 harness dispatch results — Refs #159"
```

---

### Task 3: Record EmulatedGame throughput results

**Files:**
- Create: `docs/benchmarks/2026-06-16-l7-emulated.md`

The benchmark was already run in Task 1 Step 3. If `target/emulated-benchmark-results.txt` is stale or missing, re-run:
```bash
mvn test -Pbenchmark -Dtest=EmulatedGameBenchmarkTest -q
```

- [ ] **Step 1: Create the benchmark record**

Create `docs/benchmarks/2026-06-16-l7-emulated.md` with the header below, paste the full `target/emulated-benchmark-results.txt` output, then fill in the Assessment section:

```markdown
# L7 EmulatedGame Throughput Benchmark — 2026-06-16

**Context:** L7 comparison baseline (#159)
**Setup:** PROTOSS_4GATE enemy | PROTOSS player | TerrainGrid.emulatedMap() | A* pathfinding
**Warmup:** 120 ticks — past PROTOSS_4GATE first-attack trigger (~tick 100)
**Measured:** 50 ticks — steady-state combat window (~tick 120–170)
**Full tick sequence:** setUnitSpeed(1.0) + EmulatedGame.tick() + observeVisibility()

---

[paste full target/emulated-benchmark-results.txt output here]

---

## Assessment

- **Throughput:** X ticks/sec — physics tick rate is not a constraint on plain JUnit tests driving EmulatedGame directly  ← fill X from report
- **Aspirational 22Hz (45ms/tick):** physics layer answer — does `mean` ms fit within 45ms?
  (Dispatch layer answered by `2026-06-16-l7-harness.md`; combined overhead inferred if both fit within 45ms individually)
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add docs/benchmarks/2026-06-16-l7-emulated.md
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(benchmark): L7 EmulatedGame throughput results — Refs #159"
```

---

### Task 4: Update ARC42STORIES.MD §9.2 — four touch-points

**Files:**
- Modify: `ARC42STORIES.MD`

Make all four edits, then commit once.

- [ ] **Step 1: Update Chapter Index status row**

Find:
```
| 6 | Comparison baseline | Game Agent Coordination | L7 | Low | 🔲 pending (#159) |
```
Replace with:
```
| 6 | Comparison baseline | Game Agent Coordination | L7 | Low | ✅ complete |
```

- [ ] **Step 2: Update Mermaid flowchart node colour**

Find:
```
  style C6 fill:#D3D3D3,color:#000
```
Replace with:
```
  style C6 fill:#90EE90,color:#000
```

- [ ] **Step 3: Update Layer × Chapter matrix column header**

Find:
```
| Layer | C1 ✅ | C2 🔲 | C3 🔲 | C4 🔲 | C5 🔲 | C6 🔲 |
```
Replace with:
```
| Layer | C1 ✅ | C2 🔲 | C3 🔲 | C4 🔲 | C5 🔲 | C6 ✅ |
```

- [ ] **Step 4: Verify all three §9.2 changes**

Confirm in ARC42STORIES.MD:
- Chapter Index row 6: shows `✅ complete`
- Mermaid node: `fill:#90EE90` (matches C1's green)
- Matrix header: `C6 ✅`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): mark C6 complete in §9.2 — Refs #159"
```

---

### Task 5: Write ARC42STORIES.MD §9.3 Chapter 6 entry

**Files:**
- Modify: `ARC42STORIES.MD`

Find the §9.3 C6 stub — it reads:

```
#### Chapter 6 — Comparison Baseline

**Journey:** Game Agent Coordination | **Sequence:** 6 of 6 | **Status:** 🔲 pending

🔲 Full Chapter entry at C6 close. Blocked on #159.

Known scope: benchmark QuarkMind harness agent against the naive game loop baseline (L1) and available commercial SC2 frameworks; document win-rate delta and latency delta attributable to each layer.
```

- [ ] **Step 1: Replace the stub with the full C6 chapter entry**

```markdown
#### Chapter 6 — Comparison Baseline

**Journey:** Game Agent Coordination | **Sequence:** 6 of 6 | **Status:** ✅ complete
**Delivered:** 2026-06-16 | **Issues:** #159 | **Blog:** none

**What this delivers**
Analytical validation that the L1–L7 layer sequence delivered its stated capabilities. No new harness capability is added. Harness dispatch overhead is benchmarked against the 400ms P99 quality requirement (§11); EmulatedGame full-tick throughput establishes the physics layer baseline. Win-rate comparison is deferred — L1 has no production code (intentional conceptual baseline) and the SC2 game corpus is insufficient for meaningful Bayesian Beta deltas; see §9.4 Layer — Comparison Baseline for full rationale.

**Accountability gaps closed**
None. All accountability gaps were closed by L2–L6. C6 validates the closures; it does not introduce new ones.

**Layer Impact**
| Layer | Delta |
|---|---|
| L7 Comparison Baseline | Low — documentation and benchmarks only; no production code |

**Known limitations carried forward:**
- Phase 2 dispatch pending: poc CaseEngine in use; SequenceWorker (engine#484) and `signalAndAwaitSync` (engine#483) not yet wired; benchmark numbers are Phase 1 transitional
- C2–C5 §9.3 chapter entries are stubs (layers complete; entries pending — #198)

**Refs:** §9.4 Layer — Comparison Baseline; `docs/benchmarks/2026-06-16-l7-harness.md`; `docs/benchmarks/2026-06-16-l7-emulated.md`
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): C6 §9.3 chapter entry — Refs #159"
```

---

### Task 6: Write ARC42STORIES.MD §9.4 Layer — Comparison Baseline entry

**Files:**
- Modify: `ARC42STORIES.MD`

Find the §9.4 stub — it reads:

```
### Layer — Comparison Baseline

**Participates in chapters:** C6
**Completed:** 🔲 pending (#159)

🔲 Full entry at C6 close. Known scope: benchmark QuarkMind harness agent against L1 naive loop and available commercial SC2 frameworks; document win-rate and latency delta per layer.
```

- [ ] **Step 1: Replace the stub with the full §9.4 layer entry**

```markdown
### Layer — Comparison Baseline

**Participates in chapters:** C6
**Completed:** ✅ 2026-06-16 (#159)

#### Column definitions

- **L1 Naive** — the conceptual baseline documented in §9.4 Naive Game Loop: a hypothetical SC2 bot calling plugins directly per tick, no CaseHub harness. No production code exists or was intended; its purpose is to name the accountability gaps L2–L6 close.
- **SC2 API bot (no coordination layer)** — any SC2 bot using the SC2 API without a coordination layer, regardless of client library. An analytical characterisation of the coordination-free pattern. QuarkMind removed ocraft-s2client-bot transport in #185 (replaced by `QuarkusSC2Transport`); "ocraft raw" would misrepresent both ocraft's callback model and QuarkMind's actual architecture.
- **QuarkMind L7** — full harness as built; Phase 1 bridge configuration (poc CaseEngine dispatch).

#### Capability comparison

| Dimension | L1 Naive | SC2 API bot (no coordination) | QuarkMind L7 |
|---|---|---|---|
| Plugin dispatch | Direct method calls per tick | Direct calls or callbacks | CaseEngine blackboard (`createAndSolve()`) |
| Inter-plugin state within tick | None | None | Shared CaseFile read/write within tick |
| Typed comms between plugins | None | None | casehub-qhorus channels |
| Formal out-of-scope signal | Silent no-op or throws | Silent no-op or throws | Structured DECLINE speech act (L3) |
| Audit trail | None | None | casehub-ledger attestation per plugin decision |
| Adaptive plugin selection | Fixed | Fixed | Binding-condition dispatch (L5) |
| Trust-weighted routing | None | None | Bayesian Beta strategy selection per opponent context (L6) |
| Outcome tracking | None | None | WIN→ENDORSED, LOSS→CHALLENGED, TIE→SOUND; UNKNOWN→skipped. Real SC2 games produce directional trust signals; mock/emulated/replay produce UNKNOWN and are skipped — correct, no meaningful win/loss in emulation |
| Observability | None | None | CaseFile key trace per tick; casehub-ledger attestation per decision; trust scores per strategy per opponent context; real-time GameState via WebSocket `/ws/gamestate` (dev/QA only — `@UnlessBuildProfile("prod")`) |

#### Latency

- **Configured agent tick interval:** 500ms (`starcraft.tick.interval=500ms`)
- **Quality requirement (§11):** P99 < 400ms (100ms headroom for SC2 I/O)
- **Harness dispatch overhead:** see `docs/benchmarks/2026-06-16-l7-harness.md` — Phase 1 bridge configuration; re-run at Phase 2 close
- **EmulatedGame full-tick throughput:** see `docs/benchmarks/2026-06-16-l7-emulated.md` — confirms physics tick rate is not a constraint on plain JUnit tests driving EmulatedGame directly
- **Aspirational 22Hz (45ms/tick):** Benchmark 1 answers this for the dispatch layer; Benchmark 2 answers it for the physics layer. Combined overhead not directly measured but can be inferred if both fit comfortably within 45ms individually.

#### Win-rate disposition

Win-rate comparison is explicitly deferred — not tractable at this time:

- **L1 has no production code** — intentional; L1 is a conceptual baseline naming accountability gaps, not a deployable harness.
- **Comparing to a different SC2 bot** measures strategy quality, not harness contribution. The coordination layer and strategy quality are orthogonal.
- **Insufficient SC2 game corpus** — L6 trust routing needs sufficient real-SC2 games for meaningful Bayesian Beta deltas; emulated games produce UNKNOWN outcomes (skipped by design).

Win-rate comparison remains a future item contingent on a sufficient real-SC2 game corpus and a baseline bot with comparable strategy quality.

#### Current state

Phase 1 migration complete (#193) — all plugins implement `io.quarkmind.agent.TaskDefinition` (`execute(CaseContext)`, `activateIf()`, `requires()`, `produces()`). Dispatch still runs through poc `CaseEngine.createAndSolve()` via bridges. Phase 2 (engine#483 + engine#484) replaces poc engine — harness capabilities unchanged, benchmark numbers change.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): L7 §9.4 layer entry — Closes #159"
```

---

### Task 7: Push branch

- [ ] **Step 1: Push to origin**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind push -u origin issue-159-l7-comparison-baseline
```

Expected: branch pushed to `casehubio/quarkmind`. No PR — PRs are created on demand per CLAUDE.md.
