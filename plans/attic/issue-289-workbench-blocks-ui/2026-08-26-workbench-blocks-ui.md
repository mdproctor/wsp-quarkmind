# Workbench blocks-ui Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #289 — feat: workbench blocks-ui migration + commentary surfacing
**Issue group:** #289

**Goal:** Replace the vanilla JS workbench with blocks-ui Lit components and surface commentary for the first time.

**Architecture:** The visualizer workbench gets a full rebuild: `<blocks-split-workbench>` replaces the CSS grid layout, `<blocks-detail-pane>` replaces manual tab switching, and four Lit components (`<qm-pattern-page>`, `<qm-coaching-page>`, `<qm-strategy-page>`, `<qm-commentary-page>`) replace innerHTML rendering. Commentary flows from `WorkbenchEnricher` → workbench WebSocket → `<blocks-channel-feed>` via direct `QhorusMessage[]` property assignment. Quinoa + Vite handles JS module resolution.

**Tech Stack:** Quarkus, Lit 3, blocks-ui web components, Quinoa 2.8.3, Vite, Playwright

## Global Constraints

- All workbench code carries `@UnlessBuildProfile("prod")` — excluded from production builds
- Three.js canvas code must remain in non-module scripts — NOT in the Vite bundle
- Lit arrays require immutable updates (`[...old, new]`) for change detection
- `WorkbenchPayload` is sealed — permits clause must be updated when adding `CommentaryPayload`
- blocks-ui dark theme via CSS custom properties overridden at `:root`
- Quinoa is the platform-standard frontend build integration (not frontend-maven-plugin)
- IntelliJ MCP required for all Java file operations
- Responsive collapse (< 768px) hides canvas — acceptable for dev tool (desktop-only)
- Commentary message array capped at 200 — drop oldest beyond cap
- `WorkbenchSocket.onOpen` must be `@Blocking` — history query does JDBC

---

## Batch 1: Quinoa Build Infrastructure + Skeleton Layout

### Task 1: Add Quinoa + Vite build pipeline

**Files:**
- Modify: `quarkmind-sc2/pom.xml` — add Quinoa extension, blocks-ui-npm + pages-npm dependencies, maven-dependency-plugin unpack
- Create: `quarkmind-sc2/src/main/webui/package.json`
- Create: `quarkmind-sc2/src/main/webui/vite.config.ts`
- Create: `quarkmind-sc2/src/main/webui/workbench-entry.ts`
- Create: `quarkmind-sc2/src/main/webui/tsconfig.json`
- Modify: `quarkmind-sc2/.gitignore` — add `src/main/webui/.casehub-packages/`, `src/main/webui/node_modules/`, `src/main/webui/dist/`

**Interfaces:**
- Produces: `workbench-blocks.js` bundle at `META-INF/resources/blocks/workbench-blocks.js` (served by Quarkus at `/blocks/workbench-blocks.js`)

- [ ] **Step 1: Add Maven dependencies to quarkmind-sc2/pom.xml**

Add after the existing `casehub-blocks` dependency:

```xml
<!-- CaseHub npm packages (unpacked by maven-dependency-plugin) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-npm</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-blocks-ui-npm</artifactId>
    <version>0.1-SNAPSHOT</version>
</dependency>
```

Add the Quinoa extension dependency:

```xml
<dependency>
    <groupId>io.quarkiverse.quinoa</groupId>
    <artifactId>quarkus-quinoa</artifactId>
</dependency>
```

Add the maven-dependency-plugin execution inside `<build><plugins>`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>unpack-casehub-packages</id>
            <phase>initialize</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>io.casehub</groupId>
                        <artifactId>casehub-pages-npm</artifactId>
                        <version>0.2-SNAPSHOT</version>
                        <outputDirectory>${project.basedir}/src/main/webui/.casehub-packages</outputDirectory>
                    </artifactItem>
                    <artifactItem>
                        <groupId>io.casehub</groupId>
                        <artifactId>casehub-blocks-ui-npm</artifactId>
                        <version>0.1-SNAPSHOT</version>
                        <outputDirectory>${project.basedir}/src/main/webui/.casehub-packages</outputDirectory>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```

- [ ] **Step 2: Add Quinoa configuration to application.properties**

```properties
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
quarkus.quinoa.package-manager-install.node-version=22.16.0
quarkus.quinoa.ignored-path-prefixes=/ws/,/api/,/q/,/dev/
```

- [ ] **Step 3: Create package.json**

Create `quarkmind-sc2/src/main/webui/package.json`:

```json
{
  "name": "quarkmind-workbench-ui",
  "private": true,
  "scripts": {
    "build": "vite build",
    "dev": "vite"
  },
  "dependencies": {
    "lit": "^3.3.3",
    "@casehubio/blocks-ui-split-workbench": "file:.casehub-packages/packages/split-workbench",
    "@casehubio/blocks-ui-detail-pane": "file:.casehub-packages/packages/detail-pane",
    "@casehubio/blocks-ui-channel-activity": "file:.casehub-packages/packages/channel-activity",
    "@casehubio/blocks-ui-core": "file:.casehub-packages/packages/blocks-ui-core",
    "@casehubio/pages-component": "file:.casehub-packages/packages/pages-component",
    "@casehubio/pages-primitives": "file:.casehub-packages/packages/pages-primitives",
    "@casehubio/pages-data": "file:.casehub-packages/packages/pages-data",
    "@casehubio/pages-table": "file:.casehub-packages/packages/pages-table"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vite": "^8.1.3"
  }
}
```

- [ ] **Step 4: Create vite.config.ts**

Create `quarkmind-sc2/src/main/webui/vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: __dirname,
  build: {
    outDir: 'dist',
    rollupOptions: {
      input: resolve(__dirname, 'workbench-entry.ts'),
      output: {
        entryFileNames: 'blocks/workbench-blocks.js',
      },
    },
  },
  resolve: {
    alias: [
      { find: '@casehubio/blocks-ui-split-workbench', replacement: resolve(__dirname, '.casehub-packages/packages/split-workbench/src') },
      { find: '@casehubio/blocks-ui-detail-pane', replacement: resolve(__dirname, '.casehub-packages/packages/detail-pane/src') },
      { find: '@casehubio/blocks-ui-channel-activity', replacement: resolve(__dirname, '.casehub-packages/packages/channel-activity/src') },
      { find: '@casehubio/blocks-ui-core', replacement: resolve(__dirname, '.casehub-packages/packages/blocks-ui-core/src') },
      { find: '@casehubio/pages-component', replacement: resolve(__dirname, '.casehub-packages/packages/pages-component/dist') },
      { find: '@casehubio/pages-primitives', replacement: resolve(__dirname, '.casehub-packages/packages/pages-primitives/src') },
      { find: /^@casehubio\/pages-data\/dist\/(.*)/, replacement: resolve(__dirname, '.casehub-packages/packages/pages-data/src/$1') },
      { find: '@casehubio/pages-data', replacement: resolve(__dirname, '.casehub-packages/packages/pages-data/src') },
      { find: '@casehubio/pages-table', replacement: resolve(__dirname, '.casehub-packages/packages/pages-table/src') },
    ],
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: JSON.stringify({
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    }),
  },
});
```

- [ ] **Step 5: Create tsconfig.json**

Create `quarkmind-sc2/src/main/webui/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "experimentalDecorators": true,
    "useDefineForClassFields": false,
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["*.ts", "workbench/**/*.ts"]
}
```

- [ ] **Step 6: Create minimal workbench-entry.ts**

Create `quarkmind-sc2/src/main/webui/workbench-entry.ts`:

```ts
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-detail-pane';
```

- [ ] **Step 7: Update .gitignore**

Append to `quarkmind-sc2/.gitignore`:

```
src/main/webui/.casehub-packages/
src/main/webui/node_modules/
src/main/webui/dist/
src/main/webui/.yarn/
src/main/webui/yarn.lock
```

- [ ] **Step 8: Verify build produces the bundle**

Run: `mvn compile -pl quarkmind-sc2`

Expected: Build succeeds. Quinoa runs npm install + vite build. File exists at `quarkmind-sc2/src/main/webui/dist/blocks/workbench-blocks.js`.

- [ ] **Step 9: Commit**

```bash
git add quarkmind-sc2/pom.xml quarkmind-sc2/.gitignore quarkmind-sc2/src/main/webui/ quarkmind-sc2/src/main/resources/application.properties
git commit -m "feat: add Quinoa + Vite build pipeline for blocks-ui consumption Refs #289"
```

### Task 2: Skeleton layout — split-workbench + detail-pane with canvas

**Files:**
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.html` — replace CSS grid with blocks-ui components
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js` — update workbench init
- Modify: `quarkmind-sc2/src/main/webui/workbench-entry.ts` — add workbench controller import

**Interfaces:**
- Consumes: `workbench-blocks.js` bundle from Task 1
- Produces: Working visualizer with blocks-ui layout shell, Three.js canvas rendering in left panel, tab bar in right panel

- [ ] **Step 1: Replace visualizer.html with blocks-ui layout**

Replace the `<style>` and `<body>` content. Keep Three.js and visualizer.js script tags. The new HTML uses `<blocks-split-workbench>` with canvas in the `list` slot and `<blocks-detail-pane>` in the `detail` slot.

Key changes:
- Remove CSS grid (`#workbench` grid styles, `#wb-toolbar`, `#wb-canvas`, `#wb-panel`, `#wb-pages`, `.wb-tab`, `.wb-page`)
- Add `:root` dark theme CSS custom properties
- Replace `<div id="workbench">` structure with `<blocks-split-workbench>`
- Canvas container moves to `slot="list"`
- `<blocks-detail-pane>` in `slot="detail"` with `selection-topic="qm-workbench"` and `empty-message=""`
- Status bar stays outside split-workbench
- Keep camera controls (`#angle-btns`, `#mode-toggle`, `#config-panel`) inside the canvas container
- Load `workbench-blocks.js` as `type="module"` before `three.min.js`

```html
<script type="module" src="/blocks/workbench-blocks.js"></script>
<script src="/sprites/three.min.js"></script>
<script src="/visualizer.js"></script>
```

- [ ] **Step 2: Update visualizer.js — remove old tab setup, add detail-pane config**

Remove `setupWorkbenchTabs()` function and its call. Add after DOM ready:

```js
// Configure detail-pane tabs
var detailPane = document.querySelector('blocks-detail-pane');
if (detailPane) {
  detailPane.tabs = [
    { id: 'pattern',    label: 'Pattern',    tagName: 'qm-pattern-page',    order: 0 },
    { id: 'coaching',   label: 'Coaching',   tagName: 'qm-coaching-page',   order: 1 },
    { id: 'strategy',   label: 'Strategy',   tagName: 'qm-strategy-page',   order: 2 },
    { id: 'commentary', label: 'Commentary', tagName: 'qm-commentary-page', order: 3 },
  ];
  detailPane.emptyMessage = '';
}
```

Fire the selection event to activate tabs (from the ES module controller — see next step).

- [ ] **Step 3: Create workbench controller stub**

Create `quarkmind-sc2/src/main/webui/workbench/qm-workbench-controller.ts`:

```ts
import { emitPagesEvent } from '@casehubio/pages-component';

export function initWorkbench(): void {
  emitPagesEvent(document, 'qm-workbench:selected', {});
}

document.addEventListener('DOMContentLoaded', () => {
  requestAnimationFrame(() => initWorkbench());
});
```

Update `workbench-entry.ts` to import it:

```ts
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-detail-pane';
import './workbench/qm-workbench-controller.js';
```

- [ ] **Step 4: Verify Three.js canvas still renders**

Run: `mvn quarkus:dev -pl quarkmind-sc2`

Open `http://localhost:8080/visualizer.html`. Verify:
- Split workbench renders with draggable divider
- Three.js canvas shows in the left panel with sprites
- Detail pane shows tab bar with 4 tabs (tabs show "qm-pattern-page" etc. since components aren't registered yet)
- Camera controls (angle buttons, mode toggle) are visible over the canvas
- Status bar shows at the bottom

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.html quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js quarkmind-sc2/src/main/webui/
git commit -m "feat: skeleton layout — split-workbench + detail-pane with canvas Refs #289"
```

## Batch 2: Page Components (Pattern, Coaching, Strategy)

### Task 3: Pattern page Lit component

**Files:**
- Create: `quarkmind-sc2/src/main/webui/workbench/qm-pattern-page.ts`
- Modify: `quarkmind-sc2/src/main/webui/workbench-entry.ts` — add import
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js` — remove `renderPatternPage()`, `toggleAssessment()`, add data push to component

**Interfaces:**
- Consumes: `WorkbenchEvent` JSON with `type: "pattern"` from WebSocket
- Produces: `<qm-pattern-page>` custom element; `data` property accepts `PatternPayload` shape

- [ ] **Step 1: Create qm-pattern-page.ts**

Create `quarkmind-sc2/src/main/webui/workbench/qm-pattern-page.ts`:

```ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

interface Assessment {
  archetype: string;
  confidence: number;
  rationale?: string;
}

interface CounterUnit { units: string[]; action?: string; }
interface CounterInfo { strongCounters?: CounterUnit[]; weakCounters?: CounterUnit[]; }
interface EnrichedAssessment { assessment: Assessment; counters?: CounterInfo; }
interface PatternData { assessments: EnrichedAssessment[]; }

@customElement('qm-pattern-page')
export class QmPatternPage extends LitElement {
  @property({ attribute: false }) data: PatternData | null = null;
  @state() private _expandedIndex = 0;

  static override styles = css`
    :host { display: block; padding: 10px; font-size: 12px; color: #ccc; }
    .assessment-item { margin-bottom: 8px; }
    .assessment-header { cursor: pointer; font-weight: bold; padding: 4px 0; }
    .confidence-bar { background: #1a1a3e; height: 4px; border-radius: 2px; margin: 2px 0 6px; }
    .bar-fill { height: 4px; border-radius: 2px; }
    .assessment-body { padding: 4px 0 8px; }
    .rationale { font-size: 11px; color: #999; margin-bottom: 6px; }
    .counter-section { margin: 4px 0; }
    .counter-section ul { margin: 2px 0 0 16px; list-style: none; }
    .counter-unit { cursor: pointer; text-decoration: underline; }
    .counter-unit:hover { color: #88bbff; }
  `;

  private _toggle(index: number): void {
    this._expandedIndex = this._expandedIndex === index ? -1 : index;
  }

  private _selectUnit(unitType: string): void {
    if (typeof (window as any).selection?.set === 'function') {
      (window as any).selection.set({ type: 'unitType', unitType, isEnemy: false, source: 'workbench' });
    }
  }

  override render() {
    if (!this.data?.assessments?.length) {
      return html`<div>No pattern data</div>`;
    }
    return html`${this.data.assessments.map((ea, i) => this._renderAssessment(ea, i))}`;
  }

  private _renderAssessment(ea: EnrichedAssessment, i: number) {
    const a = ea.assessment;
    const conf = Math.round(a.confidence * 100);
    const barColor = conf > 70 ? '#44ff44' : conf > 50 ? '#ffaa00' : '#ff4444';
    const expanded = this._expandedIndex === i;
    const arrow = expanded ? '▼' : '▶';

    return html`
      <div class="assessment-item">
        <div class="assessment-header" @click=${() => this._toggle(i)}>${arrow} ${a.archetype} (${conf}%)</div>
        <div class="confidence-bar"><div class="bar-fill" style="width:${conf}%;background:${barColor}"></div></div>
        ${expanded ? html`
          <div class="assessment-body">
            <div class="rationale">${a.rationale ?? ''}</div>
            ${ea.counters ? this._renderCounters(ea.counters) : nothing}
          </div>
        ` : nothing}
      </div>
    `;
  }

  private _renderCounters(counters: CounterInfo) {
    return html`
      ${counters.strongCounters?.length ? html`
        <div class="counter-section"><strong>Strong Counters:</strong>
          <ul>${counters.strongCounters.map(c => html`
            <li>${c.units.map(u => html`<span class="counter-unit" @click=${() => this._selectUnit(u)}>${u}</span>`)} — ${c.action ?? ''}</li>
          `)}</ul>
        </div>
      ` : nothing}
      ${counters.weakCounters?.length ? html`
        <div class="counter-section"><strong>Weak Counters:</strong>
          <ul>${counters.weakCounters.map(c => html`
            <li>${c.units.map(u => html`<span class="counter-unit" @click=${() => this._selectUnit(u)}>${u}</span>`)} — ${c.action ?? ''}</li>
          `)}</ul>
        </div>
      ` : nothing}
    `;
  }
}
```

- [ ] **Step 2: Add import to workbench-entry.ts**

```ts
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-detail-pane';
import './workbench/qm-pattern-page.js';
import './workbench/qm-workbench-controller.js';
```

- [ ] **Step 3: Wire pattern data in visualizer.js**

In the workbench WebSocket `onmessage` handler, replace the `case 'pattern'` line:

```js
case 'pattern':
  workbenchState.pattern = event.payload;
  var patternPage = document.querySelector('qm-pattern-page');
  if (patternPage) patternPage.data = event.payload;
  break;
```

Remove `renderPatternPage()` and `toggleAssessment()` functions.

- [ ] **Step 4: Verify pattern page renders**

Run: `mvn quarkus:dev -pl quarkmind-sc2`

Open visualizer. Click "Pattern" tab. Verify pattern assessments render with expandable sections and clickable counter units.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/webui/ quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat: pattern page Lit component — replaces renderPatternPage() Refs #289"
```

### Task 4: Coaching + Strategy page Lit components

**Files:**
- Create: `quarkmind-sc2/src/main/webui/workbench/qm-coaching-page.ts`
- Create: `quarkmind-sc2/src/main/webui/workbench/qm-strategy-page.ts`
- Modify: `quarkmind-sc2/src/main/webui/workbench-entry.ts` — add imports
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js` — remove old render functions, wire components

**Interfaces:**
- Consumes: `WorkbenchEvent` JSON with `type: "coaching"`, `"coaching_compliance"`, `"strategy"`
- Produces: `<qm-coaching-page>` with `data` property (coaching items array), fires `coaching-response` custom event; `<qm-strategy-page>` with `data` property

- [ ] **Step 1: Create qm-coaching-page.ts**

Create `quarkmind-sc2/src/main/webui/workbench/qm-coaching-page.ts`:

```ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';

interface CoachingItem {
  advice: string;
  domain: string;
  urgency?: string;
  gameFrame: number;
  correlationId: string;
  complianceStatus?: string;
}

@customElement('qm-coaching-page')
export class QmCoachingPage extends LitElement {
  @property({ attribute: false }) data: CoachingItem[] = [];

  static override styles = css`
    :host { display: block; padding: 10px; font-size: 12px; color: #ccc; }
    .coaching-item { border-bottom: 1px solid #1a1a3e; padding: 6px 0; }
    .coaching-header { font-size: 11px; color: #88bbff; }
    .coaching-advice { margin: 4px 0; }
    .coaching-status { font-size: 11px; color: #999; }
    .coaching-controls { display: flex; align-items: center; gap: 4px; margin-top: 4px; }
    .coaching-btn { cursor: pointer; padding: 2px 8px; border: 1px solid #555; background: #222; color: #ccc; border-radius: 3px; font-size: 12px; }
    .coaching-accept:hover { background: #1a3a1a; border-color: #4a4; }
    .coaching-dismiss:hover { background: #3a1a1a; border-color: #a44; }
  `;

  private _formatTime(gameFrame: number): string {
    const secs = Math.floor(gameFrame / 22.4);
    const mins = Math.floor(secs / 60);
    const rem = secs % 60;
    return `${mins}:${String(rem).padStart(2, '0')}`;
  }

  private _respond(correlationId: string, response: string): void {
    this.dispatchEvent(new CustomEvent('coaching-response', {
      bubbles: true, composed: true,
      detail: { correlationId, response },
    }));
  }

  override render() {
    if (!this.data.length) {
      return html`<div>No coaching advice yet</div>`;
    }
    return html`${this.data.map(c => this._renderItem(c))}`;
  }

  private _renderItem(c: CoachingItem) {
    const isPending = !c.complianceStatus;
    return html`
      <div class="coaching-item">
        <div class="coaching-header">${this._formatTime(c.gameFrame)} [${c.domain}] ${c.urgency ?? ''}</div>
        <div class="coaching-advice">${c.advice}</div>
        <div class="coaching-controls">
          ${isPending ? html`
            <button class="coaching-btn coaching-accept" @click=${() => this._respond(c.correlationId, 'DONE')}>✓ Accept</button>
            <button class="coaching-btn coaching-dismiss" @click=${() => this._respond(c.correlationId, 'DECLINE')}>✗ Dismiss</button>
          ` : nothing}
          <span class="coaching-status">${c.complianceStatus ?? '⏳ Pending'}</span>
        </div>
      </div>
    `;
  }
}
```

- [ ] **Step 2: Create qm-strategy-page.ts**

Create `quarkmind-sc2/src/main/webui/workbench/qm-strategy-page.ts`:

```ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

interface StrategyData {
  strategyId: string;
  archetype: string;
  confidence: number;
  pivotCount: number;
}

@customElement('qm-strategy-page')
export class QmStrategyPage extends LitElement {
  @property({ attribute: false }) data: StrategyData | null = null;

  static override styles = css`
    :host { display: block; padding: 10px; font-size: 12px; color: #ccc; }
    .strategy-label { font-weight: bold; color: #88bbff; margin-bottom: 4px; }
    .strategy-value { font-size: 16px; margin-bottom: 8px; }
    .strategy-row { font-size: 12px; margin: 2px 0; }
    .strategy-row span { color: #999; }
  `;

  override render() {
    if (!this.data) return html`<div>No strategy data</div>`;
    const s = this.data;
    return html`
      <div>
        <div class="strategy-label">Active Strategy</div>
        <div class="strategy-value">${s.strategyId}</div>
        <div class="strategy-row"><span>Archetype:</span> ${s.archetype}</div>
        <div class="strategy-row"><span>Confidence:</span> ${(s.confidence * 100).toFixed(0)}%</div>
        <div class="strategy-row"><span>Pivots:</span> ${s.pivotCount}</div>
      </div>
    `;
  }
}
```

- [ ] **Step 3: Update workbench-entry.ts**

```ts
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-detail-pane';
import './workbench/qm-pattern-page.js';
import './workbench/qm-coaching-page.js';
import './workbench/qm-strategy-page.js';
import './workbench/qm-workbench-controller.js';
```

- [ ] **Step 4: Wire coaching + strategy in visualizer.js**

Update the WebSocket `onmessage` handler:

```js
case 'coaching':
  workbenchState.coaching.unshift(event.payload);
  if (workbenchState.coaching.length > 20) workbenchState.coaching.pop();
  var coachingPage = document.querySelector('qm-coaching-page');
  if (coachingPage) coachingPage.data = [...workbenchState.coaching];
  break;
case 'coaching_compliance':
  var statusMap = { ENDORSED: '✅ Endorsed', CHALLENGED: '❌ Challenged', NEUTRAL: '⏸ Neutral', SUPERSEDED: '⏭ Superseded' };
  var label = statusMap[event.payload.status] || event.payload.status;
  var entry = workbenchState.coaching.find(function(c) { return c.correlationId === event.payload.correlationId; });
  if (entry) {
    entry.complianceStatus = label;
    var cp = document.querySelector('qm-coaching-page');
    if (cp) cp.data = [...workbenchState.coaching];
  }
  break;
case 'strategy':
  workbenchState.strategy = event.payload;
  var strategyPage = document.querySelector('qm-strategy-page');
  if (strategyPage) strategyPage.data = event.payload;
  break;
```

Add coaching response listener (replaces `sendCoachingResponse()`):

```js
document.addEventListener('coaching-response', function(e) {
  if (window.__workbenchWs && window.__workbenchWs.readyState === 1) {
    window.__workbenchWs.send(JSON.stringify({
      type: 'coaching_response',
      correlationId: e.detail.correlationId,
      response: e.detail.response,
    }));
  }
});
```

Remove `renderCoachingPage()`, `applyComplianceUpdate()`, `sendCoachingResponse()`, `renderStrategyPage()`.

- [ ] **Step 5: Remove old CSS classes from visualizer.html**

Remove all `.assessment-*`, `.coaching-*`, `.strategy-*`, `.counter-*` CSS rules from visualizer.html — these are now in the Lit component Shadow DOM.

- [ ] **Step 6: Verify all three pages work**

Run: `mvn quarkus:dev -pl quarkmind-sc2`

Open visualizer. Verify:
- Pattern tab: assessments render, expand/collapse works, counter units are clickable
- Coaching tab: advice renders, accept/dismiss buttons work (send WS message)
- Strategy tab: strategy data renders

- [ ] **Step 7: Commit**

```bash
git add quarkmind-sc2/src/main/webui/ quarkmind-sc2/src/main/resources/META-INF/resources/
git commit -m "feat: coaching + strategy page Lit components Refs #289"
```

## Batch 3: Commentary Server-Side + Commentary Page

### Task 5: Commentary server-side — payload, enricher, history

**Files:**
- Create: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/CommentaryPayload.java`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchPayload.java` — add to permits clause
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchEnricher.java` — add `onCommentaryCompleted` observer
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchBroadcaster.java` — add commentary case in `updateSnapshot`
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/WorkbenchSocket.java` — add history-on-connect, reorder `onOpen`
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/WorkbenchEventTest.java` — add serialization test
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/WorkbenchEnricherTest.java` — add observer test

**Interfaces:**
- Consumes: `CommentaryCompleted` CDI event, `CommentaryChannelBroker.channelId()`, `MessageService.history()`
- Produces: `WorkbenchEvent("commentary", CommentaryPayload)` broadcast, `WorkbenchEvent("commentary_snapshot", CommentaryPayload)` on connect

- [ ] **Step 1: Write failing test — CommentaryPayload serialization**

Add to `WorkbenchEventTest.java`:

```java
@Test
void commentary_payload_serializes_correctly() throws Exception {
    var payload = new CommentaryPayload("Great expansion!", "commentary.reactive",
        "REACTIVE", 1500, "commentator-atlas", 120L, Instant.parse("2026-08-26T12:00:00Z"));
    var event = new WorkbenchEvent("commentary", payload);
    String json = objectMapper.writeValueAsString(event);
    assertTrue(json.contains("\"type\":\"commentary\""));
    assertTrue(json.contains("\"text\":\"Great expansion!\""));
    assertTrue(json.contains("\"gameFrame\":1500"));
    assertTrue(json.contains("\"workerId\":\"commentator-atlas\""));
    assertTrue(json.contains("\"commentaryType\":\"REACTIVE\""));
    assertTrue(json.contains("\"capability\":\"commentary.reactive\""));
    assertTrue(json.contains("\"latencyMs\":120"));
}
```

Note: WorkbenchPayload has no `@JsonTypeInfo` annotations — the server only serializes, never deserializes. Test serialization output, not round-trip.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkbenchEventTest -q`

Expected: FAIL — `CommentaryPayload` class does not exist.

- [ ] **Step 3: Create CommentaryPayload and update WorkbenchPayload permits**

Create `CommentaryPayload.java`:

```java
package io.quarkmind.qa.workbench;

import java.time.Instant;

public record CommentaryPayload(
    String text,
    String capability,
    String commentaryType,
    long gameFrame,
    String workerId,
    long latencyMs,
    Instant createdAt
) implements WorkbenchPayload {}
```

Update `WorkbenchPayload.java`:

```java
package io.quarkmind.qa.workbench;

public sealed interface WorkbenchPayload permits PatternPayload, CoachingPayload, CoachingCompliancePayload, StrategyPayload, CommentaryPayload {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkbenchEventTest -q`

Expected: PASS

- [ ] **Step 5: Write failing test — WorkbenchEnricher onCommentaryCompleted**

Add to `WorkbenchEnricherTest.java`:

```java
@Test
void broadcasts_commentary_completed_event() {
    var event = new CommentaryCompleted("commentator-atlas", "commentary.reactive",
        1500, "Great expansion timing!", CommentaryType.REACTIVE, 120L);
    enricher.onCommentaryCompleted(event);

    assertEquals(1, broadcaster.events.size());
    var wbEvent = broadcaster.events.getFirst();
    assertEquals("commentary", wbEvent.type());
    var payload = (CommentaryPayload) wbEvent.payload();
    assertEquals("Great expansion timing!", payload.text());
    assertEquals("commentary.reactive", payload.capability());
    assertEquals("REACTIVE", payload.commentaryType());
    assertEquals(1500, payload.gameFrame());
    assertEquals("commentator-atlas", payload.workerId());
    assertEquals(120L, payload.latencyMs());
    assertNotNull(payload.createdAt());
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkbenchEnricherTest -q`

Expected: FAIL — `onCommentaryCompleted` method does not exist.

- [ ] **Step 7: Implement onCommentaryCompleted in WorkbenchEnricher**

Add to `WorkbenchEnricher.java`:

```java
void onCommentaryCompleted(@Observes CommentaryCompleted event) {
    broadcaster.broadcast(new WorkbenchEvent("commentary",
        new CommentaryPayload(event.text(), event.capability(),
            event.commentaryType().name(), event.gameFrame(),
            event.workerId(), event.latencyMs(), java.time.Instant.now())));
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn test -pl quarkmind-sc2 -Dtest=WorkbenchEnricherTest -q`

Expected: PASS

- [ ] **Step 9: Update WorkbenchBroadcaster.updateSnapshot**

Add explicit case in `updateSnapshot()`:

```java
private void updateSnapshot(WorkbenchEvent event) {
    switch (event.type()) {
        case "pattern"  -> latestPattern  = event;
        case "strategy" -> latestStrategy = event;
        case "coaching" -> latestCoaching = event;
        case "commentary", "commentary_snapshot" -> {}
        default -> {}
    }
}
```

- [ ] **Step 10: Update WorkbenchSocket — history-on-connect**

Inject `CommentaryChannelBroker` and `MessageService`. Reorder `onOpen` to send history before registering for live broadcasts. Add `@Blocking` because `messageService.history()` does JDBC — cannot run on the Vert.x event loop:

```java
@Inject CommentaryChannelBroker commentaryChannelBroker;
@Inject io.casehub.qhorus.runtime.message.MessageService messageService;

@io.smallrye.common.annotation.Blocking
@OnOpen
public void onOpen(WebSocketConnection connection) {
    sendCommentaryHistory(connection);
    broadcaster.addSession(connection);
}

private void sendCommentaryHistory(WebSocketConnection connection) {
    UUID channelId = commentaryChannelBroker.channelId();
    if (channelId == null) return;
    try {
        var all = messageService.history(channelId, 0L, 500);
        var recent = all.size() > 100 ? all.subList(all.size() - 100, all.size()) : all;
        for (var msg : recent) {
            try {
                var completed = objectMapper.readValue(msg.content(), CommentaryCompleted.class);
                var event = new WorkbenchEvent("commentary_snapshot",
                    new CommentaryPayload(completed.text(), completed.capability(),
                        completed.commentaryType().name(), completed.gameFrame(),
                        completed.workerId(), completed.latencyMs(), msg.createdAt()));
                connection.sendText(objectMapper.writeValueAsString(event))
                    .subscribe().with(ignored -> {}, err -> {});
            } catch (Exception e) {
                // skip malformed messages
            }
        }
    } catch (Exception e) {
        // channel not yet initialized — skip history
    }
}
```

- [ ] **Step 11: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`

Expected: All existing tests pass. No regressions.

- [ ] **Step 12: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/qa/workbench/ quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/
git commit -m "feat: commentary server-side — payload, enricher observer, history-on-connect Refs #289"
```

### Task 6: Commentary page Lit component

**Files:**
- Create: `quarkmind-sc2/src/main/webui/workbench/qm-commentary-page.ts`
- Modify: `quarkmind-sc2/src/main/webui/workbench-entry.ts` — add imports
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js` — handle commentary + commentary_snapshot events

**Interfaces:**
- Consumes: `WorkbenchEvent` JSON with `type: "commentary"` and `"commentary_snapshot"` from WebSocket
- Produces: `<qm-commentary-page>` custom element; `messages` property accepts `QhorusMessage[]`

- [ ] **Step 1: Create qm-commentary-page.ts**

Create `quarkmind-sc2/src/main/webui/workbench/qm-commentary-page.ts`. This component embeds `<blocks-channel-feed>` and receives `QhorusMessage[]` per the design spec §4:

```ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { QhorusMessage } from '@casehubio/blocks-ui-channel-activity/types.js';
import '@casehubio/blocks-ui-channel-activity/channel-feed.js';

@customElement('qm-commentary-page')
export class QmCommentaryPage extends LitElement {
  @property({ attribute: false }) messages: QhorusMessage[] = [];

  static override styles = css`
    :host { display: flex; flex-direction: column; height: 100%; }
    blocks-channel-feed { flex: 1; }
  `;

  private _formatSender = (sender: string): string => {
    const short = sender.replace(/^commentator-/, '');
    return short.charAt(0).toUpperCase() + short.slice(1);
  };

  override render() {
    return html`
      <blocks-channel-feed
        .messages=${this.messages}
        .channelId=${'quarkmind-commentary'}
        .channelName=${'Commentary'}
        .autoScroll=${true}
        .terminalDimming=${false}
        .eventStyling=${false}
        .formatSender=${this._formatSender}
      ></blocks-channel-feed>
    `;
  }
}
```

- [ ] **Step 2: Update workbench-entry.ts**

```ts
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-detail-pane';
import './workbench/qm-pattern-page.js';
import './workbench/qm-coaching-page.js';
import './workbench/qm-strategy-page.js';
import './workbench/qm-commentary-page.js';
import './workbench/qm-workbench-controller.js';
```

- [ ] **Step 3: Wire commentary events in visualizer.js**

Add to the `workbenchState` object:

```js
var workbenchState = { pattern: null, coaching: [], strategy: null, commentary: [] };
var commentaryCounter = 0;
```

Add WebSocket message handling. Map `CommentaryPayload` to `QhorusMessage` per design spec §4:

```js
case 'commentary':
case 'commentary_snapshot':
  var qm = {
    id: 'commentary-' + (commentaryCounter++),
    channelId: 'quarkmind-commentary',
    sender: event.payload.workerId,
    messageType: 'STATUS',
    actorType: 'AGENT',
    content: event.payload.text,
    topic: event.payload.commentaryType,
    topicId: '',
    replyCount: 0,
    artefactRefs: [],
    createdAt: event.payload.createdAt,
  };
  workbenchState.commentary = [...workbenchState.commentary, qm];
  if (workbenchState.commentary.length > 200) {
    workbenchState.commentary = workbenchState.commentary.slice(-200);
  }
  var commentaryPage = document.querySelector('qm-commentary-page');
  if (commentaryPage) commentaryPage.messages = workbenchState.commentary;
  break;
```

Note: immutable array update (`[...old, new]`) for Lit change detection. Capped at 200 messages.

- [ ] **Step 4: Verify commentary page works**

Run: `mvn quarkus:dev -pl quarkmind-sc2`

Open visualizer. Click "Commentary" tab. Initially shows "No commentary yet". When commentary events arrive (requires commentary pipeline active), messages appear in chronological order.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/webui/ quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js
git commit -m "feat: commentary page Lit component with direct message rendering Refs #289"
```

## Batch 4: Replay Commentary + Cleanup

### Task 7: Replay commentary — activation guard + profile wiring

**Files:**
- Modify: `quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java` — add `commentaryEnabled` config, guard in `wireCommentary()`
- Modify: `quarkmind-sc2/src/main/resources/application.properties` — add `quarkmind.commentary.enabled` per profile
- Test: `quarkmind-sc2/src/test/java/io/quarkmind/agent/QuarkMindCaseHubTest.java` (if exists) or verify via dev mode

**Interfaces:**
- Consumes: `quarkmind.commentary.enabled` config property
- Produces: Commentary pipeline active only when config is `true` AND ChatModel is available

- [ ] **Step 1: Add commentaryEnabled config to QuarkMindCaseHub**

Add field:

```java
@ConfigProperty(name = "quarkmind.commentary.enabled", defaultValue = "false")
boolean commentaryEnabled;
```

- [ ] **Step 2: Add guard to wireCommentary()**

Insert at the top of `wireCommentary()`, before the existing ChatModel check:

```java
if (!commentaryEnabled) {
    log.info("[CASEHUB] Commentary disabled via quarkmind.commentary.enabled=false");
    return 0;
}
```

- [ ] **Step 3: Add config properties**

Add to `application.properties`:

```properties
quarkmind.commentary.enabled=false
%replay.quarkmind.commentary.enabled=true
%replay.quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
%replay.quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY:}
```

- [ ] **Step 4: Run tests to verify no regression**

Run: `mvn test -pl quarkmind-sc2 -q`

Expected: All tests pass. The default `false` means commentary is disabled in test/mock profiles (existing behaviour unchanged).

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/java/io/quarkmind/agent/QuarkMindCaseHub.java quarkmind-sc2/src/main/resources/application.properties
git commit -m "feat: commentary activation guard + replay profile wiring Refs #289"
```

### Task 8: Playwright test rewrites

**Files:**
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/WorkbenchRenderTest.java` — rewrite for blocks-ui layout
- Modify: `quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/WorkbenchSocketIT.java` — rewrite for Lit components

**Interfaces:**
- Consumes: blocks-ui custom elements in Shadow DOM
- Produces: Passing Playwright test suite

- [ ] **Step 1: Update WorkbenchRenderTest**

The existing test queries `#page-pattern .assessment-item` and `.wb-tab` — both removed. Rewrite to:
- Assert `blocks-split-workbench` element exists
- Assert `blocks-detail-pane` element exists with 4 tabs
- Assert canvas container exists in the list slot
- Update `window.__test.workbenchPatternCount` to query via component `.data` property:

```js
workbenchPatternCount: () => {
  var pp = document.querySelector('qm-pattern-page');
  return pp && pp.data ? pp.data.assessments.length : 0;
},
```

- [ ] **Step 2: Update WorkbenchSocketIT**

Rewrite to verify data flows through Lit components:
- Pattern: verify `qm-pattern-page` has `.data` set after WebSocket message
- Coaching: verify `qm-coaching-page` has `.data` set, coaching-response event fires
- Strategy: verify `qm-strategy-page` has `.data` set
- Commentary: verify `qm-commentary-page` has `.messages` set

- [ ] **Step 3: Run Playwright tests**

Run: `mvn test -pl quarkmind-sc2 -Pplaywright`

Expected: All Playwright tests pass.

- [ ] **Step 4: Commit**

```bash
git add quarkmind-sc2/src/test/java/io/quarkmind/qa/workbench/ quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js
git commit -m "test: rewrite Playwright tests for blocks-ui workbench Refs #289"
```

### Task 9: Delete old workbench code + final cleanup

**Files:**
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.js` — remove dead code
- Modify: `quarkmind-sc2/src/main/resources/META-INF/resources/visualizer.html` — remove dead CSS

**Interfaces:**
- Produces: Clean codebase with no dead vanilla JS workbench code

- [ ] **Step 1: Remove dead functions from visualizer.js**

Remove these functions if they still exist (should have been removed incrementally):
- `setupWorkbenchTabs()`
- `renderPatternPage()`
- `toggleAssessment()`
- `renderCoachingPage()`
- `applyComplianceUpdate()`
- `sendCoachingResponse()`
- `renderStrategyPage()`

Remove old workbench page CSS classes from visualizer.html if not already removed.

- [ ] **Step 2: Remove stale window.__test workbench references**

Check `window.__test` API in visualizer.js for stale references to removed workbench functions. Update `workbenchPatternCount` to query the Lit component instead:

```js
workbenchPatternCount: () => {
  var pp = document.querySelector('qm-pattern-page');
  return pp && pp.data ? pp.data.assessments.length : 0;
},
```

- [ ] **Step 3: Verify full application**

Run: `mvn quarkus:dev -pl quarkmind-sc2`

Open visualizer. Verify:
- Split workbench renders correctly
- All four tabs work
- Pattern/coaching/strategy data flows correctly
- Coaching accept/dismiss works
- Canvas renders with camera controls
- Status bar shows frame/phase/game/intel
- Draggable divider works
- Counter unit clicks select units on canvas

- [ ] **Step 4: Run full test suite**

Run: `mvn test -pl quarkmind-sc2 -q`

Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add quarkmind-sc2/src/main/resources/META-INF/resources/
git commit -m "chore: remove dead vanilla JS workbench code Refs #289"
```

## References

- [2026-08-26-workbench-blocks-ui-design.md] — design spec this plan implements
- [quarkmind-sc2/.../visualizer.html:1-175] — current HTML layout
- [quarkmind-sc2/.../visualizer.js:783-921] — current rendering functions
- [quarkmind-sc2/.../WorkbenchSocket.java] — WebSocket endpoint
- [quarkmind-sc2/.../WorkbenchEnricher.java] — CDI event observer
- [quarkmind-sc2/.../WorkbenchBroadcaster.java:62-69] — updateSnapshot switch
- [quarkmind-sc2/.../WorkbenchPayload.java:3] — sealed interface permits clause
- [quarkmind-sc2/.../QuarkMindCaseHub.java:406-477] — wireCommentary method
- [quarkmind-sc2/.../CommentaryCompleted.java] — CDI event record
- [blocks-ui/components/split-workbench/src/split-workbench.ts] — layout shell
- [blocks-ui/components/detail-pane/src/detail-pane.ts] — tabbed detail panel
- [blocks-ui/components/channel-activity/src/channel-feed.ts] — message feed
- [chat-app/pom.xml:86-152] — reference Quinoa + blocks-ui Maven consumption pattern
- [chat-app/src/main/webui/package.json] — reference portal resolutions pattern
- [GitHub #289] — focal issue
- [GitHub #290] — deferred: synchronized replay-commentary
- [GitHub #291] — deferred: channel-activity sidebar panels
