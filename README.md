# Production RAG Handbook

Engineering reference material on taking Retrieval-Augmented Generation from a
working prototype to something that survives production: ingestion, embeddings,
chunking, vector storage, retrieval, and the measurement discipline that decides
which of those techniques is actually worth its cost.

These are **notes on other people's arguments**, not a framework and not a
tutorial. Each article states a position, gives the reasoning, and — where it
matters — says explicitly when the position is wrong. The recurring theme across
all of them is the same: sophistication has a scale at which it starts to pay,
and most systems are below it.

## The system the articles are about

The articles are not independent essays that happen to share a topic. They are a
sequence about **one AI-engineering project carried forward across sessions: an
IT project estimator.** It ingests a company's historical budgets (JSON) and
client meeting transcripts, and produces an estimate for a new project grounded
in what comparable past projects actually cost.

Its architecture is the thing that changes. The system was **born as CAG** —
static context in the system prompt rather than retrieval — and session 05 is
where that chapter closes.

> **What the code shows, which is not quite what the articles say.** s06-01 and
> s09-01 describe the CAG as "roughly 30 budgets pasted into the system prompt".
> At branch `session_05_live` the few-shot context is
> `app/prompts/estimation/v1/examples.j2` — 57 lines, a handful of illustrative
> projects, included at `v1/system.j2:68` — and its own header instructs the
> model *"do not copy their numbers, use them only as a calibration anchor"*. It
> is a style and rigour anchor, not a grounding corpus. More: `v2` and `v3` have
> no examples file at all, and the conversational path already defaults to `v3`
> (`CONVERSATIONAL_PROMPT_VERSION`), whose system prompt grounds itself in
> "industry rates and standard delivery practices" plus a `<project_metadata>`
> block. So by the end of session 05 the live prompt carried no historical
> budgets. The articles' "frozen knowledge" argument still stands — parametric
> rates go stale the same way — but the 30 budgets are a teaching simplification,
> not a description of the code. **The jump to RAG happens across sessions 6 and
7**, and it is a crossing rather than a switch: session 6 goes back to the
sources and builds a real ingest side, session 7 turns that corpus into
embeddings and chunks. By session 8 the estimator has somewhere to put them, and
only in session 9 is the RAG flow itself named and taken apart stage by stage.
The residual CAG layer survives the crossing — the decision was to add
retrieval, not to delete the static context.

Part 5 is the CAG chapter, and it is where the handbook now starts; s06-01 is
where that architecture is argued *out of*, reading its failure modes as four
constraints rather than one and landing on RAG with a residual CAG layer. The
handbook now covers the decision from both sides.

Everything from there is the same estimator, **made more production-ready one
article at a time** — audited sources, then a real extraction pipeline, then
chunking that respects the JSON's structure, then pgvector, then an index, then a
retriever that is not only cosine, then reranking, hybrid search and query
rewriting on top.

Two consequences for reading. The worked examples recur on purpose — the Stripe
budget, the forty-minute transcript that mixes catalogue with billing
integration, the module trees in the code blocks are one codebase seen at
different stages, not unrelated illustrations. And **later articles routinely
revise earlier ones**, because the system has grown between them: a technique
ruled out at one scale is adopted at the next. Where that happens it is flagged
rather than smoothed over (see s09-02 and s10-04).

### The code paths are snapshots, not one layout

The paths in the code blocks disagree across the arc, and that is history rather
than sloppiness: the project was **refactored into a layered architecture**
partway through — commit `eb66136`, "reorganize app/ into layered architecture
(CAG/RAG/Agentic)" — so an article's paths date it. That commit is also where
`app/cache/semantic.py` became `app/generation/cag/semantic.py` (a 98% rename),
which is how a caching module ended up wearing the CAG name. `src/estimator/retrieval/` in
s09-05 and `app/generation/rag/retrieval/` in part 10 are the same component
before and after that split. Read the last segment — `retrieval/`, `ingest/`,
`generation/` — as the stable part and the prefix as a timestamp.

**Retrieval belongs to RAG**, which sets the layout to build for a new project
from this handbook: start with `cag/`, create `rag/` at the point you actually
adopt retrieval, and put the pipelines and everything retrieval-related inside
it — ingestion, chunking, the retriever, reranking, fusion, routing. Keeping the
two side by side rather than replacing one with the other is what the estimator
did, and it is why its **residual CAG layer** survived the crossing.

> **CAG is two different things in this material, sharing an acronym.** In the
> articles it is *context*-augmented generation: roughly 30 budgets pasted into
> the system prompt, the architecture s06-01 argues its way out of. In the code,
> `app/generation/cag/` is *cache*-augmented generation — `EstimationCache`, an
> exact Redis cache keyed on a SHA-256 of the full prompt, and
> `EstimationSemanticCache`, which matches on a `prompt_version:project_type:
> detail_level:output_format` bucket plus cosine ≥ 0.92 on the description
> embedding. The prompt-stuffing the articles mean lives in prompt building,
> under what the code calls the Session 2 controls (`num_examples`,
> `example_format`, `ACTIVE_OUTPUT_PROMPT`). Do not read the folder name as the
> articles' CAG.

### Article to module map

Checked against the reference implementation at `lidr/ai-engineering`, branch
`session_11`, commit `3292744`. **The articles are the handbook's source of
truth and are reproduced verbatim; where a name below differs, the code is what
you will actually find.** Rows marked *not implemented* are articles running
ahead of the codebase.

| Article | Article names it | In the code |
|---|---|---|
| s05-01 | `extract_text_from_pdf` | `app/attachments/extractor.py` — `extract_text`, dispatching to `pypdf` / `python-docx` |
| s05-01 | Tavily / SERP search, `find_similar_projects` | *not implemented* |
| s05-02 | `ProjectMetadata`, `Session`, `ConversationHistory` | `app/sessions/models.py` — 4 metadata fields not 6, `Session.metadata` not `project_metadata`, no `updated_at` |
| s05-02 | `update_metadata_llm` | `app/sessions/metadata_extractor.py` — `update_metadata`, structured output, `METADATA_EXTRACTOR_MODEL` |
| s05-02 | sliding window, `MAX_TURNS = 6` | `max_turns=6` **plus** `anchors` and `summary` — hybrid, via `sessions/compression/` |
| s05-02 | session TTL / archiving | *not implemented* — no `updated_at`, no expiry job |
| s06-01 | `/index/run`, `/query` | `app/api/ingestion.py` (`/api/v1/ingestion`), `app/api/search.py` |
| s06-02 | `data_catalog.yaml` | `app/ingestion/catalog/` — `loader.py`, `models.py`, `inspect.py` |
| s06-03 | parsers, registry | `app/ingestion/parsers/` — `budget_json.py`, `transcript_txt.py`, `registry.py`, `protocol.py` |
| s06-04 | cleaning + validation | `app/ingestion/cleaning/` — `budget_records.py`, `policy.py`, `schemas.py` |
| s06-05 | Presidio analyzer, pseudonymiser | `app/ingestion/pii/` — `analyzer.py`, `pseudonymizer.py`, `recognizers.py`, `mapping_store.py` |
| s07-03/04 | chunking strategies | `app/generation/rag/chunking/` — `base.py` (`Chunker` ABC), `structural.py`, `strategies/*.py` |
| s08-04 | `documents` + `chunks` | `app/generation/rag/store/models.py` — but **three** chunk tables, see below |
| s09-02 | `query_reformulator.py` | same path; `compose_search_text(query: EstimationQuery)` |
| s09-03 | retriever | `app/generation/rag/retriever.py` — `SemanticRetriever` |
| s09-04 | `context_assembler.py` | same path — but `reorder_u_pattern` is **absent**, see below |
| s09-05 | `src/estimator/api/routers/` | `app/api/routers/` — prefixes `/v1/retrieval` and `/v1/estimate` exactly as described |
| s09-05 | `require_retrieval_key`, `require_estimate_key` | `app/api/security.py`, both names exact, `secrets.compare_digest` as argued |
| s09-05 | `SearchRequest`, `SearchResponse` | `RetrievalRequest`, `RetrievalResult` |
| s09-05 | 120/min, 10/min | exact, in `routers/retrieval.py` and `routers/estimate.py` |
| s10-01 | reranker | `app/generation/rag/retrieval/reranker.py` — `CrossEncoderReranker` |
| s10-02 | golden set, metrics | `evals/` — `metrics.py`, `dataset.py`, `run.py` |
| s10-03 | `reciprocal_rank_fusion` | `retrieval/fusion.py`, name exact |
| s10-04 | `query_expansion.py` | `retrieval/query_transform.py` |
| s10-04 | `interleave_rankings` | `round_robin_merge` in `retrieval/fusion.py` |
| s10-05 | `SearchTarget` | `Collection(StrEnum)` in `retrieval/collections.py`, alongside `CollectionSpec`, `HardFilters`, `match_rules` |
| s10-05 | `route_query`, `RoutingDecision` | `retrieval/router.py` — `RoutingDecision` exact, plus `RouteClassification` |
| s10-06 | `temporal_weight(document_date, …)` | `decay_weight(age_days, half_life_days)` and `apply_temporal_decay` in `retrieval/temporal.py` |
| s10-06 | per-stage toggles | `retrieval/advanced_pipeline.py` — `StageConfig.from_settings` |
| s09-04 | `validate_citations` | `verify_citations` in `rag/validation.py` |
| s11-01 | `compress_chunk`, `extract_key_points`, `BudgetEvidence` | *not implemented* |
| s11-02 | `aggregate_components`, `weighted_median`, `SourceWeight` | *not implemented* |
| s11-03 | `check_citation_integrity`, `CitationIntegrityReport` | `verify_citations`, `CitationReport` in `rag/validation.py` — per line, with a third `insufficient` status |
| s11-03 | `Citation.locator`, `char_span` | `SourceReference.evidence` — a model-copied verbatim span, not an ingest-captured offset |
| s11-03 | `CitationLinkResolver` (Rails) | *not implemented* |
| s11-04 | `numeric_grounding`, `verify_claim`, `gate_line`, `consistency_spread` | *not implemented* |
| s11-04 | consistency as dispersion | `rag/task_hours.py` — same formula, over source hours, feeding `reliability` |
| s11-04 | `ClaimVerdict`, the strict judge | `agentic/critic.py` — `Critic`, `CriticFeedback`; fails open on error |
| s11-04 | `status="insufficient"` | `validation.check_coherence` enforces it as an invariant |
| s11-05 | `embedding_version`, `source_hash` columns | *absent from the schema and from the Alembic migrations* |
| s11-05 | `EmbeddingVersion`, `is_stale`, `reindex_incremental`, shadow index | *not implemented* |
| s11-05 | `FROM chunks` | `budget_chunks`, `transcript_chunks`, `technical_doc_chunks` — the filter belongs on all three |
| s11-06 | `run_ragas`, `build_eval_dataset` | `scripts/eval_ragas_s11.py`, plus `score_ragas_s11.py` and `evals/` |
| s11-06 | golden set | `evals/golden_generation_s11.json` — 5 queries, no abstention case |
| s11-06 | "pin the RAGAS version" | `ragas>=0.2` in `pyproject.toml`, `0.4.3` in `uv.lock`, NaN fallback in `_per_query_scores` |

Two rows deserve more than a rename.

**s08-04 versus s10-05 is settled by the code.** `store/models.py` declares
`documents` plus **three** chunk tables — `budget_chunks`, `transcript_chunks`,
`technical_doc_chunks`. That is s10-05's "option B", so the implementation went
with the later article against the earlier one, and the editor's note in s08-04
records an argument the codebase has already resolved.

**s09-04 claims a function the repo does not have.** It says `reorder_u_pattern`
is "left as a configurable option in `context_assembler.py`"; that file contains
only `_wrap_chunk`, `build_context_block` and `truncate_to_token_budget`. The
function exists in the article and nowhere else — which matters, because s11-01's
note points readers at it as the correct implementation of edge loading. It is
correct, and it is theirs to write.

**The code is sometimes more specific than the article.** s10-04 describes query
routing as "a humble heuristic (query length and structure)"; `query_transform.py`
has the real policy — `_SHORT_WORDS = 6`, `_LONG_WORDS = 25`,
`_MULTITOPIC_CONNECTORS = 2`, and `choose_technique()` returning
`DIRECT` / `EXPAND` / `DECOMPOSE`. When an article waves at a heuristic, the
constants are usually in the module.

### Reading it for a different domain

This handbook is reference material for building **several RAG systems in
different contexts**, not only the estimator it was written from. That makes one
reading habit essential: the articles argue from a specific corpus, and some of
their conclusions are properties of that corpus wearing the clothes of general
principles.

The mechanisms transfer; the domain premises do not. Chunking strategy,
index anatomy, fusion by position, recall-then-rerank, the pipeline ordering
principle — these are about how retrieval works and hold anywhere. But claims
that depend on what the documents *are* need re-deriving each time. The sharpest
example is time: s10-06 argues recency is worth more "universally", which is true
of budgets and false of a clinical corpus retrieving on symptom similarity, where
an old case presentation is not a worse match and a half-life would suppress the
rare presentations the corpus exists for.

Where a claim is domain-bound in a way that would mislead on transfer, it carries
an `> *(Editor's note — domain transfer: ...)*` block working out what survives
the move and what does not. When you hit one that is not yet flagged, flag it —
those notes are the difference between a handbook and a case study.

## Reading order

The articles form an arc through the pipeline, in the order the pipeline runs.

| Part | Covers |
|---|---|
| [5 — Before RAG](#part-5--before-rag) | The CAG system, and reaching outside it without retrieval |
| [6 — Ingest](#part-6--the-ingest-side) | Choose the architecture, then audit the sources, extract them, clean and validate, make them safe to embed |
| [7 — Embeddings and chunking](#part-7--embeddings-and-chunking) | What an embedding is, which model to pick, how to cut the text |
| [8 — Vector databases](#part-8--vector-databases) | Whether you need one, which one, what the index actually builds, and its ceiling |
| [9 — The RAG flow](#part-9--the-rag-flow) | The four canonical stages — reformulation, retrieval, augmentation, generation — and serving them as an operable service |
| [10 — Advanced retrieval](#part-10--advanced-retrieval) | Reranking, hybrid search, query rewriting, and how to measure whether any of them helped |
| [11 — Augmentation and synthesis](#part-11--augmentation-and-synthesis) | Distilling retrieved chunks into evidence, then reconciling sources that disagree |

If you read only three: **s07-03** (chunking), **s09-03** (retrieval is not only
cosine), **s10-02** (measure the cost, not only the gain).

## Part 5 — before RAG

The estimator as a CAG system: static context in the prompt, and how it reaches
past that without retrieval yet. This is where the handbook now starts.

| # | Article | What it argues |
|---|---|---|
| 1 | [Dynamic context from external sources](articles/s05-01-dynamic-context-from-external-sources.md) | Static context lives in code and is paid for once; dynamic context is fetched per request, costs tokens every turn, adds latency the user feels, and is **input, not program** — delimit it or invite injection. Attachments go multimodal *or* locally extracted, never both. The business database is reached by HTTP contract and never shared, and that rule survives into RAG. |
| 2 | [Conversational memory vs history](articles/s05-02-conversational-memory-vs-history.md) | History answers "what was said on turn 7?", memory answers "what do we know about this project?", and merging them answers both badly. Memory is distilled facts in a typed structure, independent of the turn that produced them, so it survives truncation — which is what makes the *simplest* history strategy safe. Forgetting needs three policies in three different places. |

## Part 6 — the ingest side

| # | Document | What it argues |
|---|---|---|
| 1 | [Data quality and architecture decisions](articles/s06-01-data-quality-and-architecture-decisions.md) | The CAG ceiling is four constraints at once, and cost usually kills the project before capacity does. No clever chunking fixes bad data, which is why three sessions go to data before anything is embedded. Splits the six-step pipeline into offline indexing and online query, gives a four-axis decision tree, and warns that RAG can cost *more* than CAG — so the choice is made on viability, not price. |
| 2 | [Enterprise data audit and inventory](articles/s06-02-enterprise-data-audit-inventory.md) | Audit before vectorising. A factual source census, four non-averaging quality dimensions, "context erosion" as the reason lineage is a *condition of usefulness* in RAG, and a versioned typed `data_catalog.yaml` the pipeline reads at runtime. |
| 3 | [Multi-format extraction pipeline](articles/s06-03-multi-format-extraction-pipeline.md) | A modular `ingest/` (loaders → parsers → normalizers) converging on a canonical `Document`, instead of delegating everything to `unstructured.partition()`. Per-format strategy, three-source metadata propagation, and why `hi_res` PDF parsing everywhere is a waste. |
| 4 | [Data cleaning, normalization and validation](articles/s06-04-data-cleaning-normalization-validation.md) | The *form* contract (Pydantic) is not the *content* contract. Four families of dirty data, cleaning as one auditable layer between parser and normalizer, Pandera as a versioned data contract, and an explicit repair / quarantine / discard policy. |
| 5 | [PII, anonymization and GDPR](articles/s06-05-pii-anonymization-gdpr.md) | Access control cannot protect a vector corpus. Three leakage modes, four GDPR concepts that drive technical decisions, and consistent reversible pseudonymization (Presidio + Faker + mapping table) over generic redaction, which costs 15-25% of retrieval quality. |

Article 1 is where the architecture is chosen; 2-5 are the arc that follows from
choosing RAG — audit the sources, extract them, clean and validate what came out,
then make it safe to embed. Article 2 opens by treating the decision as already
made, which is exactly where article 1 leaves off.

## Part 7 — embeddings and chunking

| # | Document | What it argues |
|---|---|---|
| 1 | [Embeddings: from text to semantic geometry](articles/s07-01-embeddings-semantic-geometry.md) | What an embedding is, why contrastive training makes distance mean similarity, and the three metrics — with the argument that metric choice is a *property of the model*, not an architectural decision. Four honest limits, including that embeddings cannot reason about numbers, dates or ids. |
| 2 | [Embedding model selection: production trade-offs](articles/s07-02-embedding-model-selection.md) | The 2026 landscape, why MTEB is a coarse filter rather than an oracle, Matryoshka truncation and its renormalisation trap, and five decision axes — with a worked decision and an explicit account of when it would be wrong. |
| 3 | [Professional chunking strategies](articles/s07-03-professional-chunking-strategies.md) | Twelve strategies in four families (mechanical, structural, semantic, advanced/contextual). Recursive 400-512 is hard to beat, sophistication does not guarantee improvement, JSON needs a custom chunker, and Contextual Retrieval is the one advanced technique worth adopting early. |
| 4 | [Applied chunking: JSON budgets and transcripts](articles/s07-04-project-chunking-json-and-transcripts.md) | A worked case landing on two document types: a structural JSON chunker at business-unit granularity with parent context prepended, topic segmentation for transcripts, an explicit text-vs-metadata rule, and an ingest router keyed on explicit document type. |

## Part 8 — vector databases

| # | Document | What it argues |
|---|---|---|
| — | [Hands-on: migration to pgvector + search endpoint](articles/s08-00-hands-on-pgvector-migration-and-search.md) | Practical brief: pgvector in compose, Alembic with the `vector` type registered, a `documents`/`chunks` schema, atomic ingestion in one transaction, cosine top-k search — and deliberately **no** vector index, so sequential scan is the measured baseline. |
| 1 | [Why vector databases exist and when you actually need them](articles/s08-01-why-vector-databases-exist.md) | Similarity search is a different computational problem from exact lookup; the four properties an in-memory array lacks; three scale thresholds including where a vector DB is *over-engineering*; and how ANN's non-determinism changes testing and debugging. |
| 2 | [State of the vector database market 2026](articles/s08-02-vector-database-market-2026.md) | Five options on four axes (operating model, practical scale, native features, total cost). pgvector's ceiling, Qdrant's filtering, Weaviate's opinionated hybrid search, Milvus at billion scale, Pinecone's real bill — and an explicit list of when pgvector stops being the answer. |
| 3 | [Anatomy of a vector index: HNSW, IVFFlat, DiskANN](articles/s08-03-anatomy-of-a-vector-index.md) | What `CREATE INDEX USING hnsw` actually builds. IVFFlat's Voronoi cells and silent degradation under inserts, HNSW's layered graph and its three parameters, DiskANN beyond RAM — and a decision table that includes "no index at all". |
| 4 | [Schema design and semantic search](articles/s08-04-schema-design-and-semantic-search.md) | The applied piece: two-table schema with the typed-column/JSONB split, the three distance operators and why cosine beats the faster inner product, the operator-class trap with its `CREATE INDEX`, `EXPLAIN ANALYZE` as the reflex, and `hnsw.iterative_scan` for selective filters. Its single chunk table with a `document_type` column is revised by **s10-05** once the corpus stops being only budgets. |
| 5 | [From prototype to production: tuning, monitoring, pgvector's ceiling](articles/s08-05-from-prototype-to-production.md) | Memory sizing as 80% of performance, index-build parameters (`maintenance_work_mem`, parallel workers, `CONCURRENTLY`), `halfvec` quantisation, monitoring via `pg_stat_user_indexes`, the VACUUM/REINDEX cycle, and three measurable migration signals. |

## Part 9 — the RAG flow

| # | Document | What it argues |
|---|---|---|
| 1 | [From static CAG to the RAG flow](articles/s09-01-from-cag-to-rag-four-stages.md) | The four canonical stages, five operational differences from CAG, an honest account of when CAG still wins (Chan et al. 2024), and the rule that retrieval quality is the ceiling on the whole system. |
| 2 | [Query reformulation](articles/s09-02-query-reformulation.md) | Why embedding a raw transcript fails (length dissolves signal, noise drowns keywords, anaphora encodes nothing), the five reformulation families, and the case for structured extraction on debuggability and downstream filter utility. Its verdict on sub-query decomposition is softened by **s10-04**. |
| 3 | [Retrieval that is not only cosine](articles/s09-03-retrieval-topk-threshold-filters.md) | The two failure modes of a top-K-only retriever, threshold set from the empirical distance distribution, soft-fail with `low_confidence`, pre/post/in-query filtering chosen by selectivity, four anti-patterns, and precision over recall when the answer has economic consequences. |
| 4 | [Augmentation: assembling context](articles/s09-04-augmentation-assembling-context.md) | Why `"\n\n".join` yields invented citations and silently ignored context. XML `<source>` delimiters with metadata, lost-in-the-middle and chunk order, whole-chunk truncation counting the wrapper, a prompt with grounding and a licence to refuse, and post-generation citation validation. Its `reorder_u_pattern` is the implementation **s11-01** should have reused. |
| 5 | [The data layer as a service](articles/s09-05-data-layer-as-a-service-securing-the-retriever.md) | The retriever and the generator are two logical services sharing a codebase, and blast radius, rate limits and credential granularity all prove it. Two FastAPI routers with deliberately asymmetric contracts, two keys compared in constant time, 120/min against 10/min because one call costs milliseconds and the other costs euros, idempotency keys so a retry does not buy a second estimate, and per-stage logging tied by `request_id`. |

## Part 10 — advanced retrieval

| # | Document | What it argues |
|---|---|---|
| 1 | [Reranking: when vector top-k is not enough](articles/s10-01-reranking.md) | Bi-encoders find well and order badly because compression averages a document and never lets query and document read each other; cross-encoders fix the order and cannot scale; recall-then-rerank buys both. Plus an explicit list of when reranking is the wrong tool. |
| 2 | [Artisanal relevance measurement](articles/s10-02-artisanal-relevance-measurement.md) | Turning "it seems better" into a number: golden sets of 5-20 annotated queries, precision@k at the k you actually use, latency measured warm and by median, and a gain-versus-cost table — including the treacherous quadrant where a small gain at a small cost still is not worth the complexity toll. |
| 3 | [Hybrid search](articles/s10-03-hybrid-search.md) | Semantic search dilutes proper nouns; lexical search nails them and cannot read paraphrase. Both live in PostgreSQL (pgvector + `tsvector`/GIN), and Reciprocal Rank Fusion joins them on positions alone, sidestepping two incomparable score scales. |
| 4 | [Query expansion and decomposition](articles/s10-04-query-expansion-and-decomposition.md) | The only improvement that acts on the query, not the documents. One intent said many ways is *expanded*; many intents said at once are *decomposed* — and they must not fuse alike: expansion rewards consensus with RRF, decomposition needs round-robin coverage or the majority topic floods the context. An LLM on a short leash, and an LLM generation on the critical path of every search. |
| 5 | [Multi-index and routing](articles/s10-05-multi-index-and-routing.md) | A single index answers "what resembles this query?" when the question was "which *budget* resembles it?". Four degradation mechanisms once families mix, diverging metadata schemas as the signal to split tables, and a router hierarchy whose level zero is **no router** — put the collection in the API contract, since the caller knows what the service would guess. Partition on observed contamination, never "for when we grow". |
| 6 | [Contextual and temporal filtering](articles/s10-06-contextual-and-temporal-filtering.md) | An embedding encodes what the text says, not when it was written. Hard filters go as early as possible and soft weights as late as possible — the asymmetry is the point — with time handled by hard windows for a categorical reason and exponential decay for ordinary erosion. Closes the part by assembling every stage into one pipeline: cheap and excluding first, expensive and fine last, soft at the close. |

Article 6 is the part's capstone: it states the ordering principle for the whole
retrieval pipeline and places every technique from parts 9 and 10 in it, so it
reads well as a summary of the arc even before the individual articles.

Article 4 revisits ground **s09-02** already covered: decomposition is one of the
five reformulation families listed there, and s09-02 rules it out as the default
in favour of structured extraction. s10-04 is warmer on it for the same
estimation system, and adds the fusion asymmetry s09-02 does not raise. Both are
the same author, so `source` does not settle it; read s09-02 for *whether* to
reformulate and s10-04 for *how* the fan-out is built and paid for.

## Part 11 — Augmentation and synthesis

| # | Article | What it argues |
|---|---|---|
| 1 | [Content augmentation](articles/s11-01-content-augmentation-preparing-context.md) | The layer between retrieval and generation, where raw chunks become distilled evidence. Noise costs tokens, attention and hallucination risk — irrelevant figures are fuel for grabbing the wrong one. Extractive compression cannot invent and wins for a corpus of figures; abstractive adds a second generation point and must be fenced by a schema where a missing figure stays `None`. Preserve `chunk_id` or traceability dies before it is built. |
| 2 | [Synthesising contradictory budgets](articles/s11-02-synthesising-contradictory-budgets.md) | Three comparable budgets say 40h, 90h and 55h, and averaging them into 62 throws away the most valuable thing the data had: the disagreement names a variable the client never mentioned. Weight on three signals not seven, compute a deterministic anchor in code so the model reasons over the arithmetic instead of inventing it, and return the range — collapsing a real disagreement into a clean number is lying with the appearance of rigour. |

| 3 | [Citation and verifiable attribution](articles/s11-03-citation-and-verifiable-attribution.md) | An identifier is not a citation. A verifiable one resolves, locates the line, and is traceable — and line-level citation is an *ingestion* decision, enabled long before generation by how the sources were stored. Dangling citations are checked in code, never trusted to the model, because they look exactly like real ones. Structure is the source of truth; inline versus footnote is presentation. The AI service emits `document_id`, never URLs. |

| 4 | [Hallucination detection and mitigation](articles/s11-04-hallucination-detection-and-mitigation.md) | Referential integrity proves the source exists, not that it says what the claim says. Three kinds of hallucination, each caught by a different technique, layered cheap-first because what an `if` can reject should not spend a model call. The judge is itself a model and has a floor no more LLM can raise. Abstention is a feature — but over-abstaining is not prudence, it is declining to do the work. |

| 5 | [Reindexing and embedding versioning](articles/s11-05-reindexing-and-embedding-versioning.md) | The index rots two ways and neither raises an exception: the document changed and the vector did not, or two models' vectors share a table and cosine between them is a plausible number meaning nothing. Every vector records how it was made and queries never cross versions. Incremental is the daily tool and a trap outside its version; a model change is blue/green, verified before promotion. |

| 6 | [Quality evaluation with RAGAS](articles/s11-06-quality-evaluation-with-ragas.md) | Guardrails answer "is this estimate trustworthy?"; nothing answers "is the system getting better?". Four metrics split two/two between retrieval and generation, which is what makes them a diagnosis rather than a score. The golden set is the ceiling: with no case whose right answer is "not enough data", you are rewarding the system for always answering. Offline gates regressions; production has no reference, so only two metrics survive there. |

Articles 1-4 chain directly: 1 ends at two clean sources that disagree, 2 starts
there and ends at "where exactly does this figure come from", 3 answers that and
ends at a citation pointing at a real source that does not back it, 4 hunts
exactly that. Article 5 steps outside the request path to the index underneath
it. **4 and 5 close on the same unresolved thing** from different directions —
per-request guardrails and a healthy index both prevent damage, neither measures
whether a change was an improvement — and article 6 is what they were both
pointing at. 6 closes the part and hands off to session 12: coordinating many
specialised answers instead of generating one big one.

Against the code: 3 and 6 are implemented, 4's warnings land on code that
already exists, **5's central recommendation is the one the codebase has not
taken** (no `embedding_version`, no `source_hash`), and 1 and 2 are not written
yet. 6's gap is its golden set — five queries, none of which tests abstention.

This part revisits part 9's augmentation article rather than replacing it.
**s09-04** assembles the retrieved chunks; **s11-01** distils them first, so the
thing being assembled is structured evidence rather than raw text. Where they
overlap — chunk order, truncation — s11-01 goes further, and in one place gets
it wrong: its edge-loading function inverts its own intent, and s09-04's
`reorder_u_pattern` is the correct implementation. Flagged in place.

## Conventions

**Documents are stored in English.** Several arrive as Spanish originals and are
translated on the way in, with `lang: en` and `translated_from: es` recorded in
the frontmatter. Code blocks, identifiers and library names are kept exactly as
written — translating an identifier would break the correspondence with the code
being described.

One file per article under `articles/`, named `s{session}-{article}-{slug}.md`,
so reading order and grouping are both visible from the directory listing:
`s07-01-embeddings-semantic-geometry.md`. The session prefix is not decoration —
each session restarts its article numbering, so several articles share the
number 1 and a flat scheme would collide on every one of them.

Every file opens with YAML frontmatter:

```yaml
---
title: Multi-format extraction pipeline
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 6
series_part: 3
scope: evergreen
source: <publication, author, URL, or "user notes">
added: 2026-08-11
summary: >
  One or two sentences on what this document argues.
keywords: [ingestion, chunking, metadata]
---
```

Blockquoted markers are editorial rather than the author's; everything outside
them is the article as argued. There are four, all greppable by prefix:

| Marker | Carries |
|---|---|
| `> *(Figure in the original: ...)*` | Where an image stood. Later articles describe what the figure showed, because the rendered image was supplied with the text; earlier ones only name the missing file. |
| `> *(Editor's note: ...)*` | A cross-reference, or a tension between two articles. |
| `> *(Editor's note — domain transfer: ...)*` | A claim that holds only because of what the estimator's corpus is. See [reading it for a different domain](#reading-it-for-a-different-domain). |
| `> *(Editor's note — checked against the code: ...)*` | A difference between the article and the reference implementation. See [the article-to-module map](#article-to-module-map). |

### Reading the code in these articles

**Article bodies are reproduced as argued, and that includes their defects.**
Several articles print functions that do not do what their own prose and figures
say — an edge-loading reorder that puts the strongest evidence in the middle
(s11-01), a contradiction test that fires on the weak outlier it claims to ignore
(s11-02), a status value that is declared and never returned (s11-04). These are
flagged in an editor's note directly beneath the code, never silently rewritten,
because the argument is the author's and the correction is not.

The practical consequence: **do not lift a function out of an article without
reading the note under it.** Where a defect has a correct counterpart elsewhere
in the handbook, the note names it.

One exception to reproducing verbatim: a factual error about the outside world
gets corrected in the body, with an editor's note recording what it said before.
So far that is one — the pgvector version in s09-03.

The code-checked notes and the module map were verified against
`lidr/ai-engineering` at branch `session_11`, commit `3292744`. **They are a
snapshot and will drift.** Re-run the check when the implementation moves: grep
the repo for the identifiers an article names, and update the map rather than the
article.

**`source` is provenance**, and it does the real work later: when two articles
disagree, it is the only thing that resolves it. An entry with no traceable
source is not a fact.

**`scope` is shelf life**, and it takes one of two values. `evergreen` is the
default: the argument does not depend on facts that expire, so it is as true in
five years as today. `time-sensitive` marks an article resting on the state of
the world when it was written — a model comparison, a market survey, a price —
where the reasoning survives but the specifics will not. Two articles carry it:
[s07-02](articles/s07-02-embedding-model-selection.md) and
[s08-02](articles/s08-02-vector-database-market-2026.md). Read those against
`added` and check the numbers before acting on them.

Use `##` headings for genuinely separate ideas: **headings are chunk boundaries**,
so a heading is a claim about what belongs together. Prefer several short
sections over one long one — a chunk that argues two things retrieves well for
neither. These documents are written to be ingested by the kind of system they
describe.

## Provenance

The series is by **Antonio Perez** (`series: servicio-ia`), read and translated
from the Spanish originals. Article bodies are reproduced as argued; the
worked examples inside them (construction budgets, meeting transcripts) are the
author's own.

The articles were written for a taught course and keep its vocabulary. It is
not explained anywhere in them, so:

| Term | Means |
|---|---|
| **the programme** | The course these articles are the reading for. |
| **Project 2** | The IT project estimator — the system described in [the section above](#the-system-the-articles-are-about). |
| **Module 2 / Module 3** | Blocks of the course. Module 2 built the CAG; Module 3 is the RAG arc, parts 6-8 here. |
| **Session NN**, **SNN** | One session of the course, one part here. **S15 is referenced but not in this repo** — several articles defer production concerns (key rotation, distributed rate limiting, observability tooling) to it. |
| **pre-session exercise** | Homework preceding a session. Its results are sometimes assumed by an article's opening. |
| **the live session** | The taught class. Sections titled "connection with the live session" describe what was demonstrated there and are the least useful part of an article read standalone. |

These articles were originally collected inside a private application repository,
where each carried an `applies_to` frontmatter field mapping the advice onto that
specific codebase. Those project-specific mappings were removed when the material
moved here — what remains is the general argument.

## Adding an article

1. Translate to English if needed; keep code and identifiers verbatim.
2. Save as `articles/s{session}-{article}-{slug}.md` with the frontmatter above.
3. Add a row to the relevant part's table, in the **what it argues** style — a
   claim, not a topic list. "Covers chunking" is useless; "recursive 400-512 is
   hard to beat" is the reason to open the file.
4. If it contradicts an article already here, say so in the row and let `source`
   settle it. When it is the same author revising his own earlier position as the
   estimator grows — the common case — `source` cannot settle it: flag it both
   ways instead. A clause on the earlier article's row, and an
   `> *(Editor's note: ...)*` blockquote in the earlier article's body at the
   paragraph being revised, linking forward.
5. Sessions arrive in order and each one restarts its article numbering, so the
   next file is usually the next `series_part` of the current session. Check the
   figure filenames in the source text — they carry the article number
   (`articulo-04-figura-01-...`).
6. Keep the estimator in view. An article that reads as generic RAG advice has
   usually lost the thread: the author argues from this system's constraints —
   budgets estimated line item by line item, transcripts as the raw input, one
   Postgres — and the row should say what it changes *for that system*.
7. Then read it once more against the domains you are actually building for, and
   add a `> *(Editor's note — domain transfer: ...)*` wherever a conclusion holds
   only because of what the estimator's corpus is. Say what survives the move,
   not merely that the claim is narrow.
8. Grep the reference implementation for every identifier and path the article
   names, and update the article-to-module map. Renames go in the table only;
   add a `checked against the code` note when the article asserts something
   about the codebase that is not true, or when the code pins down a constant
   the article leaves vague.
9. Run the article's code in your head, or actually run it. Three articles so
   far print a function that contradicts its own docstring or figure. Flag it
   beneath the code and name the correct counterpart if the handbook has one;
   do not rewrite the author's body.
