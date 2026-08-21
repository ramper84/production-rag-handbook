---
title: "Retrieval that is not only cosine: top-K, threshold and filters over pgvector"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 9
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 30 min
added: 2026-08-14
summary: >
  Two failure modes of a top-K-only retriever — ten results from one document,
  and ten uniformly mediocre ones — and the two levers that fix them. Threshold
  set from the empirical distance distribution rather than intuition, soft-fail
  with `low_confidence` when nothing clears it, three metadata-filter strategies
  (pre / post / in-query) with the selectivity rule that chooses between them,
  four named anti-patterns, and the argument for precision over recall when the
  answer has economic consequences.
keywords: [top-k, similarity threshold, distance distribution, soft-fail,
           low confidence, pre-filtering, post-filtering, iterative scan,
           selectivity, recall vs precision, lost in the middle, operator class]
---

# Retrieval that is not only cosine: top-K, threshold and filters over pgvector

*Antonio Perez* · 🔴 30 min

At the close of Session 08 you had built a search endpoint that receives a query vector and returns the K most similar chunks according to pgvector's `<=>` (cosine distance) operator over your HNSW index. It is exactly what a textbook didactic RAG system does and, during the first tests with invented queries on the seed, it works reasonably. The friction appears the first time you pass it the output of your new `query_reformulator.py` over a real transcript, and looking at the results you get one of two surprises.

**The first surprise is clone results.** You asked the retriever for the ten most similar chunks for a B2B fintech marketplace transcript; inspecting the response, you discover that eight of the ten chunks belong to the same historical budget. The system found a single genuinely similar project in the database, and since top-K=10 demands ten results, the remaining positions are filled with chunks from that same project. The estimate you are about to generate is grounded in one historical case, not ten, and that is not what the system promises.

**The second surprise is the opposite** and appears when the corpus contains no genuinely similar budgets. You asked for ten results and the retriever returned ten, but looking at the cosine distances you realise they are all compressed around 0.7-0.8 in pgvector: none is really similar, all are mediocre. The retriever has kept its literal promise — "the ten most similar" — but the retrieved chunks are not going to ground anything, because they do not resemble the query enough. The estimate generated from that context will be a hallucination propped up on budgets that are generically alike but not really comparable.

Both surprises have a common cause: the retriever is using a single lever, top-K, and lacks discipline over two critical dimensions. The first, **minimum quality**: no chunk below a similarity threshold should enter the context, even if that means returning fewer than K results. The second, **structural filtering**: not every historical chunk is a candidate for every query; a retail project from 2018 should not compete with a fintech one from 2024 when the transcript explicitly talks about healthcare. This article adds those two levers and shows how they fit inside your AI service's retriever using the operators pgvector already gives you.

## Top-K: the obvious lever and its two biases

top-K is the lever that ships with any vector database and the only one the Session 08 system exposes. It defines how many chunks we want the retriever to return, ordered by ascending distance. Apparently there is not much to discuss: ask for five, get five; ask for twenty, get twenty. The real discussion is *how many to ask for, and why*.

Many students' reflex on seeing the first poor results is to raise K. The intuition is reasonable: if there is nothing good among the ten, perhaps there is among the twenty. And sometimes it works: in large corpora with ambiguous queries, widening K brings diversity and sometimes rescues relevant chunks that were "further down the queue". But in production, raising K by reflex leads to three consequences with measurable cost.

**First, the generation prompt's cost multiplies almost linearly with K.** Each chunk entering the context is between 200 and 400 tokens; going from K=10 to K=30 adds between four and eight thousand tokens to the generation prompt. If your gpt-5 call to generate the estimate costs fifty cents per request at K=10, at K=30 it costs a euro and a half. Multiply that by thousands of requests a month.

**Second, noise in the context degrades the answer's quality rather than improving it.** Modern models suffer the "lost in the middle" phenomenon — which Article 4 examines in depth — and when you put thirty partially relevant chunks in the prompt, the model tends to ignore most of them and lean disproportionately on the first and the last. The critical chunk that sat at position fourteen will probably receive less attention than the irrelevant chunk at position one.

**Third, raising K hides the real problem.** If your retriever only finds three genuinely similar chunks and you ask it for ten, you are not doing better retrieval; you are doing retrieval of the same three chunks with seven distractions. The system is telling you "your corpus has no more relevant material for this query", and the correct response is not to disguise that signal with low-quality results, but to propagate it forward so the generation stage knows.

The programme's operating rule is to keep K at a moderate, stable value — ten is reasonable for the project's case — and let the threshold take care of discarding results that do not deserve to enter the context. **The correct number of chunks per request is not a fixed retriever parameter, it is an emergent of the threshold applied over the K candidates**: sometimes it will be ten, sometimes three, sometimes zero. And that last one — zero — is valid information, not a system failure.

## Threshold: the missing discipline

Threshold is the filter that decides which distance is low enough for a chunk to deserve entering the context. In pgvector, where the `<=>` operator produces cosine distance between 0 (identical vectors) and 2 (opposite vectors), a typical threshold for OpenAI-embedding corpora would sit between 0.5 and 0.7: below that, the chunk is reasonably similar to the query; above it, it is tangential content that will probably confuse the model more than help it.

**The threshold's exact number is not decided by intuition.** It is decided by looking at the empirical distribution of distances over real queries against your corpus. The procedure is direct: take twenty or thirty representative transcripts, pass them through your reformulator, run the search with K=50 and no threshold, and plot the resulting distances. You will see two recognisable patterns. A group of chunks with low distances (typically between 0.3 and 0.5 for your corpus) that on manual inspection are genuinely relevant to the query. And a larger group with distances around 0.7-0.9 that on manual inspection are noise. **The threshold goes in the valley between the two groups**; for `text-embedding-3-small` over a reasonably specialised corpus, that valley usually falls around 0.6-0.65. The live session will do exactly that exercise over your own corpus.

> *(Figure in the original: `art_3_figura-7-distribucion-distancias.jpg` — image not included in this repo.)*

Once the threshold is fixed, the search endpoint applies it as an extra `WHERE` on the SQL you already had:

```sql
SELECT
    c.id,
    c.content,
    c.metadata,
    c.embedding <=> :query_embedding AS distance
FROM chunks c
WHERE c.embedding <=> :query_embedding < :distance_threshold
ORDER BY c.embedding <=> :query_embedding
LIMIT :top_k;
```

There is an important operational detail in this query. The `WHERE` clause evaluates the distance, and that means pgvector computes it twice per candidate if you are not careful — once for the `WHERE` and once for the `ORDER BY`. In practice Postgres's planner is clever and reuses the computation when the HNSW index is properly configured with the right operator class (`vector_cosine_ops`, aligned with the query's `<=>` operator — the antipattern we covered in Session 08). If through carelessness the index's operator class were `vector_l2_ops` and the query used `<=>`, the index would not apply, the planner would fall back to a sequential scan, and `EXPLAIN ANALYZE` would tell you the search takes seconds instead of milliseconds. **Keeping the index's operator class aligned with the query's operator is the precondition for any threshold adjustment to behave as expected.**

The retriever's behaviour when no chunk clears the threshold deserves an explicit decision. The option the programme adopts is **soft-fail**: the endpoint returns an empty list and a `low_confidence: true` field in the response body. The orchestrator, seeing the empty list, does not call the generator with an empty context — that only invites the model to hallucinate — but produces a direct response to the business backend along the lines of *"there is not enough evidence in the historical corpus to estimate this project; the commercial team should review it manually"*. **This semantics is what distinguishes a serious RAG system from a didactic one: the system recognises its limits and communicates them upward, instead of always generating something that looks like an estimate.**

An alternative some systems adopt is to relax the threshold dynamically when the initial search returns fewer than a minimum number of results. The retriever starts with a strict threshold (0.55), and if it returns fewer than three chunks, relaxes it to 0.65 and repeats. It is a valid optimisation but introduces non-determinism and makes debugging harder; the programme does not adopt it by default but leaves it as an optional improvement in the live session.

## Metadata filters: three strategies in pgvector

Vector similarity captures semantic resemblance but ignores structural data that can be critical. When the transcript explicitly talks about the healthcare sector, you do not want the retriever returning chunks from retail budgets even if they are semantically close in some respects. When the client asks for a project starting next year, you do not want to anchor the estimate on 2019 budgets with already-obsolete technologies. Structural metadata filters solve exactly this: they add `WHERE` clauses over fields of your tables that restrict the universe of candidates before, during or after the vector search.

There are three operational patterns for applying these filters, and it is worth knowing all three because the choice between them affects both query performance and effective recall.

**Pre-filtering** applies the structural clauses before the vector search: pgvector first restricts the candidate universe according to the `WHERE`, and only then applies the distance operator over the resulting subset. The SQL looks like this:

```sql
SELECT
    c.id,
    c.content,
    c.metadata,
    c.embedding <=> :query_embedding AS distance
FROM chunks c
JOIN documents d ON c.document_id = d.id
WHERE d.sector = ANY(:sectors)
  AND d.project_year >= :year_min
  AND c.embedding <=> :query_embedding < :distance_threshold
ORDER BY c.embedding <=> :query_embedding
LIMIT :top_k;
```

Pre-filtering is the right option when the filter has **high selectivity**, that is, when it reduces the corpus to a small subset before searching. If your `sector = 'healthcare' AND project_year >= 2022` clause leaves only 5% of your chunks, pgvector runs the vector search over 50 chunks instead of 1000 and the milliseconds show.

For years, the operational warning was that pre-filtering destroyed the HNSW index: the execution plan fell back to a sequential scan because the index did not know how to combine vector similarity with the structural filter. In modern pgvector versions (from 0.7), the situation has changed significantly thanks to **iterative scans over filtered HNSW**, which let the index navigate the graph discarding candidates that do not satisfy the `WHERE`. The behaviour is no longer optimal in every case but stops being catastrophic, and for selectivities below 20% the performance is reasonable. The programme adopts pre-filtering as the default strategy on the basis of this behaviour.

**Post-filtering** inverts the order: first you search vectorially with a widened K, and then apply the `WHERE` to discard those that fail the structural criteria:

```sql
WITH top_candidates AS (
    SELECT
        c.id,
        c.content,
        c.metadata,
        c.document_id,
        c.embedding <=> :query_embedding AS distance
    FROM chunks c
    WHERE c.embedding <=> :query_embedding < :distance_threshold
    ORDER BY c.embedding <=> :query_embedding
    LIMIT :wide_k
)
SELECT t.*
FROM top_candidates t
JOIN documents d ON t.document_id = d.id
WHERE d.sector = ANY(:sectors)
ORDER BY t.distance
LIMIT :top_k;
```

Post-filtering is the right option when the filter has **low selectivity** and the HNSW index works better unrestricted. If your clause only removes 10% of the corpus, pre-filtering barely reduces the index's work and does constrain the navigation graph; in that case it is preferable to search first with `wide_k = top_k × 3` or thereabouts, and then discard the few that fail. **Post-filtering's risk is losing recall when the filter is very selective**: if your `wide_k` is 50 but only 2 of those satisfy the filter, you have failed and should have raised `wide_k` further; without instrumentation, you never find out.

**In-query filtering** is the modern fusion pgvector enables through iterative scans: the query is written as pre-filtering but the optimiser internally decides the best strategy. The `WHERE` clause mixes structural and distance filters, and pgvector evaluates the filtered HNSW index or the sequential scan as a function of estimated selectivity. For the project's case this is in fact the query you run in production; "pre" vs "post" becomes an internal planner decision, not something you have to write yourself.

On the project's concrete schema, the four most useful filters worth exposing in the retrieval API are: `sectors` (list of acceptable sectors), `project_year_range` (year range, to limit to recent budgets), `tech_stack` (mentioned technologies, using the JSONB `@>` operator over the metadata field), and `chunk_types` (limit to certain chunk types from the Session 07 schema: `scope_block`, `line_item`, `phase`). The SQL with all four integrated:

```sql
SELECT
    c.id,
    c.content,
    c.chunk_type,
    c.metadata,
    c.embedding <=> :query_embedding AS distance
FROM chunks c
JOIN documents d ON c.document_id = d.id
WHERE (:sectors IS NULL OR d.sector = ANY(:sectors))
  AND (:year_min IS NULL OR d.project_year >= :year_min)
  AND (:year_max IS NULL OR d.project_year <= :year_max)
  AND (:tech_filter IS NULL OR c.metadata @> :tech_filter::jsonb)
  AND (:chunk_types IS NULL OR c.chunk_type = ANY(:chunk_types))
  AND c.embedding <=> :query_embedding < :distance_threshold
ORDER BY c.embedding <=> :query_embedding
LIMIT :top_k;
```

The `(:filter IS NULL OR ...)` pattern lets filters be optional: if the reformulator did not extract a sector from the transcript, the field arrives `null` and the filter is ignored; if it did, it applies. **This chains directly into Article 2's structured output: each field of the Pydantic schema maps to an optional retriever filter, and coherence between the two layers is maintained by construction.**

> *(Figure in the original: `art_3_figura-8-pre-vs-post-filtering.jpg` — image not included in this repo.)*

## Anti-patterns the system invites you to commit

There are four recurring anti-patterns in RAG systems worth naming so you recognise them when they appear, because they will appear.

**The first is raising K to fix quality.** We discussed it above but it bears repeating: if your results are bad, the solution is not to bring more; it is to bring better. If three chunks are relevant and the other seven are noise, adding noise to the context does not improve the answer. Raising K only makes sense when a manual inspection confirms there are relevant chunks beyond the top-10 that the system is leaving out.

**The second is trusting the LLM as the final filter.** The narrative is seductive: "we put twenty chunks in the context and let the model choose the relevant ones". In practice the model is not a good retriever: it does not systematically compare chunks against each other, it does not reject irrelevant information with discipline, and under narrative pressure it ends up synthesising information from chunks it should have ignored. **Filtering happens in the retriever; the LLM synthesises, it does not filter.**

**The third is omitting the threshold because "there's almost always something in the corpus".** The assumption is sometimes correct — dense corpora with wide coverage rarely produce empty searches — but as an architectural decision it is fragile. The day the assumption fails, the system will generate an estimate based on irrelevant chunks and nobody will notice until a client questions the result. Having the threshold as part of the retriever's contract from day one means that the first time a project is genuinely new, the system recognises it explicitly.

**The fourth is mixing chunk_types without filtering.** Your Session 07 chunk schema distinguishes types (a `scope_block` describes a functional block, a `line_item` describes a budgeted task). A search that mixes the two can retrieve chunks of different types with similar scores but different utility for generation. If the reformulator's query is clearly about "cost estimation", you probably want `line_item`; if it is about "functional scope", probably `scope_block`. Filtering by type when the reformulator gives clear hints is free and improves precision.

## Recall vs precision: the real trade-off

Behind all the decisions above lies a deeper debate worth naming: the system has to choose between **recall** (retrieve everything potentially relevant, even if it includes some noise) and **precision** (retrieve only what is clearly relevant, even if it leaves out a gem). The two cannot be maximised simultaneously; raising one lowers the other.

In didactic RAG systems the convention is to prioritise recall: bring more so the LLM has material to work with. In production RAG systems, especially when the LLM will synthesise an answer with economic consequences — and a software estimate is one — the correct operational position is **to prioritise precision**: bring less but better, accept that sometimes the system answers "I do not have enough evidence", and leave the mechanisms for raising recall without sacrificing precision to later stages (reranking in Session 10, hybrid search in the same session).

The reason is this: **a hallucination propped up on partially relevant chunks is more dangerous than an honest "I don't know".** A system that produces a €250,000 estimate for a project where it has no solid evidence is creating an expectation that neither the company nor the client can honour. A system that says "I do not have sufficiently similar budgets; review this manually" preserves trust in the estimates it does produce. **The asymmetry between the two errors is what justifies favouring precision over recall in this domain.**

> *(Figure in the original: `art_3_figura-9-impacto-k-threshold.jpg` — image not included in this repo.)*

## Connection with the live session

The session's fourth block is an iteration over the retriever's parameters. We are going to take the same ambiguous transcript from previous articles and explore three axes: how the retrieved chunks vary as `top_k` changes between 3 and 30, how the set changes as the threshold is adjusted between 0.5 and 0.8, and how structural filters (sector, year, chunk type) affect recall and latency. Each combination will produce a different result and we will measure three observable metrics: the number of chunks effectively returned, the percentage of chunks belonging to the correct sector, and the median query latency over the seed you have loaded.

The iteration's goal is not to find the "optimal" parameters — that search is partly folklore in RAG: what is optimal on today's corpus may not be on your corpus in six months — but to internalise the system's sensitivity to each lever. When you finish the session you should be able to predict, given a threshold change from 0.6 to 0.5, what will happen to the cost per request and what will happen to the rate of `low_confidence` responses.

There is also a productive debate worth anticipating. Some students will arrive defending the pattern opposite to the one the programme adopts: prioritise recall, bring thirty chunks, let the LLM filter. The discussion we will have is not theoretical; it is based on the module's empirical evidence. We will run both strategies over the same transcript and inspect the two generated estimates: with the high-recall one you will see the concrete problems (lost-in-the-middle, cross-contamination between projects), with the high-precision one you will see its own (insufficient coverage when the corpus is sparse). Neither is universally correct; the programme's is the one that best fits the estimation use case, but the other side's arguments are legitimate and it is worth articulating them in order to defend your own decision with a basis.

What closes this article, and connects with the next, is an idea: however well you tune the retriever, the chunks it returns will always reach the LLM as a block of text the model has to understand. The way you assemble that block — what order, what delimiters, what metadata accompanies each chunk, what the prompt does with all of it — is what determines whether retrieval quality translates into estimate quality or is wasted. Article 4 goes deep into that augmentation stage.
