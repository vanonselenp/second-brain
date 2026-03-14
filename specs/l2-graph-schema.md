# L2 Knowledge Graph Schema

## Overview

This document defines the graph schema for the L2 (System 2) layer of a cognitive architecture for LLMs. The graph serves as a persistent, evolving knowledge store that enriches LLM prompts (L2→L1) and absorbs learning from LLM interactions (L1→L2).

The architecture draws on Kahneman's dual-process theory: the LLM operates as System 1 (fast, pattern-matching, stateless), while this graph provides System 2 capabilities (persistent memory, structured knowledge, contextual reasoning, belief evolution).

**Target database:** Neo4j with native vector indexing.

---

## Design Principles

1. **Epistemic pluralism.** The graph supports multiple conflicting ideas simultaneously. Contradiction is a feature, not a bug. Confidence is contextual and Bayesian — incrementally adjusted through evidence, never binary.

2. **Layered epistemology.** Four node types form a hierarchy from concrete to abstract: Facts ground reality. Evidence captures observations. Concepts represent interpreted beliefs. Contexts condition when beliefs apply.

3. **Bidirectional flow.** L2→L1: the graph enriches prompts with relevant concepts, grounding facts, and known tensions. L1→L2: learning flows back through three channels — experiential learning, book learning, and session compaction.

4. **Provenance everywhere.** Every belief should be traceable to the evidence and facts that support it, and every piece of evidence should reference its source.

---

## Node Types

### Fact

The ground truth layer. Facts are verifiable statements about reality. They are not on a confidence spectrum in the same way concepts are — they are either true or they aren't. When facts change, the change should cascade to dependent concepts.

| Property | Type | Description |
|---|---|---|
| `id` | string | Unique identifier |
| `statement` | string | The factual claim in natural language |
| `embedding` | vector | Semantic embedding for retrieval |
| `verification_status` | enum | `verified` · `unverified` · `outdated` · `contested` |
| `source` | string | Where this fact originated (document, API, observation) |
| `domain` | string | Knowledge area (e.g. "infrastructure", "market", "regulation") |
| `valid_as_of` | datetime | When this fact was last known to be true |
| `created_at` | datetime | When this node was created |
| `updated_at` | datetime | Last modification timestamp |

**Design notes:**
- A fact at low confidence is a problem, unlike a concept at low confidence. The `verification_status` enum captures this differently from a numeric weight.
- When a fact becomes `outdated` or `contested`, this should trigger review of dependent concepts. The harness is responsible for this cascade, not the graph itself.
- Facts are not immutable. They update. But the old state is preserved via SUPERSEDES relationships to maintain history.

---

### Evidence

The observation layer. Evidence is something that happened, was read, or was synthesised. Each piece of evidence may support or weaken one or more concepts and may reference one or more facts.

| Property | Type | Description |
|---|---|---|
| `id` | string | Unique identifier |
| `content` | string | Natural language description of the evidence |
| `embedding` | vector | Semantic embedding for retrieval |
| `source_type` | enum | `experiential` · `book` · `compaction` |
| `source_reference` | string | Task ID, document path, or session ID |
| `outcome` | enum (nullable) | `success` · `failure` · `partial` · `null` (for non-experiential) |
| `context_description` | string | The circumstances under which this evidence was observed |
| `timestamp` | datetime | When the evidence was captured |
| `created_at` | datetime | When this node was created |

**Design notes:**
- Experiential evidence (source_type: `experiential`) is the highest-value learning signal, especially failures. The `outcome` field is critical here — failure evidence is often more diagnostic than success.
- Book learning evidence (source_type: `book`) comes from document ingestion. It may be decomposed into multiple evidence nodes, each linked to relevant concepts and facts.
- Compaction evidence (source_type: `compaction`) is synthesised at the end of a work session. It represents distilled learning from many interactions, compressed into key insights.
- The `context_description` captures the conditions under which this evidence was observed. This feeds into contextual confidence — the same evidence might strengthen a concept in one context and be irrelevant in another.

---

### Concept

The belief layer. Concepts are ideas, heuristics, principles, or interpretations that the system holds with varying degrees of confidence. Two conflicting concepts can both have high confidence if they apply in different contexts.

| Property | Type | Description |
|---|---|---|
| `id` | string | Unique identifier |
| `name` | string | Short label for the concept |
| `description` | string | Full natural language description |
| `embedding` | vector | Semantic embedding for retrieval |
| `confidence` | float | Aggregate confidence [0.0 – 1.0], derived from contextual confidences |
| `confidence_history` | list | Timestamped history of confidence values, e.g. `[{t: datetime, v: float}, ...]` |
| `stability` | float | How much confidence has oscillated over time [0.0 – 1.0]. High = stable belief, low = volatile |
| `contexts_summary` | string | Human-readable summary of when this concept applies |
| `evidence_count` | integer | Number of evidence nodes linked to this concept |
| `first_introduced_by` | enum | `experiential` · `book` · `compaction` — which channel created it |
| `status` | enum | `active` · `merged` — merged concepts preserved for provenance only |
| `created_at` | datetime | When this node was created |
| `updated_at` | datetime | Last modification timestamp |

**Design notes:**
- `confidence` is the aggregate, but the real nuance lives in the per-context confidence on APPLIES_IN relationships. The aggregate exists for quick filtering and ranking.
- `stability` is derived from the history of confidence changes. A concept that has steadily grown from 0.3 to 0.7 over 15 updates is qualitatively different from one bouncing between 0.4 and 0.9. Stability informs how much weight the harness should give this concept.
- The `contexts_summary` is a denormalised human-readable field. The structured context data lives on the APPLIES_IN relationships. Both exist because the summary is useful for L2→L1 prompt enrichment, while the structured data is useful for the harness's reasoning.
- Concepts may carry a `status` of `active` or `merged`. Merged concepts are retained for provenance but excluded from L2→L1 enrichment queries. The DERIVED_FROM relationship links them to their successor.

---

### Context

The conditioning layer. Contexts represent the circumstances under which concepts apply. They allow the system to hold conflicting beliefs without incoherence — the beliefs aren't really conflicting, they're conditional.

| Property | Type | Description |
|---|---|---|
| `id` | string | Unique identifier |
| `name` | string | Short label (e.g. "early-stage market", "high uncertainty", "technical audience") |
| `description` | string | Full description of this context |
| `embedding` | vector | Semantic embedding for retrieval |
| `context_type` | enum | `domain` · `situational` · `temporal` · `audience` · `resource` |
| `created_at` | datetime | When this node was created |
| `updated_at` | datetime | Last modification timestamp |

**Design notes:**
- `context_type` helps organise contexts into categories. Domain contexts are about subject area. Situational contexts are about circumstances. Temporal contexts are about timing. Audience contexts are about who you're communicating with. Resource contexts are about constraints.
- Contexts should be reusable across many concepts. "High-uncertainty environment" might condition dozens of different concepts.
- New contexts can emerge from evidence — if the system notices a concept holds in some situations but not others, the distinguishing factor becomes a new context node.

---

## Relationship Types

### SUPPORTS

**Between:** Concept → Concept

One concept provides support for another. Directional — A supports B doesn't mean B supports A.

| Property | Type | Description |
|---|---|---|
| `weight` | float | Strength of the support relationship [0.0 – 1.0] |
| `created_at` | datetime | When this relationship was established |
| `updated_at` | datetime | Last modification |

---

### CONTRADICTS

**Between:** Concept ↔ Concept

Two concepts are in tension. This is not an error state — it's an explicit acknowledgement that both ideas have merit and the tension is informative. Typically bidirectional (if A contradicts B, B contradicts A), but stored as two directed edges so weights can differ.

| Property | Type | Description |
|---|---|---|
| `weight` | float | Strength of the contradiction [0.0 – 1.0] |
| `resolution_notes` | string (nullable) | If the tension has been partially resolved or contextualised, notes on how |
| `created_at` | datetime | When this relationship was established |
| `updated_at` | datetime | Last modification |

**Design notes:**
- When the harness encounters a CONTRADICTS relationship during L2→L1 enrichment, it has a decision: surface the tension to the LLM, or pick the frame that fits the current context. The `resolution_notes` and contextual confidence of each concept inform that decision.

---

### EVIDENCED_BY

**Between:** Concept → Evidence

Links a concept to a piece of evidence that informs its confidence.

| Property | Type | Description |
|---|---|---|
| `direction` | enum | `strengthens` · `weakens` |
| `magnitude` | float | How much this evidence moved confidence [0.0 – 1.0] |
| `context_at_time` | string (nullable) | What context was active when this evidence was applied |
| `created_at` | datetime | When this relationship was established |

**Design notes:**
- The same evidence node can strengthen one concept and weaken another.
- `context_at_time` records what context the system was operating in when this evidence was captured. This is important because the same outcome (e.g. "the deployment failed") might mean different things in different contexts.

---

### GROUNDED_IN

**Between:** Concept → Fact

This concept's validity depends (in part) on this fact being true.

| Property | Type | Description |
|---|---|---|
| `relevance` | float | How central this fact is to the concept [0.0 – 1.0] |
| `created_at` | datetime | When this relationship was established |

**Design notes:**
- When a fact's `verification_status` changes, the harness should traverse GROUNDED_IN relationships to identify concepts that may need confidence re-evaluation.
- A concept with all its grounding facts verified is more trustworthy than one resting on unverified facts, regardless of the concept's own confidence score.

---

### REFERENCES

**Between:** Evidence → Fact

This piece of evidence contains or references this fact.

| Property | Type | Description |
|---|---|---|
| `created_at` | datetime | When this relationship was established |

---

### SUPERSEDES

**Between:** Fact → Fact

A newer fact replaces an older one. The old fact is marked `outdated` and linked to its replacement.

| Property | Type | Description |
|---|---|---|
| `reason` | string | Why the fact changed |
| `created_at` | datetime | When the supersession was recorded |

**Design notes:**
- Old facts are never deleted, only marked outdated. The SUPERSEDES chain gives you a history of how the factual ground has shifted.
- Concepts that were GROUNDED_IN the old fact don't automatically switch to the new fact. The harness must evaluate whether the new fact still supports the concept or whether the concept needs revision.

---

### APPLIES_IN

**Between:** Concept → Context

This concept holds (to some degree) within this context.

| Property | Type | Description |
|---|---|---|
| `conditional_confidence` | float | How strongly this concept holds in this specific context [0.0 – 1.0] |
| `evidence_count` | integer | How many pieces of evidence inform this conditional confidence |
| `last_updated` | datetime | When this conditional confidence was last adjusted |

**Design notes:**
- This is where the Bayesian heart of the system lives. A concept might have global confidence of 0.7 but conditional confidence of 0.9 in "early-stage markets" and 0.3 in "regulated industries." The harness uses the current situation to determine which conditional confidence to weight.
- If a concept has no APPLIES_IN relationships, it's treated as context-independent (its global confidence applies everywhere). As the system learns, context-independent concepts should gradually acquire APPLIES_IN edges as the system discovers when they do and don't hold.

---

### DERIVED_FROM

**Between:** Concept → Concept

One concept was synthesised from another (typically during compaction).

| Property | Type | Description |
|---|---|---|
| `derivation_type` | enum | `generalisation` · `specialisation` · `synthesis` · `decomposition` |
| `created_at` | datetime | When this derivation was recorded |

**Design notes:**
- This is a provenance relationship. It lets you trace why the system believes something.
- `generalisation`: a broader concept extracted from a more specific one.
- `specialisation`: a more specific variant of a broader concept.
- `synthesis`: a new concept formed by combining multiple existing ones.
- `decomposition`: an existing concept broken into constituent parts.

---

### RELATED_TO

**Between:** Concept ↔ Concept

A soft associative link. Not as strong as SUPPORTS or CONTRADICTS — these concepts are adjacent or share thematic territory.

| Property | Type | Description |
|---|---|---|
| `strength` | float | How strongly associated [0.0 – 1.0] |
| `created_at` | datetime | When this relationship was established |

---

## Confidence Evolution Model

### How confidence updates work

Confidence is not set directly. It evolves through evidence:

1. New evidence arrives (from any of the three channels).
2. The harness determines which concepts the evidence is relevant to.
3. For each relevant concept, an EVIDENCED_BY relationship is created with a `direction` (strengthens/weakens) and `magnitude`.
4. The concept's `confidence` is recalculated as a weighted function of all EVIDENCED_BY relationships, accounting for recency, source reliability, and contextual relevance.
5. The concept's `stability` is updated based on how much this change moved confidence relative to recent history.
6. If the evidence was captured in a specific context, the relevant APPLIES_IN `conditional_confidence` is also updated.

### Stability calculation

Stability reflects the volatility of belief. A simple approach:

- Maintain a rolling window of the last N confidence values.
- Stability = 1 - (standard deviation of the window / max possible deviation).
- A concept that has been at 0.7 for 20 updates has stability near 1.0.
- A concept bouncing between 0.3 and 0.9 has stability near 0.0.

### Aggregate confidence

The global `confidence` on a Concept node is derived from its contextual confidences:

- If APPLIES_IN relationships exist: weighted average of `conditional_confidence` values, weighted by `evidence_count` for each context.
- If no APPLIES_IN relationships exist: derived directly from the EVIDENCED_BY relationships.

---

## Learning Channels

### Channel 1: Experiential Learning

**Trigger:** A task is completed (success, failure, or partial).

**Process:**
1. Extract factual observations from the task outcome → create/update Fact nodes.
2. Create an Evidence node with `source_type: experiential` and the appropriate `outcome`.
3. Identify relevant existing Concepts. For each, create EVIDENCED_BY with direction and magnitude.
4. If the evidence reveals a new contextual distinction (worked here, failed there), create or link to Context nodes and update APPLIES_IN conditional confidences.
5. If the evidence suggests a genuinely new idea, create a new Concept node.

### Channel 2: Book Learning

**Trigger:** A document or knowledge source is ingested.

**Process:**
1. Decompose the source into atomic factual claims → create Fact nodes.
2. Identify higher-level ideas and principles → create Concept nodes.
3. Create Evidence nodes (source_type: `book`) linking concepts to their source material.
4. Link Concepts to Facts via GROUNDED_IN.
5. Connect new Concepts to existing graph via SUPPORTS, CONTRADICTS, RELATED_TO by comparing embeddings and using the LLM to assess relationships.

### Channel 3: Session Compaction

**Trigger:** End of a work session in L1.

**Process:**
1. Review the session transcript.
2. Extract key insights — "what did I learn?" — with a high compression ratio.
3. For each insight, determine: is this a new concept, evidence for an existing concept, or a refinement of an existing concept?
4. Create Evidence nodes (source_type: `compaction`) and link to relevant Concepts.
5. Update confidence and stability for affected Concepts.
6. Identify any new contextual distinctions that emerged during the session.

---

## L2→L1 Enrichment Query Pattern

When the harness prepares to enrich an L1 prompt, the query follows this pattern:

1. **Semantic retrieval.** Use the prompt embedding to find the top-N most relevant Concept nodes via vector similarity.
2. **Graph expansion.** For each retrieved Concept, traverse:
   - CONTRADICTS → surface tensions with opposing concepts.
   - SUPPORTS → add reinforcing ideas.
   - GROUNDED_IN → pull in supporting facts (flag any that are `outdated` or `contested`).
   - APPLIES_IN → check conditional confidence against the current context.
3. **Filtering and ranking.** Weight results by: relevance (vector similarity), confidence (contextual if available, global otherwise), stability (prefer stable beliefs), and fact verification status.
4. **Context resolution.** If conflicting concepts are both relevant, decide whether to surface the tension or commit to the frame that fits the current context. Factors: how different are the conditional confidences in the current context? How stable are the competing beliefs? Is the task one that benefits from exploring tension or one that needs a clear direction?
5. **Prompt composition.** Assemble the enrichment payload: relevant concepts with their confidence and context, grounding facts, any tensions worth surfacing, and any caveats (e.g. "this belief rests on a fact that may be outdated").

---

## Graph Maintenance: Dreaming

Over time the graph accumulates nodes, evidence, and weak connections. It needs a consolidation process — analogous to what biological dreaming appears to do: strengthen important patterns, merge redundant memories, and let go of noise.

This is distinct from the three learning channels. Learning channels add to the graph. Dreaming reorganises it.

### Trigger

Dreaming runs periodically — after a threshold of new evidence has been added, or on a schedule, or both. It is not triggered by a single session but by cumulative graph growth.

### Process

1. **Cluster detection.** Identify groups of Concept nodes that are semantically similar (via embedding proximity) and/or heavily interconnected via SUPPORTS and RELATED_TO relationships.

2. **Merge candidates.** Within each cluster, identify concepts that are essentially expressing the same idea with different emphasis. Criteria: high embedding similarity, shared evidence base, similar confidence profiles.

3. **Merge execution.** For each merge:
   - Create a new Concept node (or promote the stronger of the two).
   - Combine all EVIDENCED_BY relationships from both source concepts. Recalculate confidence from the combined evidence base.
   - Union all APPLIES_IN relationships. Where both concepts had conditional confidence in the same context, recalculate from combined evidence.
   - Preserve all GROUNDED_IN relationships (union of both fact bases).
   - Before merging, examine divergent relationships — if concept A contradicts X but concept B doesn't, investigate whether this reveals genuine nuance. If so, preserve the distinction (perhaps as a context-dependent note) rather than flattening it.
   - Link the new concept to the originals via DERIVED_FROM (derivation_type: `synthesis`).
   - Mark original concepts as merged (don't delete — preserve for provenance).

4. **Connection strengthening.** For SUPPORTS, CONTRADICTS, and RELATED_TO relationships that have been reinforced by multiple independent pieces of evidence, increase their weight. For relationships that haven't been touched in a long time and have low weight, flag for potential pruning.

5. **Pruning.** Remove (or archive) nodes and relationships that meet decay criteria:
   - Evidence nodes that are very old, linked to no current concepts, and have been superseded by more recent evidence.
   - RELATED_TO relationships with very low strength that haven't been reinforced.
   - Context nodes that no APPLIES_IN relationships point to.
   - Fact nodes marked `outdated` where the SUPERSEDES chain has been fully resolved.

6. **Emergent context discovery.** Look for patterns where a concept has high conditional confidence in some contexts and low in others, but the distinguishing contexts haven't been explicitly captured. If the evidence suggests a pattern, create new Context nodes and APPLIES_IN relationships.

### Design notes

- Dreaming should be conservative. Merging is destructive (even with provenance) and should require high confidence that two concepts are genuinely redundant.
- Pruning should also be conservative. The cost of keeping a low-value node is low (storage). The cost of removing something that turns out to be relevant later is high.
- The dreaming process itself uses the LLM (System 1) to evaluate merge candidates, assess relationship divergences, and identify emergent contexts. This is another instance of L1 serving L2 — using the pattern matcher to maintain the knowledge store.

---

## Resolved Design Decisions

1. **Confidence history storage.** For the MVP, store confidence history as a list property on the Concept node (e.g. a list of `{timestamp, value}` pairs). Simpler, avoids node proliferation. Can be promoted to separate nodes later if queryability over confidence trajectories becomes important.

2. **Context granularity.** Start at medium granularity and adjust based on what the system actually learns. This can't be hypothesised in advance — it needs to be discovered through experimentation. The dreaming process can help here: if a coarse context consistently fails to disambiguate, it's a signal to split it. If fine-grained contexts are never used distinctly, merge them.

3. **Embedding model choice.** Use a single good general-purpose embedding model across all node types. The model space is churning too fast to optimise for specific node types. Treat the embedding model as a swappable component — pick one, build the system, swap when something better arrives. Re-embedding the graph is a batch operation, not an architectural change.

4. **Graph maintenance.** Resolved via the Dreaming process described above.

5. **Concept merging.** Resolved via the Dreaming process. Key principles: combine evidence, recalculate confidence from the combined base, preserve all provenance via DERIVED_FROM, examine divergent relationships before flattening (they may reveal genuine nuance rather than duplication), and check for contradictions that should be preserved rather than resolved.
