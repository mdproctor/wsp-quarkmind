# Visualizer Workbench — Design Spec

**Issue:** #259 (primary), enables #248 (coaching UI — acknowledgment buttons are out of scope), strategy display
**Date:** 2026-08-03

## Summary

Replace the full-viewport Three.js canvas with an application shell (the "workbench") where the 3D canvas is one pane among several. The workbench provides a general-purpose output surface for intelligence data — pattern assessments, coaching advice, strategy routing — with bidirectional selection correlation between the text panels and the 3D scene.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Workbench is the application shell; canvas is a pane within it | Scales to all future output types without bolting panels onto a full-screen canvas |
| Layout | Fixed (dynamic rearrangement deferred) | Sufficient for three pages; dynamic docking is future paging system work |
| Data transport | Dedicated `/ws/workbench` WebSocket, single endpoint with type discriminator | Intelligence events are event-driven, not per-frame; decoupled from GameState tick rate |
| Visual indicator | Pulsing ring at unit feet | RTS convention, no shader work, colour-coded by context |
| Unit detail | Persistent section in workbench, always visible | Clicking a unit shouldn't disrupt the active page |
| Rendering | Pure DOM (CSS grid + HTML) with Three.js canvas embedded | DOM handles text/scroll/layout natively; Three.js stays focused on 3D |
| Initial pages | Pattern Assessment, Coaching, Strategy | Three pages exercise the paging system; server-side data exists but new CDI events and a broadcaster are needed to deliver it |

## Application Shell Layout

```
┌──────────────────────────────────────────────────────────┐
│  Toolbar: [Pattern] [Coaching] [Strategy]    Min/Gas/Sup │
│──────────────────────────────────────┬───────────────────│
│                                      │                   │
│                                      │   Active Page     │
│          3D Canvas                   │   (scrollable)    │
│          (Three.js)                  │                   │
│                                      │                   │
│                                      ├───────────────────│
│                                      │   Unit Detail     │
│                                      │   (persistent)    │
│──────────────────────────────────────┴───────────────────│
│  Status bar: Frame N │ Phase: Early │ Connected          │
└──────────────────────────────────────────────────────────┘
```

- **Toolbar** — page tabs on the left, HUD info (minerals/gas/supply) migrated from floating overlay on the right
- **Main area** — CSS grid, two columns. Left: Three.js canvas sized via `ResizeObserver`. Right: page content (top, scrollable) + unit detail (bottom, fixed height ~160px)
- **Status bar** — frame count, game phase, connection status
- **Right panel width** — fixed at ~300px
- **Existing 3D overlays** (angle buttons, mode toggle, camera toggle, enemy toggle) remain as overlays on the canvas region — they are view controls, not workbench chrome
- **Replay control bar** stays at the bottom of the canvas region, not the full window

## Selection Model

Shared JS object coordinating bidirectional correlation between the 3D view and the workbench.

```javascript
const selection = {
  type: null,       // 'unit' | 'building' | 'unitType' | null
  tag: null,        // specific unit/building tag (from 3D click)
  unitType: null,   // UnitType string (from workbench click)
  isEnemy: null,
  source: null,     // 'canvas' | 'workbench'
  _listeners: [],

  subscribe(fn) {
    this._listeners.push(fn);
    return () => { this._listeners = this._listeners.filter(l => l !== fn); };
  },

  set(sel) {
    if (this._dispatching) return;  // re-entrancy guard
    Object.assign(this, sel);
    this._dispatching = true;
    for (const fn of this._listeners) {
      try { fn(this); } catch (e) { console.error('selection listener error', e); }
    }
    this._dispatching = false;
  },

  clear() { this.set({ type: null, tag: null, unitType: null, isEnemy: null, source: null }); }
};
```

**Two selection modes:**

1. **Tag selection** (3D click) — selects a specific unit by tag. Ring on that unit. Unit detail shows live stats updated per GameState frame.
2. **Type selection** (workbench click) — selects a unit type. Rings on all matching units. Unit detail shows type info + count (e.g. "STALKER × 4 (friendly)"). If no units of the selected type exist on screen, the detail section shows the type name and portrait with "× 0" — no rings appear, but the selection remains valid (counter info is useful even before you build the unit).

**Clear:** clicking empty canvas or Escape clears selection, removes all rings, resets detail section.

**Listener lifecycle:**
- `subscribe()` returns an unsubscribe function — callers hold the reference and call it on cleanup
- Each listener is called inside a try/catch — one failing listener does not prevent others from firing
- Re-entrancy guard: if a listener calls `set()`, the nested call is ignored (prevents infinite loops)

**Unit death during tag selection:** The `syncUnits()` function (already called per GameState frame) checks whether the selected tag still exists. If the unit is gone, call `selection.clear()`. This removes the ring and resets the detail section.

## Pulsing Ring Indicator

Visual indicator in the 3D scene when a selection is active.

- **Geometry:** `THREE.RingGeometry` with `MeshBasicMaterial`, transparent, double-sided
- **Positioning:** flat on ground (rotated 90° on X), at unit's world coordinates
- **Pulse:** sine wave on opacity, 0.3–0.8, ~1 second cycle, driven in `animate()` loop
- **Ring map:** `Map<number, THREE.Mesh>` tracks active rings by unit tag
- **Size:** inner radius = sprite scale × 0.6, outer radius = sprite scale × 0.8
- **Colours:** green (friendly), red (enemy)
- **Per-frame sync:** rings follow parent sprite position in `animate()`. Ring map is keyed by unit tag; `syncUnits()` removes rings whose parent sprite was removed (unit died or left vision).
- **Lifecycle:** on selection change, clear all existing rings (remove from scene + dispose geometry/material), create new set. On unit death (sprite removed by `syncUnits()`), remove the orphaned ring and — if it was the only ring (tag selection) — clear the selection.

## WebSocket — `/ws/workbench`

### Server Side

**WebSocket endpoint:**

`WorkbenchSocket` — `@WebSocket(path = "/ws/workbench")`, `@UnlessBuildProfile("prod")`. Same pattern as `GameStateSocket`. On `@OnOpen`, calls `broadcaster.addSession(connection)` which immediately pushes current state (see Reconnect below).

**Envelope — typed payloads, not raw Object:**

```java
public sealed interface WorkbenchPayload permits
    PatternPayload, CoachingPayload, CoachingCompliancePayload, StrategyPayload {}

public record WorkbenchEvent(String type, WorkbenchPayload payload) {}
```

Each payload type is a concrete record:
- `PatternPayload(List<EnrichedAssessment> assessments)` where `EnrichedAssessment(PatternAssessment assessment, CounterInfo counters)`
- `CoachingPayload(String advice, CoachingDomain domain, CoachingUrgencyTier urgency, long gameFrame)`
- `CoachingCompliancePayload(long gameFrame, CoachingDomain domain, String status)` — `status` is "complied", "ignored", or "neutral"
- `StrategyPayload(String strategyId, StrategyArchetype archetype, double confidence, int pivotCount)`

**Architecture — enrichment separated from broadcasting:**

`WorkbenchEnricher` — `@ApplicationScoped`. Observes CDI events, enriches payloads, and forwards to `WorkbenchBroadcaster`. This separates the enrichment concern (calling `StrategyTaxonomy.countersFor()`) from the broadcasting concern (serializing and pushing to WebSocket sessions). If enrichment fails for one archetype, the assessment is still included with `counters: null` — the event is never dropped entirely.

`WorkbenchBroadcaster` — `@ApplicationScoped`. Receives pre-enriched `WorkbenchEvent` objects from the enricher, serializes to JSON, and sends to all connected sessions. Also maintains a `latestState` snapshot for reconnect (see below). Pure broadcaster — no domain logic.

**CDI event sources — concrete new events:**

All three event types require new CDI events. The spec does not reuse existing CDI events that serve other purposes.

| Type | CDI Event (new) | Fired by | Package |
|------|----------------|----------|---------|
| `"pattern"` | `PatternAssessmentPublished(List<PatternAssessment>)` | `DroolsScoutingTask` — fire alongside existing `broker.update()` call (line ~302) | `io.quarkmind.agent.plugin` (alongside `ScoutingIntelPayload`) |
| `"coaching"` | `CoachingAdvicePublished(CoachingAdvice, CoachingUrgencyTier, long gameFrame)` | `CoachingChannelBroker` — fire after dispatching to Qhorus channel (line ~100) | `io.quarkmind.plugin.coaching` |
| `"coaching"` (compliance) | `CoachingComplianceResolved(long gameFrame, CoachingDomain, String status)` | `CoachingComplianceEvaluator` — fire when a commitment resolves to ENDORSED/CHALLENGED/NEUTRAL | `io.quarkmind.plugin.coaching` |
| `"strategy"` | `StrategySelectionPublished(String strategyId, StrategyArchetype, double confidence, int pivotCount)` | `SC2StrategyRouterTask` — fire after writing CaseFile keys (line ~130) | `io.quarkmind.agent.cbr` |

`CoachingCompleted` already exists but serves the channel broker's internal handoff. `CoachingAdvicePublished` is a separate event for the workbench — it carries the fields the workbench needs without coupling to the channel broker's internal contract.

**Reconnect — server pushes current state on connect:**

`WorkbenchBroadcaster` maintains a `WorkbenchSnapshot` of the latest event per type:

```java
private volatile PatternPayload latestPattern;
private volatile CoachingPayload latestCoaching;  // most recent only
private volatile StrategyPayload latestStrategy;
```

On `addSession(connection)`, the broadcaster immediately sends any non-null snapshots as regular `WorkbenchEvent` messages. The client treats them identically to live events — no special reconnect handling needed on the JS side. The coaching feed (which is a rolling list) only recovers the most recent entry on reconnect — historical entries are not replayed. This is acceptable: coaching advice is time-sensitive, and stale entries from before a disconnect are less useful than a clean state.

**Race awareness for counter enrichment:**

`StrategyTaxonomy.countersFor(archetype)` defaults to `Race.PROTOSS`. For now this is correct — QuarkMind plays Protoss. When multi-race support lands (#74), the enricher reads the player's race from `GameState` and calls `countersFor(archetype, playerRace)`. No spec change needed — the two-arg overload already exists.

### Client Side

Second WebSocket connection at `ws://${host}/ws/workbench`. Same reconnect pattern (2s delay). On message, dispatch by `type` to page update functions.

```javascript
const workbenchState = {
  pattern: null,    // latest pattern assessments + counters
  coaching: [],     // rolling list, newest first, capped at 20
  strategy: null    // latest strategy selection
};
```

State persists across tab switches — switching to the Pattern page shows the last received data, not a blank page. On reconnect, the server pushes current state — the client processes these like normal events, repopulating `workbenchState`.

**Coaching compliance updates:** When a `coaching` event with type `"coaching_compliance"` arrives, the client finds the matching entry in `workbenchState.coaching` by `gameFrame` + `domain` and updates its status badge. If the entry has already been evicted from the capped list, the update is silently dropped.

## Page Designs

### Pattern Assessment Page

Primary deliverable for #259.

- Assessments sorted by confidence descending
- Top assessment expanded by default, others collapsed (click to expand)
- Each assessment shows: archetype name, confidence bar (green >70%, yellow >50%, red <50%), rationale text
- Strong counters and weak counters listed per assessment
- Counter unit names are **clickable** — triggers type selection, pulsing rings appear on matching units
- Updates in place on new `pattern` event — no flicker

### Coaching Page

Shows coaching advice as a chronological feed.

- Rolling feed, newest at top, capped at ~20 entries
- Each entry: game time, domain tag (BUILD/MILITARY/EXPAND/TECH), urgency tier badge, advice text
- Compliance status updates live: ⏳ Pending → ✓ Complied / ✗ Ignored / — Neutral. Updates arrive as separate `coaching_compliance` WebSocket events (see WebSocket section). Matched to feed entries by `gameFrame` + `domain`.
- **Empty state:** "No coaching advice yet" (shown in all modes). The coaching page is always visible in the tab bar regardless of game mode — in non-coach modes it simply never receives events. This avoids the workbench needing to know about game modes.

### Strategy Page

Shows current strategy routing state.

- Key-value display: active strategy name, routing method (CBR match), archetype, confidence, pivot count
- Updates on strategy selection change

## Unit Detail Section

Persistent bottom strip of the right panel. Replaces the current floating `#unit-panel`.

- Portrait canvas (64×64), unit type name, team, HP bar (colour-coded), shield bar (Protoss only)
- Adds: position coordinates, unit tag (useful for lab/debugging)
- **Tag selection:** live stats, updated per GameState frame
- **Type selection:** type name, portrait, count (e.g. "STALKER × 4 (friendly)")
- **Empty state:** "Click a unit or counter to inspect"

## Data Flow

Two WebSockets, two concerns:

- `ws/gamestate` — high frequency, per-frame spatial data → 3D view
- `ws/workbench` — event-driven intelligence data → workbench pages

The selection model is pure client-side. Both WebSocket handlers and click handlers interact with it. The two WebSockets are independent connections but **not fully decoupled** — the unit detail section reads from GameState frames to update live stats for a tag-selected unit, and `syncUnits()` (GameState-driven) clears selection when a selected unit dies. This is a deliberate coupling point at the selection model.

**Partial failure UX:** If one WebSocket disconnects while the other stays connected:
- `ws/gamestate` down, `ws/workbench` up: 3D view freezes, workbench pages still update. Status bar shows "Game: Disconnected" in red.
- `ws/workbench` down, `ws/gamestate` up: 3D view runs normally, workbench pages show stale data. Status bar shows "Intel: Disconnected" in yellow.
- Both down: full "Disconnected" state.

Each WebSocket reconnects independently on its own 2s timer. The status bar shows per-connection state so the user knows which data is live.

## Testing

**Unit tests (plain JUnit):**
- `WorkbenchEventTest` — JSON serialization of envelope with each typed payload (PatternPayload, CoachingPayload, CoachingCompliancePayload, StrategyPayload)
- `WorkbenchEnricherTest` — CounterInfo enrichment: given assessments and a mocked StrategyTaxonomy, verify enriched payloads. Verify partial enrichment failure (one archetype's counters missing) still produces the event with `counters: null` for that assessment.
- `WorkbenchBroadcasterTest` — snapshot maintenance: after receiving events, verify `latestPattern`/`latestStrategy`/`latestCoaching` are updated. Verify `addSession()` pushes non-null snapshots.

**Integration tests (`@QuarkusTest`):**
- `WorkbenchSocketIT` — WebSocket connect, fire CDI event, assert correct JSON arrives. Also test reconnect: connect, receive a pattern event, disconnect, reconnect, assert the snapshot is pushed on the new connection.

**Playwright render tests (`@Tag("browser")`):**
- `WorkbenchRenderTest`:
  - Shell renders (toolbar, page area, unit detail)
  - Tab switching shows/hides correct pages
  - Simulated pattern WebSocket message populates Pattern page
  - Counter unit name click produces pulsing rings
  - 3D unit click populates unit detail
  - Empty canvas click clears selection and rings

**`window.__test` API additions:**
- `workbenchPage()` — active page name
- `workbenchPatternCount()` — number of displayed assessments
- `selectionRingCount()` — active pulsing ring count
- `selectionState()` — current selection object
- `unitDetailName()` — name in unit detail section

## Scope Boundary

**In scope:**
- Application shell layout (CSS grid, toolbar, status bar)
- Three initial pages (pattern, coaching, strategy)
- Selection model with bidirectional correlation
- Pulsing ring indicators
- `/ws/workbench` WebSocket with `WorkbenchBroadcaster`
- Unit detail section replacing floating inspect panel
- `window.__test` API extensions
- Playwright render tests for the workbench

**Out of scope (future work):**
- Dynamic/rearrangeable panel layout (paging system TODO)
- Coaching acknowledgment UI — DONE/DECLINE buttons (#248, separate issue)
- LLM-evaluated compliance verification (#245)
- Coaching personality model (#251)
