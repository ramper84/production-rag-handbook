---
title: Schema design and semantic search in pgvector
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 8
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 32 min
added: 2026-08-14
summary: >
  The applied piece: a two-table documents/chunks schema with the typed-column
  versus JSONB split, the three distance operators and why cosine is chosen over
  the marginally faster inner product, the operator/operator-class misalignment
  that silently disables an index, EXPLAIN ANALYZE as the diagnostic reflex, and
  hnsw.iterative_scan for the selective-filter case.
keywords: [schema design, documents, chunks, JSONB, GIN index, cosine, L2,
           inner product, operator class, EXPLAIN ANALYZE, hnsw.iterative_scan,
           ON DELETE CASCADE]
---

# Schema design and semantic search in pgvector

*Antonio Perez* · 🔴 32 min

The three previous articles have built the theory: why vector databases exist, what is on the market and why we choose pgvector, and how HNSW and IVFFlat indexes work inside. What is missing is the most important thing for the pre-session exercise to go well: the concrete model of the tables that will sustain the whole system, the SQL operators used to do semantic search, and the silent antipattern that destroys performance without raising errors.

This article is the session's applied piece. We land the programme project's schema, the three distance metrics with the particular case of OpenAI's normalised embeddings, and the operational rule that closes half the session's value: **the query's operator and the index's operator class have to be aligned, always, without exception.** When they are not, Postgres emits no error: it simply disables the index silently, falls back to a sequential scan, and latency multiplies by a thousand. It is the most expensive and easiest-to-commit operational bug in pgvector, and it is worth leaving this article knowing it.

## The relational model: two tables, not one

When a student first faces the modelling of a RAG system, the natural reflex is to create a single `chunks` table containing everything needed: the chunk's text, its embedding, and the metadata of the document it belongs to. It works in terms of the final product, but it introduces duplication and loses guarantees a well-designed relational model gives for free. If a budget produces 17 chunks, you are going to repeat the document's metadata (type, sector, date, client) 17 times. If you update the document's metadata, you have to touch 17 rows coherently. If you delete the document, you have to remember to delete its chunks. When you reach several hundred documents with thousands of chunks, that duplication is a real operational problem.

The correct model for the project is exactly the one the pre-session exercise asks you to build: **two tables with a one-to-many relation.** The `documents` table represents each historical budget once, with its metadata; the `chunks` table contains the fragments derived from each document's structural chunking, each with its embedding and its own block of flexible metadata. Referential integrity between the two, with `ON DELETE CASCADE`, guarantees that deleting a document automatically deletes all its chunks with no need for application logic.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id            BIGSERIAL PRIMARY KEY,
    source_path   TEXT NOT NULL,
    document_type VARCHAR(50) NOT NULL,
    ingested_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata      JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE chunks (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT NOT NULL
                  REFERENCES documents(id) ON DELETE CASCADE,
    chunk_type    VARCHAR(50) NOT NULL,
    content       TEXT NOT NULL,
    embedding     vector(1536),
    metadata      JSONB NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

> *(Figure in the original: `articulo-04-figura-01-schema-relacional.jpg` — image not included in this repo.)*

Five decisions in this schema deserve an explicit note because you will have to defend them (in the project's README, and possibly in any RAG system you design outside the programme).

**The separation between typed columns and JSONB.** The stable metadata you know you will query structurally — document type, chunk type, dates — goes in typed columns. The metadata the chunker may enrich with arbitrary fields (sector inferred from content, technologies mentioned, scope, tags) goes in JSONB. This separation gives you the best of both worlds: queries on typed columns are fast and exploit conventional B-tree indexes, and queries on JSONB are flexible and exploit GIN indexes. The opposite error — putting all metadata into a single JSONB column "for flexibility" — works but loses efficiency and legibility. The inverse error — adding a new column to the schema every time the chunker wants to persist one more field — turns every pipeline change into a migration. The proposed division avoids both extremes.

**The GIN index over `metadata`.** Without it, a query like `WHERE metadata->>'sector' = 'fintech'` does a sequential scan over the whole table. With it, Postgres can use the index to drastically reduce the rows examined. The GIN index's maintenance cost is low for volumes in the project's range, and the benefit on queries with structured filters is enormous. We will see it live when we compare execution plans.

**`vector(1536)`.** The dimensionality of `text-embedding-3-small`, the project's model. It is hardcoded in the schema and that is deliberate: changing it implies re-embedding the whole corpus, a costly operation nobody should trigger accidentally. If in future you migrate to `text-embedding-3-large` (3072 dimensions), the migration goes through **a new column, not modifying the existing one.**

**`embedding` nullable.** Allows inserting a chunk in a transaction and filling the embedding afterwards if the computation failed. In the exercise we do not exploit it that way (we ingest chunk+embedding atomically), but it leaves the door open to asynchronous ingestion patterns we will see later in the programme.

**There is no vector index yet.** The pre-session exercise's schema deliberately omits the HNSW index. The live session starts by measuring the `/search` endpoint's latency without an index, creating the index, and measuring again. It is the only way to land empirically the order of magnitude the index contributes.

## The three distance metrics: cosine, L2 and inner product

pgvector exposes three distance operators between vectors, each with its own geometric semantics and its own operator class for index creation. It is worth being clear about all three, because choosing between them is not an aesthetic preference: it depends on the embedding model you use and the type of problem you are solving.

> *(Figures in the original: a comparison table image and `articulo-04-figura-02-metricas-distancia.jpg` — not included in this repo.)*

**Cosine distance (`<=>`).** Measures the angle between two vectors ignoring magnitude. Two vectors pointing in the same direction have cosine distance 0, regardless of whether one is 1 unit long and the other 1000. It is the standard metric for text embeddings from modern models (OpenAI, Cohere, Voyage, the main Hugging Face ones), because those models are trained so that semantic meaning is encoded in the vector's direction, not its length. For the programme's project, with `text-embedding-3-small` embeddings, it is the reasonable default.

**L2 distance (`<->`).** Measures Euclidean distance, the "straight-line distance" between the two vectors' endpoints. It is sensitive to magnitude: two vectors pointing in the same direction but with different lengths have L2 distance greater than zero. It is the natural metric for data where magnitude carries information — spatial coordinates, physical sensor data, image embeddings where pixel intensity matters. For text embeddings it is rarely the right choice.

**Negative inner product (`<#>`).** Measures the dot product, sensitive to both angle and magnitude. pgvector returns the negated value because Postgres only supports ascending ordering in index operators, and we want `ORDER BY ... ASC` to return the most similar first. For normalised vectors (length = 1), inner product is mathematically equivalent to cosine in ordering terms, but computationally more efficient because it saves the step of dividing by the norms.

For `text-embedding-3-small` all three operators work, but only one is the optimal choice. The key is an under-emphasised property of the model: **OpenAI normalises its embeddings.** Every vector coming out of the API has Euclidean norm equal to 1. That has an important practical consequence: for normalised vectors, cosine distance and inner product produce **exactly the same result ordering**. The choice between `<=>` and `<#>` is indifferent in terms of which chunks are retrieved; the only thing that changes is computational efficiency, where `<#>` wins slightly because it saves the norm-computation step it does not actually need (the norms are already 1).

Despite that efficiency, in the programme's project we use `<=>` and `vector_cosine_ops`. There are two reasons. The first is conventional: the RAG literature and most public tutorials use cosine, and learning with the dominant convention reduces friction when consulting external sources. The second is practical: if in future some team migrates the system to an embedding model that does **not** normalise (a local Sentence Transformer, for example), the SQL query will keep working with no changes and no surprises. Using `<#>` "for efficiency" today obliges you to remember forever that it is only safe while the embeddings are normalised.

## The antipattern that destroys performance without raising errors

Here comes the operational piece that is the heart of the article, the silent bug that costs most in pgvector. When you create an HNSW index with `vector_cosine_ops`, that index only accelerates queries using the `<=>` operator. If the query uses `<->`, the index does not activate. **Postgres emits no error or warning.** The query works and returns results. The only thing that changes is that internally Postgres has fallen back to a sequential scan, recomputing L2 distance against every row of the table. The result is correct in terms of ordering by L2 distance, but the latency that on a million-row table should be a few milliseconds becomes tens of seconds.

The operational rule is strict and simple:

> The query's operator (`<=>`, `<->`, `<#>`) has to match the index's operator class (`vector_cosine_ops`, `vector_l2_ops`, `vector_ip_ops`).

> *(Figure in the original: `articulo-04-figura-03-antipatron.jpg` — image not included in this repo.)*

Any misalignment between the two breaks the index's use without emitting warnings. It is one of the most expensive and easiest-to-commit bugs in pgvector. It happens when a developer copies code from an external source that used another metric, when the team changes embedding model without updating the queries, or simply when someone writes `<->` out of inertia because it is the most intuitive operator (it looks like a "distance" arrow).

In the programme's project, the index is created with `vector_cosine_ops` and all queries use `<=>`. This means that when we build the `CREATE INDEX` live, we will do it exactly like this:

```sql
CREATE INDEX chunks_embedding_idx
ON chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 128);
```

And when we run the search query, we will do it exactly like this:

```sql
SELECT id, document_id, chunk_type, content, metadata,
       embedding <=> :query_vector AS distance
FROM chunks
ORDER BY embedding <=> :query_vector
LIMIT :k;
```

The two `<=>` in the SELECT (one to return the distance as a column, another to order) are obligatory and are the same operator. The distance is computed once per row thanks to the query planner's optimisation; conceptually, what matters is that the operator in the `ORDER BY` is exactly the same as the index's operator class.

## How to verify it: EXPLAIN ANALYZE

The way to confirm the index is being used is `EXPLAIN ANALYZE` over the query. If the output shows `Index Scan using chunks_embedding_idx`, the index is active. If it shows `Seq Scan on chunks`, it is not, and there is a problem to debug. The three most common causes in order of frequency are:

1. **Operator / operator-class misalignment.** The most frequent. Given the rule above, it almost never happens if you keep the discipline of copying the operator from the index's operator class.
2. **Very selective relational filters.** If the `WHERE` reduces the result to a handful of rows, Postgres may decide it is cheaper to do a sequential scan over the already-filtered rows than to go through the vector index. This is resolved with `hnsw.iterative_scan`, which we see live.
3. **Stale statistics.** After loading many new vectors, the table's statistics can become outdated and the planner takes suboptimal decisions. A periodic `ANALYZE chunks` keeps the planner informed.

The correct operational reflex when a semantic query works but is inexplicably slow is: **`EXPLAIN ANALYZE` first**, before assuming anything else. It is the most useful tool pgvector inherits free from PostgreSQL, and it is the difference between a team that produces performant RAG systems and one that fights mysterious latencies for weeks.

## The complete query: three layers in one atomic statement

So far we have talked only about pure vector search, without filters. In practice, almost nobody wants "the k chunks nearest the query" and nothing else; what they want is something like "the k chunks nearest the query, from the fintech sector, ingested in the last 24 months, from budgets between 50k and 200k". Those filters live in typed columns (`metadata`, `ingested_at`) and in relations (`document_id` pointing at `documents`). In pgvector, all of that fits in a single atomic SQL query:

```sql
SELECT c.id, c.content, c.chunk_type,
       c.embedding <=> :query_vector AS distance,
       d.metadata->>'sector' AS sector,
       d.ingested_at
FROM chunks c
JOIN documents d ON d.id = c.document_id
WHERE d.metadata->>'sector' = 'fintech'
  AND d.ingested_at > NOW() - INTERVAL '24 months'
  AND (d.metadata->>'budget')::numeric BETWEEN 50000 AND 200000
ORDER BY c.embedding <=> :query_vector
LIMIT 5;
```

This query mixes semantic search with relational filters, filters over JSONB, joins between tables, and projection of arbitrary fields — all with ACID guarantees, all in a single atomic operation. It is exactly the type of query a RAG system over enterprise data needs and that Pinecone, Qdrant or Weaviate cannot do without coordination with another system. It is the main reason pgvector is the right choice for the programme's project, and **the property you will miss most the day you migrate to a dedicated vector database for scale reasons.**

There is an important nuance the live session addresses in depth: when the `WHERE` filters are very selective (reducing the result to a small fraction of the rows), the HNSW index can struggle because it is optimised for finding the k neighbours in the complete set, not in a pre-filtered subset. **pgvector 0.8 introduced `hnsw.iterative_scan`** precisely to resolve this case: when the engine detects it has returned fewer than k results after applying the filter, it iteratively expands the search until the `LIMIT` is satisfied. It is a relatively recent and very useful feature; we will activate it and measure its impact live.
