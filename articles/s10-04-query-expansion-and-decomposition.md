---
title: Query expansion and decomposition
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 22 min
added: 2026-08-21
summary: >
  Every other retrieval improvement acts after the query; this one acts on the
  query itself. Expansion generates alternative formulations of one intent and
  insures against the formulation lottery; decomposition splits a multi-topic
  query into one focused sub-query per workstream. They are not
  interchangeable, and they must not fuse the same way — expansion rewards
  consensus (RRF), decomposition must guarantee per-topic coverage
  (round-robin), or the majority topic floods the context. The bill is an LLM
  generation on the critical path of every search, and the largest mitigation
  is not applying the technique when the query does not need it.
keywords: [query expansion, multi-query, query decomposition, sub-queries,
           structured output, Pydantic, reciprocal rank fusion, round-robin
           interleaving, deduplication, asyncio.gather, latency, query routing]
---

# Query expansion and decomposition

*Antonio Perez* · 🔴 22 min

Up to now, every retrieval improvement one tends to consider acts *after* the query: better indexes, better rankings, better filters. This article looks at the other side of the counter, because in the project-estimation system there is a problem no amount of document-side improvement can fix: the query itself.

The concrete case. The system's usual input is not a clean search-box query — it is a forty-minute meeting transcript. The client starts talking about the product catalogue, jumps to the fact that they also want a mobile app, spends ten minutes on the integration with their billing system, and somewhere in the middle the admin panel and the monthly reports appear. When that transcript (or its summary) is turned into a search query, its embedding is the average of all those topics at once: a vector that is moderately close to everything and genuinely close to nothing. The search returns budgets "for big projects with lots of things in them", which is the worst possible category for estimating, because estimates are built line item by line item: the catalogue is estimated against catalogue references, the billing integration against integration references.

There is a second, subtler problem, and it affects even single-topic queries: **the formulation lottery**. The client says *"let the sales reps see their numbers from their phones"*; the relevant historical budget said *"KPI dashboard with a responsive version"*. Same concept, different vocabularies, and although embeddings cross paraphrase better than any earlier technology, they are not immune: the query's concrete wording decides which neighbours it has in the vector space, and an unlucky wording retrieves worse than a lucky one. Retrieval quality depending on the luck of the drafting is exactly the kind of fragility a production system cannot afford.

The two techniques in this article attack these two problems by reformulating the query before searching — and they are worth learning together precisely because they are easy to confuse and are not interchangeable.

## Two techniques that look like one

**Query expansion (multi-query)** generates several alternative formulations of the same intent and searches with all of them. If the original query is *"let the sales reps see their numbers from their phones"*, the variants might be *"metrics dashboard for mobile devices"*, *"sales KPI visualisation in a mobile application"* and *"sales reports accessible from a smartphone"*. One of them will land near the relevant budget even if the original did not. Expansion is insurance against the formulation lottery: instead of one ticket, you play four.

**Query decomposition** splits a query that mixes several intents into independent sub-queries, one per topic. The client's transcript becomes *"product catalogue with inventory management"*, *"mobile application for customers"*, *"integration with billing system"* and *"admin panel with reporting"*. Each sub-query has a sharp embedding pointing at its own corner of the vector space, and each one retrieves budgets from its topic — which is how estimating actually works.

The question that separates them fits on one line: **does the query ask for one thing that can be said in many ways, or many things said at once?** The former is expanded; the latter is decomposed. Applying the wrong technique is not neutral: expanding a multi-topic query produces four equally mixed variants (four tickets for the wrong draw), and decomposing a single-topic query manufactures artificial sub-topics that retrieve noise.

> *(Figure in the original: `articulo-04-figura-01-dos-caminos.jpg` — image not included in this repo.)*

## Generating the variants: an LLM on a short leash

Who writes the variants or the sub-queries? Years ago this was done with thesauri and rules; today the natural answer is an LLM, which understands the query and can reformulate or slice it with judgment. But "just call the LLM" is the naive version: the production version demands two disciplines.

**First discipline: structured output, not free text.** The sub-queries are input to the pipeline's next stage, not prose for a human. Asking for them as text and parsing them with regular expressions is manufacturing a breaking point. The output is defined as a schema and the model is required to comply:

```python
# app/generation/rag/retrieval/query_expansion.py

from pydantic import BaseModel, Field


class SubQuery(BaseModel):
    """A self-contained search query targeting a single workstream."""

    topic: str = Field(description="Short workstream label, e.g. 'billing integration'")
    query: str = Field(description="Standalone search query for this workstream")


class QueryDecomposition(BaseModel):
    """Decomposition of a project description into independent sub-queries."""

    sub_queries: list[SubQuery] = Field(min_length=1, max_length=4)
```

**Second discipline: instructions that fence in, not instructions that inspire.** The specific risk of putting an LLM to work rewriting queries is that it "improves" too much: that it invents requirements the client never mentioned, that it translates domain terminology into generic synonyms, or that it fabricates eight sub-queries where there were two topics. Every one of those creative touches contaminates retrieval downstream. The instructions should read like a short leash:

```python
DECOMPOSITION_INSTRUCTIONS = """
You split a software project description into independent search queries,
one per distinct workstream, to retrieve similar past project budgets.

Rules:
1. Produce at most 4 sub-queries. Fewer is better than fragmented.
2. Each sub-query must be self-contained and understandable without the others.
3. Preserve the exact domain terms used in the description (product names,
   technologies, acronyms). Never replace them with generic synonyms.
4. Never add requirements, features or technologies that the description
   does not mention.
5. If the description covers a single topic, return exactly one sub-query
   that rephrases it cleanly.
"""


async def decompose_query(self, raw_query: str) -> list[SubQuery]:
    """Split a multi-topic query into focused sub-queries."""
    response = await self._client.responses.parse(
        model=settings.query_expansion_model,
        instructions=DECOMPOSITION_INSTRUCTIONS,
        input=raw_query,
        text_format=QueryDecomposition,
    )
    sub_queries = response.output_parsed.sub_queries
    logger.info(
        "query_decomposed",
        sub_query_count=len(sub_queries),
        topics=[sub_query.topic for sub_query in sub_queries],
    )
    return sub_queries
```

Two details in that code are decisions, not accidents. The limit of four sub-queries lives in two places — in the schema (`max_length=4`, which the model cannot violate) and in the instructions (which explain *why*) — because the schema guarantees and the instruction orients, and in production you want both. And the model is chosen by configuration: reformulating queries does not need the most capable model in the catalogue, it needs the fastest one that does a small, tightly bounded task well — **this call sits on the critical path of every search, and every point of spare capability is paid for in latency.**

Multi-query expansion is the same pattern with different instructions (generate N alternative formulations of the same intent, preserving the exact domain terms); it does not deserve separate code.

## Fusing without losing sight of what you were searching for

Each sub-query produces its own ranking of budgets. One step remains: turning N rankings into the single set the rest of the pipeline will consume. And here there is a subtlety almost all introductory material glosses over: **expansion and decomposition must not fuse the same way, because they are chasing different things.**

In **expansion**, the N variants were all looking for the same thing. A budget that appears well positioned across several variants is a strong relevance signal, so the correct fusion rewards consensus. The natural tool is position-based fusion — Reciprocal Rank Fusion style: each document accumulates a score inversely proportional to its position in every ranking it appears in — which floats up the documents all formulations respect.

In **decomposition**, the N sub-queries were deliberately looking for different things. Rewarding consensus here sabotages the objective: a catalogue budget will never appear in the billing-integration ranking, and if we fuse by global consensus, the topic with the most budgets in the history will flood the result and the minority topics will be left unrepresented — exactly the problem we were running away from. The correct fusion for decomposition **guarantees per-topic coverage**: an allocation of the context budget across sub-queries (the top two from each, say) or a round-robin interleaving that takes the next best from each ranking in turn. For estimating, this is not a technical nuance: it is the difference between a context with references for every line item of the project and a single-topic context.

```python
# app/generation/rag/retrieval/fusion.py (fragment)

def interleave_rankings(
    rankings: list[list[RetrievedChunk]],
    top_k: int,
) -> list[RetrievedChunk]:
    """Round-robin across rankings to guarantee per-topic coverage."""
    fused: list[RetrievedChunk] = []
    seen_ids: set[str] = set()

    for position in range(max(len(ranking) for ranking in rankings)):
        for ranking in rankings:
            if position < len(ranking) and ranking[position].id not in seen_ids:
                fused.append(ranking[position])
                seen_ids.add(ranking[position].id)
            if len(fused) == top_k:
                return fused
    return fused
```

The deduplication visible in the code (`seen_ids`) is not defensive on a whim: when a budget covers two topics — and some do — it appears in two rankings, and without deduplication it would consume two slots of the context while counting as a single piece of information. The N searches, for their part, are independent of one another, so they are launched in parallel (`asyncio.gather`); the latency cost of searching four times is roughly that of searching once, not four times over.

## The price: an LLM call before every search

Time for the invoice, because these techniques carry a structurally different cost from the other retrieval improvements: they put an LLM generation **on the critical path, before the search has even begun.**

The cost has three line items. **Latency**: a reformulation call with a small model and short output typically runs between two hundred milliseconds and one second; it is by far the most expensive addend these techniques add, and it arrives before the search has done anything. **Tokens**: every search now pays for a generation; with small models it is loose change per query, but it is loose change multiplied by every query in the system, and it belongs on the cost dashboard from day one. **Load**: N parallel searches are N queries to the database and a candidate set N times larger entering the pipeline's later stages, which also charge by volume.

The sensible mitigations, in order of return: use the smallest model that does the task reliably (and check with real examples that it does — reformulation is a humble task, but "humble" is not "free to verify"); cap the variants at three or four, because the marginal gain of the fifth formulation is indistinguishable from zero; and cache the reformulations — the same query repeated does not need rethinking, and in systems where queries resemble one another closely the hit rate of that cache is surprising.

And the biggest mitigation of all: **do not apply the technique when it does not apply.** A short, sharp, single-topic query needs neither expansion nor decomposition; reformulating it is paying latency to stir a ranking that was already fine. The system can decide this with a humble heuristic (query length and structure as a first approximation: long transcripts are always decomposed, short queries pass straight through) and leave the decision recorded in the logs, so that when a retrieval is audited it is clear which path the query took and why.

> *(Editor's note — the implementation is more specific than this paragraph. `retrieval/query_transform.py` sets the heuristic in constants: `_SHORT_WORDS = 6` (below this, pass through untouched), `_LONG_WORDS = 25` (at or above, treat as multi-topic) and `_MULTITOPIC_CONNECTORS = 2` (this many "and"/comma/semicolon joins imply multi-topic), with `choose_technique()` returning `DIRECT`, `EXPAND` or `DECOMPOSE`. Note also that this article's `query_expansion.py` is `query_transform.py` in the code, and `interleave_rankings` ships as `round_robin_merge` in `retrieval/fusion.py`.)*

> *(Figure in the original: `articulo-04-figura-02-arbol-decision.jpg` — image not included in this repo.)*

## Where it lives and how it plugs in

In the AI service's architecture, reformulation is one more stage of the retrieval layer, with the same composable property as the rest: it takes a query, returns a list of queries (of size one when there is nothing to reformulate), and the next stage searches with all of them and fuses. Turning it on, turning it off or changing its strategy is configuration, not surgery — and therefore its impact can be measured like that of any other piece: the same fixed reference set of annotated queries, with and without the technique, and let the numbers decide. With one caveat of honesty: these techniques shine on the *difficult* queries (the long ones, the mixed ones, the badly worded ones), so if the query set used to measure contains only clean laboratory queries, the verdict will come out unfairly lukewarm. **A measurement is worth exactly as much as its resemblance to real traffic.**
