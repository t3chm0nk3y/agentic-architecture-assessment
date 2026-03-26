# agent.web — Workbench UI Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** Workbench UI, Run Inspection, SSE Consumption, Step Trace Viewer, Artifact Viewer  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.web is the first-party UI for the platform. It owns the visual interface — nothing else renders to users.

agent.web is a consumer, not a controller. It calls agent.api exclusively and renders the results. It does not access agent.core, agent.adapters, or agent.workflows directly. Every piece of data it displays comes from an HTTP endpoint or SSE stream defined in agent.api.

agent.web is the single source of truth for:

- how users interact with the platform visually
- how workflows and skills are presented for selection
- how run input is collected and submitted
- how run execution is displayed in real time
- how step traces are inspected
- how artifacts are viewed
- how approvals are presented and resolved

---

## 2. What agent.web Owns vs Delegates

**Owns**

- React application structure and routing
- UI component library
- Layout, styling, and interaction design
- SSE client connection and event rendering
- Run state reconstruction from the event stream
- Form construction for workflow input
- Approval resolution UI
- Artifact rendering

**Does Not Own**

- Execution logic → agent.core (via agent.api)
- Data persistence → agent.adapters (via agent.api)
- Authentication enforcement → agent.api
- Canonical contract definitions → agent.core
- Workflow/skill definitions → agent.workflows
- HTTP endpoint contracts → agent.api
- Correlation ID generation → agent.api

---

## 3. Internal Component Map

```
agent.web
├── src/
│   ├── app.tsx                        # Root application, routing
│   ├── api/
│   │   ├── client.ts                  # HTTP client wrapper (fetch + auth headers)
│   │   ├── types.ts                   # TypeScript types mirroring agent.api DTOs
│   │   └── sse.ts                     # SSE client with reconnection logic
│   ├── pages/
│   │   ├── Dashboard.tsx              # Landing: recent runs, quick launch
│   │   ├── WorkflowLauncher.tsx       # Workflow selection, input form, submit
│   │   ├── AutonomousLauncher.tsx     # Goal input, skill selection, submit
│   │   ├── RunView.tsx                # Real-time run monitoring
│   │   └── RunHistory.tsx             # Past runs list with filtering
│   ├── components/
│   │   ├── run/
│   │   │   ├── RunTimeline.tsx        # Vertical timeline of run events
│   │   │   ├── StepCard.tsx           # Individual step with status, timing, expand
│   │   │   ├── ToolInvocationCard.tsx # Tool call detail within a step
│   │   │   ├── LLMStreamView.tsx      # Live token streaming display
│   │   │   ├── ApprovalGate.tsx       # Approval context + resolve controls
│   │   │   └── RunStatusBadge.tsx     # Status indicator with color coding
│   │   ├── artifacts/
│   │   │   └── ArtifactViewer.tsx     # Render artifact by content type
│   │   ├── forms/
│   │   │   ├── DynamicForm.tsx        # JSON Schema → form fields
│   │   │   └── SkillSelector.tsx      # Multi-select skill picker
│   │   └── layout/
│   │       ├── Shell.tsx              # App shell: sidebar, header, content area
│   │       ├── Sidebar.tsx            # Navigation
│   │       └── Header.tsx             # Branding, status indicators
│   ├── hooks/
│   │   ├── useRunStream.ts            # SSE connection + state management for a run
│   │   ├── useRun.ts                  # Fetch and poll run state
│   │   └── useWorkflows.ts            # Fetch workflow/skill lists
│   └── styles/
│       └── tailwind.config.ts         # Tailwind configuration with design tokens
```

---

## 4. Design Language

### 4.1 Aesthetic

The UI follows a **dark ops** aesthetic: dark backgrounds, high-contrast text, monospace accents for technical data, and restrained use of color for status signaling.

### 4.2 Typography

| Role | Font | Usage |
|------|------|-------|
| Headings | Rajdhani | Page titles, section headers, run IDs |
| Body | IBM Plex Sans | Descriptions, form labels, general text |
| Code / Data | JetBrains Mono | Step outputs, tool payloads, JSON, timestamps |

### 4.3 Color System

Colors are functional, not decorative. They communicate status.

| Token | Usage |
|-------|-------|
| `--color-bg-primary` | Main background (near-black) |
| `--color-bg-surface` | Card and panel backgrounds (dark gray) |
| `--color-bg-elevated` | Hover states, active elements |
| `--color-text-primary` | Primary text (high-contrast white) |
| `--color-text-secondary` | Labels, timestamps, metadata (muted) |
| `--color-text-code` | Monospace data values |
| `--color-status-running` | Blue — execution in progress |
| `--color-status-completed` | Green — terminal success |
| `--color-status-failed` | Red — terminal failure |
| `--color-status-awaiting` | Amber — paused, awaiting human action |
| `--color-status-cancelled` | Gray — externally cancelled |
| `--color-accent` | Primary action buttons, links |

### 4.4 Motion

framer-motion is used for:

- Step cards expanding/collapsing
- New events entering the timeline (slide-in from top)
- Status badge transitions
- Page transitions

Animations are fast (150-250ms) and functional — they communicate state changes, not decoration.

### 4.5 Icons

lucide-react is the sole icon set. Common mappings:

| Icon | Usage |
|------|-------|
| `Play` | Start run |
| `Square` | Cancel run |
| `CheckCircle` | Completed |
| `XCircle` | Failed |
| `Clock` | Awaiting approval |
| `Wrench` | Tool invocation |
| `Brain` | Agent/LLM step |
| `Cog` | Automated step |
| `Shield` | Human gate |
| `ChevronDown` / `ChevronRight` | Expand/collapse |

---

## 5. Pages

### 5.1 Dashboard

The landing page. Shows:

- **Recent runs** — last 10 runs with status badges, workflow name, timing
- **Quick launch** — cards for the most-used workflows with a single-click start (pre-filled defaults)
- **System health** — summary from `GET /health` (server status, connection state)

### 5.2 Workflow Launcher

Triggered when the user selects a workflow to execute.

1. `GET /v1/workflows` → display workflow list with names, descriptions, tags
2. User selects a workflow → `GET /v1/workflows/{workflow_id}` → load full definition
3. **DynamicForm** renders input fields from the workflow's `input_schema` (JSON Schema → form)
4. User optionally overrides default skills via **SkillSelector**
5. Submit → `POST /v1/runs` with `workflow_id`, `input`, and `skills`
6. On 201 → navigate to **RunView** for the new run

### 5.3 Autonomous Launcher

Triggered when the user wants to run the agent without a predefined workflow.

1. Free-text goal input (textarea)
2. **SkillSelector** for choosing which skills to activate
3. Submit → `POST /v1/runs` with no `workflow_id`, `input: { goal: "..." }`, and `skills`
4. On 201 → navigate to **RunView**

### 5.4 RunView

The primary monitoring page. Displays a single run in real time.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│ Header: Run ID | Workflow Name | Status Badge        │
├──────────────────────┬──────────────────────────────┤
│                      │                              │
│   Run Timeline       │   Detail Panel               │
│   (left, scrollable) │   (right, context-sensitive)  │
│                      │                              │
│   ┌─ step.started    │   Selected step detail:       │
│   │  Step: Normalize │   - Input (JSON tree)         │
│   │                  │   - Output (JSON tree)        │
│   ├─ tool.invoked    │   - Tool invocations          │
│   │  sentinel.query  │   - LLM stream (if agent)     │
│   │                  │   - Timing                    │
│   ├─ tool.result     │   - Error detail (if failed)  │
│   │  230ms           │                              │
│   │                  │                              │
│   ├─ step.completed  │                              │
│   │                  │                              │
│   ├─ step.started    │   Or: Approval gate           │
│   │  Step: Classify  │   - Context data              │
│   │  ...             │   - Approve / Reject / Modify │
│   │                  │                              │
│   └─ run.completed   │                              │
│                      │                              │
└──────────────────────┴──────────────────────────────┘
```

**Behavior:**

- On mount, connect to `GET /v1/runs/{run_id}/stream`
- Reconstruct run state from the event stream
- Each event appends to the timeline and updates the relevant step card
- Clicking a step card opens its detail in the right panel
- If a `run.awaiting_approval` event arrives, the **ApprovalGate** component renders in the detail panel with context data and resolution controls
- When the run reaches a terminal state, the SSE connection closes and the timeline shows a final status indicator

### 5.5 RunHistory

A filterable list of past runs.

- `GET /v1/runs?status={status}&limit=50&offset=0`
- Filter by status (tabs or dropdown)
- Pagination
- Click a run → navigate to **RunView**

---

## 6. SSE Client

### 6.1 useRunStream Hook

The primary mechanism for real-time run monitoring.

```typescript
interface UseRunStreamReturn {
  events: Event[];
  runStatus: RunStatus | null;
  steps: Map<string, StepState>;
  error: Error | null;
  isConnected: boolean;
}

function useRunStream(runId: string): UseRunStreamReturn {
  // 1. Connect to GET /v1/runs/{run_id}/stream
  // 2. Parse SSE events into typed Event objects
  // 3. Maintain derived state:
  //    - Ordered event list
  //    - Step state map (step_id → current status, timing, output, tool calls)
  //    - Overall run status
  // 4. Handle reconnection with since_sequence
  // 5. Clean up on unmount
}
```

### 6.2 Reconnection

If the SSE connection drops:

1. Record the highest `sequence` number received
2. Wait 1 second (with exponential backoff, max 30 seconds)
3. Reconnect with `?since_sequence={last_sequence}`
4. Merge replayed events with local state (deduplicate by sequence number)

### 6.3 State Reconstruction

The client maintains a local state model derived entirely from the event stream. This model drives all UI rendering.

```typescript
interface RunState {
  runId: string;
  status: RunStatus;
  events: Event[];
  steps: Map<string, StepState>;
  currentApproval: Approval | null;
}

interface StepState {
  stepId: string;
  name: string;
  stepType: StepType;
  status: StepStatus;
  startedAt: string | null;
  completedAt: string | null;
  toolInvocations: ToolInvocationState[];
  llmTokens: string[];           // Accumulated streaming tokens
  error: ErrorState | null;
}
```

Every event type maps to a state update:

| Event Type | State Update |
|------------|-------------|
| run.started | Set status to RUNNING |
| run.completed | Set status to COMPLETED |
| run.failed | Set status to FAILED, set error |
| run.cancelled | Set status to CANCELLED |
| run.awaiting_approval | Set status to AWAITING_APPROVAL, set currentApproval |
| step.started | Add/update step in map with RUNNING status |
| step.completed | Update step status to COMPLETED, set output |
| step.failed | Update step status to FAILED, set error |
| tool.invoked | Add tool invocation to step's list |
| tool.result | Update tool invocation with result and latency |
| llm.token | Append token to step's llmTokens array |
| approval.requested | Set currentApproval with context |
| approval.resolved | Clear currentApproval, update status |

---

## 7. Key Components

### 7.1 DynamicForm

Generates form fields from a JSON Schema (the workflow's `input_schema`).

Supported field types:

| JSON Schema Type | Form Element |
|-----------------|--------------|
| `string` | Text input |
| `string` with `format: "date"` | Date picker |
| `string` with `enum` | Select dropdown |
| `number` / `integer` | Number input |
| `boolean` | Toggle switch |
| `object` | Nested fieldset |
| `array` of `string` | Tag input / multi-line |

For complex schemas that exceed what DynamicForm can render, fall back to a raw JSON editor (textarea with syntax highlighting via JetBrains Mono).

### 7.2 ApprovalGate

Renders when a run reaches AWAITING_APPROVAL.

- Displays the `context` data from the approval entity (formatted JSON tree)
- Three action buttons: **Approve**, **Reject**, **Modify**
- Approve and Reject send `POST /v1/runs/{run_id}/approvals/{approval_id}/resolve` with the resolution
- Modify opens an editable JSON view of the context, and the edited payload is sent as the resolution

### 7.3 RunTimeline

A vertical timeline of events, rendered from the `events` array in RunState.

- Events are grouped by step
- Each step group shows the step name, type icon, status badge, and timing
- Within a step group, tool invocations are shown as sub-items
- LLM tokens are not shown individually in the timeline — they render in the detail panel's **LLMStreamView**
- New events animate in (framer-motion slide from top)

### 7.4 ArtifactViewer

Renders artifacts produced by a run. Artifact rendering is content-type-aware:

| Content Type | Rendering |
|-------------|-----------|
| `application/json` | Formatted JSON tree with expand/collapse |
| `text/plain` | Monospace preformatted text |
| `text/markdown` | Rendered markdown |
| `text/html` | Sandboxed iframe |
| Other | Download link |

---

## 8. API Client

### 8.1 HTTP Client

A thin wrapper around `fetch` that handles:

- Base URL configuration (environment variable: `VITE_API_BASE_URL`)
- Authentication headers (`Authorization: Bearer {key}`)
- JSON serialization/deserialization
- Error response parsing

```typescript
class ApiClient {
  constructor(private baseUrl: string, private apiKey?: string) {}

  async post<T>(path: string, body: unknown): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        ...(this.apiKey && { Authorization: `Bearer ${this.apiKey}` }),
      },
      body: JSON.stringify(body),
    });
    if (!response.ok) throw new ApiError(response);
    return response.json();
  }

  async get<T>(path: string, params?: Record<string, string>): Promise<T> {
    const url = new URL(`${this.baseUrl}${path}`);
    if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    const response = await fetch(url.toString(), {
      headers: this.apiKey ? { Authorization: `Bearer ${this.apiKey}` } : {},
    });
    if (!response.ok) throw new ApiError(response);
    return response.json();
  }
}
```

### 8.2 TypeScript Types

`api/types.ts` mirrors the agent.api DTOs as TypeScript interfaces. These are manually maintained — they are the frontend's contract with the API.

```typescript
interface RunResponse {
  run_id: string;
  workflow_id: string | null;
  workflow_version: string | null;
  status: RunStatus;
  input: Record<string, unknown>;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
  step_count: number;
  current_step_index: number | null;
  error: ErrorResponse | null;
  artifacts: ArtifactResponse[];
}

type RunStatus = "CREATED" | "RUNNING" | "AWAITING_APPROVAL" | "COMPLETED" | "FAILED" | "CANCELLED";
type StepType = "AUTOMATED" | "AGENT" | "HUMAN_GATE";
type StepStatus = "PENDING" | "RUNNING" | "COMPLETED" | "FAILED";

// ... other interfaces mirror agent.api DTOs
```

---

## 9. Configuration

agent.web is configured via environment variables at build time:

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_API_BASE_URL` | agent.api base URL | `http://localhost:8000` |
| `VITE_API_KEY` | API key for authentication (optional in dev) | — |

No runtime configuration files. No access to `agent.yaml`.

---

## 10. Boundary Rules

agent.web enforces these rules:

- **Consumes agent.api exclusively** — no direct imports from agent.core, agent.workflows, or agent.adapters
- **No execution logic** — the UI submits requests and renders responses; it never decides what to execute
- **No data persistence** — all state is derived from API responses and the SSE event stream
- **No canonical contract imports** — TypeScript types are maintained independently, mirroring DTOs
- **No server-side rendering** — agent.web is a static SPA served independently
- **CORS is required** — agent.api must allow agent.web's origin

---

## 11. Build and Deployment

### 11.1 Build

```bash
npm run build
```

Produces a static bundle (HTML, JS, CSS) in `dist/`. The bundle is framework-agnostic — it can be served by any static file server, CDN, or embedded in the FastAPI application.

### 11.2 Serving Options

- **Standalone**: Serve `dist/` via nginx, Caddy, or any static server
- **Embedded**: Mount `dist/` as static files in the FastAPI application (convenient for single-process development)
- **CDN**: Deploy to Cloudflare Pages, Vercel, Netlify, etc.

### 11.3 Development

```bash
npm run dev
```

Runs Vite dev server with hot module replacement. Proxies API requests to the local agent.api instance.

---

## 12. Observability

agent.web's observability is limited to what it can observe from the API and SSE stream. It does not emit Logfire spans or structlog entries.

### 12.1 Run Timeline Visualization

The RunTimeline component is the primary observability surface for users. It visualizes:

- Event sequence and timing
- Step durations (startedAt → completedAt)
- Tool invocation latency
- Approval wait time
- Error context on failure

### 12.2 Step Trace Viewer

The detail panel for a selected step shows:

- Resolved input (what the step received from RunContext)
- Output (what the step produced)
- Each tool invocation with input, output, and latency
- LLM token stream (for AGENT steps)
- Error detail with diagnostic context (for failed steps)

This provides the same information as Logfire traces but in a user-friendly format.

---

## 13. Implementation Notes for Claude Code

- Use Vite as the build tool — it is the standard for React + TypeScript projects
- Use React Router for client-side routing
- Use `EventSource` API for SSE connections — it handles reconnection natively, but the custom `useRunStream` hook adds `since_sequence` support which `EventSource` alone does not provide
- Tailwind configuration should define the design tokens from Section 4.3 as CSS custom properties
- All API calls go through the `ApiClient` class — no raw `fetch` calls scattered in components
- State management is local (React state + hooks) — no global store needed for v1. Each page manages its own data fetching.
- JSON tree rendering can use a simple recursive component — no need for a heavy library
- The DynamicForm component is the most complex piece; scope it to the supported types in Section 7.1 and fall back to raw JSON for anything else
- agent.web has no test dependencies on other modules — it can be developed and tested against a mock API

---

## 14. Example User Flow

```
1. User opens the dashboard
   → GET /v1/runs?limit=10 (recent runs)
   → GET /health (system status)
   Dashboard renders with recent runs and health indicators

2. User clicks "Classify TTP" workflow card
   → GET /v1/workflows/classify-ttp
   WorkflowLauncher renders with DynamicForm for input_schema

3. User enters adversary description, clicks "Run"
   → POST /v1/runs { workflow_id: "classify-ttp", input: { description: "..." } }
   ← 201 Created { run_id: "run-001", status: "CREATED" }
   Navigate to RunView for run-001

4. RunView mounts
   → GET /v1/runs/run-001/stream (SSE connection opens)
   Events flow in: run.started → step.started → tool.invoked → tool.result → step.completed → ...
   Timeline builds in real time, step cards animate in

5. Run reaches HUMAN_GATE
   ← run.awaiting_approval event
   ApprovalGate renders with classification results for review

6. User reviews and clicks "Approve"
   → POST /v1/runs/run-001/approvals/apr-001/resolve { resolution: "APPROVED", resolved_by: "user" }
   ← approval.resolved event via SSE
   Run resumes, final steps execute

7. Run completes
   ← run.completed event
   SSE connection closes
   Final status badge: green ✓
   User clicks a step to inspect detail in right panel
```