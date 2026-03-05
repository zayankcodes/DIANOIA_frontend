# DIANOIA Frontend (Week 1) — FROM 0 → DEMO-READY REASONING IDE
# PURE LITERAL MARKDOWN DOCUMENT (everything is inside this single black box)

## Audience
Brand new first-time frontend engineer. Assume:
- you can code, but you may be new to Next.js / React Flow / Tailwind
- you need explicit “what to do next” steps and guardrails
- you should not invent backend behaviors

## Mission (Friday demo definition)
Ship **DIANOIA FE v0.1**: a 3-pane “Reasoning IDE” that consumes the DIANOIA API and lets a user:
1) Paste text → Analyze
2) See **span-accurate highlights** for each ADU in source text
3) See the **argument graph** (nodes=ADUs, edges=relations)
4) See **findings** (defeaters, coherence failures, fallacies, warnings)
5) Click any finding → the UI **jumps/focuses the correct target** (node/edge/inference)
6) Toggle **I-node overlay**
7) Export the analysis JSON

If the demo cannot do the above reliably, it’s not done.

---

# 1) HARD CONSTRAINTS (do not violate)

## 1.1 Mandatory tech stack (locked)
- TypeScript (strict)
- Next.js (App Router)
- Tailwind CSS

## 1.2 Approved additions (use these; don’t bike-shed)
- shadcn/ui (standard components)
- reactflow (graph rendering)
- dagre (layout)
- zod (runtime validation)
- clsx + tailwind-merge (via shadcn utils)

Optional, if time:
- TanStack Query (fetching)
- Playwright (E2E smoke)
- Vitest (unit tests)

## 1.3 Product invariants (locked)
- Use real `adus`, `edges`, and `inference_nodes` from API. No fake graph.
- Highlight spans ONLY by API offsets (`adu.span.start/end`) — never guess.
- IDs are authoritative. Never generate ids client-side.
- Clicking any finding must cause visible focus change (node/edge/inference or detail drawer).
- No backend changes required this week.

---

# 2) WHAT DIANOIA API RETURNS (contract you must implement)

You will call:
- POST `/analyze` with `{ "text": "..." }`
- POST `/analyze/structured` with CanonDoc (ignore for this week unless asked)

Top-level response includes (non-exhaustive):
- doc_id: string
- source_text: string
- n_adus: number
- n_edges: number
- sanity_warnings: SanityWarning[]
- coherence_failures: CoherenceFailure[]
- witnesses: Witness[]
- fallacies: FallacyResult[]
- unsupported_claims: string[]
- grounded_extension: string[]
- preferred_extensions_available: boolean
- preferred_extensions: string[][] | null
- bipolar_degrees_original: NodeDegree[]
- bipolar_degrees_repaired: NodeDegree[]
- repairs_applied: number
- repairs_skipped: number
- defeaters: Defeater[]
- warrant_strength: Record<string, number>
- argument_components: string[][] | null
- extracted: boolean
- extraction_meta: object

NEW (critical for frontend):
- adus: ADU[]
- edges: Edge[]
- inference_nodes: InferenceNode[]

### ADU (ADUResponseSchema)
- id: string                 # ex: "a0"
- span: { start: number, end: number }   # char offsets in source_text
- text: string
- type: string               # major_claim | claim | premise | etc.
- modality: string | null
- speaker_commitment: number | null
- provenance: string | null
- verifiability_type: string | null
- anchor_id: string | null
- propositional_core: string | null

### Edge (EdgeResponseSchema)
- id: string                 # ex: "r0"
- src: string                # ADU id
- tgt: string                # ADU id
- label: string              # support | attack | undercut | other
- weight: number             # post-NLI weight
- nli_scores?: { entailment: number, neutral: number, contradiction: number } | null

### InferenceNode (InferenceNodeSchema)
- id: string                 # ex: "i_custom_a0_a1"
- scheme: string             # analogy | authority | practical | ...
- premises: string[]         # ADU ids
- conclusion: string         # ADU id
- warrant?: string | null

### Defeater (DefeaterSchema)
- target_type: "claim" | "inference" | "edge" | "premise" | ...
- target_id: string
- effect: "undercut" | "rebut" | "attack" | ...
- label: string
- evidence_span: string
- reasoning: string
- severity: number (0..1)
- confidence: number (0..1)
- context_edge_id: string | null

---

# 3) WEEK 1 DELIVERABLES (strict)

## Day 1 (PR1): Project boot + API plumbing + schema validation
Deliver:
- Next.js project running with TS+Tailwind
- API client with env var base URL
- zod schemas for full AnalysisResult (including adus/edges/inference_nodes)
- fixture mode (load JSON from /fixtures instead of network)
- /analyze page skeleton renders basic stats, major claim preview

Acceptance:
- Paste text → calls backend → shows “received N adus / M edges”
- If backend down → fixture mode still works
- Any schema mismatch → user sees clear error panel, not a blank screen

## Day 2 (PR2): SourcePane with span highlighting + selection sync
Deliver:
- Source pane renders `source_text`
- Highlights each ADU by `span.start/end`
- Hover highlight ↔ hover node (soft highlight)
- Click highlight selects ADU and scrolls into view
- Selection state is centralized (one source of truth)

Acceptance:
- No span crashes on weird offsets
- Overlaps handled deterministically (see policy below)
- Click highlight always selects the correct ADU

## Day 3 (PR3): GraphPane (ADU graph) with layout + node/edge selection
Deliver:
- Graph pane renders nodes from `adus` and edges from `edges`
- Dagre layout (left-to-right)
- Node style encodes:
  - ADU type (major_claim/claim/premise)
  - verifiability_type badge
  - warrant strength (rounded)
- Click node selects ADU
- Click edge selects edge (shows tooltip or side panel)
- Selecting from SourcePane focuses graph; selecting from graph focuses source

Acceptance:
- Graph renders legibly at 5–50 ADUs
- Layout is stable across rerenders
- Selection sync is bidirectional

## Day 4 (PR4): FindingsPane with “jump to target” navigation
Deliver:
- Findings pane lists:
  - defeaters (sorted by severity*confidence desc)
  - coherence failures
  - structural fallacies
  - sanity warnings
- Clicking a finding focuses the correct target:
  - claim → ADU node + highlight span
  - edge → edge highlight
  - inference → inference focus (via overlay or fallback drawer)
- Detail drawer shows full reasoning text and evidence span

Acceptance:
- Clicking any finding causes visible focus change every time
- Sorting stable and deterministic
- No “dead” findings

## Day 5 (PR5): I-node overlay toggle + export + tests + polish
Deliver:
- Toggle: Show Inference Nodes
- When on:
  - inference nodes rendered as small nodes
  - edges: premise ADU → inference → conclusion ADU
- Defeaters targeting inference nodes become visible targets
- Export JSON button downloads the full response
- Unit tests for span normalization + mapping + target resolver
- Performance guardrails (memoization; no layout on hover)

Acceptance:
- Overlay mode works and doesn’t break base mode
- Export works
- Tests run in CI/local

---

# 4) SETUP: FROM ZERO COMMANDS (copy/paste)

## 4.1 Prereqs
- Node.js v20.x
- pnpm recommended

Check:
  node -v
  pnpm -v

If Node is not 20.x:
- Volta (recommended): https://volta.sh/
- or nvm

## 4.2 Create project
From your workspace folder:
  npx create-next-app@latest dianoia-frontend --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
  cd dianoia-frontend

## 4.3 Install packages
  pnpm add zod reactflow dagre
  pnpm add -D @types/dagre

## 4.4 Install shadcn/ui
  pnpm dlx shadcn@latest init

Prompts:
- TypeScript: yes
- Tailwind: yes
- App Router: yes
- components: src/components
- utils: src/lib/utils

Add components:
  pnpm dlx shadcn@latest add button card tabs badge separator textarea input scroll-area tooltip dialog dropdown-menu

## 4.5 Env vars
Create .env.local:
  touch .env.local

Add:
  NEXT_PUBLIC_DIANOIA_API_BASE_URL=http://localhost:8000

## 4.6 Run
  pnpm dev

Open:
- http://localhost:3000/analyze

---

# 5) REPO STRUCTURE (exact)

Create this structure:

src/
  app/
    analyze/
      page.tsx
  components/
    analyze/
      AnalyzeLayout.tsx
      AnalyzeToolbar.tsx
      SourcePane.tsx
      GraphPane.tsx
      FindingsPane.tsx
      DetailDrawer.tsx
      NodeDetails.tsx
      EdgeDetails.tsx
      InferenceDetails.tsx
  lib/
    api/
      client.ts
      routes.ts
    schema/
      analysis.ts
    state/
      analyzeStore.tsx
      selectors.ts
      types.ts
    graph/
      layout.ts
      mapping.ts
      styles.ts
      reactflowTypes.ts
      focus.ts
    text/
      spans.ts
      highlight.ts
    util/
      clamp.ts
      sort.ts
      download.ts
  fixtures/
    ai_tutors.json
    slow_by_default.json
    analogy_country_business.json

Notes:
- All API parsing and zod lives in src/lib/schema
- All state lives in src/lib/state
- All graph mapping/layout lives in src/lib/graph
- All span/highlight logic lives in src/lib/text

---

# 6) STATE MODEL (single source of truth)

## 6.1 Central rule
Selection state must be centralized. No component should have its own “selected id” that diverges.

## 6.2 Store shape (minimal)
Selection can target:
- aduId
- edgeId
- inferenceId
- findingKey (optional)
- hoverAduId (soft hover)
- hoverEdgeId

Also keep:
- analysis: AnalysisResult | null
- loading/error
- ui flags (showInferenceOverlay)

## 6.3 Resolver rule (critical)
Every click from Findings must be resolved through a single function:
resolveTarget(defeater/failure/fallacy) → { kind: "adu"|"edge"|"inference"|"drawer", id: string, fallbackAduId?: string }

No scattered if/else in UI components.

---

# 7) SPAN HIGHLIGHTING POLICY (must implement)

## 7.1 Span normalization
Given ADU span {start,end}:
- clamp start/end to [0, source_text.length]
- if start >= end → drop span
- sort spans by start asc, then end asc

## 7.2 Overlap policy (deterministic)
Overlaps can happen. For Week 1:
- If overlaps exist, prefer rendering the smallest span “on top” for clickability.
- Do not crash or misalign text.

Simplest robust approach:
- Build “events” for all span boundaries (start/end)
- Sweep line across text; maintain active spans list
- At each segment, choose a single “active winner” span:
  - winner = smallest length; tie-breaker by earlier start; then by adu id
- Render each segment as either plain text or highlighted with winner ADU

This yields:
- stable, deterministic highlight rendering
- correct click target even in overlaps

## 7.3 Click behavior
- Clicking highlight selects that ADU id
- Must scroll highlight into view
- Must also focus graph node

---

# 8) GRAPH POLICY (must implement)

## 8.1 Base graph (Week 1 default)
- Nodes: ADUs only
- Edges: edges[] (src/tgt are ADU ids)

## 8.2 Inference overlay (toggle)
When enabled:
- Render inference_nodes as separate node type
- Render edges:
  - premise ADU → inference node
  - inference node → conclusion ADU
- Keep original ADU edges optionally visible (recommended: keep them visible; they help readability)

## 8.3 Layout rules
Use dagre directed layout:
- direction: LR
- nodesep/ranksep: reasonable defaults (start with 40/80)
- Run layout ONLY when:
  - analysis changes
  - overlay toggle changes
- Do NOT relayout on hover or selection

## 8.4 Styling rules
Node styling:
- major_claim: prominent border
- claim: normal
- premise: lighter
- show warrant strength as badge (rounded 2 decimals)
- show verifiability_type as small chip

Edge styling:
- support: solid
- attack: solid + red-ish (or label-based class)
- undercut: dashed

Tooltip:
- show weight
- show NLI scores if present

---

# 9) FINDINGS POLICY (must implement)

## 9.1 Groups
Render findings in this order:
1) Defeaters (defeaters[])
2) Coherence failures (coherence_failures[])
3) Structural fallacies (fallacies[])
4) Sanity warnings (sanity_warnings[])

## 9.2 Sorting
Defeaters sorted by:
- score = severity * confidence desc
- tie-breaker: label asc

Coherence failures sorted by severity desc.

## 9.3 Click routing rules
On click:
- If target_type === "claim" and target_id is an ADU id → select ADU
- If target_type === "edge" and target_id matches edges[].id → select edge
- If target_type === "inference" and target_id matches inference_nodes[].id:
  - If overlay enabled → select inference node
  - If overlay disabled → open detail drawer AND select fallbackAduId = conclusion (from inference_nodes mapping)
- Otherwise → open detail drawer with raw JSON

---

# 10) PERFORMANCE GUARDRAILS (non-negotiable)

- Memoize derived maps: aduById, edgeById, inodeById
- Memoize React Flow nodes/edges arrays
- Do not compute dagre layout inside render; compute in useMemo/useEffect
- Do not rerender whole graph on hover; use minimal state updates
- Avoid rendering massive lists unvirtualized if >500 findings (unlikely; ok for week 1)

---

# 11) IMPLEMENTATION: FILE-BY-FILE GUIDE

## 11.1 src/lib/api/client.ts
Purpose:
- call backend
- handle base URL
- return parsed JSON
- no zod here (zod in schema)

Implement:
- getBaseUrl() from NEXT_PUBLIC_DIANOIA_API_BASE_URL
- postJSON(path, body)
- throw descriptive errors

## 11.2 src/lib/schema/analysis.ts
Purpose:
- zod schemas for the entire response
- parse response safely
- export TypeScript types via z.infer

Add zod schemas:
- Span, ADU, Edge, InferenceNode
- Defeater, CoherenceFailure, Witness, FallacyResult, SanityWarning, NodeDegree
- AnalysisResultSchema top-level includes new fields:
  - adus, edges, inference_nodes

Rule:
- If schema fails, show user a readable error with path and snippet.

## 11.3 src/lib/state/analyzeStore.tsx
Purpose:
- Context + reducer store for analysis + selection

State:
- analysis: AnalysisResult | null
- loading: boolean
- error: string | null
- selection: { aduId?, edgeId?, inferenceId?, findingKey? }
- hover: { aduId?, edgeId? }
- ui: { showInferenceOverlay: boolean }

Actions:
- SET_ANALYSIS
- SET_LOADING
- SET_ERROR
- SELECT_ADU
- SELECT_EDGE
- SELECT_INFERENCE
- SET_HOVER_ADU
- SET_HOVER_EDGE
- TOGGLE_INFERENCE_OVERLAY
- CLEAR_SELECTION

## 11.4 src/lib/text/spans.ts
Purpose:
- normalize spans (clamp, drop invalid)
- build highlight segments (sweep algorithm)

Exports:
- normalizeSpans(sourceText, adus) → normalized list
- buildSegments(sourceText, normalizedSpans) → segments array:
  - { text: string, aduId: string | null, start: number, end: number }

## 11.5 src/components/analyze/SourcePane.tsx
Purpose:
- render source_text with segments
- click highlight selects adu
- scroll-to selection
- hover highlight sets hoverAduId

## 11.6 src/lib/graph/mapping.ts
Purpose:
- convert adus + edges into React Flow node/edge objects

Exports:
- buildAduNodes(adus, warrant_strength, selection, hover)
- buildAduEdges(edges, selection, hover)
- buildInferenceOverlay(inference_nodes) → nodes + edges
- getConclusionForInference(inferenceId) → aduId

## 11.7 src/lib/graph/layout.ts
Purpose:
- dagre layout calculation

Exports:
- layoutGraph(nodes, edges, options) → positioned nodes

## 11.8 src/components/analyze/GraphPane.tsx
Purpose:
- render React Flow graph
- onNodeClick selects adu/inference depending on node type
- onEdgeClick selects edge
- uses layoutGraph when analysis changes or overlay toggled

## 11.9 src/components/analyze/FindingsPane.tsx
Purpose:
- render grouped findings lists
- click item resolves target and dispatches selection/focus
- show detail drawer for fallback cases

## 11.10 src/components/analyze/DetailDrawer.tsx
Purpose:
- right-side drawer (or modal) showing full details for:
  - selected ADU
  - selected Edge
  - selected InferenceNode
  - selected Finding

---

# 12) UI LAYOUT (must match demo needs)

## 12.1 Analyze page layout
Top:
- input textarea
- Analyze button
- Fixture mode toggle
- Export button (enabled when analysis exists)
- Toggle: “Show inference nodes”

Main:
- 3-column grid:
  - left: SourcePane
  - center: GraphPane
  - right: FindingsPane

Bottom (optional):
- status bar: doc_id, counts, latency

---

# 13) FIXTURE MODE (required)

Why:
- backend may be down
- demo must work offline
- tests need stable data

Implementation:
- Store JSON fixtures under src/fixtures
- In UI, “Use fixtures” dropdown:
  - ai_tutors
  - slow_by_default
  - analogy_country_business
- When fixture mode is on, do not call network.

---

# 14) TESTING REQUIREMENTS (minimum viable)

## 14.1 Unit tests (required)
Test functions:
- normalizeSpans
- buildSegments
- resolveTarget (defeater → selection)
- buildInferenceOverlay mapping correctness

Suggested tool:
- Vitest + @testing-library/react (optional)
If you cannot add a runner this week, at minimum:
- add a small node script to run span tests (but prefer Vitest)

## 14.2 Smoke test checklist (manual)
Before merging each PR, run:
- analyze with fixture
- analyze with backend
- click 5 highlights, 5 nodes, 5 findings
- toggle overlay on/off
- export JSON

---

# 15) PR PROCESS (strict)

## 15.1 PR naming
- PR1: FE: boot + API client + zod + /analyze skeleton
- PR2: FE: SourcePane highlight engine + selection sync
- PR3: FE: GraphPane ReactFlow + dagre layout + selection sync
- PR4: FE: FindingsPane + target resolver + detail drawer
- PR5: FE: Inference overlay + export + tests + perf

## 15.2 PR description template (copy/paste)
Title:
- FE: <short>

Description:
- What changed:
- How to run:
- Demo steps:
- Screenshots/GIF:
- Known limitations:
- Checklist:
  - [ ] fixtures work
  - [ ] backend works
  - [ ] click finding focuses target
  - [ ] no console errors

---

# 16) “DO NOT DO THIS” LIST (common time sinks)
- Do not add auth/accounts
- Do not add saving/projects
- Do not refactor into complex state libraries
- Do not attempt diff/what-if UI (future engine)
- Do not over-animate
- Do not build your own graph library

---

# 17) SAMPLE TEXTS (for demos)
Use these to generate stable graphs:

1) AI tutors (mixed practical/objection):
- "Schools should use AI tutors to give students personalized feedback..."

2) Convenience / slow-by-default essay (longer, multiple claims/objections)

3) Analogy critique (country vs business; marriage vs business partnership)

4) Consent / disclosure / coercion argument (definitional + causal + abductive)

---

# 18) IMPLEMENTATION DETAILS (HIGHEST RISK AREAS)

## 18.1 Span highlights are the most fragile
You must:
- clamp offsets
- handle overlaps
- never crash on invalid spans
- preserve original source_text exactly

## 18.2 Target routing must be centralized
The fastest way to ship bugs:
- scattered click handlers that each interpret targets differently

Must have:
- resolveTarget() in one place

## 18.3 Graph layout must not rerun constantly
Do not compute layout on hover/selection.

---

# 19) CHECKLIST (Friday demo readiness)
- [ ] /analyze loads
- [ ] analyze call works
- [ ] fixtures work
- [ ] spans highlight correctly
- [ ] graph renders correctly
- [ ] selection sync works both ways
- [ ] findings list renders and sorts
- [ ] clicking any finding focuses target
- [ ] inference overlay toggle works
- [ ] export JSON works
- [ ] no console errors
- [ ] minimal tests pass

---

# 20) NEXT: WHAT YOU WILL BUILD AFTER THIS WEEK (do not do now)
- Diff viewer (ledger diffs)
- What-if minimal repair UI
- Profile presets switching
- Persistent workspaces
- Collaboration/comments

END OF DOCUMENT
