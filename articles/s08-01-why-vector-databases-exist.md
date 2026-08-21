---
title: Why vector databases exist and when you actually need them
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 8
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 31 min
added: 2026-08-13
summary: >
  Why similarity search in high-dimensional space is a different computational
  problem from exact-value lookup, the four properties an in-memory array cannot
  provide (persistence, concurrency, relational filtering, transactions), three
  scale thresholds for when a vector database is justified — including the case
  where it is over-engineering — and how ANN's non-determinism changes both
  testing and debugging.
keywords: [vector database, ANN, KNN, recall, HNSW, IVFFlat, scale thresholds,
           over-engineering, operator class, sequential scan, ACID]
---

# Why vector databases exist and when you actually need them

*Antonio Perez* · 🔴 31 min

At the end of the previous session your AI service has a working pipeline: `chunker.py` splits each budget JSON by its logical business unit, `embedder.py` turns each chunk into a 1536-dimension vector by calling `text-embedding-3-small`, and the `POST /embeddings/ingest` endpoint returns those vectors in the response so you can inspect them. `compare.py` takes two arbitrary texts, embeds them and measures the cosine similarity between them. If you run the pipeline over the example budget corpus and look at the numbers, you see that two descriptions of backend services with authentication give high cosine, and that a backend service description against a graphic design one gives low cosine. The intuition works, the geometry we saw in the Session 07 articles is real, and the code is clean.

There is a small, almost cosmetic detail that nevertheless changes the whole conversation: **those vectors live in the AI service process's memory.** If you restart the container, they are lost. If you ingest three budgets in a row, the fourth knows nothing about the first. If you want to "search" for a budget similar to the one a client just sent you, you have to re-embed all the historical budgets from scratch, or keep a Python list with all the vectors and walk it with numpy computing cosines one by one. For ten demo budgets that is invisible. For the hundreds or thousands of historical budgets a real client would have, it stops being invisible very fast.

This article is about that gap. Not about which vector database to choose — that is article 2 — but about **why an entire category of software exists** and what problem it solves that neither a Python array nor a classic relational database solves. The question is not rhetorical. There are teams that assume they need a vector database because "everyone uses one" and end up paying operational complexity beyond their real scale. And there are teams that postpone it until they already have a serious latency or incoherence problem in production. Both mistakes are easy to avoid if you understand the exact moment when adding this piece to the stack is justified.

## The technical problem no earlier system solves

To understand the category you first have to see what does not fit in what you already know. A relational database like PostgreSQL is designed to answer questions of the type "does the record with `id = 42` exist?" or "how many budgets are there from the fintech sector in 2024?". The engine is optimised for exact-value and range searches, with B-tree indexes that exploit the fact that values are linearly orderable. When you ask "which budgets are semantically similar to this brief?", the answer is not obtained by comparing exact values: it is obtained by comparing 1536-dimension vectors and returning the k nearest. **There is no way to index "closeness in 1536 dimensions" with a B-tree.**

You could try to solve it by hand. You store the vectors as `float[]` or as JSONB in a column, write a PL/pgSQL function computing cosine distance, and do `ORDER BY similarity DESC LIMIT k`. The result works — it returns the k nearest vectors correctly — but that `ORDER BY` forces Postgres to compute the distance against *every* stored vector before it can sort them. This is called **exact search or KNN** (k-nearest neighbors): it guarantees the correct result at a cost linear in the number of vectors. With a hundred vectors it is invisible. With a hundred thousand it starts to hurt. With several million it becomes unviable for any interactive application.

The way out is not making the search faster by computing cosines in C instead of Python. **The way out is changing the question.** Instead of "give me the guaranteed k nearest", you accept "give me k that are almost certainly among the nearest". That approximation has a name: **ANN, approximate nearest neighbors.** The key is not in the word "approximate" but in what that approximation permits: specialised data structures — navigable graphs, partitions of the space into regions, quantisations — that avoid touching most vectors on each query and reduce complexity from linear to logarithmic or nearly constant in many cases. The price is a recall slightly below 1.0 (perhaps 1% of your searches miss a neighbour that was the "true" best) and a construction cost for the structure before querying. For a RAG application returning the 5 most relevant chunks to a user, 98% recall is indistinguishable from 100% in practice. **A latency of 5 ms versus 5000 ms is not.**

> *(Figure in the original: `articulo-01-figura-01-knn-vs-ann.jpg` — image not included in this repo.)*

That is exactly what a vector database offers: an implementation of ANN indexes on top of a persistent storage system, exposed through an API or an SQL dialect. The concrete indexing algorithms we will see in detail in article 3, but the underlying idea is this: **the category exists because similarity search in high-dimensional spaces is a different computational problem from exact-value search, and needs different data structures.** It is not a marginal optimisation over what you already had; it is a new primitive.

## The four properties a Python array does not give you

Let's return to the project's current state. The in-memory vectors are, in reality, a mini vector database implemented with `list` and numpy. The search works (you compute cosines and sort), it is even reasonably fast at small scales because numpy is compiled C underneath. So what is missing for this to count as production?

**Persistence.** A process that dies takes all the vectors with it. This is unacceptable in any system that is not an exploration script. Re-embedding the whole corpus on each restart is not just costly in time: it is costly in real money, because each call to the embeddings API is paid. Persisting the vectors is the first line you cross when the project stops being a prototype.

**Concurrency.** When the AI service has a single FastAPI worker, in-memory state is coherent. When you deploy two replicas behind a load balancer to support more load, each replica has its own in-memory state and the two can diverge: an ingestion arrives at replica A, a search arrives at replica B and does not find what was just ingested. The solution is not sharing memory between processes — that reopens every problem concurrent programming solved decades ago. The solution is delegating persistence and coherence to a system designed for it.

**Queries combined with relational data.** In the estimation project, you almost never want "the 5 most similar chunks" and nothing else. You want "the 5 most similar chunks from the fintech sector, ingested in the last 2 years, from budgets under 100k". Those filters live in typed columns. A vector database on a relational engine lets you mix the two things in a single atomic query. A Python array forces you to filter by hand before or after, losing efficiency either way and opening subtle bugs when the filter and the search get out of sync.

**Transactional operations.** Ingesting a budget means, logically, creating a row in `documents` and N rows in `chunks`. If the embedder fails on the fifth chunk, you do not want to be left with the document created and only four chunks embedded: you want to roll back and retry. ACID guarantees are not a luxury; they are what stops your corpus degrading silently with every network error.

These four properties are what an in-memory array does not give you and what your pipeline needs the moment it crosses the prototype frontier. The question is not whether you need them — you need them as soon as the project leaves the scratchpad — but **where you put them**. One option is building them yourself: persisting the vectors to a file, managing concurrency with locks, writing your own filter engine. It is educational and, for very small volumes, viable. It is also a bad idea almost always, because you are reimplementing half a database without the decades of hardening the existing options already ship with.

## When to add a vector database to the stack — and when not to

The decision to add this piece to the system has to be taken with operational judgement, not by reflex. There are three approximate thresholds worth having in your head when evaluating whether the moment is right.

> *(Figure in the original: `articulo-01-figura-02-rangos-escala.jpg` — image not included in this repo.)*

**Below about ten thousand vectors**, and especially if the data is static (it does not change, it is only queried), a numpy array loaded at service startup is perfectly reasonable. Exact KNN search takes milliseconds at that scale, there are no coherence problems, and the code is trivial to maintain. **Adding a vector database at that scale is over-engineering.** Most public tutorials skip over this and push the developer to set up pgvector, Pinecone or Qdrant from day one, which confuses when the complexity is justified.

**Between ten thousand and, say, fifty million vectors**, a vector database is the correct answer. It is the range in which exact search stops being viable and where ANN indexes shine. It is also the range where the great majority of B2B AI products sit, including the estimation system we are going to build. If your project falls here — and it is very likely yours does — the question is not "do I need a vector database?" but "which one?". And that is article 2's question.

**Above a hundred million vectors**, things get interesting and considerations enter that are outside this programme's focus: sharding, replication across multiple regions, indexes that no longer fit in RAM and need disk strategies like DiskANN, very different cost models. At that scale, decisions are taken with the concrete case and the team's operational ecosystem in mind. Almost no real-world project starts at this scale; you get there after years of growth.

There is a fourth, important case where none of the above applies: **when the problem you have does not require semantic similarity search at all.** If what you are looking for is exact matches on product names, client identifiers or catalogue references, a full-text search with PostgreSQL `tsvector` or a trigram index solves the problem better and with less complexity. **Semantic search is valuable when the exact terms vary but the meaning does not.** If your queries and your data share vocabulary, a vector database is a hammer on something that is not a nail.

The estimation project falls clearly in the middle range, with the addition that the data does vary in vocabulary (a client brief rarely uses the same exact words as the historical budgets). Semantic search is the correct primitive, and we need persistence, concurrency, joins with relational data and transactional operations. All the conditions are aligned: session 08 introduces the component exactly when the project needs it and not before.

## Exact KNN versus approximate ANN: what changes in your head

One last detail before closing, because it is the conceptual piece that will probably cause you the most friction when you start with pgvector in the pre-session exercise and live. Until now, every system you have touched in the programme returns deterministic answers: if you ask for the record with `id = 42`, it returns the record with `id = 42`, always. **Vector databases with ANN indexes do not work like that.**

When you build an HNSW or IVFFlat index and fire a search, the answer is "the k vectors the index *believes* are nearest, with high probability". If the same dataset is reindexed with different parameters, the results can change slightly. If you tune `ef_search` upwards, recall rises and so does latency. If you lower it, both drop. What you accept in exchange for the speed is a tolerance zone where "almost always correct" is the operational norm.

This has two practical implications worth internalising right away. The first is that **the index's recall is measured; it is not assumed.** Before putting an ANN index into production, you compare its results against ground truth (the exact search without an index) over a set of representative queries and verify the overlap is in an acceptable range for your use case. This is a new mechanic that did not exist in the relational world, and we introduce it live in session 08.

The second implication is that **debugging changes.** If a search returns a result you did not expect, before assuming the embedding model is bad or your chunking is broken, it is worth verifying the index is not being silently ignored. There is a very common antipattern (which we see in detail in article 4 and replicate live) in which the query's operator and the index's operator class are misaligned — the query uses `<=>` but the index was built with `vector_l2_ops` — and Postgres falls back to sequential scan without warning. The results remain correct in exact-search terms, but **the latency is 600× worse** and the rankings change because a different distance is being measured. It is the kind of silent bug that defines the productivity ceiling of a team that does not understand the tool well.
