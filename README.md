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

## Reading order

The articles form an arc through the pipeline, in the order the pipeline runs.

| Part | Covers |
|---|---|
| [6 — Ingest](#part-6--the-ingest-side) | Audit the sources, extract them, clean and validate, make them safe to embed |
| [7 — Embeddings and chunking](#part-7--embeddings-and-chunking) | What an embedding is, which model to pick, how to cut the text |
| [8 — Vector databases](#part-8--vector-databases) | Whether you need one, which one, what the index actually builds, and its ceiling |
| [9 — The RAG flow](#part-9--the-rag-flow) | The four canonical stages: reformulation, retrieval, augmentation, generation |
| [10 — Advanced retrieval](#part-10--advanced-retrieval) | Reranking, hybrid search, and how to measure whether either helped |

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
| 4 | [Schema design and semantic search](articles/s08-04-schema-design-and-semantic-search.md) | The applied piece: two-table schema with the typed-column/JSONB split, the three distance operators and why cosine beats the faster inner product, the operator-class trap with its `CREATE INDEX`, `EXPLAIN ANALYZE` as the reflex, and `hnsw.iterative_scan` for selective filters. |
| 5 | [From prototype to production: tuning, monitoring, pgvector's ceiling](articles/s08-05-from-prototype-to-production.md) | Memory sizing as 80% of performance, index-build parameters (`maintenance_work_mem`, parallel workers, `CONCURRENTLY`), `halfvec` quantisation, monitoring via `pg_stat_user_indexes`, the VACUUM/REINDEX cycle, and three measurable migration signals. |

## Part 9 — the RAG flow

| # | Document | What it argues |
|---|---|---|
| 1 | [From static CAG to the RAG flow](articles/s09-01-from-cag-to-rag-four-stages.md) | The four canonical stages, five operational differences from CAG, an honest account of when CAG still wins (Chan et al. 2024), and the rule that retrieval quality is the ceiling on the whole system. |
| 2 | [Query reformulation](articles/s09-02-query-reformulation.md) | Why embedding a raw transcript fails (length dissolves signal, noise drowns keywords, anaphora encodes nothing), the five reformulation families, and the case for structured extraction on debuggability and downstream filter utility. |
| 3 | [Retrieval that is not only cosine](articles/s09-03-retrieval-topk-threshold-filters.md) | The two failure modes of a top-K-only retriever, threshold set from the empirical distance distribution, soft-fail with `low_confidence`, pre/post/in-query filtering chosen by selectivity, four anti-patterns, and precision over recall when the answer has economic consequences. |
| 4 | [Augmentation: assembling context](articles/s09-04-augmentation-assembling-context.md) | Why `"\n\n".join` yields invented citations and silently ignored context. XML `<source>` delimiters with metadata, lost-in-the-middle and chunk order, whole-chunk truncation counting the wrapper, a prompt with grounding and a licence to refuse, and post-generation citation validation. |

## Part 10 — advanced retrieval

| # | Document | What it argues |
|---|---|---|
| 1 | [Reranking: when vector top-k is not enough](articles/s10-01-reranking.md) | Bi-encoders find well and order badly because compression averages a document and never lets query and document read each other; cross-encoders fix the order and cannot scale; recall-then-rerank buys both. Plus an explicit list of when reranking is the wrong tool. |
| 2 | [Artisanal relevance measurement](articles/s10-02-artisanal-relevance-measurement.md) | Turning "it seems better" into a number: golden sets of 5-20 annotated queries, precision@k at the k you actually use, latency measured warm and by median, and a gain-versus-cost table — including the treacherous quadrant where a small gain at a small cost still is not worth the complexity toll. |
| 3 | [Hybrid search](articles/s10-03-hybrid-search.md) | Semantic search dilutes proper nouns; lexical search nails them and cannot read paraphrase. Both live in PostgreSQL (pgvector + `tsvector`/GIN), and Reciprocal Rank Fusion joins them on positions alone, sidestepping two incomparable score scales. |

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
   settle it.
