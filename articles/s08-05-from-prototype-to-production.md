---
title: "From prototype to production: tuning, monitoring and pgvector's ceiling"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 8
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 40 min
added: 2026-08-14
summary: >
  What separates a working prototype from a system that survives two years in
  production: memory sizing as the variable governing 80% of performance, index
  build parameters (maintenance_work_mem, parallel workers, CONCURRENTLY),
  halfvec quantisation, operational monitoring via pg_stat_user_indexes and the
  VACUUM/REINDEX cycle, and three measurable signals that pgvector has hit its
  ceiling.
keywords: [shared_buffers, maintenance_work_mem, CREATE INDEX CONCURRENTLY,
           halfvec, quantisation, pg_stat_user_indexes, pg_stat_statements,
           REINDEX, VACUUM, ANALYZE, p99, migration signals]
---

# From prototype to production: tuning, monitoring and pgvector's ceiling

*Antonio Perez* · 🔴 40 min

On finishing the pre-session exercise, your AI service has a Postgres with pgvector, a well-designed schema, a working search endpoint and a validation script. It is a correct system, sufficient for the programme's example corpus. It is also, almost by definition, a development system: it runs in local Docker, with PostgreSQL defaults meant for a demo, without a vector index yet, without metrics, without a maintenance strategy, without any thinking about what happens when the corpus grows from hundreds to millions of chunks.

This final article covers the pieces that cross that frontier. They are not pieces the student will apply literally in the exercise — the programme project's volumes do not require them — but they are the decisions separating a RAG system that "works in session 09" from one that "works in production for two years without surprises". And they are also the decisions you will need to argue the day your team decides to take a pgvector-based system to production, inside or outside this programme.

The route has four parts. First, the most important rule: why memory sizing is the decision with the greatest performance impact, and how it is calculated. Second, the index construction parameters and why PostgreSQL's defaults are inadequate for building a serious HNSW index. Third, `halfvec` — the quantisation halving storage without significant quality loss. Fourth, operational monitoring with `pg_stat_user_indexes` and the maintenance cycle (`REINDEX`, `VACUUM`, `ANALYZE`). We close with the objective signals indicating pgvector has hit its ceiling in a concrete case and migration is worth considering.

## Memory sizing: the rule governing everything else

There is a claim that deserves to be taken literally: **the performance of a pgvector system with HNSW indexes is determined 80% by a single variable, and that variable is whether the index fits in memory.** Everything else — `ef_search`, construction parameters, embedding model, hardware — are refinements on that central question. If the HNSW index lives comfortably in `shared_buffers` and the operating system cache, queries are a few milliseconds. If the index does not fit and Postgres has to read pages from disk on every query, no parameter is going to recover the latency.

The reason is structural. HNSW is a graph: a search traverses multiple nodes jumping between neighbours. If each jump implies an SSD read, what in RAM would be 5 microseconds becomes 100 microseconds per jump, and with 30 typical jumps per query the latency goes to 3 ms from I/O alone — not counting distance computation, not counting the network round-trip. If the SSD is busy with other workloads, the I/O queue translates into unpredictable additional latency. This is the famous "long tail" that no application tuning fixes.

Practical sizing has three components. **The space the vectors occupy on disk**: for `text-embedding-3-small`, each vector is 1536 dimensions × 4 bytes = 6,144 bytes, or approximately 6 KB. A million chunks consumes about 6 GB in vectors alone. **The HNSW index**: approximately 2× to 3× the space of the underlying vectors for the default `m = 16`. A million chunks produces an HNSW index of some 12 to 18 GB. **Postgres overhead**: connection pools, `work_mem`, OS buffers, WAL buffers. In total, a conservative practical rule is that you need hardware with total RAM greater than or equal to **1.5× the size of the index plus the vectors.**

For the programme project's volumes this is not a problem: hundreds of thousands of chunks produce an index of a few GB that fits effortlessly on any reasonable hardware. The problem appears when a team extrapolates the prototype setup to a client with several million budgets and discovers, in production, that the latency which was 5 ms in tests is now 500 ms because the index no longer fits in memory. It is one of the most common failure modes in pgvector migrations to real environments.

The concrete PostgreSQL configuration controlling this reduces to three parameters that come in `postgresql.conf` or are passed as flags when starting the container:

- **`shared_buffers`** is the memory PostgreSQL reserves for its own cache. The standard heuristic is 25% of total available RAM. For a server with 32 GB of RAM, `shared_buffers = '8GB'`.
- **`effective_cache_size`** is not a real memory allocation, but a hint to the query planner about how much total memory (PostgreSQL + OS cache) is available. The heuristic is 75% of total RAM. For the 32 GB example, `effective_cache_size = '24GB'`.
- **`work_mem`** controls how much memory each operation (sort, hash join) can use. For workloads with vector search, values of 64-256 MB are reasonable.

PostgreSQL's defaults for `shared_buffers` (128 MB) and `work_mem` (4 MB) are appropriate for a small database on a shared server, not for a production RAG service. This is the first thing you change when you deploy.

> *(Figure in the original: `articulo-05-figura-01-sizing-memoria.jpg` — image not included in this repo.)*

## Index construction: parameters that are not the query's

When you run `CREATE INDEX ... USING hnsw`, Postgres builds the complete graph in an operation that is essentially a massive query pre-computing an enormous number of distances between vectors. That operation has its own parameters, distinct from the ones affecting the query.

**`maintenance_work_mem`** is the most important. It controls how much memory PostgreSQL can use during index maintenance operations. The default is 64 MB, which is **tragic** for building an HNSW index: if the graph under construction does not fit in that buffer, Postgres falls back to a disk-based construction between 10× and 50× slower. For a corpus of 5 million 1536-dimension vectors, you will need between 8 and 16 GB of maintenance memory. Before building a large index, raise it:

```sql
SET maintenance_work_mem = '4GB';
SET max_parallel_maintenance_workers = 4;

CREATE INDEX CONCURRENTLY chunks_embedding_idx
ON chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 128);
```

**`max_parallel_maintenance_workers`** controls how many parallel workers can work on the index construction. The default is 2, conservative. With 4 workers, an HNSW index over a million rows that would take 30 minutes serially can be built in 8 to 10 minutes. The ceiling is set by your number of available vCPUs. In your `docker-compose.yml`, you also need to declare sufficient `shm_size`: parallel workers share memory via `/dev/shm`, and if that area is small, the workers crash with **cryptic OOM errors at the end of an hours-long construction.**

**`CONCURRENTLY`** is critical for production. Without that keyword, `CREATE INDEX` locks the table for writes for the entire construction. For an index taking half an hour, that is half an hour unable to ingest new documents. With `CONCURRENTLY`, construction is slower (between 1.5× and 2× depending on concurrent load) but the table keeps accepting writes. In development, `CONCURRENTLY` is optional. In production, it is obligatory.

> *(Figure in the original: a sizing reference table image — not included in this repo. It gives build times and index sizes by corpus size for Postgres + pgvector 0.8, HNSW with m = 16 and ef_construction = 128, over 1536-dimension embeddings.)*

The programme's project, with its expected volumes, lives in the first row. But the numbers in the following rows are what justify `halfvec` — the next topic — being the default decision in any serious scenario.

## Halfvec: the quantisation saving half the storage

`text-embedding-3-small` embeddings are stored by default in pgvector as single-precision floats (32 bits, 4 bytes) per dimension. That is the representation the `vector(1536)` type uses. The 32-bit precision is inherited from how models generate embeddings and how PostgreSQL stores floats natively, but it is far more precision than semantic search needs.

The **`halfvec`** type pgvector introduced in version 0.7 stores each dimension in 16 bits (half-precision float) instead of 32. Space is halved, index construction times are also reduced by approximately half, and recall over OpenAI's normalised embeddings stays above 99% in published benchmarks. It is not an aggressive quantisation like binary (which reduces each dimension to 1 bit and significantly degrades recall); it is a loss of precision that in practice is indistinguishable from the full representation for the vast majority of use cases.

> *(Figure in the original: `articulo-05-figura-02-halfvec.jpg` — image not included in this repo.)*

The operational recommendation in 2026 is clear: **start with `halfvec` from day one if you are going to production.** Migrating from `vector` to `halfvec` when you already have tens of millions of vectors in the table is a painful operation requiring re-embedding or re-copying the whole corpus. Taking the right decision when designing the schema saves you that cost.

In practice, this slightly changes the schema and the index creation:

```sql
-- Schema: the column remains vector(1536) but we index halfvec
CREATE INDEX chunks_embedding_idx
ON chunks
USING hnsw ((embedding::halfvec(1536)) halfvec_cosine_ops)
WITH (m = 16, ef_construction = 128);
```

The expression `embedding::halfvec(1536)` quantises the vector in memory when building the index, without touching the original column. The corresponding operator class is `halfvec_cosine_ops` (and its equivalents `halfvec_l2_ops`, `halfvec_ip_ops`). Queries keep working with the `<=>` operator unchanged — the cast is applied automatically when the planner detects the index.

For the programme's project we do not use `halfvec` in the pre-session exercise because we keep the setup minimal, but it is the first thing you would add when moving from prototype to real production.

## Operational monitoring: seeing what your index is doing

Once the system is in production, there are three questions you should be able to answer at any moment: **is the index being used?**, **are queries returning results in the expected time?**, and **is the index degrading over time?** Postgres brings tools for all three.

**Is the index being used?** The `pg_stat_user_indexes` view maintains cumulative counters per index. The `idx_scan` field indicates how many times an index has been used since the last statistics reset, and `last_idx_scan` (PostgreSQL 16+) indicates when the most recent use was. A useful periodic checkup query:

```sql
SELECT indexrelname AS index_name,
       idx_scan AS scans,
       last_idx_scan AS last_used,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'chunks'
ORDER BY idx_scan DESC;
```

If your main vector index has `idx_scan = 0` after a significant period in production, something is not right — and it is almost always the previous article's antipattern: queries using an operator different from the index's operator class. **It is the fastest way to detect the silent bug.**

**How long does each query take?** The canonical system for this is `pg_stat_statements`, an extension recording aggregate statistics for each distinct query executed: number of calls, total time, average time, minimum and maximum time. It lets you identify which queries are the main time consumers and detect regressions when changes are introduced. For RAG systems, it is worth having this extension enabled from day one. The Logfire plugin we already have in the programme's stack also captures this data, integrating it with the traces of the LLM calls, which gives an end-to-end view of where time goes in each request.

**Is the index degrading?** This is where maintenance operations come in. An HNSW index that was optimal at initial construction can degrade over time for two reasons. First, **bloat**: updates and deletes leave pages with dead space the index does not recover automatically. Second, **stale statistics**: the query planner relies on statistics updated by `ANALYZE`, and if they are out of date, it can take suboptimal decisions (for example, ignoring the index because it believes the table is smaller than it is).

The basic maintenance cycle is composed of three operations:

- **`VACUUM ANALYZE chunks`** recovers dead space and updates statistics. Postgres does it automatically in the background with autovacuum, but for tables with high churn it is worth scheduling explicitly in a low-traffic window.
- **`REINDEX INDEX CONCURRENTLY chunks_embedding_idx`** rebuilds the index from scratch. It is the operation that recovers performance when bloat has degraded the index. The `CONCURRENTLY` keyword lets the reindex happen without blocking queries: Postgres builds a new index alongside, validates that it works, and atomically swaps old for new.
- **`ANALYZE chunks`** without VACUUM when you only need to update statistics. It is fast and cheap.

The recommended cadence for RAG systems with moderate writes (tens to hundreds of insertions per day) is: automatic `ANALYZE` with autovacuum, weekly `VACUUM ANALYZE` in a low-traffic window, and monthly `REINDEX CONCURRENTLY` or when you observe the index's latency starting to degrade perceptibly. For systems with very intensive writes, the cadences accelerate proportionally.

## The objective migration signals

pgvector is the right choice for the programme's project and for the vast majority of production RAG systems in 2026. It is not the right choice for every case. There are three objective signals indicating you have reached pgvector's ceiling in a concrete scenario and that migration to a dedicated system (Qdrant, Milvus) or a more specialised extension (pgvectorscale with DiskANN) is worth evaluating. They are not nuances or preferences: they are **measurable metrics**.

> *(Figure in the original: `articulo-05-figura-03-senales-migracion.jpg` — image not included in this repo.)*

**Signal 1: the HNSW index no longer fits in memory.** When the index's size significantly exceeds what `shared_buffers` and the OS cache can retain, p99 latency becomes unpredictable. If you check the ratio between index size and available RAM and it is above ~70%, you are in the risk zone. The short-term solution is scaling hardware vertically (more RAM); the medium-term solution is migrating to pgvectorscale with DiskANN, specifically designed to maintain low latency with indexes living partly on SSD.

**Signal 2: p99 latency exceeds your operational SLO in a sustained way.** If vector search queries start seeing p99 above the threshold your product can tolerate (typically 100-200 ms for interactive RAG), and you have ruled out PostgreSQL tuning problems, query antipatterns and bloat, what you are seeing is probably pgvector's ceiling for your volume and access pattern. At that point, a dedicated vector database like Qdrant or Milvus usually gives between 2× and 5× better p99 at equal hardware.

**Signal 3: you need native features pgvector does not have.** Multi-modal search with text + image embeddings as first-class citizens, native multi-region sharding with a strict SLA, very specific cost models. If your product genuinely requires them and they are not simple nice-to-haves, a dedicated option is going to be a better investment than fighting pgvector to imitate them.

What matters is that none of these signals is based on "what is cooler" or "what the competition uses". They are verifiable metrics about your concrete system. **If none of the three holds, keeping pgvector is almost always the right decision** — even if you are tempted to migrate to something "more serious". The operational complexity a dedicated system adds has its own cost, and it is only justified when the signals demand it.
