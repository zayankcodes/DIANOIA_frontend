# DIANOIA Frontend — Week 1 Deliverables (Feasible Mean, Max Learning)

**Stack (mandatory):** Next.js + TypeScript + Tailwind  
**Stack (recommended):** shadcn/ui, React Flow, dagre, zod, Zustand *(TanStack Query optional)*

You are building a **Reasoning IDE** that renders DIANOIA’s `AnalysisResult`:

- **Source pane:** span-accurate highlights for ADUs  
- **Graph pane:** ADU graph (+ optional inference I-node overlay)  
- **Findings pane:** defeaters/coherence/fallacies/warnings with jump-to-target  

**Constraint: NO “smart guessing” in UI.**
- Spans come from API offsets
- Graph comes from API `edges` / `inference_nodes`
- Findings come from API lists

---

## 0) Week 1 Success Definition

**0a. End-to-end analysis loop — target**  
Paste text → click **Analyze** → stable interactive UI renders.

**0b. Cross-pane selection sync — target**  
Click in any pane → selection updates all panes (immediate, consistent).

**0c. Inference overlay toggle — target**  
Toggle overlay → I-nodes appear/disappear cleanly; undercut targets stay interpretable.

**0d. Export JSON — target**  
Export downloads the **exact** in-memory `analysis` object (no transforms).

**0e. Tests — target**  
At least **3 meaningful unit tests** (spans + resolver) pass locally/CI.

**Hard Week-1 checklist**
- [ ] No console errors
- [ ] No span drift (highlights match exactly)
- [ ] Selection sync is immediate + consistent
- [ ] Loading/error states are real (not hand-waved)
- [ ] No partial/invalid analysis rendered as “fine”

---

## 1) What You’re Building (Mental Model)

DIANOIA behaves like a compiler:

- **input:** raw argument text  
- **output:** typed artifact `AnalysisResult` containing:
  - extracted units (**ADUs**) with **char spans**
  - **edges** between units (support/attack/undercut + weights)
  - **inference nodes** (I-nodes) representing hyperedges (premises → conclusion)
  - diagnostics (defeaters, fallacies, coherence failures, warnings)
  - scoring (`warrant_strength` per claim)

Frontend job:
- present the artifact
- enable exploration via **selection**
- never invent structure

---

## 2) Week 1 Deliverables — Day by Day

### Day 1 — Boot + API + schema validation + fixture mode

**1a. Create `/analyze` page — TODO**  
Create `src/app/analyze/page.tsx` (Next App Router).

**1b. API client (real calls) — TODO**  
Implement:
- `POST /analyze`
- `POST /analyze/structured`

**1c. Runtime validation (zod) — TODO**  
Add `AnalysisResultSchema` (+ nested schemas). Validate every response at runtime. On failure show: `ValidationError at path ...`.

**1d. Fixture Mode toggle — TODO**  
Add toggle + dropdown:
- `ai_tutors`
- `slow_by_default`
- `analogy`

**1e. Minimal render + debug surfaces — TODO**  
Render:
- `doc_id`, `n_adus`, `n_edges`
- collapsible **Raw JSON** panel
- explicit error panel on validation failure

**Acceptance**
- Fixtures load instantly
- Backend mode loads after network call
- Schema mismatch is readable and blocks rendering of invalid UI

---

### Day 2 — Source pane highlighting + selection core

**2a. Implement span segmentation algorithm — TODO**  
Implement in `src/lib/text/spans.ts` (see Section 8).

**2b. Render source as segments — TODO**  
Render full text as segments; highlight segments by `ownerAduId`.  
Do **not** nest `<mark>` or rely on DOM overlap.

**2c. Click highlight → global selection — TODO**  
Click highlight → `setSelection({ kind: "adu", id })`.

**2d. Select → scroll into view — TODO**  
Selecting an ADU scrolls highlight into view (smooth scroll + focus ring).

**Acceptance**
- Selected ADU becomes visible even if initially offscreen
- Overlapping spans remain clickable and deterministic

---

### Day 3 — Graph pane (ADUs + edges) + layout + sync

**3a. Map ADUs → React Flow nodes — TODO**  
Nodes come strictly from `analysis.adus`.

**3b. Map edges → React Flow edges — TODO**  
Edges come strictly from `analysis.edges`.

**3c. Dagre layout LR — TODO**  
Use `rankdir = "LR"`.

**3d. Selection sync — TODO**
- click node → select ADU
- selecting ADU → center node in viewport (light fit/zoom)

**Acceptance**
- Graph renders cleanly for ~5–50 ADUs
- Clicking node updates Source pane selection immediately

---

### Day 4 — Findings pane + target resolver + detail drawer

**4a. Normalize findings into UI rows — TODO**  
Normalize defeaters/coherence/fallacies/warnings into a single `FindingRow[]`.

**4b. Implement `resolveTarget()` — TODO**  
Central resolver maps any finding → a focus action (see Section 11).  
No per-component guessing.

**4c. Click behavior — TODO**
- defeater targeting claim → select ADU
- defeater targeting inference → select inference (or drawer fallback if overlay off)
- coherence failure → select edge

**4d. Detail drawer — TODO**
Drawer opens for:
- selected finding
- selected inference node
- selected edge (label, weight, NLI scores)

**Acceptance**
- Every findings row is clickable and triggers a visible effect

---

### Day 5 — Inference overlay + export + tests

**5a. Overlay mode (I-nodes) — TODO**
Overlay ON adds:
- inference nodes (`analysis.inference_nodes`)
- edges: premise → inode and inode → conclusion

**5b. Export JSON — TODO**
`downloadJSON(\`\${doc_id}.json\`, analysis)` downloads the current analysis object.

**5c. Tests — TODO**
Minimum:
- span normalization clamping
- overlap winner rule
- resolveTarget inference fallback when overlay OFF

**Acceptance**
- Overlay toggle changes graph without breaking selection
- Export matches current in-memory analysis object
- Tests pass locally + CI

---

## 3) Project Setup — Commands (Copy/Paste)

**Create app**
```bash
npx create-next-app@latest dianoia-frontend --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
cd dianoia-frontend
```

**Install deps**
```bash
pnpm add zod zustand reactflow dagre
pnpm add -D @types/dagre vitest @testing-library/react @testing-library/jest-dom jsdom
```

**shadcn/ui**
```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card tabs badge separator textarea input scroll-area tooltip dialog dropdown-menu
```

**Env**
```bash
cat > .env.local << 'EOF'
NEXT_PUBLIC_DIANOIA_API_BASE_URL=http://localhost:8000
EOF
```

**Run**
```bash
pnpm dev
```

---

## 4) Required Folder Structure (Don’t Freestyle)

```txt
src/app/analyze/page.tsx
src/components/analyze/
  AnalyzeLayout.tsx
  AnalyzeToolbar.tsx
  SourcePane.tsx
  GraphPane.tsx
  FindingsPane.tsx
  DetailDrawer.tsx

src/lib/api/client.ts
src/lib/schema/analysis.ts
src/lib/state/
  store.ts
  types.ts
  selectors.ts

src/lib/text/spans.ts
src/lib/graph/
  mapping.ts
  layout.ts
  focus.ts

src/lib/util/
  download.ts
  clamp.ts
  sort.ts

src/fixtures/
  ai_tutors.json
  slow_by_default.json
  analogy.json

src/__tests__/
  spans.test.ts
  resolveTarget.test.ts
```

---

## 5) API Contract — What Your UI Must Support

**5a. Endpoints**
- `POST /analyze` → `AnalysisResultSchema`
- `POST /analyze/structured` → `AnalysisResultSchema`

**5b. Top-level (minimum required)**
- `doc_id: string`
- `source_text: string`
- `n_adus: number`
- `n_edges: number`
- `sanity_warnings: SanityWarningSchema[]`
- `coherence_failures: CoherenceFailureSchema[]`
- `witnesses: WitnessSchema[]`
- `fallacies: FallacyResultSchema[]`
- `unsupported_claims: string[]`
- `grounded_extension: string[]`
- `preferred_extensions_available: boolean`
- `preferred_extensions: string[][] | null`
- `warrant_strength: Record<string, number>`

**5c. Expanded graph (Week 1 required)**
- `adus: ADUResponseSchema[]`
- `edges: EdgeResponseSchema[]`
- `inference_nodes: InferenceNodeSchema[]`

**5d. Findings**
- `defeaters: DefeaterSchema[]`

---

## 6) Frontend Domain Model (TypeScript)

In `src/lib/state/types.ts`:

```ts
export type AduId = string;
export type EdgeId = string;
export type InferenceId = string;

export type Selection =
  | { kind: "none" }
  | { kind: "adu"; id: AduId }
  | { kind: "edge"; id: EdgeId }
  | { kind: "inference"; id: InferenceId }
  | { kind: "finding"; key: string };

export type UiFlags = {
  showInferenceOverlay: boolean;
  useFixture: boolean;
  fixtureName: "ai_tutors" | "slow_by_default" | "analogy";
};

export type AnalyzeState = {
  analysis: AnalysisResult | null;
  loading: boolean;
  error: string | null;
  selection: Selection;
  hoverAduId: AduId | null;
  hoverEdgeId: EdgeId | null;
  flags: UiFlags;
  textInput: string;
};

export type AnalyzeActions = {
  setTextInput(text: string): void;
  runAnalyze(): Promise<void>;
  loadFixture(name: UiFlags["fixtureName"]): Promise<void>;
  setSelection(sel: Selection): void;
  setFlags(patch: Partial<UiFlags>): void;
};
```

---

## 7) Span Highlighting — Algorithm (Must Implement)

Problem:
- ADU spans are **char offsets** into `source_text`
- Spans may overlap
- Need deterministic clickable highlighting

Rules:
- Use spans as truth (no heuristics).
- Resolve overlaps deterministically.
- Output segments cover full text in order.

Algorithm:
1) Normalize spans:
   - clamp `start/end` to `[0, text.length]`
   - if `start >= end` → drop span
2) Boundaries: collect all `start/end` + `0` + `text.length`
3) Sort unique boundaries
4) For each segment `[b[i], b[i+1])`:
   - active spans satisfy: `span.start <= seg.start && span.end >= seg.end`
   - owner rules:
     - winner = shortest span length
     - tie → earliest start
     - tie → lexicographic `adu_id`
5) Merge adjacent segments with same owner

Output:
- `segments: Array<{ start; end; text; ownerAduId: string | null }>`

Required tests:
- clamps correctly
- overlap chooses smallest span
- adjacency merge works

---

## 8) Graph Mapping Rules (No Inventing)

**Base Graph Mode**
- Nodes = `analysis.adus`
- Edges = `analysis.edges`
- Node label = `adu.text` (truncate in node UI; full in drawer)
- Node UI can encode:
  - `adu.type` (major_claim vs claim vs premise)
  - `warrant_strength[adu.id]` (badge/heat)
  - `verifiability_type` (chip)

**Inference Overlay Mode**
- Add nodes = `analysis.inference_nodes`
- Add edges:
  - premise ADU → inference node (edge type `"premise"`)
  - inference node → conclusion ADU (edge type `"conclusion"`)

**Layout**
- dagre rankdir LR
- recompute layout only when:
  - analysis changes
  - overlay flag changes

---

## 9) Findings Rendering (Week 1)

Render sections in this order:

**9a. Defeaters (`analysis.defeaters`)**
- sort by `(severity * confidence)` desc
- show: effect badge, label, score, evidence preview (~80 chars)

**9b. Coherence failures (`analysis.coherence_failures`)**
- sort by severity desc
- show: failure_code, edge_id + src/tgt, severity + NLI preview

**9c. Structural fallacies (`analysis.fallacies`)**
- sort by score desc
- show: fallacy_type, tier, implicated_nodes

**9d. Sanity warnings (`analysis.sanity_warnings`)**
- show: code + message

Every row is clickable and goes through `resolveTarget()`.

---

## 10) Target Resolution (Central “Feels Smart” Piece)

`resolveTarget(item, analysis, flags) -> FocusAction`

```ts
export type FocusAction =
  | { kind: "selectAdu"; aduId: string }
  | { kind: "selectEdge"; edgeId: string }
  | { kind: "selectInference"; inferenceId: string }
  | {
      kind: "openDrawer";
      drawerKind: "inference" | "edge" | "finding";
      id: string;
      fallbackAduId?: string;
    };
```

Rules:
- Defeater:
  - `target_type==="claim"` → select ADU
  - `target_type==="inference"`:
    - overlay ON → select inference
    - overlay OFF → open drawer for inference (plus fallback conclusion ADU if available)
- Coherence failure → select edge
- Fallacy → select ADU
- Sanity warning → open drawer (or visible no-op)

Drawer contents:
- inference: scheme, premise texts, conclusion text, warrant if present
- edge: src/tgt texts, label, weight, NLI scores if present
- defeater: reasoning, evidence_span, severity, confidence

---

## 11) Testing (Week 1 Minimum)

Use Vitest + jsdom.

- `src/__tests__/spans.test.ts`
  - clamp
  - overlap winner
  - merge adjacency

- `src/__tests__/resolveTarget.test.ts`
  - inference target with overlay ON → selectInference
  - inference target with overlay OFF → openDrawer + fallback

---

## 12) Performance Budgets (Week 1)

Targets:
- 50 ADUs renders without lag on a laptop
- no layout recompute on hover/selection

Memoize mapping + layout. Recompute layout only when analysis/overlay changes.

---

## 13) PR Discipline

- PR1: Boot/API/schema/fixtures
- PR2: SourcePane spans + selection store
- PR3: GraphPane mapping/layout + sync
- PR4: Findings + resolver + drawer
- PR5: Overlay + export + tests

Each PR includes:
- screenshots/gif
- acceptance checklist
- known limitations

---

## 14) Common Pitfalls

- Treating spans as token offsets (they are **char** offsets)
- Nesting `<mark>` for overlaps (breaks DOM/clicks)
- Separate selection state per pane (desync hell)
- Recomputing dagre layout on every render
- Forgetting inference nodes are hyperedges (`premises[]`)
- Skipping schema validation (you will chase ghosts)
