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
roughly 30 hand-picked budgets pasted into the system prompt, closed in session
05, before this handbook starts. **The jump to RAG happens across sessions 6 and
7**, and it is a crossing rather than a switch: session 6 goes back to the
sources and builds a real ingest side, session 7 turns that corpus into
embeddings and chunks. By session 8 the estimator has somewhere to put them, and
only in session 9 is the RAG flow itself named and taken apart stage by stage.
The residual CAG layer survives the crossing — the decision was to add
retrieval, not to delete the static context.

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
than sloppiness: the project was **refactored into `cag/` and `rag/` folders**
partway through, so an article's paths date it. `src/estimator/retrieval/` in
s09-05 and `app/generation/rag/retrieval/` in part 10 are the same component
before and after that split. Read the last segment — `retrieval/`, `ingest/`,
`generation/` — as the stable part and the prefix as a timestamp.

**Retrieval belongs to RAG**, which sets the layout to build for a new project
from this handbook: start with `cag/`, since that is where a system that has not
yet earned retrieval lives; create `rag/` at the point you actually adopt
retrieval, and put the pipelines and everything retrieval-related inside it —
ingestion, chunking, the retriever, reranking, fusion, routing. Keeping the two
side by side rather than replacing one with the other is what the estimator did,
and it is why its **residual CAG layer** survived the crossing.

The CAG chapter itself is not in this repo — the handbook opens after that
decision has already been reversed.

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
| [6 — Ingest](#part-6--the-ingest-side) | Audit the sources, extract them, clean and validate, make them safe to embed |
| [7 — Embeddings and chunking](#part-7--embeddings-and-chunking) | What an embedding is, which model to pick, how to cut the text |
| [8 — Vector databases](#part-8--vector-databases) | Whether you need one, which one, what the index actually builds, and its ceiling |
| [9 — The RAG flow](#part-9--the-rag-flow) | The four canonical stages — reformulation, retrieval, augmentation, generation — and serving them as an operable service |
| [10 — Advanced retrieval](#part-10--advanced-retrieval) | Reranking, hybrid search, query rewriting, and how to measure whether any of them helped |

If you read only three: **s07-03** (chunking), **s09-03** (retrieval is not only
cosine), **s10-02** (measure the cost, not only the gain).

## Part 6 — the ingest side

| # | Document | What it argues |
|---|---|---|
| 2 | [Enterprise data audit and inventory](articles/s06-02-enterprise-data-audit-inventory.md) | Audit before vectorising. A factual source census, four non-averaging quality dimensions, "context erosion" as the reason lineage is a *condition of usefulness* in RAG, and a versioned typed `data_catalog.yaml` the pipeline reads at runtime. |
| 3 | [Multi-format extraction pipeline](articles/s06-03-multi-format-extraction-pipeline.md) | A modular `ingest/` (loaders → parsers → normalizers) converging on a canonical `Document`, instead of delegating everything to `unstructured.partition()`. Per-format strategy, three-source metadata propagation, and why `hi_res` PDF parsing everywhere is a waste. |
| 4 | [Data cleaning, normalization and validation](articles/s06-04-data-cleaning-normalization-validation.md) | The *form* contract (Pydantic) is not the *content* contract. Four families of dirty data, cleaning as one auditable layer between parser and normalizer, Pandera as a versioned data contract, and an explicit repair / quarantine / discard policy. |
| 5 | [PII, anonymization and GDPR](articles/s06-05-pii-anonymization-gdpr.md) | Access control cannot protect a vector corpus. Three leakage modes, four GDPR concepts that drive technical decisions, and consistent reversible pseudonymization (Presidio + Faker + mapping table) over generic redaction, which costs 15-25% of retrieval quality. |

Articles 2-5 form a complete arc through the ingest side: audit the sources,
extract them, clean and validate what came out, then make it safe to embed.
Article 1 of this part is missing, and nothing in 2-5 depends on it — article 2
opens by treating the RAG-versus-CAG decision as already made.

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
| 4 | [Augmentation: assembling context](articles/s09-04-augmentation-assembling-context.md) | Why `"\n\n".join` yields invented citations and silently ignored context. XML `<source>` delimiters with metadata, lost-in-the-middle and chunk order, whole-chunk truncation counting the wrapper, a prompt with grounding and a licence to refuse, and post-generation citation validation. |
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

## Conventions

**Documents are stored in English.** Several arrive as Spanish originals and are
translated on the way in, with `lang: en` and `translated_from: es` recorded in
the frontmatter. Code blocks, identifiers and library names are kept exactly as
written — translating an identifier would break the correspondence with the code
being described.

One file per article under `articles/`, named `s{session}-{article}-{slug}.md`,
so reading order and grouping are both visible from the directory listing:
`s07-01-embeddings-semantic-geometry.md`. The session prefix is not decoration —
each session restarts its article numbering, so a flat scheme would have put
session 7's article 1 exactly where session 6's *missing* article 1 belongs.

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

Two markers are editorial rather than the author's: `> *(Figure in the
original: ...)*` stands where an image was, and `> *(Editor's note: ...)*` carries
a cross-reference or a flagged tension between articles. Everything outside those
blockquotes is the article as argued.

**`source` is provenance**, and it does the real work later: when two articles
disagree, it is the only thing that resolves it. An entry with no traceable
source is not a fact.

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
