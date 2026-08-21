---
title: "Hands-on: migration to pgvector + search endpoint"
author: Antonio Perez
lang: en
translated_from: es
doc_type: hands-on-exercise
series: servicio-ia
session: 8
series_part: 0
scope: evergreen
source: user-supplied exercise brief (pre-session practical)
added: 2026-08-13
summary: >
  Practical brief for persisting a chunk+embedding pipeline into Postgres +
  pgvector: docker-compose service, Alembic migrations with the vector type
  registered, a documents/chunks schema with JSONB metadata, atomic ingestion in
  one transaction, and a cosine-distance top-k search endpoint — deliberately
  with no vector index, so sequential scan is the baseline the live session
  measures against.
keywords: [pgvector, alembic, sqlalchemy, asyncpg, cosine_distance, HNSW,
           IVFFlat, operator class, JSONB, GIN index, atomic ingestion, scoping]
---

# Hands-on: migration to pgvector + search endpoint

*Antonio Perez*

⏱ Deadline: Tuesday 12 July, end of day.

## Objective

Persist the pipeline built in Session 07 into PostgreSQL + pgvector and expose a working semantic-search endpoint over the historical budgets. On finishing, the AI service must:

- Bring up a Postgres with pgvector as a declared project dependency.
- Have its own relational schema (`documents` and `chunks` tables) managed with Alembic migrations.
- Persist each ingested budget as a `document` with its corresponding `chunks` (each with its embedding) **in a single transaction**.
- Resolve a semantic query via SQL returning the k nearest chunks by cosine distance.

## What is NOT done in this exercise

These four topics are deliberately addressed live. If you get ahead of them, you lose half the value of the live session:

- **Vector indexes (HNSW, IVFFlat).** The sequential scan is the baseline against which we will measure the index's impact live.
- **Metadata filters** (`WHERE chunk_type = 'budget_component'`, `WHERE metadata->>'sector' = 'fintech'`). We explore them live, comparing execution plans with and without selective filters.
- **Hybrid search** (full-text search + vector). Built live on top of your code.
- **Parameter tuning** (`shared_buffers`, `maintenance_work_mem`, `ef_search`). Defaults throughout the exercise.

Resist the temptation to "go further": **scoping discipline is part of the exercise.**

## Stack and dependencies

Add to the AI service's `pyproject.toml`:

```toml
sqlalchemy>=2.0
asyncpg>=0.29
pgvector>=0.3
alembic>=1.13
```

`asyncpg` is the async driver officially recommended by SQLAlchemy 2.0 for Postgres. `pgvector` is the Python package that registers the `vector` type in SQLAlchemy and exposes the distance operators (`l2_distance`, `cosine_distance`, `max_inner_product`) as methods callable from the ORM.

## Step 1 — Postgres with pgvector in docker-compose

Add to `docker-compose.yml` a `postgres` service with pgvector's official image. This image is Postgres 16 with the `vector` extension precompiled and installable with a single `CREATE EXTENSION`.

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: estimator
      POSTGRES_USER: estimator
      POSTGRES_PASSWORD: estimator
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U estimator -d estimator"]
      interval: 5s
      timeout: 5s
      retries: 10

  ai_service:
    # ... existing configuration ...
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql+asyncpg://estimator:estimator@postgres:5432/estimator

volumes:
  postgres_data:
```

Verify before continuing:

```bash
docker compose up postgres
docker compose exec postgres psql -U estimator -d estimator -c "SELECT version();"
```

If this does not work, do not move on to Step 2.

## Step 2 — Configure Alembic in the AI service

Initialise Alembic with the async template:

```bash
docker compose run --rm ai_service alembic init -t async alembic
```

Configure `alembic.ini` and `alembic/env.py` so they take the connection URL from the `DATABASE_URL` environment variable and so they recognise pgvector's `vector` type. Without this, `alembic check` does not correctly detect `vector` columns and produces inconsistent migrations.

In `env.py`, inside `do_run_migrations`:

```python
import pgvector.sqlalchemy

def do_run_migrations(connection):
    connection.dialect.ischema_names["vector"] = pgvector.sqlalchemy.Vector
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
    )
    with context.begin_transaction():
        context.run_migrations()
```

## Step 3 — Database schema

Create the first migration with the `vector` extension plus two tables. The column names are the ones the following steps expect; if you change them, adjust all the exercise's code accordingly.

```python
# alembic/versions/0001_initial_schema.py
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql
from pgvector.sqlalchemy import Vector

def upgrade():
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")

    op.create_table(
        "documents",
        sa.Column("id", sa.BigInteger, primary_key=True),
        sa.Column("source_path", sa.Text, nullable=False),
        sa.Column("document_type", sa.String(50), nullable=False),
        sa.Column("ingested_at", sa.DateTime(timezone=True),
                  server_default=sa.func.now(), nullable=False),
        sa.Column("metadata", postgresql.JSONB,
                  server_default="{}", nullable=False),
    )
    op.create_index("ix_documents_source_path", "documents", ["source_path"])

    op.create_table(
        "chunks",
        sa.Column("id", sa.BigInteger, primary_key=True),
        sa.Column("document_id", sa.BigInteger,
                  sa.ForeignKey("documents.id", ondelete="CASCADE"),
                  nullable=False),
        sa.Column("chunk_type", sa.String(50), nullable=False),
        sa.Column("content", sa.Text, nullable=False),
        sa.Column("embedding", Vector(1536), nullable=True),
        sa.Column("metadata", postgresql.JSONB,
                  server_default="{}", nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True),
                  server_default=sa.func.now(), nullable=False),
    )
    op.create_index("ix_chunks_document_id", "chunks", ["document_id"])
    op.create_index("ix_chunks_chunk_type", "chunks", ["chunk_type"])
    op.create_index("ix_chunks_metadata_gin", "chunks", ["metadata"],
                    postgresql_using="gin")
```

Run the migration:

```bash
docker compose run --rm ai_service alembic upgrade head
```

**Schema decisions you will have to justify in the README** (you will defend them live if you are asked):

- **Two tables instead of one.** A budget produces N chunks. A single table with the document's metadata duplicated on each row loses referential integrity and duplicates data. With two tables and `ON DELETE CASCADE`, deleting a budget automatically deletes all its chunks.
- **`metadata` JSONB in both tables.** Stable metadata (document type, chunk type, dates) in typed columns; variable metadata, or metadata the chunker may enrich (tags, scope, technologies mentioned) in JSONB. The GIN index over the JSONB allows querying by arbitrary keys without migrating the schema each time.
- **`vector(1536)`.** Dimensionality of `text-embedding-3-small`. It is hardcoded because changing it implies re-embedding the whole corpus, so it is not a decision that will change dynamically.
- **`embedding` nullable.** Allows inserting a chunk in a transaction and filling the embedding afterwards if the computation failed. In this exercise we will not use it that way (we ingest chunk+embedding atomically), but it leaves the door open to the asynchronous ingestion we will see in later sessions.
- **There is no vector index.** Deliberate. The live session adds it.

## Step 4 — Refactor POST /embeddings/ingest

The endpoint goes from returning chunks+vectors in the response to persisting them in a transaction and returning only the ingestion's identifiers and metrics.

Final endpoint contract:

**Request:**

```json
{
  "source_path": "data/budgets/budget_2024_q1_fintech.json",
  "document_type": "historical_budget",
  "content": { /* full budget JSON, exactly as it comes from the chunker */ }
}
```

**Response (200 OK):**

```json
{
  "document_id": 42,
  "chunks_created": 17,
  "embedding_dimension": 1536,
  "ingestion_time_ms": 1240
}
```

**Response (409 Conflict)** if a document with that `source_path` already exists:

```json
{
  "detail": "Document already ingested",
  "document_id": 42
}
```

**Implementation.** Inside a single async SQLAlchemy session:

1. Verify no document with that `source_path` already exists.
2. Create the row in `documents`.
3. Run the structural chunker over the JSON.
4. Call the embedder **in batches** (not chunk by chunk — a single `embeddings.create` with an array of inputs).
5. Create all the rows in `chunks` with `add_all`.
6. Commit.

The single transaction guarantees that a failure in the embedder does not leave orphan `documents` without chunks.

## Step 5 — New endpoint POST /search

Contract:

**Request:**

```json
{
  "query": "REST API with OAuth authentication for fintech sector",
  "k": 5
}
```

**Response:**

```json
{
  "query": "REST API with OAuth authentication for fintech sector",
  "k": 5,
  "search_time_ms": 87,
  "results": [
    {
      "chunk_id": 156,
      "document_id": 12,
      "chunk_type": "budget_component",
      "content": "Backend service implementation with JWT-based authentication...",
      "distance": 0.231,
      "metadata": { "scope": "backend", "technologies": ["python", "fastapi"] }
    }
  ]
}
```

**Query implementation.** Embed the query with the same model used at ingestion (`text-embedding-3-small`) and execute via SQLAlchemy:

```python
from sqlalchemy import select

stmt = (
    select(
        Chunk.id,
        Chunk.document_id,
        Chunk.chunk_type,
        Chunk.content,
        Chunk.metadata,
        Chunk.embedding.cosine_distance(query_vector).label("distance"),
    )
    .order_by(Chunk.embedding.cosine_distance(query_vector))
    .limit(k)
)
result = await session.execute(stmt)
```

**Why `cosine_distance` (the `<=>` operator).** OpenAI's embeddings are normalised, so `cosine_distance` and `inner_product` would give equivalent results. We use cosine for consistency with the most common convention in RAG literature, and so that when we add the HNSW index with `vector_cosine_ops` live, the operators and the index's operator class are aligned. **This alignment is important: if the query uses one operator and the index is built with a different operator class, Postgres silently ignores the index and falls back to a sequential scan without warning.** We will see it live.

**Note on performance.** At this stage there is no index. Postgres does a full sequential scan. For the volume of the programme's example corpus (tens of documents, hundreds of chunks) that is perfectly acceptable and the endpoint responds in a few hundred ms. Do not worry about latency — observing it without an index is precisely one of the live session's starting points.

## Step 6 — `query_examples.py` script

Replace Session 07's `compare.py` (which measured similarity between pairs of loose texts) with a script that calls the `/search` endpoint with five representative queries and formats the results.

The five queries must exercise the dataset from different angles:

1. **Known direct component.** A query you know should have an almost perfect match against at least one chunk of the corpus. Sanity check.
   - Example: *"REST API development with JWT authentication for financial sector"*
2. **Semantic reformulation.** The same conceptual idea with vocabulary different from the corpus's. Measures whether the embeddings capture meaning or only words.
   - Example: *"secure backend service with token-based access control for banking applications"*
3. **Different domain.** A query about something that should not be in the corpus. The results should have high distance or be clearly irrelevant.
   - Example: *"mobile application for restaurant reservations"*
4. **Ambiguous query.** Short and generic, which many chunks could partially match. Useful for observing how the ranking behaves in the absence of a dominant match.
   - Example: *"integration with external system"*
5. **Very specific query.** Precise technical vocabulary. Tests whether the model distinguishes between related technologies.
   - Example: *"migration from monolith to microservices architecture using Kubernetes"*

For each query, print the top-5 results with: `chunk_id`, `distance` (4 decimals), `chunk_type` and the first ~120 characters of `content`. Format is free, but readable in a terminal.
