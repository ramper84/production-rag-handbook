---
title: "Anatomy of a vector index: HNSW, IVFFlat and the DiskANN horizon"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 8
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 40 min
added: 2026-08-13
summary: >
  What actually happens on CREATE INDEX USING hnsw. IVFFlat as Voronoi cells with
  lists/probes, HNSW as a multi-layer navigable graph with m/ef_construction/
  ef_search, DiskANN as the beyond-RAM horizon — plus a decision table placing
  each (including sequential scan) and how to verify with EXPLAIN ANALYZE that
  the index is being used at all.
keywords: [HNSW, IVFFlat, DiskANN, pgvectorscale, ef_search, ef_construction,
           lists, probes, operator class, EXPLAIN ANALYZE, sequential scan, recall]
---

# Anatomy of a vector index: HNSW, IVFFlat and the DiskANN horizon

*Antonio Perez* · 🔴 40 min

If you have got this far following the two previous articles, you already know you are going to use pgvector and you know why. What you do not know yet — and what separates the developer who copies the defaults from the one who takes operational decisions with judgement — is what exactly happens when you run `CREATE INDEX ... USING hnsw` and why the difference between the parameters you put there marks two orders of magnitude in latency, recall and memory consumption.

This article opens that black box. We are not going to derive mathematics; we are going to build the geometric intuition of the algorithms and land each one on the concrete parameters pgvector exposes, with the real values the student will touch in production. There are three relevant families in the current landscape: the IVFFlat index (based on partitioning the space), the HNSW index (based on navigable graphs) and the DiskANN horizon (the next generation, appearing as pgvectorscale and as `diskann` in Azure). All three solve the same problem with different geometric strategies, and knowing all three is what lets you recognise the moment the default choice stops being the right one.

Before going into each, it is worth recalling the baseline they are compared against. With no index at all, pgvector does a **sequential scan**: for each query, it computes the distance between the query and the embeddings of the rows surviving the relational filters (if any), sorts, and returns the k nearest. It is the simplest method, guarantees 100% recall and returns deterministic results. Latency grows linearly with the number of vectors, and the moment that number enters the tens of thousands, calling it "interactive" stops being realistic. ANN indexes exist to break that linearity. Each does it differently.

## IVFFlat — split the space into cells and look only at the ones that matter

IVFFlat's intuition is the simplest of the three. If you have a map of a city with thousands of restaurants and a client asks you for the five nearest to their location, the reasonable thing is not to compare distances one by one with every restaurant in the city. The reasonable thing is to identify the neighbourhood the client is in, look at the restaurants in that neighbourhood and, if there are few, expand to neighbouring ones. IVFFlat does exactly that, but in a 1536-dimension space.

The process has two phases. First, during index construction, a clustering algorithm (k-means) is applied over the existing set of vectors, identifying `lists` centroids — the "neighbourhoods". The space ends up divided into regions called **Voronoi cells**: each point of the space corresponds to the cell whose centroid is nearest. Each stored vector is assigned to its corresponding cell. The resulting structure is an "inverted list": for each centroid, a list of the vectors living in its cell. Hence the name IVFFlat: *Inverted File index with Flat (uncompressed) vectors.*

> *(Figure in the original: `articulo-03-figura-01-ivfflat.jpg` — image not included in this repo.)*

During the query, the logic is inverted. Given a query, its distance to the `lists` centroids is computed — a fast operation, because `lists` is usually between 100 and 10,000, far smaller than the total number of vectors. The `probes` centroids nearest the query are chosen, and the search is restricted to the vectors living in those cells. If `probes = 1`, you look only at the client's neighbourhood; if `probes = 10`, you also look at the nine nearest neighbours. Recall rises with `probes`, and so does latency.

The `lists` parameter is chosen during index construction and the standard heuristic is `lists ≈ sqrt(rows)` for corpora up to a million vectors, scaling to `rows/1000` for larger ones. For the hundreds of thousands of chunks the programme's project will have, this translates to values between 100 and 1000 — small, easy numbers. The `probes` parameter is chosen per query (or by default in the session) and the standard heuristic is `probes ≈ sqrt(lists)`: if `lists = 100`, try `probes = 10`. Raising `probes` improves recall and increases latency approximately linearly.

IVFFlat has three virtues and two important defects. The virtues: fast construction (k-means over the embeddings is reasonable up to hundreds of thousands of vectors), moderate memory (you only need to store the centroids plus the grouped vectors, no additional structures), and it works well on mostly-static corpora. The defects are the ones that almost always rule it out in real RAG: **it needs training** (you cannot create the index over an empty table; you need representative data for k-means to produce useful centroids), and **it suffers with dynamic data** (as you insert new vectors, the cells the original k-means defined start not representing the real distribution, and recall degrades silently until you rebuild the index). For a RAG system where the corpus grows with each newly ingested budget, that silent degradation is a concrete operational risk.

IVFFlat's other classic problem — that two nearby vectors can fall into different cells if they are near a border — is mitigated with higher `probes`, but never disappears. The quality of results depends fundamentally on the shape of the clusters k-means found, and that is not something you control directly.

## HNSW — a multi-layer graph acting as a hierarchical GPS

HNSW solves the same problem from a completely different angle, and the most useful geographic analogy is not the map with neighbourhoods but a country's road system. When you want to reach a specific address in a village 800 kilometres away, you do not travel every possible road. You take a motorway bringing you near the general region, then a national road bringing you near the area, then a local road taking you to the village, and only at the end do you cross local streets to the exact address. Each level of the hierarchy covers different distances, and the combination is what makes the journey logarithmic instead of linear.

HNSW builds exactly that hierarchy. It is a graph divided into layers. The bottom layer — layer 0 — contains every stored vector, each connected to its nearest neighbours by short edges. The upper layers contain progressively smaller subsets of vectors, chosen at random with an exponentially decreasing distribution: approximately 1% of the vectors appear in layer 1, 0.01% in layer 2, and so on. The edges of the upper layers cover long distances; those of the lower ones, short.

The search works top-down. You start from an entry point in the highest layer and advance greedily towards the neighbour nearest the query. When there are no more neighbours in that layer bringing you closer, you drop to the next layer and continue the process at finer resolution. In layer 0, where you are already very close, you explore a wide neighbourhood to find the final k nearest vectors. The structure is essentially a **skip list applied to graphs**: what skip lists are to linked lists, HNSW is to neighbour graphs. Malkov and Yashunin's original paper, published in 2018, showed this produces logarithmic complexity with consistently high recall even in very high-dimensional spaces.

> *(Figure in the original: `articulo-03-figura-02-hnsw.jpg` — image not included in this repo.)*

In pgvector, HNSW is governed by three parameters. The first two are **build-time** (decided when creating the index and not changeable afterwards without rebuilding): `m` and `ef_construction`. The third, `ef_search`, is **query-time** (adjusted per session or transaction without touching the index).

**`m`** is the maximum number of bidirectional connections each vector has in each layer of the graph. The default is 16 and the community has converged on it being correct for 1536-dimension embeddings (the project's OpenAI `text-embedding-3-small`). Raising `m` to 32 or 48 slightly improves recall at the cost of doubling the index's memory and practically doubling construction time. The original paper names it the most important parameter, and almost nobody should change it without having first measured that recall is insufficient with the other parameters exhausted. For the programme's project, `m = 16` is the right choice.

**`ef_construction`** is the size of the dynamic candidate list the algorithm maintains while building the graph. pgvector's default is 64, but the 2026 community has converged on `ef_construction = 128` or even 200 being a more reasonable starting point for production, especially with high-dimensional embeddings. The cost of raising it is index construction time (roughly doubling from 64 to 128), but since construction happens once and search happens many times, it is almost always worth it. For the project, start at 128 and only lower it if construction time becomes an operational bottleneck.

**`ef_search`** is the most interesting and the most used. It controls the size of the candidate list during the query. The default is 40 and it can be changed per session, per transaction or per query. Raising it to 80 or 100 improves recall at the cost of latency (the curve is not linear: the marginal recall improvement decreases, while latency rises approximately linearly). Lowering it to 20 speeds up queries at a recall cost that may be acceptable where the k results are going to be re-ranked afterwards anyway. It is the parameter worth tuning empirically: sweep `ef_search` between 10 and 200 over a representative set of queries, measure recall and latency, and pick the point best balancing both for your use case.

HNSW's virtues are the opposite of IVFFlat's defects. **It needs no training**: you can build an HNSW index over a table with zero rows and then insert. **It absorbs insertions without rebuilds** (new rows are added to the graph incrementally). And its recall is consistently high, typically above 95% with the defaults, without IVFFlat's silent degradations. The cost is memory: HNSW consumes between 2 and 5 times more memory than IVFFlat for the same number of vectors, because it has to store the complete graph in addition to the vectors. For volumes up to about ten million vectors on reasonable hardware, that cost is perfectly manageable. For larger volumes, it starts to squeeze.

> *(Figure in the original: `articulo-03-figura-03-parametros-hnsw.jpg` — image not included in this repo.)*

For the programme's project, HNSW is the right choice and the starting parameters are `m = 16`, `ef_construction = 128`, `ef_search = 40`. These are the values we are going to use live when we create the index and the ones you will touch empirically when we measure latency and recall.

## DiskANN — the horizon extending pgvector beyond RAM

There is a third algorithm worth knowing, even though you will not use it in the programme's project: **DiskANN**. It is the natural evolution when volumes exceed what fits comfortably in RAM and you need to keep maintaining low latencies. The original paper is from Microsoft Research, published in 2019; Microsoft's implementation reaches a billion vectors with 95% recall and 5 ms latency on a single machine with an SSD — a scale where HNSW becomes unviable because its index does not fit in memory.

DiskANN's key intuition is that it replaces HNSW's multi-layer hierarchy with a **single flattened graph** but with "long" edges strategically inserted during construction, via an algorithm called Vamana. Those long edges play the role of HNSW's jumps between layers, but since the graph is flat, its disk layout can be optimised to minimise SSD reads during search. The operational trick has two parts: a quantised, compressed version of the graph lives in memory, allowing fast navigation among approximate candidates; and only when final distances must be compared is the complete vector read from the SSD. The result is that indexing a billion vectors requires a few GB of RAM instead of hundreds.

In the Postgres ecosystem, DiskANN appears as **pgvectorscale**, an open-source extension from Tiger Data (formerly Timescale) adding a third index type — `USING diskann` — over pgvector's `vector` type. The syntax is the same and so are the distance operators; only the underlying algorithm changes. Microsoft also offers its own version as a managed extension in Azure Database for PostgreSQL Flexible Server. Both implementations add, besides the algorithm itself, a quantisation technique (Statistical Binary Quantization in pgvectorscale, Product Quantization in Azure's version) further reducing the memory footprint.

When to migrate from HNSW to DiskANN? The two clear signals are: the HNSW index no longer fits in `shared_buffers` and latency degrades because disk I/O happens on every query; or the RAM cost of sustaining the index exceeds the SSD cost needed for DiskANN. In 2026 Tiger Data's public benchmarks report pgvectorscale reaching 471 QPS at 99% recall over 50 million vectors, while pure HNSW starts degrading in that range. For volumes below several million, HNSW remains faster and simpler. For the programme's project we are orders of magnitude below any DiskANN migration threshold, so we mention it as a horizon and continue with HNSW.

## The decision table: when each algorithm wins

The choice between the three is not a personal preference. There is a reasonably clear map:

**Sequential scan (no index).** Up to a few thousand vectors, especially if the data is very dynamic (many insertions, relatively rare reads) or if you need **guaranteed 100% recall** for auditing or evaluation reasons. It is also the baseline against which you measure the impact of adding an ANN index: if you build HNSW and latency does not drop significantly, something is wrong (typically: the execution plan is not using the index — we will see it live).

**IVFFlat.** Up to a few million vectors, mostly-static dataset, strict memory budget, critical construction time. It is reasonable, for example, in batch analytical systems where the database is rebuilt periodically and real-time search is not the main use case. For RAG with a growing corpus, avoid it: silent degradation with insertions is a real operational problem.

**HNSW.** The workhorse for almost all RAG in the range of tens of thousands to tens of millions of vectors, with active writes, high recall required and reasonable memory available. It is the programme project's choice and the choice of the vast majority of production RAG systems in 2026.

**DiskANN (via pgvectorscale or equivalent).** From several million vectors where HNSW starts squeezing memory, or tens of millions where HNSW is no longer viable. It provides the highest ceiling without migrating to a dedicated system.

There is an important detail worth keeping in mind, which we see live: the three ANN algorithms share in pgvector the same three **operator classes** — `vector_cosine_ops` for cosine (`<=>`), `vector_l2_ops` for L2 (`<->`), and `vector_ip_ops` for inner product (`<#>`). When creating the index you choose one of the three, and from that moment only queries using the corresponding operator will exploit the index. **If you build the index with `vector_cosine_ops` and then run a query with `<->`, Postgres will fall back to sequential scan silently, with no error, no warning, only a brutal latency degradation.** It is one of the most common antipatterns and we cover it in detail in the next article.

## How to verify the index is being used

Although we will see the detailed mechanics live, it is worth anticipating the main tool: `EXPLAIN ANALYZE`. Postgres shows a query's execution plan and, crucially, indicates whether the plan is using the index or doing a sequential scan. A well-planned query using HNSW will look like this:

```sql
EXPLAIN ANALYZE
SELECT id, content, embedding <=> :query AS distance
FROM chunks
ORDER BY embedding <=> :query
LIMIT 5;
```

```
Limit (cost=... rows=5 ...)
  -> Index Scan using chunks_embedding_idx on chunks (cost=...)
       Order By: embedding <=> '[...]'::vector
       ...
Planning Time: 0.234 ms
Execution Time: 4.821 ms
```

The key is the line `Index Scan using chunks_embedding_idx`. If `Seq Scan on chunks` appears instead, the index is not being used, and it is almost always for one of three reasons: the query's operator does not match the index's operator class, the `WHERE`'s relational filters are too selective and Postgres considers it cheaper to scan than to go through the index, or the index has not yet been vacuumed after its construction.

## Bridge to the next article

You now have the map of the three algorithms. You know what each does underneath, what parameters pgvector exposes to tune them and under what conditions each wins. You also know that for the programme's project the choice is HNSW with `m = 16`, `ef_construction = 128`, `ef_search = 40`, and that this choice carries an additional and very important decision: which operator class to build the index with and, therefore, which operator to use in the queries.

That decision, together with the design of the relational schema sustaining the whole system, is the content of the next article: *Schema design and semantic search in pgvector*. There we land the concrete model of the `documents` and `chunks` tables for the project, the three distance metrics with the particular case of OpenAI's embeddings (which are normalised, which has practical consequences), and the silent trap of operator-index misalignment we have just mentioned. Read it just before the pre-session exercise: it contains the executable SQL schema you will use as reference.
