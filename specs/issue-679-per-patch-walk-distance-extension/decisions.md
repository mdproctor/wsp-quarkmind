## D1: Scope — walk asymmetry only

**Choice:** Model outbound/return walk asymmetry. Effective radii (mining radius 1.215 vs EM 1.0, deposit distance 3.755 vs EM 2.75) deferred to a separate follow-up issue.
**Alternatives:**
- Walk asymmetry + effective radii — more accurate but changes walk pipeline substantially
- Walk asymmetry + radii in same issue — scope creep, two distinct phenomena conflated
**Rationale:** The cadence data shows the return walk "shortening" is primarily explained by effective radius differences (workers stop further from both buildings). Fixing radii changes edge-to-edge targeting, which affects every walk calculation. Keep #679 focused on the structural path shape asymmetry.
**Trade-offs:** Half the measured asymmetry (return side) won't improve until the radii issue lands.
**Sources:** CycleDecompositionTest output (8w cadence data), sc2-physics-constants.md §3
**Exploration:** quick
**Status:** captured

## D2: Calibration source — SC2 measurements

**Choice:** Use the 8w cadence data (CycleDecompositionTest) as ground truth — compute outbound extensions directly from measured WalkTo/WalkBack per worker.
**Alternatives:**
- Geometry-derived — compute from mineral line geometry (patch count, angles); more generalizable but needs own validation
- Hybrid — derive formula from geometry, calibrate against measurements
**Rationale:** Direct SC2 per-worker measurements exist (8 data points across E2E 2.27–3.88). Empirical calibration avoids modelling assumptions about SC2's pathfinding.
**Trade-offs:** Only 8 data points from one map. If the parametric model (D3) doesn't generalise, may need geometry.
**Sources:** cal-worker-cadence-8w.json, CycleDecompositionTest.java
**Exploration:** quick
**Status:** captured

## D3: Generalisation — parametric from E2E distance

**Choice:** Derive outbound walk extension as a function of E2E distance (linear fit from cadence data). Apply to arbitrary mineral layouts.
**Alternatives:**
- Per-patch-index heuristic — use position in arc (inner/outer); indices aren't stable across maps
- Fixed per-distance-band — two values (close/far); coarse but simple
**Rationale:** 8 data points across E2E 2.27–3.88 provide enough range for a linear fit. The slope captures how path deviation scales with distance. Per-band is too coarse; per-index is fragile.
**Trade-offs:** Linear fit R²=0.104 for outbound extension vs E2E — weak correlation. The relationship may be non-linear or dominated by angular position. Acceptable as a first approximation.
**Sources:** Calibration analysis: `outbound_ext = -0.2346 * E2E + 0.9459` (R²=0.104)
**Exploration:** quick
**Status:** captured

## D4: Mechanism — loop delay

**Choice:** Apply walk extensions as loop delays (extra time cost), not as physical distance modifications to walk targets.
**Alternatives:**
- Walk distance — move targets further; changes collision model, harder to calibrate from time data
- Target offset — offset perpendicular to straight line; most realistic but very complex
**Rationale:** Same mechanism as current approach docking — proven, simple. The extension represents path deviation time, not physical distance. Keeps movement physics unchanged.
**Trade-offs:** Workers walk the same straight-line path as before — only the time overhead changes. Collision timing during walks is unchanged.
**Sources:** SimulatedWorker.moveOneLoop(), MineralPatch.computeApproachDocking()
**Exploration:** quick
**Status:** captured

## D5: Return delay integration — not needed (outbound-only approach)

**Choice:** Outbound-only delay extraction (Approach A). No return delay needed. The return walk stays unchanged.
**Alternatives:**
- Return delay in RETURNING_MINERALS on arrival — symmetric with outbound; adds delay before deposit
- New APPROACHING_BASE state — explicit state machine change; breaks existing tests
- Increase deposit per-patch — conflates path deviation with deposit animation
**Rationale:** Approach A chosen: extract path deviation from approach docking into outbound-only delay. The return being shorter is primarily an effective radius effect (deferred issue), not path shape. Adding a return delay without fixing radii adds complexity without accuracy gain.
**Trade-offs:** Only outbound asymmetry modeled. Accurate return modelling requires the radii follow-up.
**Sources:** Calibration analysis: return extension mean=-1.115u (explained by effective radius diff of ~1.0u)
**Exploration:** quick
**Depends on:** D1 (scope decision deferred radii)
**Status:** captured

## D6: Approach — outbound-only delay extraction

**Choice:** Extract path deviation from approach docking into a separate `outboundPathDelay` per-patch field. Approach docking reduced to animation-only.
**Alternatives:**
- Bilateral delay split — split approach into outbound + return; return overhead conceptually awkward without radii fix
- Walk speed asymmetry — speed factor per direction; violates compensation constant prohibition
**Rationale:** Minimal code change. Preserves total cycle time exactly (`outboundPathDelay + reducedApproach ≈ currentApproachDocking`). Clean stepping stone for radii follow-up. Achieves all acceptance criteria.
**Trade-offs:** Only half the asymmetry modeled. Collision timing during walks unchanged.
**Sources:** MineralPatch.computeApproachDocking(), WorkerSimulation.stepOneMineralLoop(), sc2-physics-constants.md §3/§14
**Exploration:** quick
**Depends on:** D4 (mechanism), D5 (no return delay)
**Status:** captured
