# ARC42STORIES.MD §9.3 C2–C5 Chapter Entries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the four 🔲 stub entries in ARC42STORIES.MD §9.3 (C2–C5) with complete chapter entries, and fix all dependent metadata in §9.2 and §9.4.

**Architecture:** Single file edit — `ARC42STORIES.MD` in the project repo. Each chapter entry replaces its stub wholesale. Supporting fixes update the §9.2 chapter table, Mermaid diagram, sequencing rationale, and C6 capability table DECLINE row. A new GitHub issue is created first for the DECLINE wiring work so its number can be referenced in C2's Known limitations.

**Tech Stack:** Markdown, GitHub CLI (`gh`), IntelliJ MCP for class-existence checks.

---

## File Map

| File | Action |
|------|--------|
| `ARC42STORIES.MD` (project repo root) | Modify — replace C2–C5 stubs, fix §9.2 metadata, fix C6 DECLINE row |

No new files created.

---

### Task 1: Create DECLINE wiring issue

**Files:**
- No file edits — GitHub issue creation only

- [ ] **Step 1: Create the issue**

```bash
gh issue create \
  --repo casehubio/quarkmind \
  --title "feat: wire DECLINE speech act in QuarkMind game-loop dispatch (Quality Goal 2)" \
  --body "Quality Goal 2 (Formal DECLINE) from §3 of ARC42STORIES.MD is unmet: the DECLINE speech act is platform-defined in casehub-qhorus but not wired in QuarkMind's game-loop dispatch path. When a plugin is out of scope, it currently produces a silent no-op rather than a structural DECLINE record.\n\nWork required:\n- Identify the dispatch path in AgentOrchestrator where DECLINE should be emitted\n- Wire the casehub-qhorus DECLINE speech act for plugins that return false from canActivate()\n- Add ledger entry for each DECLINE event\n- Update C2 Known limitations and C6 capability table once wired"
```

Expected output: a URL like `https://github.com/casehubio/quarkmind/issues/NNN`

- [ ] **Step 2: Record the issue number**

Note the issue number from the URL (e.g. `201`). You will use it as `#NNN` throughout this plan wherever `#DECLINE` appears.

---

### Task 2: Write C2 chapter entry

**Files:**
- Modify: `ARC42STORIES.MD` — replace C2 stub (§9.3)

- [ ] **Step 1: Replace the C2 stub**

Find this exact text in `ARC42STORIES.MD`:

```
#### Chapter 2 — Formal Plugin Obligation

**Journey:** Game Agent Coordination | **Sequence:** 2 of 6 | **Status:** 🔲 pending

🔲 Full Chapter entry at C2 close. casehub-qhorus integration complete (#155).

Known scope: typed COMMAND/RESPONSE between plugin agents; DECLINE recorded structurally when a plugin is out-of-scope; commitment lifecycle per plugin engagement; `MessageLedgerEntry` per speech act.
```

Replace it with (substitute `#NNN` with the actual issue number from Task 1):

```
#### Chapter 2 — Plugin Intel Channels

**Journey:** Game Agent Coordination | **Sequence:** 2 of 6 | **Status:** ✅ complete
**Delivered:** 2026-06-01, formalised 2026-06-04 (#177) | **Issues:** #155, #177 | **Blog:** `blog/2026-06-01-mdp01-phase5-complete.md`; `blog/2026-06-04-mdp01-three-cleanups-strategy-upgrade.md`

**What this delivers**
`ScoutingIntelBroker` dual-stack — a synchronous in-memory store (Stack 1) for plugin-to-plugin intel delivery, and a Qhorus channel (Stack 2, `quarkmind-scouting-intel`) as the advisory surface for external consumers (LLM advisors, Commentator). Plugins — `DroolsTacticsTask`, `DroolsStrategyTask`, `FlowEconomicsTask` — implement `ScoutingIntelConsumer` and read typed scouting intel via `broker.current()` (Stack 1 only); the Qhorus channel carries scouting intel payloads for advisory consumers, not for plugin-to-plugin coordination. `TacticsIntelCache` and `TacticsMessageBridge` removed; `MessageObserver` wiring eliminated.

**Accountability gaps closed**
- No typed inter-plugin intel delivery (L1 gap) → `ScoutingIntelBroker` Stack 1 synchronous in-memory delivery to plugin consumers

**Layer Impact**
| Layer | Delta |
|---|---|
| L3 casehub-qhorus | High — `ScoutingIntelBroker` dual-stack; `ScoutingIntelConsumer` interface; `quarkmind-scouting-intel` Qhorus channel (advisory); `TacticsIntelCache`, `TacticsMessageBridge` removed |
| L2 casehub-engine | Low — dispatch unchanged; plugin intel flows via broker alongside CaseFile |

**Known limitations / open gaps**
- Quality Goal 2 (Formal DECLINE) not closed: DECLINE speech act is platform-defined in casehub-qhorus but not wired in QuarkMind game-loop dispatch (#NNN)
```

- [ ] **Step 2: Verify the replacement**

Open `ARC42STORIES.MD` and confirm:
- The header reads "Chapter 2 — Plugin Intel Channels"
- Status shows ✅ complete
- The Known limitations references the correct `#NNN` issue number
- No stub text (`🔲 Full Chapter entry`) remains in the C2 section

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): §9.3 C2 chapter entry — Plugin Intel Channels — Refs #198"
```

---

### Task 3: Write C3 chapter entry

**Files:**
- Modify: `ARC42STORIES.MD` — replace C3 stub (§9.3)

- [ ] **Step 1: Replace the C3 stub**

Find this exact text in `ARC42STORIES.MD`:

```
#### Chapter 3 — Outcome Tracking

**Journey:** Game Agent Coordination | **Sequence:** 3 of 6 | **Status:** 🔲 pending

🔲 Full Chapter entry at C3 close. L3 (#155) now complete — C3 unblocked.

Known scope: plugin outcome recording (plugin ID, decision, game state context, result); Bayesian Beta trust scoring per plugin (EigenTrust not activated — ADR-0009); async non-blocking writes; in-memory backend for mock/emulated profiles.
```

Replace it with:

```
#### Chapter 3 — Outcome Tracking

**Journey:** Game Agent Coordination | **Sequence:** 3 of 6 | **Status:** ✅ complete
**Delivered:** 2026-06-05 | **Issues:** #156 | **Blog:** `blog/2026-06-02-mdp03-eigentrust-inert-single-attestor.md`; `blog/2026-06-06-mdp01-layer4-ledger-integration.md`

**What this delivers**
Every plugin decision is recorded via casehub-ledger: plugins fire `PluginDecisionEvent` on state transitions, and `PluginOutcomeAuditor` observes asynchronously (`@ObservesAsync`) to write a ledger entry via `outcomeRecorder.record()`. Bayesian Beta trust scores accumulate per plugin over time. Writes are async non-blocking — game-loop latency is unaffected. In-memory backend used for mock and emulated profiles (lightweight mode — tamper-evident guarantees not needed for game agent decisions). EigenTrust inert; single-attestor mode is correct for this domain (ADR-0009).

**Accountability gaps closed**
- No audit trail (L1 gap) → casehub-ledger attestation per plugin decision via `PluginOutcomeAuditor`
- No outcome tracking (L1 gap) → Bayesian Beta trust scores accumulated per plugin via casehub-ledger

**Layer Impact**
| Layer | Delta |
|---|---|
| L4 casehub-ledger | High — `PluginOutcomeAuditor` (@ObservesAsync `PluginDecisionEvent`) writes ledger entries; Bayesian Beta trust scoring; async non-blocking writes; in-memory backend for dev profiles |
| L2 casehub-engine | None |
```

- [ ] **Step 2: Verify the replacement**

Confirm:
- Status shows ✅ complete
- `PluginOutcomeAuditor` is named in the Layer Impact column
- Two distinct gap items (audit trail, outcome tracking) — no spurious forward pointer to C5
- No stub text remains in the C3 section

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): §9.3 C3 chapter entry — Outcome Tracking — Refs #198"
```

---

### Task 4: Write C4 chapter entry

**Files:**
- Modify: `ARC42STORIES.MD` — replace C4 stub (§9.3)

- [ ] **Step 1: Replace the C4 stub**

Find this exact text in `ARC42STORIES.MD`:

```
#### Chapter 4 — Adaptive Plugin Selection

**Journey:** Game Agent Coordination | **Sequence:** 4 of 6 | **Status:** 🔲 pending

🔲 Full Chapter entry at C4 close. L3 (#155), L4 (#156), and L5 (#157) all complete — C4 unblocked.

Known scope: `casehub-engine` binding conditions evaluate game state; plugins dispatched only when conditions are satisfied (e.g. `TacticsTask` skips when no enemy units); removes hardwired all-plugins-every-tick dispatch.
```

Replace it with:

```
#### Chapter 4 — Adaptive Plugin Selection

**Journey:** Game Agent Coordination | **Sequence:** 4 of 6 | **Status:** ✅ complete
**Delivered:** 2026-06-03 | **Issues:** #157 | **Blog:** `blog/2026-06-03-mdp01-layer5-adaptive-plugin-selection.md`; `blog/2026-06-04-mdp01-three-cleanups-strategy-upgrade.md`

**What this delivers**
Two structurally distinct gate mechanisms replace hardwired all-plugins-every-tick dispatch. (1) `requires()` — CaseFile key-presence ordering dependency: `ENEMY_ARMY_SIZE` on `DroolsStrategyTask` ensures scouting always runs before strategy in the CaseEngine re-evaluation loop. (2) `activateIf()` broker state gate — `broker.current(THREAT_POSITION).isPresent()` on `DroolsTacticsTask` skips tactics when no threat position is known; `broker.current(POSTURE).isPresent()` on `DroolsStrategyTask` suppresses strategy in early game until scouting has classified the enemy posture. The Pattern 2 broker gates were introduced by the #177 dual-stack redesign (delivered under C2), replacing the earlier CaseFile `NEAREST_THREAT` key gate from the initial #157 commit; they belong to the L5 story as the final form of the conditional skip pattern. `canActivate(CaseFile)` combines `requires()` and `activateIf()` via `testActivation()`. `GameTickExecutor` extracted from `AgentOrchestrator.gameTick()` to surface `CaseFile` state for testing.

**Accountability gaps closed**
- Fixed all-plugins-every-tick dispatch (L1 gap) → binding-condition dispatch via `requires()` + `activateIf()`

**Layer Impact**
| Layer | Delta |
|---|---|
| L5 Adaptive Plugin Selection | High — `requires()` ordering gates and `activateIf()` broker-state gates on all plugins; `GameTickExecutor` extraction |
| L2 casehub-engine | Medium — `AgentOrchestrator.gameTick()` refactored; `testActivation()` override pattern established across all plugins |

**Known limitations / open gaps**
- `canActivate(CaseFile)` in the installed poc casehub-core snapshot returns `true` unconditionally — does not call `testActivation()`. All four plugin classes explicitly override `canActivate()` with `testActivation(new CaseFileContext(caseFile))`, which evaluates both `requires()` and `activateIf()`. Overrides can be removed once the foundation corrects the default.
```

- [ ] **Step 2: Verify the replacement**

Confirm:
- Two gate patterns described, not three (StrategySelector gate is absent — it belongs to C5)
- Pattern 2 attribution note mentions #177 dual-stack redesign
- Known limitations names `testActivation()`, not `entryCriteria().stream().allMatch()`
- No stub text remains

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): §9.3 C4 chapter entry — Adaptive Plugin Selection — Refs #198"
```

---

### Task 5: Write C5 chapter entry

**Files:**
- Modify: `ARC42STORIES.MD` — replace C5 stub (§9.3)

- [ ] **Step 1: Replace the C5 stub**

Find the start of the C5 stub in `ARC42STORIES.MD`. The stub begins with:

```
#### Chapter 5 — Trust-weighted Routing

**Journey:** Game Agent Coordination | **Sequence:** 5 of 6 | **Status:** 🔲 pending
```

And ends just before `#### Chapter 6`. Replace the entire C5 stub section (from the `#### Chapter 5` header up to but not including the `---` separator before Chapter 6) with:

```
#### Chapter 5 — Trust-weighted Routing

**Journey:** Game Agent Coordination | **Sequence:** 5 of 6 | **Status:** ✅ complete
**Delivered:** 2026-06-10 | **Issues:** #158 | **Blog:** `blog/2026-06-12-mdp01-wiring-playerresult-to-trust.md`; `blog/2026-06-12-mdp02-trust-routing-is-degenerate-cbr.md`; `blog/2026-06-15-mdp01-casehub-engine-phase1.md`

**What this delivers**
Among three competing `StrategyTask` implementations (`DroolsStrategyTask`, `EarlyPressureStrategyTask`, `EconomicExpansionStrategyTask`), `StrategyTrustRouter` selects using opponent-context-keyed Bayesian Beta scores from casehub-ledger. A four-phase trust maturity model (BOOTSTRAP phaseScore=0.5, QUALIFIED threshold 0.838) ensures any candidate that has accumulated qualified outcomes beats any unproven one — correct cold-start behaviour without requiring prior game data. `strategy.drools` is the designated fallback, winning all ties and exempt from BORDERLINE exclusion. At game start, routing uses `STRATEGY_VS_UNKNOWN`; at the first mid-game checkpoint (when `DroolsScoutingTask` fires `EnemyPostureClassifiedEvent`), the router re-selects for the classified opponent context. `LedgerLifecycleAdapter` removed — it was clearing the in-memory ledger between games, capping trust accumulation at one decision.

**Accountability gaps closed**
- No per-outcome trust signal → `GameStopped` carries `GameResult`; WIN→ENDORSED, LOSS→CHALLENGED, TIE→SOUND, UNKNOWN→skipped; Bayesian Beta scores evolve from real game outcomes

C5 introduces a new architectural capability — opponent-context-keyed strategy selection — enabled by L4 trust data; this is not closure of an L1 gap.

**Layer Impact**
| Layer | Delta |
|---|---|
| L6 Trust Routing | High — `StrategyTrustRouter`, `StrategyTrustObserver`, `StrategySelector`, `GameOutcomeRecorder`; `EarlyPressureStrategyTask` and `EconomicExpansionStrategyTask` introduced; `DroolsStrategyTask.activateIf()` extended to add `strategySelector.isSelected(getId())` alongside the existing #177 POSTURE broker gate — enforces exactly one strategy fires per tick; four-phase trust maturity model; `strategy.drools` designated fallback |
| L4 casehub-ledger | Medium — `TrustGateService` drives routing decisions; trust scores read per opponent context |
| L2 casehub-engine | Low — Phase 1 migration (#193): `StrategyTrustRouter` implements `io.quarkmind.agent.TaskDefinition`; `execute(CaseContext)` writes selected strategy ID to `agent.strategy.selected.id` |

**Known limitations / open gaps**
- Phase 1 configuration: `StrategyTrustObserver` (CDI event observer) still drives selection; Phase 2 migration to `SequenceWorker` (engine#484) will make trust routing structural and event-log-visible
- Win/loss detection: mock, emulated, and replay profiles produce `UNKNOWN` and are skipped — correct, no meaningful win/loss signal outside real SC2. Trust scores only evolve from real SC2 games.
```

- [ ] **Step 2: Verify the replacement**

Confirm:
- `EarlyPressureStrategyTask` and `EconomicExpansionStrategyTask` named as C5 deliveries
- `strategySelector.isSelected()` gate named as C5 addition to `DroolsStrategyTask.activateIf()`
- BOOTSTRAP/QUALIFIED maturity model in "What this delivers"
- C5 note: not L1 gap closure — new capability
- No stub text remains

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): §9.3 C5 chapter entry — Trust-weighted Routing — Refs #198"
```

---

### Task 6: Fix §9.2 metadata and C6 capability table

**Files:**
- Modify: `ARC42STORIES.MD` — §9.2 chapter table, Mermaid diagram, sequencing rationale, Layer × Chapter matrix, C6 DECLINE row

All five sub-steps edit the same file; commit once at the end.

- [ ] **Step 1: Update §9.2 chapter table — C2 title and C2–C5 status**

Find and update the four pending rows in the §9.2 chapter index table. Change each `🔲 pending (#NNN)` to `✅ complete (#NNN)` and rename C2's title:

| Before | After |
|--------|-------|
| `\| 2 \| Formal Plugin Obligation \| ... \| 🔲 pending (#155) \|` | `\| 2 \| Plugin Intel Channels \| ... \| ✅ complete (#155, #177) \|` |
| `\| 3 \| Outcome Tracking \| ... \| 🔲 pending (#156) \|` | `\| 3 \| Outcome Tracking \| ... \| ✅ complete (#156) \|` |
| `\| 4 \| Adaptive Plugin Selection \| ... \| 🔲 pending (#157) \|` | `\| 4 \| Adaptive Plugin Selection \| ... \| ✅ complete (#157) \|` |
| `\| 5 \| Trust-weighted Routing \| ... \| 🔲 pending (#158) \|` | `\| 5 \| Trust-weighted Routing \| ... \| ✅ complete (#158) \|` |

- [ ] **Step 2: Update §9.2 Mermaid diagram — C2 label and C2–C5 colours**

In the Mermaid diagram block, make these changes:

Change the C2 label:
```
  C2["C2: Formal Plugin Obligation\n+ L3"]
```
to:
```
  C2["C2: Plugin Intel Channels\n+ L3"]
```

Change the four pending node styles from grey to green (matching C1 and C6):
```
  style C2 fill:#D3D3D3,color:#000
  style C3 fill:#D3D3D3,color:#000
  style C4 fill:#D3D3D3,color:#000
  style C5 fill:#D3D3D3,color:#000
```
to:
```
  style C2 fill:#90EE90,color:#000
  style C3 fill:#90EE90,color:#000
  style C4 fill:#90EE90,color:#000
  style C5 fill:#90EE90,color:#000
```

- [ ] **Step 3: Update §9.2 Layer × Chapter matrix headers**

Find the matrix header row:
```
| Layer | C1 ✅ | C2 🔲 | C3 🔲 | C4 🔲 | C5 🔲 | C6 ✅ |
```

Replace with:
```
| Layer | C1 ✅ | C2 ✅ | C3 ✅ | C4 ✅ | C5 ✅ | C6 ✅ |
```

- [ ] **Step 4a: Update §9.2 sequencing rationale — C2→C3**

Find (in the §9.2 detailed sequencing rationale):
```
- C2 before C3: formal plugin obligation tracking is meaningful only when inter-plugin channels exist
```

Replace with:
```
- C2 before C3: layer sequence — L3 (typed inter-plugin channels) precedes L4 (ledger) in the delivery sequence; no runtime dependency exists between the broker and `PluginOutcomeAuditor`
```

- [ ] **Step 4b: Update summary sequencing rationale — C2→C3**

There is a second, earlier rationale summary block ("Chapter Sequencing Rationale (summary — full rationale in §9.2)") that also has a wrong C2→C3 line. Find:
```
- C2 before C3: formal plugin obligation tracking is meaningful only when inter-plugin channels exist
```

Replace with:
```
- C2 before C3: layer sequence — L3 (typed inter-plugin channels) precedes L4 (ledger) in delivery order; no runtime dependency between the broker and the auditor
```

- [ ] **Step 5: Fix C6 capability table — DECLINE row**

Find in the §9.4 Comparison Baseline capability table (substitute `#NNN` with the actual issue number from Task 1):
```
| Formal out-of-scope signal | Silent no-op or throws | Silent no-op or throws | Typed inter-plugin channels (L3); DECLINE speech act is platform-defined, not yet wired in QuarkMind's game-loop dispatch (pending C2 chapter entry — #198) |
```

Replace with:
```
| Formal out-of-scope signal | Silent no-op or throws | Silent no-op or throws | Typed inter-plugin channels (L3); DECLINE speech act is platform-defined, not yet wired in QuarkMind's game-loop dispatch (#NNN) |
```

- [ ] **Step 6: Verify all five changes**

Scan `ARC42STORIES.MD` and confirm:
- C2 row title is "Plugin Intel Channels" in both the table and the Mermaid diagram
- C2–C5 Mermaid nodes are `fill:#90EE90` (green)
- C2–C5 Layer × Chapter matrix headers show ✅
- C2→C3 sequencing rationale no longer mentions "commitment chain"
- C6 DECLINE row parenthetical references the new issue number, not #198

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): §9.2 metadata — C2 retitle, C2-C5 status complete, sequencing rationale, DECLINE ref — Refs #198"
```

---

### Task 7: Quality sweep

**Files:**
- Modify: `ARC42STORIES.MD` if any sweep item requires a fix (commit immediately after each fix)

- [ ] **Step 1: Verify issue statuses**

```bash
gh issue view 155 --repo casehubio/quarkmind --json state,title --jq '"#\(.number) [\(.state)] \(.title)"'
gh issue view 156 --repo casehubio/quarkmind --json state,title --jq '"#\(.number) [\(.state)] \(.title)"'
gh issue view 157 --repo casehubio/quarkmind --json state,title --jq '"#\(.number) [\(.state)] \(.title)"'
gh issue view 158 --repo casehubio/quarkmind --json state,title --jq '"#\(.number) [\(.state)] \(.title)"'
```

Expected: all four OPEN (they are delivery issues, not closure issues). If any are CLOSED, note it in the chapter entry's header status line but do not change ✅ complete — the chapter closed when the work shipped, regardless of issue state.

- [ ] **Step 2: Verify class existence via IntelliJ MCP**

Using the `mcp__intellij-index__ide_find_class` tool with `project_path: /Users/mdproctor/claude/casehub/quarkmind`, confirm each class exists in the project:

- `ScoutingIntelBroker` — L3, C2
- `PluginOutcomeAuditor` — L4, C3
- `GameTickExecutor` — L5, C4
- `StrategyTrustRouter` — L6, C5
- `StrategyTrustObserver` — L6, C5
- `StrategySelector` — L6, C5
- `GameOutcomeRecorder` — L6, C5
- `EarlyPressureStrategyTask` — L6, C5
- `EconomicExpansionStrategyTask` — L6, C5

If any class is not found: update the entry that references it to say "provided by [foundation-module]" or remove the name — do not leave a reference to a non-existent class.

- [ ] **Step 3: Verify forward references**

Check that every `#NNN` reference in the four new chapter entries refers to a real GitHub issue. Issues referenced: #155, #177, #156, #157, #158, #193, #198, the new DECLINE issue, engine#484.

```bash
gh issue view 155 --repo casehubio/quarkmind --json number --jq '.number' 2>/dev/null && echo "exists" || echo "MISSING"
gh issue view 177 --repo casehubio/quarkmind --json number --jq '.number' 2>/dev/null && echo "exists" || echo "MISSING"
gh issue view 193 --repo casehubio/quarkmind --json number --jq '.number' 2>/dev/null && echo "exists" || echo "MISSING"
gh issue view 484 --repo casehubio/engine --json number --jq '.number' 2>/dev/null && echo "exists" || echo "MISSING"
```

- [ ] **Step 4: Commit any sweep fixes**

If step 2 or 3 required any edits to `ARC42STORIES.MD`:

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/quarkmind commit -m "docs(arc42): quality sweep fixes — Refs #198"
```

If no edits were needed, skip this step.

- [ ] **Step 5: Close issue #198**

```bash
gh issue close 198 --repo casehubio/quarkmind --comment "C2–C5 §9.3 chapter entries written. Quality sweep passed. §9.2 metadata updated. DECLINE wiring tracked in #NNN."
```

- [ ] **Step 6: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/quarkmind push --set-upstream origin issue-198-arc42-c2-c5-chapters
```
