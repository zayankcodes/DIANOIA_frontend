# DIANOIA Frontend — Week 1 Deliverables (FEASIBLE MEAN, MAX LEARNING)
# Stack (mandatory): Next.js + TypeScript + Tailwind
# Stack (recommended): shadcn/ui, React Flow, dagre, zod, Zustand, TanStack Query (optional)
#
# You are building a “Reasoning IDE” that renders DIANOIA’s AnalysisResult:
# - Source pane: span-accurate highlights for ADUs
# - Graph pane: ADU graph (+ optional inference I-node overlay)
# - Findings pane: defeaters/coherence/fallacies/warnings with jump-to-target
#
# Constraint: NO “smart guessing” in UI.
# - Spans come from API offsets
# - Graph comes from API edges/inference_nodes
# - Findings come from API lists
#
# ============================================================
# 0) WEEK 1 SUCCESS DEFINITION
# ============================================================
#
# By end of week:
# 1) A user can paste text, click Analyze, and get a stable, interactive analysis UI.
# 2) Clicking in any pane synchronizes selection across all panes.
# 3) Inference overlay toggle works (I-nodes appear; undercut targets make sense).
# 4) Export JSON works (downloads the exact analysis).
# 5) There are at least 3 meaningful unit tests (spans + resolver).
#
# Hard “done” checklist:
# - No console errors
# - No span drift (highlights match exactly)
# - Selection sync is immediate + consistent
# - Loading/error states are real, not hand-waved
#
# ============================================================
# 1) WHAT YOU’RE BUILDING (MENTAL MODEL)
# ============================================================
#
# Think of DIANOIA as a compiler:
# input: raw argument text
# output: a typed artifact (AnalysisResult) that contains:
# - extracted units (ADUs) with char spans
# - edges between units (support/attack/undercut + weights)
# - inference objects (I-nodes) representing hyperedges (premises -> conclusion)
# - diagnostics (defeaters, fallacies, coherence failures, warnings)
# - scoring (warrant_strength per claim)
#
# Frontend job:
# - present that artifact
# - allow exploration via “selection”
# - never silently distort it
#
# ============================================================
# 2) WEEK 1 DELIVERABLES — DAY BY DAY
# ============================================================
#
# Day 1 — Boot + API + schema validation + fixture mode
# ----------------------------------------------------
# Goals:
# - FE can develop without backend (fixtures)
# - FE can develop with backend (real calls)
# - Runtime validation prevents silent breakage
#
# Tasks:
# 1) Create /analyze page (Next App Router)
# 2) Build API client: POST /analyze + POST /analyze/structured
# 3) Add zod schema for AnalysisResultSchema (+ nested schemas)
# 4) Add “Fixture Mode” toggle (dropdown: ai_tutors / slow_by_default / analogy)
# 5) Render:
#    - doc_id, n_adus, n_edges
#    - collapsible “Raw JSON” panel
#    - clear error panel if validation fails
#
# Acceptance:
# - With fixtures: analysis loads instantly
# - With backend: analysis loads after network call
# - If schema mismatch: user sees “ValidationError at path …”
#
# Day 2 — Source pane highlighting + selection core
# ------------------------------------------------
# Goals:
# - Precise span highlighting using API offsets
# - Handles overlaps deterministically
# - Clicking a highlight selects that ADU globally
#
# Tasks:
# 1) Implement span segmentation algorithm (see Section 8)
# 2) Render text as segments; highlight segments owned by an ADU
# 3) Hover behavior (optional but recommended): hover highlight brightens
# 4) Click highlight -> store.selectAdu(aduId)
# 5) Selecting an ADU scrolls into view (smooth scroll + focus ring)
#
# Acceptance:
# - Selected ADU has visible highlight even if offscreen (scroll)
# - Overlapping spans do not produce “broken” DOM or misclicks
#
# Day 3 — Graph pane (ADUs + edges) + layout + sync
# -------------------------------------------------
# Goals:
# - React Flow graph from API adus + edges
# - Clicking node selects ADU, highlights span
#
# Tasks:
# 1) Map ADUs -> React Flow nodes
# 2) Map edges -> React Flow edges
# 3) Dagre layout LR (left-to-right)
# 4) Selection sync:
#    - click node -> selectAdu
#    - selectAdu -> center node in viewport (fit/zoom lightly)
#
# Acceptance:
# - Graph renders correctly for 5–50 ADUs without chaos
# - Clicking any node updates Source pane selection
#
# Day 4 — Findings pane + target resolver + detail drawer
# -------------------------------------------------------
# Goals:
# - Show defeaters + coherence failures + fallacies + warnings
# - Clicking a finding jumps to its target (ADU / edge / inference)
# - Drawer shows full details (reasoning, evidence_span, scores)
#
# Tasks:
# 1) Normalize findings into a UI model (see Section 10)
# 2) Implement resolveTarget() (see Section 11)
# 3) Clicking finding sets selection:
#    - if target_type=claim -> adu
#    - if target_type=inference -> inode (or drawer fallback if overlay off)
#    - if coherence failure -> edge
# 4) Detail drawer opens for:
#    - selected finding
#    - selected inference node
#    - selected edge (shows label, weight, NLI scores)
#
# Acceptance:
# - Every row in findings list is clickable and does something visible
#
# Day 5 — Inference overlay + export + tests
# -----------------------------------------
# Goals:
# - Toggle overlay: adds inference nodes and hyperedges
# - Export button downloads analysis JSON
# - Minimum unit tests exist and pass in CI/local
#
# Tasks:
# 1) Graph overlay mode:
#    - add inode nodes
#    - add premise->inode edges and inode->conclusion edges
# 2) Export:
#    - downloadJSON(`${doc_id}.json`, analysis)
# 3) Tests:
#    - span normalization clamping
#    - overlap winner rule
#    - resolveTarget inference fallback behavior
#
# Acceptance:
# - Toggle changes graph without breaking selection
# - Export produces identical object to current analysis in memory
#
# ============================================================
# 3) STACK DECISIONS (LOCK THESE IN WEEK 1)
# ============================================================
#
# Mandatory:
# - Next.js (App Router)
# - TypeScript
# - Tailwind
#
# Recommended (do it):
# - zod (runtime validation)
# - React Flow (graph)
# - dagre (layout)
# - shadcn/ui (fast, consistent components)
#
# State management:
# - Use Zustand for Week 1 (simple, centralized, avoids prop drilling).
#
# Data fetching:
# - Plain fetch is fine.
# - TanStack Query optional; use only if FE is comfortable. (Not required Week 1.)
#
# ============================================================
# 4) PROJECT SETUP — COMMANDS (COPY/PASTE)
# ============================================================
#
# Create app:
#   npx create-next-app@latest dianoia-frontend --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
#   cd dianoia-frontend
#
# Install deps:
#   pnpm add zod zustand reactflow dagre
#   pnpm add -D @types/dagre vitest @testing-library/react @testing-library/jest-dom jsdom
#
# shadcn/ui:
#   pnpm dlx shadcn@latest init
#   pnpm dlx shadcn@latest add button card tabs badge separator textarea input scroll-area tooltip dialog dropdown-menu
#
# Env:
#   cat > .env.local << 'EOF'
#   NEXT_PUBLIC_DIANOIA_API_BASE_URL=http://localhost:8000
#   EOF
#
# Run:
#   pnpm dev
#
# ============================================================
# 5) REQUIRED FOLDER STRUCTURE (DON’T FREESTYLE)
# ============================================================
#
# src/app/analyze/page.tsx
# src/components/analyze/
#   AnalyzeLayout.tsx
#   AnalyzeToolbar.tsx
#   SourcePane.tsx
#   GraphPane.tsx
#   FindingsPane.tsx
#   DetailDrawer.tsx
#
# src/lib/api/client.ts
# src/lib/schema/analysis.ts
# src/lib/state/
#   store.ts
#   types.ts
#   selectors.ts
#
# src/lib/text/spans.ts
# src/lib/graph/
#   mapping.ts
#   layout.ts
#   focus.ts
#
# src/lib/util/
#   download.ts
#   clamp.ts
#   sort.ts
#
# src/fixtures/
#   ai_tutors.json
#   slow_by_default.json
#   analogy.json
#
# src/__tests__/
#   spans.test.ts
#   resolveTarget.test.ts
#
# ============================================================
# 6) API CONTRACT — WHAT YOUR UI MUST SUPPORT
# ============================================================
#
# POST /analyze -> AnalysisResultSchema
# POST /analyze/structured -> AnalysisResultSchema
#
# Top-level (minimum required):
# - doc_id: string
# - source_text: string
# - n_adus: number
# - n_edges: number
# - sanity_warnings: SanityWarningSchema[]
# - coherence_failures: CoherenceFailureSchema[]
# - witnesses: WitnessSchema[]
# - fallacies: FallacyResultSchema[]
# - unsupported_claims: string[]
# - grounded_extension: string[]
# - preferred_extensions_available: boolean
# - preferred_extensions: string[][] | null
# - warrant_strength: Record<string, number>
#
# Expanded graph (required for Week 1 UI):
# - adus: ADUResponseSchema[]
# - edges: EdgeResponseSchema[]
# - inference_nodes: InferenceNodeSchema[]
#
# Findings:
# - defeaters: DefeaterSchema[]
#
# ============================================================
# 7) FRONTEND DOMAIN MODEL (TS TYPES YOU SHOULD DEFINE)
# ============================================================
#
# In src/lib/state/types.ts define:
#
# type AduId = string
# type EdgeId = string
# type InferenceId = string
#
# type Selection =
#   | { kind: "none" }
#   | { kind: "adu"; id: AduId }
#   | { kind: "edge"; id: EdgeId }
#   | { kind: "inference"; id: InferenceId }
#   | { kind: "finding"; key: string }  // stable key for findings list
#
# type UiFlags = {
#   showInferenceOverlay: boolean
#   useFixture: boolean
#   fixtureName: "ai_tutors" | "slow_by_default" | "analogy"
# }
#
# type AnalyzeState = {
#   analysis: AnalysisResult | null
#   loading: boolean
#   error: string | null
#   selection: Selection
#   hoverAduId: AduId | null
#   hoverEdgeId: EdgeId | null
#   flags: UiFlags
# }
#
# type AnalyzeActions = {
#   setTextInput(text: string): void
#   runAnalyze(): Promise<void>
#   loadFixture(name: UiFlags["fixtureName"]): Promise<void>
#   setSelection(sel: Selection): void
#   setFlags(patch: Partial<UiFlags>): void
# }
#
# Keep selection as a discriminated union to avoid “two sources of truth”.
#
# ============================================================
# 8) SPAN HIGHLIGHTING — THE ALGORITHM (YOU MUST IMPLEMENT THIS)
# ============================================================
#
# Problem:
# - ADU spans are char offsets into source_text
# - Spans may overlap
# - You need deterministic clickable highlighting
#
# Requirements:
# - No heuristic splitting. Use spans.
# - Overlaps resolved deterministically.
# - Output segments cover full text in order.
#
# Steps:
# 1) Normalize spans:
#    - clamp start/end to [0, text.length]
#    - if start >= end -> drop span (invalid)
# 2) Create boundary points: collect all start/end + 0 + text.length
# 3) Sort unique boundaries
# 4) For each segment [b[i], b[i+1]):
#    - determine active spans where span.start <= seg.start && span.end >= seg.end
#    - if none active: segment owner = null
#    - else pick winner by rule:
#         winner = shortest span length
#         tie -> earliest start
#         tie -> lexicographic adu_id
# 5) Merge adjacent segments with same owner (to reduce DOM nodes)
#
# Output:
# segments: Array<{ start: number; end: number; text: string; ownerAduId: string|null }>
#
# Unit tests you MUST write:
# - clamps correctly
# - overlap chooses smallest span
# - adjacency merge works
#
# ============================================================
# 9) GRAPH — MAPPING RULES (NO INVENTING)
# ============================================================
#
# Base Graph Mode:
# - Nodes = analysis.adus (filter out synthetic/defeater nodes; API already filters)
# - Node label = adu.text (truncate in node UI, full in drawer)
# - Node style encodes:
#     - adu.type (major_claim vs claim vs premise)
#     - warrant_strength[adu.id] (badge/heat)
#     - verifiability_type (chip)
#
# - Edges = analysis.edges
#   - edge.label: "support" | "attack" | "undercut" | ...
#   - edge.weight: number (post NLI)
#   - edge.nli_scores optional -> show in drawer
#
# Inference Overlay Mode:
# - Add nodes = analysis.inference_nodes
# - Add edges:
#     - premise ADU -> inference node (edge type "premise")
#     - inference node -> conclusion ADU (edge type "conclusion")
#
# Layout:
# - Use dagre rankdir LR
# - Layout recompute ONLY when:
#     - analysis changes
#     - overlay flag changes
#
# Performance:
# - Memoize nodes/edges mapping
# - Don’t recompute layout on hover/selection
#
# ============================================================
# 10) FINDINGS — WHAT TO RENDER (WEEK 1)
# ============================================================
#
# Render 4 sections, in this order:
#
# Section 1: Defeaters (analysis.defeaters)
# - Sort by (severity * confidence) desc
# - Display:
#   - effect badge (undercut/rebut/attack)
#   - label (cq_violation_..., possible_...)
#   - score = severity*confidence (0..1)
#   - short evidence preview (evidence_span first ~80 chars)
#
# Section 2: Coherence Failures (analysis.coherence_failures)
# - Sort by severity desc
# - Display:
#   - failure_code (SUPPORT_CONTRADICTED / ATTACK_ENTAILED)
#   - edge_id + src/tgt
#   - severity + NLI scores preview
#
# Section 3: Structural Fallacies (analysis.fallacies)
# - Sort by score desc
# - Display:
#   - fallacy_type (missing_support/circularity)
#   - tier
#   - implicated_nodes
#
# Section 4: Sanity Warnings (analysis.sanity_warnings)
# - Display code + message
#
# Each row is clickable and goes through resolveTarget().
#
# ============================================================
# 11) TARGET RESOLUTION (THE MOST IMPORTANT “FEELS SMART” PIECE)
# ============================================================
#
# Central function:
#   resolveTarget(item, analysis, flags) -> Action
#
# Return type:
# type FocusAction =
#   | { kind: "selectAdu"; aduId: string }
#   | { kind: "selectEdge"; edgeId: string }
#   | { kind: "selectInference"; inferenceId: string }
#   | { kind: "openDrawer"; drawerKind: "inference"|"edge"|"finding"; id: string; fallbackAduId?: string }
#
# Rules:
# A) DefeaterSchema:
#    - target_type === "claim" -> selectAdu(target_id)
#    - target_type === "inference":
#        if flags.showInferenceOverlay -> selectInference(target_id)
#        else -> openDrawer("inference", target_id, fallbackAduId = inference.conclusion)
#    - target_type === "edge" -> selectEdge(target_id)   (if you ever add)
#
# B) CoherenceFailureSchema:
#    - selectEdge(edge_id)
#
# C) FallacyResultSchema:
#    - selectAdu(adu_id)
#
# D) SanityWarningSchema:
#    - openDrawer("finding", warning.code)  (or no-op; but should be visible)
#
# Drawer contents:
# - for inference: show scheme, premises texts, conclusion text, warrant if present
# - for edge: show src/tgt texts, label, weight, nli_scores
# - for defeater: show reasoning + evidence_span + severity/confidence
#
# ============================================================
# 12) UI SPEC (MINIMUM VIABLE “SERIOUS” LOOK)
# ============================================================
#
# Layout:
# - 3 columns with resizable-ish feel (optional)
# - Each pane scrolls independently
#
# Top toolbar:
# - textarea input
# - Analyze button
# - Fixture toggle + dropdown
# - Inference overlay toggle
# - Export JSON button
#
# Source pane:
# - Display text in a readable style
# - Highlight segments with subtle background
# - Selected highlight has strong outline
# - Hover highlight brightens
#
# Graph pane:
# - FitView on first render
# - Selected node has outline
# - Optional mini-map (can skip Week 1)
#
# Findings pane:
# - Group headers with counts
# - Each row shows: badge + label + score
# - Clicking row selects and scrolls/centers target
#
# Accessibility (Week 1 minimum):
# - All clickable rows are buttons or have role/button + keyboard support
# - Drawer is focus-trapped (shadcn Dialog handles this)
# - Prefer reduced motion respects OS setting (optional Week 1)
#
# ============================================================
# 13) ERROR HANDLING (NO SILENT FAILURES)
# ============================================================
#
# Possible failures:
# - network error
# - non-200 response
# - JSON parse error
# - zod validation error
#
# Requirements:
# - show a readable error string
# - include “raw response snippet” if possible
# - never render partial state as if it’s correct
#
# ============================================================
# 14) FIXTURE MODE (CRITICAL FOR FE SPEED)
# ============================================================
#
# Add fixture JSON files that are EXACT API responses.
# - FE can iterate on UI without running backend
# - Use fixture toggle to swap analysis in store
#
# Implementation:
# - store.flags.useFixture
# - store.flags.fixtureName
# - loadFixture() reads from /src/fixtures/*.json (import or fetch via next static)
#
# ============================================================
# 15) TESTING (WEEK 1 MINIMUM, LEARNING MAXIMUM)
# ============================================================
#
# Use Vitest + jsdom.
#
# Tests to implement:
# 1) spans.test.ts:
#    - clamp
#    - overlap winner
#    - merge adjacency
#
# 2) resolveTarget.test.ts:
#    - inference target with overlay ON -> selectInference
#    - inference target with overlay OFF -> openDrawer + fallbackAduId
#
# Optional:
# - mapping tests (adus->nodes count, edges->edges count)
#
# ============================================================
# 16) PERFORMANCE BUDGETS (WEEK 1 TARGETS)
# ============================================================
#
# - 50 ADUs should render without lag on a laptop
# - No layout recomputation on hover/selection
# - Source segmentation is O(n_boundaries * active_spans) worst-case; keep spans small.
#
# If perf becomes an issue:
# - index spans by boundaries to speed active lookup
# - virtualize findings list (later)
#
# ============================================================
# 17) PR DISCIPLINE (HOW TO SHIP WITHOUT CHAOS)
# ============================================================
#
# PR1: Boot/API/schema/fixtures
# PR2: SourcePane spans + selection store
# PR3: GraphPane mapping/layout + sync
# PR4: Findings + resolver + drawer
# PR5: Overlay + export + tests
#
# Each PR includes:
# - screenshots/gif
# - acceptance checklist
# - “known limitations”
#
# ============================================================
# 18) COMMON PITFALLS (AVOID THESE)
# ============================================================
#
# - Treating spans as token offsets (they are char offsets)
# - Rendering overlaps by nesting <mark> (breaks DOM and clicks)
# - Having separate selection state per pane (desync hell)
# - Recomputing dagre layout on every render
# - Forgetting that inference_nodes are hyperedges (premises list)
# - Ignoring schema validation (you will chase ghosts later)
#
# ============================================================
# 19) STRETCH (ONLY IF ALL ABOVE IS DONE)
# ============================================================
#
# - Search within source text (jump to ADU by id)
# - Node filter by verifiability_type
# - “Copy link” to selection state (URL query string)
# - Add a small “Warrant Strength” sorted list
#
# ============================================================
# 20) WHAT I STILL NEED FROM YOU (TO MAKE THIS 100% EXECUTABLE)
# ============================================================
#
# Provide fixture JSON responses (copy/paste from real backend outputs):
# - ai_tutors.json
# - slow_by_default.json
# - analogy.json
#
# Each file must include:
# - source_text
# - adus, edges, inference_nodes
# - defeaters at least in one file
#
# ============================================================
# END — WEEK 1 GUIDE
# ============================================================
