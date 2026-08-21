---
title: Multi-index and routing
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-08-21
summary: >
  A single index answers "what resembles this query?" when the real question was
  "which budget resembles this query?". Four degradation mechanisms when
  document families mix, and the rule for partitioning: diverging metadata
  schemas mean separate tables, variations of one thing mean a discriminator
  column. The router is a cost hierarchy whose level zero is having no router —
  put the collection in the API contract, because the caller already knows what
  the service would be guessing. Scores from different collections are not
  comparable, so fuse by position or quota, and often do not fuse at all.
  Partition on an observed contamination signal, never "for when we grow".
keywords: [multi-index RAG, routing, query router, collection partitioning,
           document_type, StrEnum, structured output, LLM classifier, fallback,
           provenance, rank fusion, agent delegation, when not to partition]
---

# Multi-index and routing

*Antonio Perez* · 🔴 19 min

During its first weeks of life, the estimation system had a homogeneous corpus: historical budgets, chunked and vectorised. But real RAG systems put on weight, and ours already stores three quite distinct families of document: the **budgets** (structured, telegraphic, full of line items and figures), the **meeting transcripts** (spoken language, redundant, meandering, where the useful information swims among filler words) and the **internal technical documentation** (descriptive, dense, written to be consulted as reference).

And with the mixture comes an uncomfortable phenomenon. Faced with the query *"how much did the SAP integration cost in previous projects?"*, the document that answers is a budget. But in a single index mixing the three families, the top-5 comes contaminated: two transcript chunks where a client talked about SAP at length, a fragment of technical documentation about connectors, and only then the budgets. Semantically, they all orbit "SAP integration"; functionally, only some of them are any use for estimating. **The single index answers "what resembles this query?" when the real question was "which *budget* resembles this query?"** — and that difference, which a human resolves without thinking, the index cannot resolve because nobody has told it.

This article is about telling it: how to partition the corpus into specialised collections and how to decide, query by query, where to search. The technique is called **multi-index RAG with routing**, and like everything in its family, it is as valuable when it applies as it is counterproductive when adopted out of fashion.

## Why a single index degrades with a mixed corpus

It is worth being precise about the mechanism of the degradation, because it is not merely "results of the wrong type come out".

**Document families have different semantic textures.** A forty-minute transcript and a three-page budget do not embed alike: spoken language produces diffuse, thematically mixed chunks; budget language produces dense, single-topic chunks. When both compete in the same space for the same query, the distances are not comparable *in terms of usefulness* — a transcript chunk can land closer to the query embedding precisely because of its thematic verbosity, while being less useful.

**The dominant type floods.** If the history holds ten times more transcript chunks than budget chunks (and it will: meetings generate text at a rate budgets cannot keep up with), the top-k of any query will have the proportion volume dictates, not the one usefulness dictates.

**Each family wants its own preprocessing.** The chunking that is reasonable for a transcript (by speaker turns, by topic blocks) is not the one for a budget (by line items) nor the one for documentation (by sections). A single index pushes toward a single compromise preprocessing, mediocre for all three.

**And operations suffer.** Reindexing the transcripts — because their chunking strategy changed, say — should not touch the budgets. In a single index, every change is a global change, with its global risk.

The architectural answer is to partition: separate collections, each with its preprocessing, its metadata schema and its vector index. Which opens two design questions: how to partition physically, and who decides where to search.

## Partitioning in PostgreSQL: discriminator column or separate tables

With the corpus in PostgreSQL, there are two ways to materialise the collections, and the choice is not cosmetic.

**Option A — one table with a discriminator column.** All chunks live together in one table with a `document_type` column, and every search filters by type. It is the lower-friction option: a trivial migration, a single data model, a single query with one more `WHERE`. It works reasonably when the families share a schema (the same metadata serves all of them) and when many queries want to search across all of them at once.

**Option B — one table per family.** `budget_chunks`, `transcript_chunks`, `technical_doc_chunks`: each with its own columns, its own vector index and its own life cycle. More pieces, more migrations — and in exchange, each family can have the schema its nature asks for (budgets carry amounts and line items; transcripts carry speakers and meeting dates; documentation carries versions), each HNSW index is built and tuned over a homogeneous population, and reindexing one family is a local operation that does not graze the others.

> *(Figure in the original: `articulo-05-figura-01-matriz-particionado.jpg` — image not included in this repo.)*

The decision rule this article defends: **if the metadata schemas diverge, separate tables; if the families are variations of the same thing, a discriminator column.** The metadata is the design's involuntary confession — when you catch yourself adding columns that only make sense for one family (`speaker_count` that is `NULL` on every budget, `total_amount` that is `NULL` on every transcript), the single table is telling you these are really distinct entities cohabiting unhappily. In the estimation system the schemas diverge unambiguously, so the road is option B. With one piece of operational honesty: three tables are three migrations, three indexes to monitor and three ingestion pipelines to maintain; the price is real and it is paid in maintenance, not in performance.

## The router: who decides where to search

With the collections separated, every query needs a destination. The piece that decides it is the **router**, and its design has a hierarchy of solutions worth walking in order of cost — because the expensive, showy version has become the default reflex, and it is almost never the one to reach for first.

**Level zero: the best router is no router.** In a real system, many queries arrive with their destination implicit in the context of whoever issues them. The business backend's estimation flow always searches budgets: there is nothing to classify. The correct way to capture that knowledge is **the API contract** — an explicit collection parameter, or outright separate endpoints per use case. It is free, it is deterministic, it is traceable, and it turns routing into a decision made by the party that knows (the caller) instead of a guess by the party that does not (the service). Before building any classifier, the obligatory question is: does the AI service really have to guess something the business backend already knows?

**Level one: deterministic rules.** When the query arrives with no destination — the case of the internal free-text search, where a user asks whatever they like — a layer of cheap rules resolves a surprising share of the traffic: unambiguous vocabulary patterns (*"how much did ... cost?"*, *"what did we budget for ...?"* → budgets; *"what did the client say ...?"*, *"what was agreed in the meeting ...?"* → transcripts). Rules are fragile in the face of linguistic creativity, but free in latency and transparent when debugging; as a first filter before the expensive classifier, they earn their keep.

**Level two: the LLM as classifier.** For what the rules do not resolve, a call to a small model with structured output. The discipline is the same as for any LLM with an infrastructure function: closed schema, instructions that fence in, and an output design that accommodates honest doubt:

```python
# app/generation/rag/retrieval/router.py

from enum import StrEnum

from pydantic import BaseModel, Field


class SearchTarget(StrEnum):
    BUDGETS = "budgets"
    TRANSCRIPTS = "transcripts"
    TECHNICAL_DOCS = "technical_docs"


class RoutingDecision(BaseModel):
    """Which collections a query should be searched against."""

    targets: list[SearchTarget] = Field(
        min_length=1,
        max_length=3,
        description="Collections to search. Use several only when the query genuinely spans them.",
    )
    reason: str = Field(description="One short sentence explaining the choice")


ROUTING_INSTRUCTIONS = """
You classify search queries for a project estimation system into the
collections they should be searched against.

Collections:
- budgets: historical project budgets, with line items, effort and cost figures.
- transcripts: meeting transcripts between the team and clients.
- technical_docs: internal technical documentation and architecture references.

Rules:
- Choose the single most appropriate collection whenever possible.
- Choose several collections only when the query genuinely needs them.
- Questions about cost, effort or estimates belong to budgets, even if the
  query mentions meetings or documents.
"""


async def route_query(self, query: str) -> RoutingDecision:
    """Classify a query into the collections it should be searched against."""
    response = await self._client.responses.parse(
        model=settings.router_model,
        instructions=ROUTING_INSTRUCTIONS,
        input=query,
        text_format=RoutingDecision,
    )
    decision = response.output_parsed
    logger.info(
        "query_routed",
        targets=[target.value for target in decision.targets],
        reason=decision.reason,
    )
    return decision
```

Three decisions in this design deserve a defence. **The output is a list of destinations, not a destination with a confidence number**: when the classifier hesitates between two collections, the correct action is not to choose wrongly at 0.55 confidence, it is to search both — and modelling the output as a list turns the doubt into well-defined behaviour instead of an arbitrary threshold to calibrate. **The `reason` field is not decorative**: it costs one sentence of tokens and turns every routing decision into something auditable when, months later, somebody asks why a query ended up searching where it did. And **the `StrEnum` closes the universe of answers** — the model cannot invent a collection that does not exist, because the schema does not let it.

**The last rung: when not even the classifier decides, search everything.** Searching the three collections in parallel and combining is the honest fallback, and its latency cost is that of the slowest collection, not the sum. The degradation is graceful: in the worst case, the multi-index system behaves like the single index we came from — never worse.

> *(Figure in the original: `articulo-05-figura-02-cascada-routing.jpg` — image not included in this repo.)*

## Combining results from different collections: carefully

When a query ends up searching several collections, a trap we already know appears in an aggravated version: **the similarity scores of different collections are not comparable with one another** — each collection has its texture, its distance distribution, its density. Fusing by raw score lets the collection with generous distances devour the rest.

There are two sensible ways out. If the consumer needs a single ranking, fuse by positions or by per-collection quotas (the top two from each), never by raw score. But very often the right answer is **not to fuse at all**: present the results grouped by provenance — *"this is what the budgets say; this is what was discussed in meetings"* — because the final consumer (human or generating LLM) does different things with each family, and flattening them into one list destroys information a router was paid to obtain. For that, **every retrieved chunk must travel with its provenance label.** This is not a detail: provenance is what will later allow attributing, auditing and debugging, and losing it in the fusion is losing it for good.

## The seed of something larger

One architectural observation before closing, because it connects this piece to the system's future. The pattern we have just built — a component that examines a request and delegates it to the appropriate specialist, with the option of consulting several and combining — is not exclusive to search. It is a general **delegation** pattern, and it is exactly the embryo of how agent-based systems divide the work among themselves: a coordinator deciding which specialist handles which request. The difference is one of degree and of freedom: our router performs a bounded classification with a closed schema, with no open-ended reasoning and no tools. That containment is deliberate — to decide where to search, a cheap, auditable classification pays better than any sophistication — but the mental pattern it leaves installed will be reused, enlarged, later in the programme.

## When not to partition

As always, the technique has its contraindication, and in this case it is especially tempting to ignore because partitioning looks like serious architecture.

Do not partition if the corpus is **functionally homogeneous**, however different the documents' origins. Do not partition if one collection would concentrate 95% of the queries: the router would be a toll booth that almost always gives the same answer. And do not partition **"for when we grow"**: three tables, three ingestions and a router are maintenance debt contracted today against a hypothetical need. The legitimate signal for partitioning is observable and concrete: **results from one family contaminating queries aimed at another, recurrently and measurably.** In the estimation system that signal exists — the SAP example from the opening is not hypothetical, it is the real behaviour of a mixed index — ; in another system, perhaps not. Partitioning without the signal is adding complexity to solve a problem you do not have.
