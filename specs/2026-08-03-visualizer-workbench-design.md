# Visualizer Workbench — Design Spec

**Issue:** #259 (primary), also serves #248 (coaching UI), strategy display
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
| Initial pages | Pattern Assessment, Coaching, Strategy | Three pages exercise the paging system; all data pipelines already exist server-side |

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
  listeners: [],
  set(sel) { /* update fields, notify listeners */ },
  clear() { /* reset all, notify listeners */ }
};
```

**Two selection modes:**

1. **Tag selection** (3D click) — selects a specific unit by tag. Ring on that unit. Unit detail shows live stats updated per GameState frame.
2. **Type selection** (workbench click) — selects a unit type. Rings on all matching units. Unit detail shows type info + count (e.g. "STALKER × 4 (friendly)").

**Clear:** clicking empty canvas or Escape clears selection, removes all rings, resets detail section.

**Listeners:** canvas renderer (rings), unit detail section, active page (row highlight). Each registered via `selection.listeners.push(fn)`.

## Pulsing Ring Indicator

Visual indicator in the 3D scene when a selection is active.

- **Geometry:** `THREE.RingGeometry` with `MeshBasicMaterial`, transparent, double-sided
- **Positioning:** flat on ground (rotated 90° on X), at unit's world coordinates
- **Pulse:** sine wave on opacity, 0.3–0.8, ~1 second cycle, driven in `animate()` loop
- **Ring map:** `Map<number, THREE.Mesh>` tracks active rings by unit tag
- **Size:** inner radius = sprite scale × 0.6, outer radius = sprite scale × 0.8
- **Colours:** green (friendly), red (enemy), blue (neutral/informational)
- **Per-frame sync:** rings follow parent sprite position. Parent removed → ring removed.
- **Lifecycle:** on selection change, clear all existing rings (remove + dispose), create new set.

## WebSocket — `/ws/workbench`

### Server Side

`WorkbenchSocket` — `@WebSocket(path = "/ws/workbench")`, `@UnlessBuildProfile("prod")`. Same pattern as `GameStateSocket`.

`WorkbenchBroadcaster` — `@ApplicationScoped`. Observes CDI events, serializes and pushes to connected sessions.

**Envelope:**
```java
public record WorkbenchEvent(String type, Object payload) {}
```

**Event sources:**

| Type | Payload | CDI Source |
|------|---------|-----------|
| `"pattern"` | `List<PatternAssessment>` enriched with `CounterInfo` per archetype | New `PatternAssessmentPublished` CDI event fired by `DroolsScoutingTask` alongside existing `ScoutingIntelBroker.update()` call |
| `"coaching"` | `CoachingAdvice` + urgencyTier, gameFrame, compliance status | Observe `CoachingCompleted` (already exists) |
| `"strategy"` | strategy ID, archetype, confidence, pivot count | Observe CaseFile key change on `STRATEGY_SELECTED_ID` |

**Enrichment:** `WorkbenchBroadcaster` calls `StrategyTaxonomy.countersFor()` to attach counter info per archetype to pattern events. The client receives ready-to-render data.

### Client Side

Second WebSocket connection at `ws://${host}/ws/workbench`. Same reconnect pattern (2s delay). On message, dispatch by `type` to page update functions.

```javascript
const workbenchState = {
  pattern: null,    // latest pattern assessments + counters
  coaching: [],     // rolling list, newest first, capped at 20
  strategy: null    // latest strategy selection
};
```

State persists across tab switches — switching to the Pattern page shows the last received data, not a blank page.

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
- Compliance status updates live: ⏳ Pending → ✓ Complied / ✗ Ignored / — Neutral
- Empty state message when not in coach mode

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

The selection model is pure client-side. Both WebSocket handlers and click handlers interact with it. The two WebSockets never couple — they meet only at the selection model when a GameState frame updates a tag-selected unit's detail.

## Testing

**Unit tests (plain JUnit):**
- `WorkbenchEventTest` — JSON serialization of envelope with each payload type
- `WorkbenchBroadcasterTest` — CounterInfo enrichment logic with mocked StrategyTaxonomy

**Integration tests (`@QuarkusTest`):**
- `WorkbenchSocketIT` — WebSocket connect, fire CDI event, assert correct JSON arrives

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
