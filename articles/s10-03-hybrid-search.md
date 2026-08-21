---
title: Hybrid search
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 23 min
added: 2026-08-18
summary: >
  Semantic search is short-sighted about the literal — proper nouns, acronyms,
  versions and codes are the terms with least general semantic mass and most
  discriminating value, exactly the combination that survives compression into
  an embedding worst. Lexical search nails them and cannot read paraphrase.
  Build both in PostgreSQL (pgvector plus tsvector with a GIN index) and fuse
  them with Reciprocal Rank Fusion, which uses positions only and so sidesteps
  calibrating two incomparable score scales.
keywords: [hybrid search, lexical search, full-text search, tsvector, tsquery,
           GIN index, ts_rank, BM25, reciprocal rank fusion, RRF, score
           normalisation, proper nouns, stemming]
---

# Hybrid search

*Antonio Perez* · 🔴 23 min

A scene from the project-estimation system. The description of a new project arrives: a booking app needing *"payment integration with Stripe, including subscriptions and billing webhooks"*. The retrieval pipeline searches the historical budgets by semantic similarity and returns reasonable results: budgets for projects with payment gateways, recurring charges, financial integrations. All from the right semantic field.

But in the history there is a budget that integrated **exactly Stripe**, with its subscriptions and its webhooks, a year and a half ago. For estimating, that document is gold: it contains the real effort it took to wrestle with that particular API, its surprises and its line items. And it appears in position fourteen, behind half a dozen projects with other gateways.

Why? Because to an embedding model, "Stripe" is approximately synonymous with "payment gateway". That generalisation is precisely the virtue of embeddings — they understand that "recurring charges" and "subscriptions" are talking about the same thing — and here it is exactly the problem: **the proper noun, the exact term that distinguishes the perfect document from the merely similar ones, dissolves into a vector that averages the whole chunk's content. Semantic search is short-sighted about the literal.**

The irony is that the problem embeddings do not solve was solved by the previous generation of search technology: exact term matching. **Hybrid search consists of not choosing** — running both searches, semantic and lexical, and fusing their results into a single ranking. This article builds that idea from the ground up: what each family of search sees that the other does not, how to set up lexical search without leaving PostgreSQL, and how to fuse two rankings that do not speak the same language of scores.

## Two families of search, two blind spots

**Lexical search** — that of classic search engines — operates on the literal terms of the text. Its question is: which documents contain the query's words, and with what weight? Words that are rare in the corpus discriminate a lot (if "Stripe" appears in three documents out of a thousand, those three matter); ubiquitous words discriminate little. Decades of information retrieval were built on that intuition.

**Semantic search** operates on representations of meaning: query and documents are projected into a vector space where closeness approximates topical affinity, whatever the exact words say.

Each is blind exactly where the other sees:

**Lexical does not understand paraphrase.** "Recurring charges" and "payment subscriptions" share not one word; to lexical search they are unrelated queries. A budget history written by different people over years is full of these variations: what one author called "admin panel" another called "backoffice". Semantics crosses those variations effortlessly.

**Semantics dilutes the literal.** Proper nouns, acronyms, versions, internal codes: "Stripe", "SAP", "ISO 27001", "PostGIS". They are the terms with the least general semantic mass and the most discriminating value in a technical corpus — the exact combination that survives compression into an embedding worst. When the query asks for a specific identifier, lexical search nails it and semantic search approximates it.

In an estimation system both types of query coexist — conceptual descriptions of projects and mentions of concrete technologies — and very often within the same query. The conclusion is not to choose better between families: it is **to stop choosing.**

## Full-text search in PostgreSQL: the piece you already have

The instinctive reaction on hearing "keyword search in production" is to think of Elasticsearch. Resist the instinct for a moment: if the system's vectors already live in PostgreSQL, PostgreSQL itself ships a mature full-text search engine, and using it means **zero new infrastructure, zero synchronisation between stores, and both searches one SQL query away.**

The pieces of full-text in PostgreSQL:

**`tsvector`: the preprocessed document.** A `tsvector` is a text's representation optimised for search: tokenised, lowercased, stripped of stopwords, and with each word reduced to its root (stemming) — "integrations", "integration" and "integrate" collapse to the same root and find each other. That normalisation depends on the language, which is why the linguistic configuration is not a detail: a corpus of Spanish budgets must be processed with the `'spanish'` configuration for stemming and stopwords to be correct. Terms the dictionary does not recognise — "Stripe", "webhook" — pass through almost intact, which is exactly what we want: they are the identifiers we came to find.

**`tsquery`: the preprocessed query.** The user's query goes through the same normalisation so both sides speak the same language of roots. The `websearch_to_tsquery` function accepts natural search-engine syntax (loose terms, quotes for phrases, OR) and tolerates imperfect input, which makes it the sensible option when the query comes from a user or from free text.

**The GIN index and `ts_rank`.** The `@@` operator checks whether a `tsvector` satisfies a `tsquery`; the GIN index (Generalized Inverted Index — an inverted index, the classic structure of search engines: from each term to the documents containing it) makes that fast over millions of rows; and `ts_rank` scores each match by term frequency and proximity so results can be ordered.

In practice the piece is set up with a generated column — PostgreSQL keeps the `tsvector` synchronised with the content automatically, with no triggers and no application code — and its index:

```sql
ALTER TABLE budget_chunks
ADD COLUMN content_tsv tsvector
GENERATED ALWAYS AS (to_tsvector('spanish', content)) STORED;

CREATE INDEX ix_budget_chunks_content_tsv
ON budget_chunks USING gin (content_tsv);
```

And the lexical search is one query:

```sql
SELECT chunk_id, ts_rank(content_tsv, query) AS lexical_rank
FROM budget_chunks,
     websearch_to_tsquery('spanish', :query_text) AS query
WHERE content_tsv @@ query
ORDER BY lexical_rank DESC
LIMIT 50;
```

> *(Figure in the original: `articulo-03-figura-01-arquitectura-hibrida.jpg` — image not included in this repo.)*

Two honesties before going on. **First:** `ts_rank` is not BM25, the lexical ranking algorithm that is the de facto standard in dedicated search engines, and it is somewhat cruder at scoring — it does not normalise by document length with the same sophistication. PostgreSQL extensions exist that add BM25, but for a company corpus of thousands or tens of thousands of chunks, the difference between `ts_rank` and BM25 is noise compared with the gain of having a lexical branch at all. **Second:** Elasticsearch (or OpenSearch) still has its place — enormous corpora, advanced lexical needs like custom analysers or fuzzy typo-tolerant search, teams already operating it. This article's position is not "never Elasticsearch"; it is **"do not add a second data store until the first one becomes too small for you, because every extra store is synchronisation, monitoring and new failure modes."**

## The problem of joining two rankings

We now have two searches, each returning its top-50: the semantic one with its cosine distances, the lexical one with its `ts_rank` scores. And here a subtler problem than it appears surfaces: **the two scores are not comparable.** Cosine similarity lives in a bounded range with its own distribution; `ts_rank` produces values on a completely different scale with no intuitive upper bound. Adding them directly is adding metres to kilograms.

The engineer's temptation is to normalise: rescale both scores to a common range and combine them with weights. It works in the demo and breaks in production, because **the distribution of the scores changes with every query** — a query with very rare terms produces sky-high lexical scores; a conceptual one, very low — and a normalisation calibrated with yesterday's queries is out of calibration with today's. Maintaining that calibration is permanent work nobody asked for.

The elegant solution sidesteps the problem entirely: **ignore the scores and use only the positions.**

## Reciprocal Rank Fusion: fusing by consensus

**Reciprocal Rank Fusion (RRF)** fuses rankings with a one-line rule: each document receives, from each ranking it appears in, a score inversely proportional to its position, and the scores are summed.

```
rrf_score(d) = Σ 1 / (k + rank_i(d))
```

where `rank_i(d)` is the document's position in ranking *i* (starting at 1) and *k* is a smoothing constant, typically 60. The final score of a document depends only on which positions it landed in — never on each engine's raw scores, which is exactly what makes them incomparable.

Seeing it with numbers clarifies why it works. With k = 60:

- A budget that comes 2nd in semantic search and 5th in lexical: `1/62 + 1/65 ≈ 0.0315`.
- A budget that comes 1st in semantic but does not appear in lexical: `1/61 ≈ 0.0164`.

**The document both searches consider good beats the champion of a single one.** RRF is, in essence, a machine for rewarding consensus: appearing reasonably high in several rankings is worth more than dominating one. For the Stripe budget this is exactly the rescue we needed — the lexical branch places it high for the exact term, the semantic one keeps it respectable for the topic, and the fusion lifts it into the leading positions of the combined ranking.

> *(Figure in the original: `articulo-03-figura-02-calculo-rrf.jpg` — image not included in this repo.)*

The constant *k* deserves thirty seconds of attention because it is the technique's only dial. With small *k*, the top positions dominate the fusion (the difference between 1/1 and 1/2 is enormous); with large *k*, positional differences flatten and the fusion becomes more democratic across the whole top. The value 60 comes from the original paper that proposed the technique and has proved robust across very different domains; changing it is rarely the lever that moves quality, and starting by tuning it is premature optimisation.

The implementation fits in a pure function:

```python
# app/generation/rag/retrieval/fusion.py

from collections import defaultdict

RRF_SMOOTHING_K = 60

def reciprocal_rank_fusion(
    rankings: list[list[str]],
    k: int = RRF_SMOOTHING_K,
) -> list[str]:
    """Fuse multiple ranked lists of chunk ids into a single ranking."""
    scores: dict[str, float] = defaultdict(float)

    for ranking in rankings:
        for rank, chunk_id in enumerate(ranking, start=1):
            scores[chunk_id] += 1.0 / (k + rank)

    return [
        chunk_id
        for chunk_id, _ in sorted(scores.items(), key=lambda item: item[1], reverse=True)
    ]
```

Note that it receives a *list of rankings*, not exactly two. That is deliberate: **RRF neither knows nor cares how many sources it fuses**, and that generality makes it the pipeline's universal fusion piece — today it fuses the semantic branch with the lexical one; the day the system produces results from several parallel searches, the same function will fuse them without changing a line.

## Hybrid search in the AI service

Orchestrating the hybrid is composing the above. The two branches are independent queries to the same database, so the natural implementation in an asynchronous service is to launch them in parallel: **the hybrid's latency is that of the slower branch, not the sum of both.**

```python
# app/generation/rag/retrieval/hybrid_search.py (fragment)

import asyncio

async def hybrid_search(self, query: str, limit: int = 50) -> list[RetrievedChunk]:
    """Run semantic and lexical search in parallel and fuse with RRF."""
    semantic_results, lexical_results = await asyncio.gather(
        self._vector_search.search(query, limit=limit),
        self._fulltext_search.search(query, limit=limit),
    )

    fused_ids = reciprocal_rank_fusion(
        [
            [chunk.id for chunk in semantic_results],
            [chunk.id for chunk in lexical_results],
        ]
    )

    chunks_by_id = {
        chunk.id: chunk
        for chunk in [*semantic_results, *lexical_results]
    }
    return [chunks_by_id[chunk_id] for chunk_id in fused_ids[:limit]]
```

The contract is the same as that of any other search in the system: a query goes in, an ordered list of chunks comes out. **That uniformity is an architectural decision, not a coincidence** — when every retrieval strategy respects the same contract, switching from vector search to hybrid is swapping one piece for another behind a configuration, and comparing them becomes a one-boolean experiment instead of a three-week branch. The hybrid produces a ranking which, like any other, can feed the generator directly or pass first through a fine reordering stage; it is one more composable piece of the pipeline, not a different pipeline.

The hybrid's operational cost is modest and worth naming precisely: one additional SQL query per search (in parallel, so the latency impact is small), a generated column that fattens the table and is recomputed on every content write, and a GIN index that occupies space and is maintained on every insert. In a company budget corpus, all of that is small change. **The real cost is where it always is: one more piece to understand, configure and debug.**

## When hybrid wins — and when it adds nothing

Hybrid search is not better than semantic in the abstract; it is better on a concrete query profile, and it is honest to delimit it.

**Where it clearly wins:** queries with exact identifiers — technologies, products, acronyms, standards, client names. In an estimation system this is not the rare case but daily bread: project descriptions are strewn with "Stripe", "Salesforce", "GDPR", "React Native". It also wins on short, specific queries, where there is little semantic signal to exploit and every literal term counts.

**Where it barely moves the needle:** purely conceptual, well-paraphrased queries — "internal process digitalisation project with approval flows" — where semantics was already doing a good job and the lexical branch returns more or less the same in another order. In these cases the fusion does not get in the way (RRF degrades gracefully: if both rankings agree, so does the fused one), but it does not shine either.

**Where it needs watching:** if corpus and queries are in mixed languages — Spanish budgets riddled with English terminology, as is the norm in the sector — the `tsvector`'s linguistic configuration will process one part well and leave the other without stemming. It is not usually serious (English technical terms function as exact identifiers, which is the case where lexical shines), but it is the kind of detail that explains baffling results and is worth knowing before debugging them blind.

The question of whether hybrid pays **in your system, with your queries** has the same answer as any question of this type: measure against a fixed reference, configuration against configuration, and let the numbers decide. What this article contributes is the conviction that, in technical domains with their own vocabulary, the a priori bet is clearly favourable — and the cost of checking, an afternoon.

## Stop choosing

The idea to take away: semantic and lexical search do not compete, they cover each other's blind spots. Semantics understands paraphrase and lets identifiers slip; lexical nails identifiers and does not understand paraphrase. PostgreSQL offers both over the same table — pgvector for one, `tsvector` with GIN for the other — and Reciprocal Rank Fusion joins them without the swamp of calibrating incomparable scores: positions only, rewarding the document both searches respect.

In the live session, hybrid search will be one of the pieces we work on in the estimation system: we will watch it rescue, live, budgets that purely semantic search left buried, and check with real domain queries where it makes the difference and where it does not.
