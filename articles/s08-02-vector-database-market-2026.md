---
title: State of the vector database market 2026
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 8
series_part: 2
scope: time-sensitive
source: user-supplied article (market state as of 2026)
reading_time: 40 min
added: 2026-08-13
summary: >
  Five production options (pgvector, Qdrant, Weaviate, Milvus, Pinecone) judged
  on four axes that matter operationally — operating model, practical scale,
  native features, and total cost including operations and migration. Concludes
  with a worked case for pgvector and, more usefully, an explicit list of the
  conditions under which that answer stops being correct.
keywords: [pgvector, qdrant, weaviate, milvus, pinecone, pgvectorscale, HNSW,
           hybrid search, tsvector, total cost of ownership, migration]
---

# State of the vector database market 2026

*Antonio Perez* · 🔴 40 min

The previous article closed with an open question. You already know the project is in the range where adding this piece to the stack is justified, you know what properties it must have (persistence, concurrency, joins with relational data, transactions), and you know the new technical primitive is approximate search with ANN indexes. What you do not have yet is the market map.

This article is that map. It covers the five options that dominate production deployments in 2026, positions them on the axes that matter operationally, and ends by justifying the programme's locked decision — using pgvector — without selling it. It is deliberately honest about when that decision is correct and when it would stop being so, because the question "which vector database do we use?" is not answered once for a whole career: it is answered for each project, and the projects of your working life outside the programme will probably point in different directions.

Before going into each option, it is worth being clear about the four axes we are going to evaluate them on. Any comparison based only on QPS or only on price leaves you halfway.

## The four axes that matter

**Operating model.** Do you manage it or does the provider? Self-hosted means your team has total control — but also that your team is responsible for updates, backups, monitoring, tuning and incident response at 3 in the morning. Managed means you pay not to have to think about any of that, in exchange for less control and a cost model that scales with usage. There is no universally better answer; there is a correct answer for your team's size and operational maturity.

**Practical scale.** The number of vectors the system reasonably supports. "Supports" does not mean "can technically store"; it means it maintains acceptable latencies, reasonable recall, and a cost that does not blow up quadratically. Public 2026 benchmarks mark three clear ranges: below ten million vectors any of the five options is viable, between ten and a hundred million the field narrows, and above a billion the list reduces to systems designed specifically for that scale.

**Native features.** Does hybrid search (vector + keyword) come as standard or do you have to compose it yourself? Is metadata filtering first-class or an add-on? Does the system understand multi-modal (text + image) without reconfiguration? Does it support multi-region sharding? Each of these features, when not included, means real engineering work to build, maintain and monitor it.

**Cost model.** The monthly invoice price is only one of the three components of real cost. The second is the operating cost (DevOps, on-call, incident management). The third is the cost of migrating the day you need to change — and every 2026 benchmark agrees that changing vector database in production is not a weekend project, especially when the corpus has tens of millions of vectors to re-index.

With these axes in mind, the five options.

## pgvector — the Postgres extension

pgvector is an open-source PostgreSQL extension adding a `vector` data type and a handful of distance operators, along with ANN indexes (HNSW since version 0.5, IVFFlat before that). It is not a new database: it is Postgres doing more things. If your team already operates Postgres, pgvector appears as one more extension enabled with `CREATE EXTENSION vector`. There is no new service to monitor, no new credentials to manage, no new system to learn to debug.

The narrative that "Postgres is slow for vectors" comes from the IVFFlat index era and no longer holds. Recent public benchmarks show that with well-sized HNSW indexes pgvector competes seriously with dedicated systems up to volumes of around ten million vectors, and with the pgvectorscale extension (which adds DiskANN and quantised compression) the ceiling rises significantly. An April 2026 analysis over 50M vectors reports pgvectorscale at 471 QPS against Qdrant at 41 QPS under equivalent conditions — an order of magnitude in pgvector's favour. These numbers have to be taken with the usual caution about vendor benchmarks, but the general pattern is consistent: the gap between pgvector and dedicated systems has narrowed drastically in the last two years.

The conceptual advantage no dedicated system matches is **the ability to cross vector search with relational data in a single atomic query.** You want the chunks most similar to the client's brief filtered by sector, by year, by budget range, ordered by a combination of similarity and recency, with joins to the clients table — and all that in a single SQL statement with ACID guarantees. Pinecone and Qdrant can do the vector part well, but the relational data lives in another system and coherence between the two is your responsibility.

pgvector's ceiling arrives when the vector volume exceeds what fits comfortably in `shared_buffers` and the index starts doing disk I/O on every query, or when you need very specific features (hybrid search with tuned BM25, multimodal embeddings as first-class citizens, native multi-region sharding). In those situations, dedicated systems clearly win. For the rest, pgvector is probably the correct answer and almost nobody tells you because selling Postgres generates revenue for nobody.

## Qdrant — the open-source speed leader

Qdrant is a dedicated vector database written in Rust, open-source, with a very clear profile: pure performance and metadata filtering as first-class citizens. Its public benchmarks consistently show p50 latencies below 5 ms at high recall, and its pre-search filter system is genuinely the best among open-source options — a detail that matters more than it seems when real queries are almost never "the k nearest in absolute terms" but "the k nearest that satisfy these conditions".

Where Qdrant shines is read-heavy loads with complex filtering, where speed matters, and where the team is comfortable operating an additional service in their stack. Where it falls short is operations with non-vector data: there are no joins, no transactional guarantees over relational data outside the system, and integration with the rest of the backend depends on keeping two sources of truth coherent.

Its practical sweet spot runs from a few hundred thousand vectors up to several tens of millions. Below that range it is over-engineering. Above it, public reports suggest performance starts degrading in write-intensive scenarios, although for predominantly read loads it scales better than the documentation admits.

## Weaviate — native hybrid search

Weaviate is open-source, with a managed service available, and its hallmark is that hybrid search — combining vector similarity with BM25 over keywords — is native and well integrated, not a pattern you have to build yourself. It also brings integrated vectorisation modules: you can insert raw text and Weaviate calls OpenAI, Cohere or Hugging Face underneath to generate the embedding before storing it. For teams valuing "developer experience" over fine flexibility, this reduces real friction.

The price of that integration is that Weaviate's conceptual model is more opinionated than Qdrant's or pgvector's. There is a class schema defining how fields are vectorised, there is a GraphQL API as the primary interface, and there is a certain coupling with the ecosystem the Weaviate team designs. For projects where hybrid search is the centre of the product (legal documents, e-commerce with SKUs and descriptions, compliance where exact match on proper nouns matters as much as semantic similarity), Weaviate saves work. For projects where vector search is one more piece within a system with a lot of relational logic, that integrated opinion feels like a restriction.

## Milvus — billion scale

Milvus is the option designed from the start for extreme scale. Distributed architecture with separation between storage, computation and indexing; mature sharding; GPU support; backed by Zilliz, which offers the managed service. By GitHub stars it is the most popular open-source option, and its community of users at scale is the most active of the five.

For projects below a hundred million vectors, Milvus is oversized: the operational complexity of managing its multiple components (etcd, MinIO, several distinct services) is not justified for scales pgvector or Qdrant handle with a single process. But above a billion vectors, Milvus is one of the two or three options that genuinely work in production, and almost every publicly documented RAG architecture at that scale uses it or uses Vespa.

It is not the likely choice for the programme's project. It is worth knowing because it defines the real ceiling of the open-source market and because several use cases you will face in your career — if you work with very large e-commerce catalogues, IoT telemetry, or consumer-scale recommendation systems — end up there.

## Pinecone — the managed option

Pinecone is fully managed, proprietary, and sells "zero ops" as its central value proposition. You create a serverless index, upload the vectors, fire the queries. There are no servers to provision, no indexes to tune, no rebalancing to watch. For small teams where the cost of a senior engineer dedicated to vector-database DevOps exceeds the cost of the monthly invoice, this is exactly what you need to pay for.

The cost model has three components — storage at $0.33/GB/month, read units, write units — plus a $50 monthly minimum Pinecone introduced in 2025 which changed the calculation for very light loads. The thing no Pinecone tutorial tells you is that cost does not scale linearly with visible usage: read units are consumed faster than the documentation suggests, especially with metadata filters, and a recent benchmark over standard RAG loads reports that the real bill averages **between 2.5× and 4× Pinecone's own calculator estimate**. It is not exactly misleading advertising — it is that the workloads people deploy are different from those modelled in the calculator.

Up to around ten million vectors with moderate loads, Pinecone is competitive and sometimes cheaper than the self-hosted alternative once you add the real DevOps cost. From there, the economics change: at fifty or a hundred million vectors with sustained traffic, public reports consistently show that self-hosted Qdrant, Milvus or pgvector are 3× to 10× cheaper in TCO. For teams in regulated industries with data sovereignty requirements, Pinecone does not enter the conversation at all.

## Comparative summary

> *(Figures in the original: `articulo-02-figura-01-matriz-posicionamiento.jpg` and a comparison table image — not included in this repo.)*

The p50 latencies of the five well-tuned options are all in the 5 to 50 ms range for volumes up to 10M vectors, so the QPS differences published in benchmarks are not what decides. What decides are the four axes above, in this order: **operating model, expected practical scale, native features you will genuinely use, and cost model integrating the three components.**

## The programme's decision: pgvector

For the estimation system we build throughout the programme, pgvector is the choice. The reasons are four and it is worth articulating them precisely, because exactly the same reasons — or their absence — are what justify or reject pgvector in any project you evaluate outside the programme.

**First, alignment with the business backend's stack.** The programme's reference implementation uses Ruby on Rails over PostgreSQL for the business backend. The AI service can talk directly to that same PostgreSQL — or to a dedicated Postgres, but with the same technology — and exploit everything your team already knows how to operate. Had the choice been Qdrant or Pinecone, we would have added to the stack a new component with its own failure model, its own backups, its own monitoring, its own learning curve. For the project's volumes, that operational cost is not compensated by any technical advantage.

**Second, the simplicity of transactional joins.** The project constantly needs to cross vector search results with relational data: clients, sectors, dates, amounts. In pgvector that is an SQL query with a JOIN and a WHERE, atomic, with ACID guarantees. In any of the other four options it is coordination between two systems, with all the failure modes that coordination introduces. For a project that is by nature "RAG with many structured filters", this is decisive.

**Third, natural hybrid search.** PostgreSQL has had `tsvector` and `ts_rank` for decades. Combining vector similarity with full-text search in a single query is direct, with no bolt-on needed. Weaviate offers more sophisticated hybrid search, but the cost of adding an entire system to the stack just for that feature is not justified in this case.

**Fourth, expected scale.** The programme's project, replicated at a real client, is going to handle hundreds to a few thousand historical budgets. Each budget produces between 10 and 50 chunks. The reasonable total corpus is in the range of tens or hundreds of thousands of vectors. A hundred thousand vectors with HNSW run in pgvector with latencies of a few milliseconds on modest hardware. We are at least two orders of magnitude below any plausible ceiling.

This decision is not ideological. It is the correct answer for this specific project under these specific constraints. It is worth being very clear now about the conditions under which that correct answer would stop being correct.

## When this decision would stop being correct

If your project outside the programme resembles the estimation one — relational data with semantic search as an important but not exclusive piece, volumes in the range of tens to a few million vectors, a team familiar with Postgres — the decision is pgvector again. If it differs on any of the axes, it is worth reconsidering.

> *(Figure in the original: `articulo-02-figura-02-arbol-decision.jpg` — image not included in this repo.)*

- If your real volume is going to be consistently **above fifty million vectors**, evaluate Qdrant or Milvus before committing. pgvectorscale extends the ceiling but introduces its own operational complexity.
- If your product is predominantly search where **exact match on proper nouns, identifiers and SKUs** matters as much as semantic similarity, Weaviate or Elasticsearch with its vector support are probably a better investment than pgvector.
- If your team has **no experience operating PostgreSQL** but does have budget for SaaS and prioritises developer velocity, Pinecone is the rational choice, especially below ten million vectors where its cost is still competitive.
- If your system needs **native multi-region distribution** with a strict SLA, Pinecone or Astra DB are the serious options; pgvector can do it with Postgres replication but the pattern is more artisanal.
- If your team is building something with **multimodal embeddings** (text + image + audio) as first-class citizens, LanceDB or Marqo are specifically designed for that case; pgvector supports it but without the specific optimisations.

These are not binary conditions. The reality is that many projects start on pgvector during the prototype and migrate to a dedicated option when some axis crosses a concrete threshold. The migration is never trivial but neither is it impossible if the schema design and the data-access layer contemplate this possibility from the start — something we are going to do in the pre-session exercise and live.
