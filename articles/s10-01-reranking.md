---
title: "Reranking: when vector top-k is not enough"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 24 min
added: 2026-08-18
summary: >
  Why a bi-encoder finds well and orders badly — compression to one vector
  averages a document's content, and query and document never read each other.
  Cross-encoders fix the ordering by reading both texts jointly and cannot
  scale, so the standard answer is recall-then-rerank: a wide cheap net, then
  an expensive careful read over the finalists only. Includes local vs hosted
  model trade-offs, the asyncio thread-pool trap, and an explicit list of when
  reranking is the wrong tool.
keywords: [reranking, cross-encoder, bi-encoder, recall-then-rerank, sentence-transformers,
           ms-marco-MiniLM, bge-reranker, cohere rerank, latency budget, top-k,
           candidate pool, event loop blocking]
---

# Reranking: when vector top-k is not enough

*Antonio Perez* · 🔴 24 min

Picture this in the project-estimation system. A meeting transcript arrives describing an e-commerce platform: product catalogue, cart, inventory management, admin panel. The retrieval pipeline searches the historical budgets and returns, in first position, the budget for a mobile payments app built two years ago.

It is not an absurd result. E-commerce and payments share vocabulary (transactions, gateways, checkout, security), share business context, and probably share technologies. In the vector space their embeddings are genuinely close. But for estimating an e-commerce platform that budget is nearly useless: the bulk of the effort in e-commerce is the catalogue, the inventory and the administration, not the payment gateway. If the LLM generates the estimate with that context, the estimate will be skewed toward a project that is not the one in front of us.

This is the article's central problem: **vector search is excellent at finding candidates and mediocre at ordering them.** The distinction matters because the solution is not to change embedding model or tune the chunking — it is to add a second stage that does well what the first does badly.

## Why the bi-encoder orders badly: compression has a price

The embedding model we use to vectorise budgets and queries is a **bi-encoder**: it encodes each text separately and compresses it into a single fixed-dimension vector. The query becomes a vector, each budget chunk became another one at ingest time, and relevance is approximated by measuring the distance between them.

That independence is exactly what makes the bi-encoder viable in production. Because documents are encoded once at ingest, searching is cheap: vectorise the query and compare against precomputed vectors with an approximate index. Millions of documents, milliseconds of search.

But compression to a single vector has two consequences that are paid for in fine ranking:

**First: the vector averages.** An e-commerce budget with a minor section on payment integration produces an embedding that blends all its content. A payments-app budget produces an embedding where payments dominate. Faced with a query that mentions payments in passing, both may end up at similar distances, because the vector does not distinguish between "this is mainly about that" and "it mentions it among ten other things".

**Second: query and document never look at each other.** The bi-encoder encodes each text without knowing what it will be compared against. There is no point in the process at which the model can reason *"this query asks for e-commerce and this document is about payments; they resemble each other, but it is not what was asked"*. Cosine similarity is geometry over two compressed summaries, not a joint reading.

The practical result: among the 50 budgets closest to a query, the genuinely relevant ones are almost always present. But the order within those 50 is unreliable, and for a RAG pipeline about to pass 5 documents to the LLM, that order is everything.

> *(Figure in the original: `articulo-01-figura-01-biencoder-vs-crossencoder.jpg` — image not included in this repo.)*

## Cross-encoders: reading both texts at once

A **cross-encoder** attacks exactly that limitation. Instead of encoding query and document separately, it concatenates them into a single input and processes them together through the transformer. The attention mechanisms operate over the tokens of both texts simultaneously: every word of the query can attend to every word of the document. The output is not a vector but directly a relevance score for the pair.

That joint reading is the qualitative difference. The cross-encoder can capture that "e-commerce platform" and "payments application" share a semantic field but not an intent, because it sees both texts in the same attention context and was trained specifically to score pair relevance rather than to produce reusable representations. In ranking benchmarks cross-encoders systematically beat bi-encoders on ordering precision, and the intuition for why is sound: they have access to information the bi-encoder destroyed when it compressed.

The price is equally clear: there is nothing to precompute. The score depends on the complete pair, so every new query forces a pass through the model for each query-document pair we want to evaluate. Scoring one document costs a transformer inference; scoring a corpus of thousands of budgets per query is unviable. **The cross-encoder is precise and slow; the bi-encoder is imprecise and fast. Neither alone solves the problem.**

## Recall-then-rerank: dividing the work

The standard solution in information retrieval — predating LLMs, in fact; web search engines have used it for decades — is to chain both in a two-stage pipeline:

1. **Recall stage (wide search).** Vector search retrieves a generous set of candidates: top-50, say. Here we do not ask for fine ordering; we ask that the relevant documents be *in the set*. It is the task the bi-encoder is good at, and it is cheap.

2. **Precision stage (reranking).** The cross-encoder scores each of those 50 query-document pairs and reorders. We keep the best: top-5. It is the task the cross-encoder is good at, and because it only evaluates 50 pairs rather than the whole corpus, its cost becomes bearable.

> *(Figure in the original: `articulo-01-figura-02-recall-then-rerank.jpg` — image not included in this repo.)*

The pipeline's two numbers deserve their own judgment, not values by inertia:

**The size of the wide set** (50 in the example) controls the **ceiling on quality**. If the relevant budget does not make the vector top-50, no reranker will rescue it: *reranking reorders, it does not retrieve*. A larger set gives the reranker more room in exchange for more latency, because each additional candidate is another inference. In small, heterogeneous corpora like a company's budget history, between 30 and 75 candidates is usually reasonable; watching what position the relevant documents occupied in the original vector ranking tells you quickly whether the margin is enough.

**The size of the final set** (5 in the example) is dictated by the consumer of the context, not by the reranker. How many complete budgets fit comfortably in the LLM's context without diluting the instruction? How many does the use case genuinely benefit from? For project estimation, passing 5 well-chosen budgets produces better results than passing 15 mediocre ones: the generating model also suffers when you bury the signal in noise.

## The model landscape: local or hosted

There are two routes to incorporating a reranker, with sharp trade-offs.

### Open-source cross-encoders locally

The `sentence-transformers` library offers the most direct way to serve a cross-encoder inside the AI service itself. The classic family is **ms-marco-MiniLM**, trained on the MS MARCO dataset of query-passage pairs: small models (tens of millions of parameters), fast on CPU for batches of 50 candidates, and surprisingly competent.

There is a nuance that is not optional in a project with Spanish data: the classic MS MARCO models are **monolingual English**. For Spanish or multilingual corpora the serious options are **mmarco-mMiniLMv2** (the multilingual variant of the same family, lightweight) or **BAAI/bge-reranker-v2-m3** (more powerful and heavier, natively multilingual). Choosing between them is the classic quality-latency balance: the multilingual MiniLM responds in tens of milliseconds per batch on CPU; the BGE raises ordering quality in exchange for needing more muscle, ideally a GPU if query volume grows.

In favour of local: zero marginal cost per query, the data never leaves your infrastructure (with client budgets this may be a requirement rather than a preference), and latency with no network in the middle. Against: the PyTorch dependency fattens the service image by hundreds of megabytes, the model consumes memory permanently, and the quality of small models, while good, is not state of the art.

### Rerankers as a service

The alternative is to delegate to an API. **Cohere Rerank** is the segment's reference: send it the query and the list of documents, get back the reordered list with scores. Its multilingual model covers Spanish with no extra configuration, quality is superior to small local cross-encoders, and integration is three lines of HTTP client without touching the Docker image.

Against: every query has a direct monetary cost, it adds a network dependency to the critical path of each retrieval (with its latency and its failures), and the documents travel to a third party — exactly what the local option avoided. For a budget history containing sensitive commercial information, that last point deserves a conversation with whoever is responsible before choosing.

### A position, not a list of pros and cons

For an internal system with a Spanish corpus, moderate query volume and sensitive data, the sensible starting point is **a lightweight multilingual cross-encoder locally**: the infrastructure cost is bearable, the improvement over pure vector ranking is immediate, and the data stays at home. Moving to a hosted reranker is justified when the lightweight model measurably falls short, or when you do not want to carry the operational burden of the model — and that decision has to be taken with numbers from your own domain in front of you, not with generic benchmarks.

## Implementation in the AI service

The reranker is a component of the retrieval layer and lives with it. In a layered FastAPI service, that means its own module inside the retrieval package, next to the vector search it complements.

The essential wrapper over `sentence-transformers`:

```python
# app/generation/rag/retrieval/reranker.py

from sentence_transformers import CrossEncoder

from app.foundation.config import settings
from app.foundation.logging import get_logger

logger = get_logger(__name__)

class Reranker:
    """Cross-encoder reranker for retrieved candidates."""

    def __init__(self, model_name: str | None = None) -> None:
        self._model_name = model_name or settings.reranker_model_name
        self._model = CrossEncoder(self._model_name)
        logger.info("reranker_loaded", model=self._model_name)

    def rerank(
        self,
        query: str,
        candidates: list[RetrievedChunk],
        top_k: int = 5,
    ) -> list[RetrievedChunk]:
        """Score query-candidate pairs jointly and return the top_k best."""
        if not candidates:
            return []

        pairs = [(query, candidate.content) for candidate in candidates]
        scores = self._model.predict(pairs)

        ranked = sorted(
            zip(candidates, scores),
            key=lambda item: item[1],
            reverse=True,
        )

        logger.info(
            "rerank_completed",
            candidates_in=len(candidates),
            candidates_out=min(top_k, len(ranked)),
        )

        return [candidate for candidate, _ in ranked[:top_k]]
```

Three decisions in this code deserve comment, because they are what separates a tutorial example from a production component:

**The model is loaded once, at construction.** Loading a cross-encoder costs seconds; doing it per query would be a latency disaster. The reranker instance must be created at application startup and shared across requests — the same lifecycle-singleton pattern the LLM client already follows. The first operational consequence: service startup becomes slower and heavier in memory, and the container healthcheck must wait for the model to be loaded before declaring the service ready.

**The reranker receives and returns the same type.** A list of retrieved chunks goes in, a list of retrieved chunks comes out, shorter and better ordered. This makes it an optional, composable pipeline stage: enabling or disabling it is deciding whether the list passes through it, without the rest of the flow noticing. **When a retrieval technique can be switched on and off with a configuration boolean, comparing its impact stops being a refactor and becomes an experiment.**

**The logging records input and output sizes.** When in a few months an estimate comes out wrong and someone has to audit why the system chose those budgets, the structured log of each stage — what went in, what came out, with which model — is the difference between diagnosing in minutes or in days.

Integration into the retrieval flow is then a simple composition:

```python
# app/generation/rag/retrieval/pipeline.py (fragment)

async def retrieve(self, query: str) -> list[RetrievedChunk]:
    candidates = await self._vector_search.search(
        query,
        limit=settings.retrieval_candidate_pool_size,  # wide net: e.g. 50
    )

    if not settings.reranking_enabled:
        return candidates[: settings.retrieval_top_k]

    return self._reranker.rerank(
        query,
        candidates,
        top_k=settings.retrieval_top_k,  # narrow output: e.g. 5
    )
```

Note that the vector search is asynchronous (I/O against the database) and the reranking is not (local computation). **In an asyncio service this matters**: a cross-encoder inference of hundreds of milliseconds executed on the event loop blocks every other request for its duration. If the local reranker enters the path of an endpoint with real concurrency, the inference must be dispatched to a thread pool (`asyncio.to_thread` or the loop's executor). It is the kind of detail that does not appear in tutorials and does appear in incidents.

## Latency: the tax of reranking

It is worth being clear about where reranking is paid for, because it is always paid, and in the worst possible place: **the critical path of every query**, between the user asking and the LLM starting to answer.

Indicative orders of magnitude for a batch of 50 candidates: a lightweight cross-encoder on CPU moves in tens to a few hundred milliseconds; a powerful model without a GPU can reach seconds (and with a GPU returns to hundreds of milliseconds); an external API adds its inference plus the network round trip, typically between one hundred and five hundred milliseconds. On top of this sits the fixed cost of a local cold start — loading the model — which does not affect each query but does affect deployment and autoscaling.

Is that a lot? It depends on a denominator only your use case knows. In the estimation system, the LLM generation that follows takes several seconds: a 200 ms reranking is noise against a substantial improvement in context. In an interactive autocomplete where the total latency budget is 300 ms, that same reranking is unaffordable. **The right question is never "how long does the reranker take?" but "what fraction of my latency budget does it consume and what does it give back?"** — and answering that requires measuring the relevance gain in your domain, with your data, not assuming it.

## When not to rerank

Reranking has become the default recommendation in any article about RAG, and it is worth resisting the default. There are scenarios where it is superfluous:

- **When the vector ranking is already sufficient.** If the relevant documents appear consistently in the first positions — small, well-differentiated corpora, very specific queries — the reranker reorders something that was already well ordered. Cost without benefit.

- **When the bottleneck is earlier.** If the relevant documents do not even enter the wide candidate set, the problem is recall: the chunking, the embeddings, or the search itself. **Reranking does not rescue what was not retrieved**, and adding it on top of poor recall is polishing the order of the wrong results.

- **When the latency budget does not allow it.** There are products where milliseconds rule, and architectural honesty consists of acknowledging that instead of degrading the experience to follow a generic best practice.

**The signal that reranking is the right tool is precise: the relevant documents are among the retrieved candidates, but not at the top.** That is exactly the case of the e-commerce budget buried under the payments app — and that is why this technique is the first stop for improving the estimation system's retrieval.

## Order matters, and it can be bought

The idea to take from this article fits in three sentences. Vector search finds well and orders indifferently, because it compresses each text into a vector that never gets to look at the query. The cross-encoder orders well and scales badly, because it reads each query-document pair jointly. The recall-then-rerank pipeline buys the best of both: a wide cheap net so nothing is missed, and a fine expensive reading over the finalists only.

In the live session we will see it working on the project: we will integrate the reranker into the budget retrieval pipeline and check, with real domain queries, how what reaches the LLM changes — and with it, the quality of the estimates it produces.
