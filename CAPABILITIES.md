# DIANOIA — Capabilities Reference


---


## Summary

DIANOIA is a governed argument compiler. It takes natural language text — an essay, a debate transcript, a policy document — and produces a structured, computable analysis of the argument: what claims are being made, how they connect, whether the logical structure is coherent, where the reasoning breaks down, and what minimal changes would fix it.

The output is not a summary or opinion. It is a formal artifact: a directed claim graph annotated with semantic scores, fallacy flags, coherence violations, scheme labels, CQ-based defeaters, and repair suggestions. Every output field is traceable to a specific text span or a computable graph property.

### What is built and working

The analysis pipeline is production-quality end-to-end. The full stack:

**Extraction (LLM-backed):** A single LLM call extracts ADUs (argument discourse units), classifies them by type and verifiability, and proposes edges. Span alignment is deterministic (exact → casefold → fuzzy → `generated_paraphrase`). An NLI verifier pass drops impossible edges (semantic contradiction on support, entailment on attack) but does not penalize normative or definitional text where neutral NLI is expected. The governance metadata (provenance, align_method, span integrity rate) is always emitted.

**Coherence checking:** Every edge is checked for semantic consistency using a cross-encoder NLI model (generic `cross-encoder/nli-deberta-v3-base` or fine-tuned `deberta-arg-v1`). Two failure types: `SUPPORT_CONTRADICTED` (support edge where texts actively contradict) and `ATTACK_ENTAILED` (attack edge where texts actually agree). Policy: drop only on clear violations (contradiction > 0.5 for support, entailment > 0.5 for attack). Neutral-dominant NLI on a support edge is normal for normative and definitional argument — never flagged.

**Structural fallacies (Tier 1):** Graph-topology detection of circular_reasoning (support-only cycles) and missing_support (claim nodes with no incoming support). No model required. Definitional, normative, interpretive, and forecast-causal nodes are exempt from `missing_support` — they are not expected to produce NLI-testable support chains.

**Structural repair:** Each coherence failure produces a `Witness` with a `repair_hint` (FLIP_LABEL or DELETE_EDGE). `apply_repairs()` applies the hints to produce a repaired graph. Original is not mutated. Full audit trail in `RepairResult.applied` and `RepairResult.skipped`.

**Argumentation semantics (Option B with I-nodes):** When `inference_object_enabled=True`, the engine builds a merged claim+I-node graph. Defeaters attack I-nodes (not claim nodes), so grounded semantics propagates defeat through the inferential chain correctly. Three distinct outputs:
- `grounded_ext`: Dung grounded extension on claim-only graph. Semantically correct but not the primary metric when I-nodes are present — on a connected graph it tends toward "all unattacked claims."
- `warranted_claims`: Claims supported by at least one undefeated I-node chain. This is the meaningful metric for "does the argument support its conclusions." Three-way split in the output: *warranted* (undefeated I-node path exists), *leaf-accepted* (premise or axiom — no support chain required), *unwarranted* (claim/major_claim lacking undefeated I-node support).
- `bipolar_degrees_original/repaired`: Per-node h-categorizer strength ∈ (0, 1].

**Inference nodes + Engine 4 (Walton CQs):** One InferenceNode per edge. Authority I-nodes are synthesized for attributed claims. Undercut edges are structurally rewired to target I-nodes (prefer premise-role I-nodes for undercutters — "that only follows if..." challenges the outgoing inference, not how the claim was derived). The top-K I-nodes are selected by leverage score (direct major_claim path → distance-1 → coherence-implicated → on-path within adaptive hop depth). The LLM classifies the argument scheme and evaluates Walton critical questions. Evidence gate: FAIL requires a verbatim quoted span. Counter-argument filtering: objections already answered in the graph (attacked or undercut by a committed author ADU) are not shown to the CQ evaluator — prevents practical_2 from firing on addressed side effects.

**Engine 5B — Informal fallacy detection:** `LLMFallacyDetector` detects typed defeaters: each has a `target_type` (claim vs inference), `effect` (rebut/undercut/credibility_downweight), `evidence_span`, `severity`, and `confidence`. The `possible_` prefix is permanent until a human edits it. CQ defeaters suppress LLM informal defeaters targeting the same I-node (CQ violations are more specific). Top-1 CQ defeater per (inode, scheme_prefix) pair prevents near-duplicate noise.

**Explain layer:** Three output formats — `narrative()` (plain-English paragraph), `bullet_points()` (structured list), `structured_report()` (JSON-ready dict). When I-nodes are active, the narrative leads with `warranted_claims` and the three-way split. The major claim status is a three-way distinction: *defeated* (not warranted), *contested* (warranted but not grounded — has accepted attackers), *accepted* (both warranted and grounded).

**REST API + CLI:** FastAPI `/analyze`, `/analyze_text`, `/health`. `LLMFallacyDetector` and `LLMSchemeClassifier` auto-wired when `ANTHROPIC_API_KEY` is set. CLI: `build-data` (corpus → JSONL), `eval` (batch analysis on corpus).

### What is not yet built

- Engine 6 (epistemic verification — SUPPORTED/REFUTED/NEI via retrieval)
- Engine 7 (rhetorical strength scoring by audience model)
- Engine 8 full (multi-signal aggregation across epistemic/inferential/rhetorical tracks)
- Engine 9 full (ILP/SMT minimal repair search, Pareto frontier)
- Engine 0 (discourse: coreference, ellipsis, speaker turns)
- Versioned ledger + governance enforcement (anchor_id/fingerprint exist; no hashing, no diff)

See the roadmap at the end of this document.

### The differentiator

DIANOIA produces outputs a chat model cannot: a versioned, auditable argument ledger; computable acceptability and minimal repairs under explicit constraints; CQ-based defeaters with scheme-specific evidence gates; three separated strength tracks (epistemic/inferential/rhetorical). The guarantee is reproducibility and formal traceability — not just "here is my analysis."


---


## What DIANOIA Takes as Input

**Pre-structured input (`CanonDoc`):** If you already have argument structure (e.g. from
a gold corpus or a previous extraction run), you pass a `CanonDoc` directly. This is a
JSON-serializable object containing:
- The full source text
- A list of Argument Discourse Units (ADUs): spans of text that are individual claims or pieces of evidence
- A list of edges: directed relations between ADUs (support, attack, undercut)

**Raw text:** If you have free text, the engine extracts structure automatically.
Two extractors are available:
- **NLI-based extractor:** Sentence segmentation + zero-shot NLI typing + NLI relation detection.
  Works offline, no API key, but limited to sentence-level granularity and support/attack only.
- **LLM-based extractor:** Single Claude Haiku call. Handles sub-sentence ADUs, coreference,
  non-adjacent relations, undercut edges, and informal text. Requires an Anthropic API key.


---


## Core Data Types

### CanonADU — One Argument Unit

Every claim or piece of evidence is a `CanonADU`. It carries:

| Field | What it means |
|---|---|
| `id` | Short identifier within the document (e.g. `a0`) |
| `span` | Character offsets `(start, end)` in the source text |
| `text` | Normalized proposition text (LLM may resolve coreference, strip hedges) |
| `type` | `major_claim`, `claim`, or `premise` |
| `modality` | How certain the speaker is: `certain`, `probable`, `possible`, `contested` |
| `speaker_commitment` | Float 0–1: how strongly the speaker endorses this (1.0 = outright assertion, 0.0 = purely attributed) |
| `attribution` | Speaker name for debates/transcripts, null for essays |
| `provenance` | `extracted` = verbatim source span; `generated_paraphrase` = LLM deviated; `generated_premise` = implicit premise surfaced |
| `anchor_id` | 24-char hex ID stable to the span position (for change tracking) |
| `span_fingerprint` | 16-char hex ID stable to the text content (for integrity checking) |
| `verifiability_type` | `empirical_factual`, `historical`, `normative_value`, `forecast_causal`, `interpretive`, `definitional`, or `unclear` |

### CanonEdge — One Relation

| Field | What it means |
|---|---|
| `id` | Short identifier (e.g. `e0`) |
| `src`, `tgt` | Source and target ADU ids |
| `label` | `support`, `attack`, or `undercut` |
| `attrs` | Extra metadata: NLI scores, extractor name, target_kind |

**support:** The source claim provides evidence or reasons for the target.
**attack:** The source claim opposes or contradicts the target (challenges its truth).
**undercut:** The source challenges the *inferential link* between two claims, not the truth of the target itself. Example: "The study was funded by the manufacturer" undercuts the inference from the study to its conclusion without directly contradicting the conclusion.

### AnalysisResult — Everything the Pipeline Produces

The `AnalysisResult` object returned by `engine.analyze()` contains every intermediate and final output:

```
AnalysisResult
├── doc_id, source_text
├── original_graph          (NetworkX DiGraph with NLI scores on edges)
├── repaired_graph          (after applying structural repairs)
├── sanity_warnings         (structural issues before any scoring)
├── coherence_failures      (list of CoherenceFailure)
├── witnesses               (repair hints, one per coherence failure)
├── fallacies               (list of FallacyResult)
├── unsupported_claims      (list of node ids)
├── repair_result           (audit log: applied + skipped repairs)
├── grounded_ext            (frozenset of claim node ids — Dung on claim-only graph)
├── preferred_exts          (list of frozensets, or None if graph too large)
├── bipolar_degrees_original (dict: node_id → float score)
├── bipolar_degrees_repaired (dict: node_id → float score)
├── extraction_meta         (populated when analyze_text() is used)
├── inference_nodes         (list of InferenceNode, if profile enables it)
├── inference_graph         (NetworkX DiGraph with I-nodes, if profile enables it)
├── defeaters               (list of Defeater — CQ violations + informal fallacy signals)
├── semantics_injected_edges (list of (attacker_id, inode_id, weight) tuples)
└── warranted_claims        (frozenset of claim node ids with undefeated I-node support)
```

`warranted_claims` is the primary acceptability metric when I-nodes are present. It is a
proper subset of all claim/major_claim nodes: those whose inferential support chain contains
at least one undefeated I-node in the grounded extension of the merged claim+I-node graph.
Premises are axioms and are not expected to appear in `warranted_claims` — see the
three-way split in `structured_report`.

---

## Capability 1 — Argument Graph Construction

DIANOIA builds a directed graph from the ADUs and edges in a `CanonDoc`. Every node gets:
- The ADU text, type, modality, provenance, verifiability type
- Graph-computed attributes populated downstream (NLI scores, weights)

Every edge gets:
- The relation label
- NLI scores (entailment, neutral, contradiction) as a cached triple
- A weight derived from the NLI scores: `1 - contradiction` for support edges (compatibility score), `contradiction` for attack edges; undercut edges are unweighted

This graph is the foundation for everything else. Nothing is re-computed from raw text after this point — all subsequent steps read from the graph.

---

## Capability 2 — Semantic Coherence Checking

For every support and attack edge, DIANOIA checks whether the edge label is semantically consistent with what the texts actually say.

It does this using NLI (Natural Language Inference): given the source ADU text as a "premise" and the target ADU text as a "hypothesis", it scores the probability of three relationships: entailment (the source implies the target), neutral, and contradiction.

**SUPPORT_CONTRADICTED:** A support edge where the source text actually contradicts the target. Example: labeling "Clinical trials show vaccines increase mortality" as *supporting* "Vaccines protect against disease." The texts contradict — the support label is wrong.

**ATTACK_ENTAILED:** An attack edge where the source text actually entails (logically implies) the target. Example: labeling "Vaccines are highly effective at preventing transmission" as *attacking* "Vaccines protect against disease." The texts are logically consistent — the attack label is wrong.

For each failure, DIANOIA reports:
- Which edge failed (edge_id, src_id, tgt_id, edge_label)
- Why it failed (failure_code)
- How bad it is (severity: the winning NLI score, 0–1)
- The full NLI scores (entailment, neutral, contradiction)

NLI scores are computed once per edge and cached on the graph. Every downstream step (coherence, unsupported claims, weights) reads from this cache — the model is called at most once per edge per document.

Undercut edges are skipped by coherence checking. They describe inferential challenges, not semantic compatibility between claim texts, so NLI is not the right tool.

---

## Capability 3 — Structural Fallacy Detection (Tier 1)

DIANOIA detects two structural fallacies from graph topology alone, with no model needed:

**circular_reasoning:** A cycle where every edge is a support edge. Example: A supports B, B supports C, C supports A. The conclusion is its own support — the argument is circular. Every node in the cycle is flagged. Cycles involving attack edges are dialectical exchanges, not circular reasoning.

**missing_support:** A `claim` or `major_claim` node that has no incoming support from any premise. The node is asserting a position without backing it up. `premise` nodes are excluded — they are expected to be unsupported atomic facts. Additionally, nodes with `verifiability_type` in `{definitional, normative_value, interpretive, forecast_causal}` are exempt — these claim types do not produce NLI-testable support chains, so flagging them for "missing support" is a false positive. This mirrors the skip logic in `find_unsupported_claims`.

Both detections have `score=1.0` (deterministic) and `tier=1` (graph-based).

**Optional Tier 2 (model-based):** The engine accepts an optional `fallacy_model` parameter. When provided, it classifies every ADU text against 13 informal fallacy categories (ad hominem, appeal to authority, false causality, slippery slope, etc.) using the `minjingbo/logic` HuggingFace classifier. This is Tier 2 — probabilistic, not deterministic. It requires the `transformers` package and downloads the model on first use.

---

## Capability 4 — Argumentation Semantics

DIANOIA implements Dung's abstract argumentation framework on the argument graph.

**Grounded extension:** The most skeptical acceptable set of arguments. Starts from arguments that are not attacked, then inductively adds anything defended by that set. Arguments in the grounded extension are "uncontroversially acceptable" — they survive under the most conservative interpretation.

When I-nodes are present (inference_object_enabled=True), the grounded extension is computed on the full merged claim+I-node graph, then I-nodes and synthetic defeater nodes are stripped from the result. The `grounded_ext` field contains only claim node IDs. Note: on connected claim graphs, the grounded extension tends toward "all unattacked claims" because claim-to-claim attack edges without I-node propagation are rare. Use `warranted_claims` as the primary acceptability metric when I-nodes are present.

**Preferred extensions:** All maximally consistent sets of arguments you could coherently accept together. Each preferred extension is one "coherent viewpoint" on the argument. A single argument can appear in some preferred extensions and not others — this captures genuine dialectical disagreement. Returns `None` if the total merged graph (claims + I-nodes + synthetic defeater nodes) has more than 20 nodes (to avoid exponential runtime). The threshold is checked on the actual merged graph size, not just the claim count.

Both are computed on the original graph (before repairs). They reflect the original argument's logical structure, not a cleaned-up version.

---

## Capability 5 — Bipolar Degree Scoring

Every node in the argument graph gets a scalar "strength" score in the range (0, 1]:
- Near 1.0: unattacked or net strongly supported
- Lower: under pressure from attack edges

The score uses the h-categorizer (Besnard & Hunter 2001) extended to treat support edges:

    strength(a) = (1 + Σ w(s→a) * strength(s)) / (1 + Σ w(s→a) * strength(s) + Σ w(b→a) * strength(b))

When I-nodes are present, the score is computed on the merged claim+I-node graph; I-node and synthetic node scores are stripped from the output so `bipolar_degrees_original` maps only claim node IDs.

Scores are computed on both the original graph and the repaired graph, so you can show "before and after repair" comparisons.

---

## Capability 6 — Structural Repairs

When DIANOIA finds a coherence failure, it suggests a repair.

**Witness classification:** Each `CoherenceFailure` becomes a `Witness` with a `repair_hint`:
- `flip_label`: Change the edge label (e.g. flip support → attack, because the source actually contradicts the target)
- `delete_edge`: Remove the edge entirely (the relationship is semantically incoherent in both directions)

**Repair application:** `apply_repairs()` produces a new graph with the suggested changes applied. The original graph is not modified — you always have both.

**Audit trail:** `RepairResult` contains the full list of applied and skipped repairs, so the system's actions are explainable.

---

## Capability 7 — Unsupported Claim Detection

A node is an "unsupported claim" if it has incoming support edges but none of them has a `weight ≥ threshold` (default 0.3). Weight is `1 - contradiction` — semantic compatibility, not entailment.

Why `1 - contradiction` and not entailment: argumentative support regularly produces neutral NLI (entailment near 0) for normative, definitional, and analogy-based arguments. A weight of 0.9 with `entailment=0.2, neutral=0.72, contradiction=0.08` is a valid well-supported edge — flagging it as "unsupported" because entailment is low would be wrong. The weight (compatibility) score only catches cases where the incoming support edge actively contradicts the conclusion.

The VerifiabilityGate type is also respected: `definitional`, `normative_value`, `interpretive`, and `forecast_causal` nodes are skipped — they are not expected to produce NLI entailment from their premises.

This is distinct from the `missing_support` structural fallacy:
- `missing_support` (Tier 1): node has no incoming support edges at all (pure structural)
- `unsupported_claims`: node has incoming support, but every support edge's source actively contradicts the conclusion

---

## Capability 8 — Verifiability Classification

Before any fact-checking or epistemic verification can run, DIANOIA classifies what kind of claim each ADU is:

| Type | What it means | Verifiable? |
|---|---|---|
| `empirical_factual` | Observable fact, statistic, measurement | Yes |
| `historical` | Past event or record | Yes, with sources |
| `normative_value` | Value judgment, ethical claim ("we should X") | No — evaluate differently |
| `forecast_causal` | Prediction or causal claim about the future | Partly — needs uncertainty |
| `interpretive` | Interpretation of meaning or intent | Often not cleanly verifiable |
| `definitional` | Defines or classifies a concept ("X is defined as Y", "meets the definition of") | No — evaluate via Engine 4, not fact-checking |
| `other` | Cannot be classified | Skip |

This runs as part of the main pipeline (step 2b) when a `VerifiabilityGate` is passed to the engine. The tag lives on the graph node as `verifiability_type`.

Classification uses a rule-based regex pass first (fast, no model). If the rule-based pass returns `unclear`, it falls back to NLI zero-shot classification (same NLI model already loaded).

---

## Capability 9 — Span Identity and Integrity

Every ADU gets two IDs:

**anchor_id** (`sha256(doc_id:span_start:span_end)[:24]`): Identifies a location in a document. Stable as long as the span doesn't move. If you re-run analysis and the same text span is found, the anchor_id will match — allowing change tracking across runs.

**span_fingerprint** (`sha256(normalize(text))[:16]`): Identifies content. Stable as long as the text doesn't change (normalized for whitespace and case). If the anchor_id is the same but the fingerprint changed, DIANOIA flags a governance violation: the span moved to cover different text without re-anchoring.

`check_anchor_integrity(anchor_id, span_fingerprint, doc_id, span, text)` returns a list of violation strings, which you can use to validate stored analysis results against updated documents.

---

## Capability 10 — Discourse Connective Pre-labeling

A rule-based, zero-cost labeler that recognizes discourse markers in text pairs and assigns a prior over relation type:

- **support markers:** "because", "since", "therefore", "as a result", "this shows that", etc.
- **attack markers:** "however", "but", "yet", "despite", "on the contrary", etc.
- **undercut markers:** "even if", "granted that", "one might object that", etc.
- **concession markers:** "although", "while it is true that", etc.

Returns `ConnectivePrior(label, score)` for any (src_text, tgt_text) pair. Used as a soft prior signal feeding into `score_candidates()`: the connective label is passed through to RelationScore as informational metadata. The relation classifier (or NLI model) always has final say. This labeler is intentionally kept simple — a trained discourse relation classifier is not planned, as improved LLM extraction prompting and better candidate pruning deliver more value per unit effort.

---

## Capability 11 — Inference Nodes + Argument Scheme Detection + Walton CQ Evaluation (Engine 4)

When `CompilationProfile(inference_object_enabled=True)` is set, DIANOIA generates one `InferenceNode` per support and attack edge. An `InferenceNode` represents the inferential step between a premise and a conclusion — separate from the claim nodes themselves.

This follows the AIF (Argument Interchange Format) model, where undercutters attack inference nodes, not claim nodes. Example: "The study was pharma-funded" undercuts the inference from the study to the conclusion — it doesn't directly contradict the conclusion.

The inference graph rewires `src → tgt` support/attack edges to `src → I-node → tgt`. Undercut edges are structurally rewired to target I-nodes. Rewiring preference order for undercutters:
1. Authority I-nodes (when the target node has an attribution) — preferred unconditionally
2. Premise-role I-nodes (I-nodes where the target ADU is a PREMISE, i.e. the outgoing inference) — preferred for propositional undercutters ("that only follows if X" challenges how the claim is used, not how it was derived)
3. Conclusion-role I-nodes (fallback)

**Authority I-node synthesis:** When an ADU carries an `attribution` field (i.e., the claim is attributed to a named speaker — "Dr. Kim says X"), an additional authority I-node is synthesized with `scheme="authority"`, `premises=()`, `conclusion=<adu_id>`, and `attribution` in `scheme_params`. This gives COI and appeal-to-authority undercut edges a structurally correct target: they target the testimony inference (Kim's assertion), not the propositional content of the claim.

**Scheme labeling + Walton CQ evaluation (Engine 4 full):**

When `CompilationProfile(scheme_engine_enabled=True)` is set and a `SchemeClassifier` is provided, DIANOIA:

1. Selects the top-K most important I-nodes (default K=8) using a leverage score:

   | Score tier | Condition |
   |---|---|
   | +100 | Conclusion IS a major_claim node |
   | +80 | Conclusion is distance-1 from major_claim (direct supporter/attacker) |
   | +60 | Premise or conclusion implicated by a coherence failure |
   | +40 | On argumentative path within adaptive hop budget |
   | +10 (up to) | High out-degree of conclusion node (downstream mass) |
   | +10 (up to) | Low edge confidence (instability bonus — more uncertain = more worth evaluating) |

   Hop budget adapts to graph size: 4 hops for ≤20 ADUs, 3 for ≤50, 2 for larger. This prevents "connected graph ⇒ everything qualifies" defeating the top_k limit.

2. Calls the `SchemeClassifier` to label the argument scheme and evaluate Walton critical questions for each selected I-node. A combined LLM call handles both scheme detection and CQ evaluation.

3. Emits `Defeater` objects for CQ failures — only when a verbatim evidence span from the text is present (no evidence, no defeater).

4. Counter-argument filtering: before passing objections to the CQ evaluator, answered objections are removed. An objection ADU is "answered" if any edge `(x, objection_id)` exists in the claim graph with `label ∈ {attack, undercut}` and the source has `speaker_commitment ≥ 0.5`. This prevents `practical_2` ("unacceptable side effects?") from firing when the argument already rebuts the stated side effect.

Supported schemes and their critical questions:

| Scheme | Key CQs |
|---|---|
| `authority` | Is the source expert? Biased or conflicted? Do other experts agree? |
| `causal` | Could a confounding factor explain both? Correct causal direction? Chain complete? |
| `analogy` | Cases sufficiently similar? Disanalogies that undermine the comparison? |
| `statistical` | Sample representative? Sample size sufficient? Confounders? |
| `abductive` | Alternatives ruled out? Is this actually the simplest explanation? |
| `practical` | Action achieves the goal? Unacceptable side effects? Better alternatives? |
| `definitional` | Is the criterion for the key predicate explicitly stated? Does the premise satisfy it? Is the criterion contested? |

CQ defeaters have `target_type="inference"`, `target_id=<inode_id>`, `effect="undercut"`, `label="cq_violation_<cq_id>"`.

**Evidence gate:** The scheme classifier must quote a verbatim span from the argument text to emit a FAIL state. Without a quoted span, the CQ state is demoted to UNKNOWN and no defeater is emitted. This prevents hallucinated CQ violations.

**Scheme confidence gate:** If the scheme classifier's confidence is below 0.55, no CQ defeaters are emitted for that I-node.

**CQ deduplication:** Top-1 CQ defeater per (inode_id, scheme_prefix) pair is kept. For example, `definitional_1` and `definitional_3` both saying "vague predicate" on the same I-node — only the highest-confidence one is emitted.

`LLMSchemeClassifier` (in `graph/schemes.py`) is the current implementation — temporary; replace with a trained cross-encoder when ArgMicro + AIFdb data is available.

---

## Capability 12 — Informal Fallacy Detection as Typed Defeaters (Engine 5B)

DIANOIA detects informal fallacies and represents them as structured `Defeater` objects — not plain text labels. Every defeater is typed by what it attacks (claim vs inference) and what logical effect it has (rebut vs undercut).

**Defeater fields:**

| Field | What it means |
|---|---|
| `label` | Fallacy type, always prefixed `possible_` (e.g. `possible_ad_hominem`) |
| `target_type` | `claim` = rebuttal (challenges truth) / `inference` = undercutter (challenges warrant) / `dialogue` = dialogue-level |
| `target_id` | ID of the attacked `CanonADU` or `InferenceNode` |
| `effect` | `rebut`, `undercut`, `credibility_downweight`, `rhetorical_penalty` |
| `evidence_span` | Text span grounding the detection — mandatory, no span = no defeater |
| `reasoning` | Human-readable explanation of why this applies |
| `severity` | Estimated strength [0, 1] |
| `confidence` | Detection confidence [0, 1] |
| `context_edge_id` | Optional edge for UX linkage only |

**Target typing is the key distinction:**

- `ad_hominem`, `appeal_to_emotion`, `hasty_generalization` → `target_type="claim"`, `effect="rebut"` — they challenge the truth of a claim node
- `conflict_of_interest`, `appeal_to_authority`, `false_dilemma`, `slippery_slope`, `false_causality` → `target_type="inference"`, `effect="undercut"` — they challenge the inferential step between premises and conclusion, not the conclusion itself. Example: "The study was pharma-funded" is not a direct contradiction of the study's conclusion — it undercuts the inference from the study to its conclusion.

**Deduplication:** Informal defeaters targeting an I-node that already has a CQ violation defeater are suppressed (CQ violations are more specific and formally grounded). This prevents noisy LLM duplicates.

**LLM-based detection:** Enabled by passing `fallacy_detector=LLMFallacyDetector(client)` to `ArgumentEngine`. Uses a structured LLM prompt with JSON output. Auto-wired by `create_app()` if `ANTHROPIC_API_KEY` is set. `LLMFallacyDetector` is a temporary implementation — replace with a trained DeBERTa classifier (MAFALDA + adjudicated data) when ready; it will satisfy the same `FallacyDetector` protocol.

**Policy:** The `possible_` prefix is never auto-removed. Upgrading to a hard label requires `provenance: user_edit` from an explicit human action.

Results live in `AnalysisResult.defeaters`.

---

## Capability 13 — Major Claim Identification

For text where the extraction layer does not explicitly mark `major_claim` nodes, DIANOIA infers top conclusions from graph structure using a centrality score:

**Score formula:** `in_support - out_support + 0.5 × in_attack`

Nodes that receive many support edges and are also attacked are likely to be central conclusions. A node that primarily emits support (acts as premise for others) scores low.

`identify_major_claims(g)` returns node IDs sorted by score.
`tag_major_claims(g)` promotes qualifying `claim`-typed nodes to `major_claim` in-place.

This requires no model call — it is pure graph topology.

---

## Capability 14 — LLM-Based Extraction (Production-Grade)

When an Anthropic API key is available, `LLMExtractionLayer` extracts argument structure from raw text in a single LLM call.

**What the LLM produces:**
- ADU segmentation (sub-sentence granularity, not sentence-by-sentence)
- ADU typing: major_claim / claim / premise
- Modality: certain / probable / possible / contested
- Speaker commitment: float 0–1
- Attribution: speaker name or null
- Edges: support / attack / undercut, with target_kind (claim / inference)
- Evidence quote: exact substring copied from source text

**Span alignment (deterministic):** The LLM's evidence quote is matched back to the source text in three stages:
1. Exact match → `provenance="extracted"`, `align_method="exact"`
2. Case-insensitive match → `provenance="extracted"`, `align_method="casefold"`
3. difflib fuzzy match with ratio ≥ 0.7 → `provenance="generated_paraphrase"`, `align_method="difflib"`
4. Below threshold → `(0, 0)` span, `provenance="generated_paraphrase"`, `align_method="none"`

**Invalid edge labels** (anything other than support/attack/undercut) are dropped, never coerced to support. `n_edges_invalid_label` appears in the meta dict so you can detect model drift.

**Extraction prompt enforces:**
- Conditional defeaters preserve framing: "that only follows if X" must stay as the conditional proposition, not be stripped to "X"
- Rebuttal chains preserve structure: C → B → objection, not C → objection and B → objection independently
- Reinforcement-of-defeater: if C explains why undercutter A is true, emit C → A (support), not C → original_target (attack)

**NLI verifier pass (optional):** After extraction, if an NLI model is provided, the verifier checks:

1. **ADU verifier:** Drops ADUs where the LLM's claim_text clearly contradicts its evidence_quote (contradiction ≥ 0.5). Catches hallucinations without touching normalizations.

2. **Edge verifier:** Drop policy:
   - **Support:** drop if `contradiction ≥ 0.5`
   - **Attack:** drop if `entailment ≥ 0.5`
   - **Undercut:** always pass through

**Governance metadata in `meta`:** Every extraction call returns:
```json
{
  "n_adus": 4,
  "n_edges_extracted": 3,
  "n_edges_invalid_label": 0,
  "align_methods": {"exact": 3, "casefold": 1},
  "align_score_min": 1.0,
  "align_score_mean": 1.0,
  "extractor": "llm",
  "llm_model": "claude-haiku-4-5-20251001"
}
```

**Span integrity rate** = fraction of ADUs with `align_method` in `{exact, casefold}`. If this drops significantly across a batch, it means the LLM is paraphrasing instead of quoting — an audit signal.

---

## Capability 15 — Serialization (Full Round-Trip)

All `CanonDoc` objects (including all v1 ADU fields) can be serialized to and deserialized from JSONL. Every field survives the round-trip:
- `anchor_id`, `span_fingerprint`, `modality`, `speaker_commitment`, `attribution`, `provenance`, `dialogue_move`, `verifiability_type`, `propositional_core`

Legacy JSONL files (missing fields) load safely with defaults. The serialization layer validates every document on load and on write.

---

## Capability 16 — Explanation Layer

Three output formats for making analysis results human-readable:

**narrative:** A paragraph-form explanation of the argument's structure, coherence issues, and fallacies. Reads like a written critique. When I-nodes are present:
- Major claim status is a three-way distinction: *defeated* (not in warranted_claims), *contested* (in warranted_claims but not in grounded_ext — has accepted attackers), *accepted* (both warranted and grounded)
- Grounded extension is replaced by the warranted/leaf-accepted/unwarranted three-way split (Dung on claims alone is misleading on connected graphs when I-nodes are present)

**bullet_points:** A structured list of issues, grouped by type (structural warnings → coherence failures → repair hints → fallacies → unsupported claims → defeaters).

**structured_report:** A full structured JSON/dict report. Machine-readable. Contains all the same information as the other formats plus raw scores. When I-nodes are present, includes a `warranted_claims` dict with three keys:
- `warranted`: claim/major_claim nodes with undefeated I-node support chains
- `leaf_accepted`: premise nodes or nodes with no incoming edges (axiomatic; no support chain required)
- `unwarranted`: claim/major_claim nodes lacking undefeated I-node support

This is what you use to build UIs.

---

## Capability 17 — REST API

A FastAPI layer wraps the engine. Endpoints:

`POST /analyze` — accepts a `CanonDoc` JSON object, returns a serialized `AnalysisResult`
`POST /analyze_text` — accepts raw text, runs extraction then analysis, returns `AnalysisResult`
`GET /health` — liveness check

The API supports the same model injection as the Python API (pass `nli_model`, `fallacy_model`, `verifiability_gate`, `profile` at startup).

---

## Capability 18 — CLI

`build-data [corpus]` — converts raw corpora (AAE2, CDCP, ArgMicro, ARIES) to normalized JSONL.

`eval [corpus] [--split train|test|dev]` — runs the pipeline on a JSONL corpus and reports coherence failure rates, fallacy detection counts, and bipolar degree distributions.

---

## What Is Not Yet Built

### Components at stub / placeholder status

| Component | Status | Blocker |
|---|---|---|
| 4-class relation classifier | Protocol + training script + 16k training examples ready; **no trained checkpoint**. NLIRelationAdapter fallback active. Deprioritized — LLM extraction labels relations; classifier is on-prem fallback only. | Run `python scripts/train_relation.py train` when needed |
| Relation classifier wired into extraction | FourClassRelationClassifier exists but analyze_text still uses NLI thresholding | Train model first, then wire into ExtractionLayer |
| Engine 4 scheme classifier (trained) | LLMSchemeClassifier works end-to-end. Trained cross-encoder replacement: not yet built. | Get ArgMicro + AIFdb data; train DeBERTa 6-class |
| Engine 5B trained classifier | LLMFallacyDetector works and is wired. Trained replacement: not yet built. | Get MAFALDA + Argotario; train DeBERTa 15-class on adjudicated output |
| VerifiabilityGate trained classifier | NLI zero-shot is real but not task-specific; acceptable for now | Get ClaimBuster / CLEF data; train DeBERTa |
| propositional_core population | CanonADU.propositional_core field defined; nothing populates it systematically | Engine 1 propositionization |
| CompilationProfile serialization | Profile dataclass defined but has no JSON serialization or registry | Implement profile_id computation + serialization |

### Engines not yet started

| Engine | Description | Priority |
|---|---|---|
| Engine 0 — Input & Discourse | Speaker segmentation, coreference, ellipsis recovery, dialogue move tagging, hierarchical topic blocks | v2 |
| Engine 1 — Propositionization (full) | Modality extraction (no stripping), conjunct splitting, propositional_core normalization, implicit premises | v1 partial → v1.5 |
| Engine 6 — Epistemic Verification Stack | Normalize → retrieve (BM25 + dense) → rerank → minimal evidence selection → SUPPORTED/REFUTED/NEI with provenance. Gate already done (verifiability_type); stack not started | v2 |
| Engine 7 — Rhetorical Strength | Per-node scoring across clarity/specificity/relevance/sufficiency/completeness/civility/dialectical conduct/verbosity; parameterized by audience_model + rhetorical_objective | v1.5 |
| Engine 8 (full) — Multi-signal Aggregation | Dung semantics + bipolar degree done. Multi-signal aggregation across epistemic/inferential/rhetorical tracks, calibrated confidence, abstention policy: not done | v2 |
| Engine 9 (full) — Minimal Repair Search | flip_label + delete_edge + greedy apply done. Minimal repair search (ILP/SMT), Pareto frontier, modality weakening action, multi-objective cost vectors: not done | v2 |

### Infrastructure not yet started

| Component | Description |
|---|---|
| Versioned ledger + governance enforcement | anchor_id + span_fingerprint fields exist; no ledger hashing, no governance contract enforcement, no cross-run diff |
| Benchmark CI/CD gating | CLI eval command exists; no commit-gated F1/precision regression gates |
| Calibration contracts | No formal precision/recall contracts per model per fallacy type |
| Discourse relation classifier | DiscourseConnectiveLabeler is rule-based (kept as debug signal only); training a discourse relation model is not planned — LLM extraction + candidate pruning deliver more value |
| Private corpus connectors | Engine 6 product track: Drive / SharePoint / Confluence / internal PDFs |

---

## Development Roadmap to v2

Two extraction tiers (settled architecture):
- **SaaS / interactive default**: LLM extraction + deterministic governance + NLI verifier (impossibility detector) + I-nodes
- **On-prem / batch fallback**: trained 4-class relation classifier (FourClassRelationClassifier)

The relation classifier does not "fix edge induction for all text" — AAE2/CDCP corpora are stylistically narrow. Its role is cheap local inference when LLM is unavailable.

---

### Pre-v1 Step 0: Governance fixes

**0a. Provenance integrity — DONE**

`_align_span` already maps: `align_method in {exact, casefold}` → `provenance="extracted"`, `align_method in {difflib, none}` → `provenance="generated_paraphrase"`. The `_parse_response` method adds a secondary guard based on similarity between `claim_text` and `evidence_quote`. The behavior is correct.

**0b. Undercut structural rewiring — DONE**

Undercut edges are structurally rewired in the inference graph to target I-nodes. Rewiring preference: authority I-nodes (pre-set scheme, unconditional) > premise-role I-nodes (undercutter challenges the outgoing inference, i.e. "that only follows if X") > conclusion-role I-nodes (fallback). Both `conclusion_to_inodes` and `premise_to_inodes` maps are built in `graph/inference.py`.

**0c. NLI verifier policy — DONE**

Policy is enforced in code:
- Support edge: drop only if `contradiction > 0.5`
- Attack edge: drop only if `entailment > 0.5`
- Neutral-dominant NLI on support = valid; not touched

`find_unsupported_claims` correctly uses `weight = 1 - contradiction` (not entailment). Documented in Cap 7.

---

### Pre-v1: Structural foundation

**1. Train the 4-class relation classifier** *(low priority — on-prem fallback only)*
- LLM extraction is the SaaS default and handles edge induction. Classifier = cheap local fallback when LLM unavailable.
- Data ready: `data/relation_train.jsonl` (~16k examples: 4775 support, 464 attack, 0 undercut, 10763 none)
- Run when needed: `python scripts/train_relation.py train` (~2–4 hours on M2 Max MPS)
- Undercut class coverage currently 0% — source ARIES if needed

**2. Wire the trained classifier into the extraction pipeline** *(depends on 1)*
- Add `relation_backend: "classifier"` path in CompilationProfile to ExtractionLayer
- Route through `score_candidates()` with ConnectivePrior signals (already implemented)

**3. CompilationProfile serialization + profile registry** *(code change)*
- Serialize profile → canonical JSON → `sha256[:16]` → profile_id
- Output always `(profile_id, ledger)` — never the ledger alone
- Stakeholder disputes become: profile bug vs legitimate profile disagreement

**4. Benchmark CI/CD gating** *(code + infra)*
- CLI eval command exists; add commit-gate wrappers
- Gates: relation F1 on AAE2 test split, coherence precision vs gold
- Dataset governance: dedup all new training data against eval splits before use

---

### v1: Structural foundation

**5. Engine 1 — Propositionization (LLM-backed initially)** *(code + LLM prompt)*
- CanonADU.propositional_core exists but nothing populates it systematically
- LLMExtractionLayer partially does this; extend to cover: conjunct splitting, modality extraction (no stripping — modality="probable", propositional_core loses the hedge word), implicit premises only when manifestly needed with `provenance="generated_premise"`
- LLM-backed acceptable here; target = trained model (needs scheme-labeled data)
- After this: NLI coherence operates on propositional_core, not raw text

**6. Anchor ID enforcement on load/write** *(code change)*
- `compute_anchor_id`, `compute_span_fingerprint`, `check_anchor_integrity` exist in `graph/identity.py`
- Wire into JSONL load/write: call check_anchor_integrity on every load, log violations
- Auto-compute anchor_id + span_fingerprint when saving docs that lack them
- Cross-run: anchor_id stable + span_fingerprint changed → governance violation

**7. Component segmentation as first-class output** *(code change)*
- Disconnected graph components are currently surfaced only as sanity warnings
- Promote to first-class: `AnalysisResult.argument_threads: List[FrozenSet[str]]` — one set of node IDs per weakly connected component
- Label components "Argument A / B / C" with a deterministic ordering (by major_claim centrality)
- This changes how the frontend, the repair engine, and Engine 4 select what to process

---

### v1.5: Inference & fallacy engines

**8. Engine 4 — Argument Scheme Detection + Walton CQs** *(done — LLM-backed)*
- `LLMSchemeClassifier` in `graph/schemes.py`. One combined LLM call per selected I-node.
- Top-K=8 selection by leverage score. Counter-argument filtering (answered objections removed before CQ evaluation).
- Evidence gate: FAIL requires verbatim quoted span. Scheme confidence gate: < 0.55 → no defeaters.
- CQ deduplication: top-1 per (inode, scheme_prefix) pair.
- TODO (you act): Fine-tune scheme classifier on ArgMicro + AIFdb. Replace `LLMSchemeClassifier` with trained cross-encoder; CQ evaluation stays LLM-backed (requires world knowledge NLI can't provide).

**9. Engine 5B — Informal fallacy detection, LLM-first** *(already done; no new training yet)*
- LLMFallacyDetector already works; this is the correct posture for v1.5
- Train a classifier LATER on your own adjudicated corrections — raw MAFALDA/Logic datasets are noisy; your adjudicated data will be higher quality signal
- Deployment gate remains: precision > 0.85 per fallacy type before replacing LLM impl
- When ready to train: build `scripts/train_fallacy.py` (mirrors train_relation.py pattern)

**10. Verifiability Classifier Training** *(you act — when NLI zero-shot is insufficient)*
- NLI zero-shot VerifiabilityGate is real and acceptable; this is an upgrade, not a bug fix
- When ready: ClaimBuster (23k, binary CFS/NFS) + CLEF CheckThat! (30k+) + CDCP proposition types
- Build `scripts/train_verifiability.py`

**11. Engine 7 — Rhetorical Strength (LLM-backed)** *(code + LLM prompt)*
- Nothing built; start with LLM-as-judge, schema-constrained
- Dimensions: clarity, specificity, relevance, sufficiency, completeness, civility, dialectical conduct, verbosity
- Output per node: `rhetorical_scores: {clarity: 0.8, specificity: 0.4, ...}`
- Shipped with parameterizable presets, ONE default (never ship without a default):
  - Default: "Reasonable critical reader" (informed lay audience, evaluating fairly)
  - Presets: Academic peer reviewer / Policy audience (risk/implementation bias) / Debate judge (dialectical conduct emphasis)
  - Store `audience_model` + `rhetorical_objective` in CompilationProfile
- Anti-leak regression gate: swap style, hold propositional content fixed → epistemic/inferential must not move (Δ ≤ 0.05). Failure = broken track separation.
- Training target (later): IBM argument quality ranking + UKP convincingness corpora

**12. Modality weakening as repair action** *(code change)*
- Add `weaken_modality(g, adu_id, target_modality) → new_graph` to repair/ops.py
- Repair cost is a multi-objective vector: `(semantic_drift, content_loss, rhetorical_damage, epistemic_plausibility)` — never collapsed to a scalar
- "certain → probable" costs less semantic_drift than deleting the support edge
- Provenance: every accepted repair logs `provenance="user_edit"`; generated_premise never upgrades silently

**13. Engine 0-lite — Speaker & Dialogue (for transcripts)** *(code + LLM prompt)*
- v1.5-lite scope (full Engine 0 is v2):
  - Speaker segmentation when explicit ("A: … B: …")
  - Dialogue move tagging per ADU (challenging | supporting | questioning | conceding | redirecting)
  - Shallow cross-turn target linking ("this claim" → most recent major_claim in current thread)
- This gives credible debate and interview support without the full discourse research overhead

---

### v2: Epistemic verification + auditability

**14. Engine 6 — Epistemic Verification Stack** *(code + architecture)*
- Gate already routes `empirical_factual` and `historical` ADUs here
- Pluggable CorpusAdapter architecture from day one (reproducibility requires it — retrieval_snapshot_id already in CompilationProfile):
  - `CorpusAdapter` protocol → `WikipediaAdapter` first → private connectors later
  - Index: BM25 + FAISS dense, local. Wikipedia snapshot pinned by version.
- Stack:
  1. Normalize: propositional_core → canonical claim form (Engine 1 must be done)
  2. Retrieve: BM25 + dense dual-encoder → top-K passages
  3. Rerank: cross-encoder reranker → top-M evidence passages
  4. Minimal evidence selection → verdict: SUPPORTED | REFUTED | NEI
  5. Provenance: doc IDs, passage spans, confidence, explanation
  6. Source quality: domain reputation, genre, COI flags from Engine 5B, corroboration count
  7. Abstention: NEI | ambiguous | depends on definition | mixed evidence | below threshold
- Mandatory pre-deployment gate: adversarial retrieval stress tests
- New fields on nodes: `verification_status`, `evidence_citations`, `verification_confidence`

**15. Engine 8 complete — Multi-signal Aggregation** *(code change)*
- Dung semantics + bipolar degree: done
- Need: multi-signal aggregation across three fully separated tracks
  - Epistemic: `verification_status` + `evidence_citations` (empirical/historical ADUs only)
  - Inferential: `bipolar_degree` + CQ Defeaters (Engine 4) + informal Defeaters (Engine 5B)
  - Rhetorical: `rhetorical_scores` (Engine 7)
- Hard rule: never collapse tracks to a single scalar. Calibrated confidence per track. Abstention per track.

**16. Ledger diff + versioned ledger + governance enforcement** *(code change)*
- anchor_id + span_fingerprint exist; need ledger hashing:
  - `sha256(profile_id ‖ doc_id ‖ sorted(model_versions) ‖ sorted(adu_ids) ‖ sorted(edge_ids))`
  - Output: `LedgerArtifact(profile_id, ledger_id, analysis_result, governance_log)`
- Governance pre-emission: all informal fallacy labels carry `possible_`; generated_premise never upgrades; tracks never collapsed
- `diff_ledgers(ledger_a, ledger_b)` → what changed between runs, why — this is the killer feature; it must ship before v2 branding
- Immutable audit trail: every operation logged with provenance, timestamp, model version

**17. Engine 9 complete — Minimal Repair Search (Pareto Frontier)** *(code + ILP/SMT)*
- apply_repairs greedy is done
- ILP/SMT: find minimum-cost set of repairs restoring coherence
- Multi-objective Pareto frontier over (semantic_drift, content_loss, rhetorical_damage, epistemic_plausibility)
- User budget parameter: max_semantic_drift → prune frontier
- Modality weakening (Step 12) is also a repair candidate

**18. Engine 0 full — Input & Discourse** *(code + LLM + eval harnesses)*
- Coreference resolution, ellipsis recovery, stance resolution
- Hierarchical structure: turn → topic block → ADU (mandatory for O(n²) avoidance on long transcripts)
- Cross-block retrieval: bi-encoder for long-range relations
- Eval harnesses: DialogRE + RST discourse resources

**19. Calibration contracts** *(code + protocol)*
- Formal precision/recall contract per component
- Deployment gate: checkpoint fails contract → cannot replace incumbent
- Contract IDs stored in CompilationProfile.calibration_contract_id

---

### Summary table

| # | Step | Milestone | Who acts |
|---|---|---|---|
| 0a | Provenance integrity enforcement | **done** | — |
| 0b | Undercut structural rewiring | **done** | — |
| 0c | NLI verifier policy clarification | **done** | — |
| 1 | Train 4-class relation classifier (on-prem fallback) | Low priority | You: run `train_relation.py train` when needed |
| 2 | Wire classifier into extraction pipeline | Depends on 1 | Code change |
| 3 | CompilationProfile serialization | Pre-v1 | Code change |
| 4 | Benchmark CI/CD gating | Pre-v1 | Code + infra |
| 5 | Engine 1 propositionization (LLM-backed) | v1 | Code + LLM prompt |
| 6 | Anchor ID enforcement on load/write | v1 | Code change |
| 7 | Component segmentation as first-class output | v1 | Code change |
| 8 | Engine 4 scheme detection + Walton CQs (LLM-backed) | **done** | — |
| 8d | Engine 4 trained scheme classifier (replaces LLM) | later | You: get ArgMicro + AIFdb |
| 9 | Engine 5B: LLM-first (already done); train on adjudicated data later | v1.5 | No action yet |
| 10 | Train verifiability classifier (when needed) | v1.5 | You: get ClaimBuster/CLEF |
| 11 | Engine 7 rhetorical strength (LLM-backed, presets) | v1.5 | Code + LLM prompt |
| 12 | Modality weakening as repair action | v1.5 | Code change |
| 13 | Engine 0-lite (speaker + dialogue_move) | v1.5 | Code + LLM prompt |
| 14 | Engine 6 epistemic stack (CorpusAdapter, Wikipedia first) | v2 | Code + Engine 1 |
| 15 | Engine 8 multi-signal aggregation | v2 | Code change |
| 16 | Ledger diff + versioned ledger + governance enforcement | v2 | Code change |
| 17 | Engine 9 minimal repair search (Pareto) | v2 | Code + ILP/SMT |
| 18 | Engine 0 full (coreference, ellipsis, hierarchical) | v2 | Code + LLM + evals |
| 19 | Calibration contracts | v2 | Code + protocol |
