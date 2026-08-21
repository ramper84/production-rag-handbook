---
title: Data quality and architecture decisions
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 6
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 24 min
added: 2026-08-21
summary: >
  The CAG ceiling is four constraints operating at once, not just the context
  window — and cost usually kills the project before capacity does, while
  attention degradation over long contexts means fitting is not the same as
  working. No amount of clever chunking fixes fundamentally bad data, which is
  why three sessions go to data before anything is embedded. Splits the
  six-step pipeline into an offline indexing half and an online query half,
  gives a four-axis decision tree over CAG / hybrid / RAG, and closes on three
  trade-offs the promotional literature omits — including that RAG can cost
  more than CAG, so the choice is made on viability, not price.
keywords: [CAG limits, context window, cost per query, latency, lost in the
           middle, data quality, six-step pipeline, offline indexing, online
           query, architecture decision tree, fine-tuning, source attribution,
           access control, hybrid CAG-RAG, residual static context]
---

# Data quality and architecture decisions

*Antonio Perez* · 🔴 24 min

If you did the pre-session exercise, you have spent the last few hours looking at a spreadsheet of results that has probably surprised you. Your CAG, which you closed in Session 05 with the legitimate feeling that it worked well, has done strange things when fed a realistic corpus. Perhaps it hallucinated about projects that do not exist, perhaps it answered with excessive confidence questions it had no data for, perhaps it took fifteen seconds on a query it used to resolve in two, perhaps a single answer cost you two euros. Or perhaps all of the above at once.

This article is not going to explain what RAG is. It is going to explain **why that CAG broke exactly where it broke**, and give you an architectural decision framework you will be able to defend in front of any stakeholder. The difference matters: at this point in the programme, applying RAG is the easy part; the hard part is knowing when to apply it, when not to, and when to combine it with what you already have.

## The real CAG ceiling: it is not only the context window

When a junior engineer looks at CAG's failure modes over a large corpus, their first reaction is usually to identify a single culprit: "it does not fit in the context". It is a useful simplification for starting the conversation, but a misleading one. **CAG has a ceiling made of four constraints operating at once**, and understanding each is what allows non-trivial architectural decisions.

**The first constraint** is the one everybody knows: the **context window**. Your model declares a maximum token capacity per call, and when the corpus exceeds that threshold, the system cannot inject it whole. This constraint is binary and obvious.

**The second constraint** is the **cost per query**. With large corpora and many queries, cost rises linearly with input tokens. A system costing three cents per query is perfectly deployable to production; one costing two euros is not, for almost any use case. This constraint is continuous and it is usually the one that kills the project before the first one does.

**The third constraint** is **latency**. Processing a 100K-token context takes several seconds even in optimised models. For a synchronous conversational assistant, that is unviable. For a nightly batch process, irrelevant. The constraint depends on the product, not on the architecture.

**The fourth constraint**, and the most underestimated, is **attention degradation over long contexts**. Models do not process a 200K-token context with the same fidelity as a 5K one. The phenomenon is documented as *lost in the middle*: information placed in the middle of the context is recovered worse than information placed at the extremes. In practical terms this means that even if your corpus fits in the context window, it is not being processed with the same quality as if only the relevant fragments were injected.

It is worth fixing the four constraints as a formal object before going on:

```python
from dataclasses import dataclass


@dataclass
class CAGViability:
    fits_in_context_window: bool      # Does it technically fit?
    cost_per_query_acceptable: bool   # Is it economically viable?
    latency_acceptable: bool          # Does it respond within the product's SLA?
    quality_holds_with_load: bool     # Does quality hold with the full corpus?

    def is_viable(self) -> bool:
        return all([
            self.fits_in_context_window,
            self.cost_per_query_acceptable,
            self.latency_acceptable,
            self.quality_holds_with_load,
        ])
```

If you filled in `CAG_LIMITS.md` with real numbers, you already have the four booleans for your case. The empirical conclusion you have probably reached is that **one failing is enough for the architecture not to be viable in production**. And it is almost never only one that fails.

## Why data quality is the real control variable

There is a claim that appears recurrently in the literature on RAG in production and is worth internalising: *no amount of clever chunking or fancy architecture can fix fundamentally bad data.* The quote is from a Towards Data Science article collecting lessons from teams that have operated RAG systems for months, and it sums up well why Module 3 opens the RAG block with three sessions dedicated to data before touching embeddings or vector databases.

The reasoning is direct. **A RAG system does not generate information: it retrieves and presents it.** If what is retrieved is noise, what is presented is well-formatted noise. If what is retrieved is out of date, what is presented is misinformation with the appearance of an authoritative answer. If what is retrieved is duplicated, inconsistent or badly structured, no reranker and no cross-encoder is going to fix it at query time.

This is the operational difference between a team that puts a RAG into production and a team that attempts it. The first invests weeks or months in auditing, normalising and validating the corpus before vectorising anything. The second vectorises immediately to see results fast and then spends the next six months trying to understand why the system answers badly, intermittently.

What makes the data-quality problem especially treacherous in RAG is that **the system appears to work well at first**. With a small corpus and test queries chosen by the team, the answers are acceptable. The degradation appears when the corpus grows, when real users start asking questions the team had not anticipated, and when the data becomes incongruent with itself (two contradictory versions of the same budget coexisting in the index, for example). By that moment it is very late to fix it, because the pipeline is already built on assumptions that do not hold.

## The RAG pipeline as a six-step abstraction

To reason architecturally about RAG, it helps to hold a shared mental model of the complete pipeline. The canonical decomposition, popularised by teams like Databricks, has six steps:

1. **Ingest** — collect the data from the enterprise sources (databases, files, external APIs, filesystems).
2. **Parse** — extract clean text and metadata from each format (PDF, DOCX, JSON, transcripts).
3. **Chunk** — split the documents into fragments of a size suitable for vectorisation.
4. **Embed** — convert each fragment into a vector via an embedding model.
5. **Retrieve** — given a query, recover the most relevant fragments.
6. **Generate** — hand the LLM the query along with the recovered fragments so it composes the answer.

The trap in this list is that it seems to suggest a linear, real-time flow. **It is not.** The six steps split across two distinct pipelines that operate at distinct moments and under distinct constraints. This distinction is Module 3's first serious architectural decision.

## Offline vs online: the line that changes the whole architecture

> *(Figure in the original: `sesion_06_article_1_visual_1_rag_pipeline.jpg` — image not included in this repo.)*

Steps 1-4 (ingest → parse → chunk → embed) constitute the **offline indexing pipeline**. It runs in the background, with no user waiting for an answer. It may take minutes or hours. It is triggered by system events: upload of a new document, scheduled nightly run, weekly refresh of a source.

Steps 5-6 (retrieve → generate) constitute the **online query pipeline**. It runs synchronously when a user asks a question. It has a strict latency budget (typically under 3 seconds for an acceptable conversational experience). It has no access to the raw data, only to the previously indexed vectors and metadata.

Materialising this separation in the AI service radically changes the project's structure. An endpoint that does everything in a single call is not the same thing as two endpoints with clearly disjoint responsibilities:

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel

app = FastAPI()

# ============================================================
# OFFLINE: indexing pipeline
# Triggered by ingestion events, not by user queries.
# Time budget: minutes to hours. No user waiting.
# ============================================================


class IngestRequest(BaseModel):
    source: str  # "budgets_json", "transcripts_txt", "proposals_pdf"
    document_ids: list[str]


@app.post("/index/run")
async def trigger_indexing(req: IngestRequest, tasks: BackgroundTasks):
    """Offline pipeline: parse -> chunk -> embed -> store."""
    tasks.add_task(run_indexing_pipeline, req.source, req.document_ids)
    return {"status": "scheduled", "documents": len(req.document_ids)}


async def run_indexing_pipeline(source: str, doc_ids: list[str]):
    documents = await load_documents(source, doc_ids)
    parsed = await parse_documents(documents)
    chunks = await chunk_documents(parsed)
    embeddings = await embed_chunks(chunks)
    await store_in_vector_db(chunks, embeddings)


# ============================================================
# ONLINE: retrieval + generation pipeline
# Triggered by user queries through the business backend.
# Time budget: under 3 seconds. User is waiting.
# ============================================================


class QueryRequest(BaseModel):
    user_question: str
    top_k: int = 5


@app.post("/query")
async def answer_query(req: QueryRequest):
    """Online pipeline: retrieve -> augment -> generate."""
    relevant_chunks = await retrieve(req.user_question, top_k=req.top_k)
    augmented_prompt = build_prompt(req.user_question, relevant_chunks)
    answer = await call_llm(augmented_prompt)
    return {
        "answer": answer,
        "sources": [c.metadata for c in relevant_chunks],
    }
```

This separation has immediate practical consequences. The offline pipeline can use heavy models (OCR extractors, large embedding models, strict validators) because latency does not matter. The online pipeline has to be surgical: only fast vector search, prompt construction and the LLM call. **Mixing the responsibilities is one of the commonest antipatterns in badly architected RAG systems.**

For the business backend (Rails or another stack), this means it is going to invoke the AI service by two completely different routes. The indexing invocation is asynchronous (fire and forget, or check the status afterwards). The query invocation is synchronous and blocking. If this distinction is not clear from day one, you end up with an AI service that hangs because it is trying to index 200 PDFs while processing a user query.

> *(Editor's note: `/index/run` versus `/query` is the first of two splits, not the final shape. [s09-05](s09-05-data-layer-as-a-service-securing-the-retriever.md) divides the online half again, on a different axis — retrieval and generation get their own routers because one call costs milliseconds and the other costs euros, which drives separate keys and rate limits an order of magnitude apart. The two splits compose rather than compete: offline/online is about latency budget, retrieval/generation is about cost per call.)*

## The architectural decision: CAG, RAG, fine-tuning and combinations

So far we have established the framework. Now for the decision tree, which is what an AI Engineer has to be able to defend in front of a Product Director who asks why money is being spent on one architecture and not another.

The decision turns on four axes:

1. **Corpus volume relative to the model's context window** (with reasonable margin: loading 95% of the context is technically possible but degrades quality).
2. **Update frequency of the data** (how often the corpus changes).
3. **Traceability requirement** (whether the system has to cite the concrete source of each claim).
4. **Data sensitivity** (presence of PII, per-user access control requirements).

> *(Figure in the original: `sesion_06_article_1_visual_2_decision_tree.jpg` — image not included in this repo.)*

Materialising the tree as code helps clarify the logic:

```python
from dataclasses import dataclass
from enum import Enum


class Architecture(Enum):
    PURE_CAG = "pure_cag"
    HYBRID_CAG_RAG = "hybrid_cag_rag"
    PURE_RAG = "pure_rag"


@dataclass
class CorpusProfile:
    total_tokens: int
    update_frequency_days: int
    requires_source_attribution: bool
    requires_per_user_access_control: bool


@dataclass
class ModelProfile:
    context_window: int
    cost_per_million_input_tokens: float


def recommend_architecture(
    corpus: CorpusProfile, model: ModelProfile
) -> Architecture:
    context_usage = corpus.total_tokens / model.context_window

    # If traceability is mandatory, there is no decision: RAG.
    # CAG cannot attribute answers to concrete fragments.
    if corpus.requires_source_attribution:
        return Architecture.PURE_RAG

    # If we need per-user access control over the corpus,
    # CAG is unviable (the whole corpus goes in every call).
    if corpus.requires_per_user_access_control:
        return Architecture.PURE_RAG

    # If it does not fit with reasonable margin, RAG.
    if context_usage > 0.7:
        return Architecture.PURE_RAG

    # If it fits but changes very often, RAG (avoids re-injecting everything).
    if corpus.update_frequency_days < 7:
        return Architecture.PURE_RAG

    # If it fits and is very stable, pure CAG is still valid.
    if corpus.update_frequency_days > 90 and context_usage < 0.3:
        return Architecture.PURE_CAG

    # Remaining cases: hybrid. Stable, small context in CAG;
    # dynamic, voluminous corpus in RAG.
    return Architecture.HYBRID_CAG_RAG
```

The function is deliberately simple and does not capture every nuance. Its pedagogical value is forcing you to make the criteria explicit. When you defend the choice of RAG in front of a stakeholder, you are going to talk about those four axes with concrete numbers, not generalities.

> *(Editor's note: the thresholds baked into the function — 0.7 context usage, 7 days, 90 days — are the author's judgment calls, not measured constants, and the article says as much by calling the function deliberately simple. [s09-01](s09-01-from-cag-to-rag-four-stages.md) revisits the same decision with research behind it (Chan et al. 2024) and is more generous to CAG than this tree is: there are corpora where the static context genuinely wins, and the tree here routes some of them to RAG. Read the tree as a way to make your criteria explicit, which is what it claims to be, rather than as a verdict.)*

One angle is missing from the function: **fine-tuning**. The reason for the omission is that fine-tuning is not an alternative to RAG; it is a layer that can be added on top of RAG (or of CAG) when specific limitations are detected that better retrieval does not resolve. The typical cases are: a response style very specific to the company, proprietary terminology the base model does not handle well, or a structured format the model does not respect consistently. If the base model's answer over correctly retrieved fragments is already good, fine-tuning is not needed. If what you retrieve is correct but the model presents it badly, you do need it. **What never works is using fine-tuning as a substitute for badly designed retrieval**: you are teaching the model to memorise what it should be searching for.

## Project 2's case against the tree

Let us apply the tree to the programme's use case. We are building a system that receives client meeting transcripts and generates software project estimates based on a history of past budgets.

- **Volume**: grows linearly with the business. Each new client contributes one more transcript and, eventually, one more signed budget. Year over year that is hundreds of documents.
- **Update frequency**: high. Every week there are new meetings; every month, closed budgets.
- **Traceability**: critical. If the system proposes an estimate of €80,000 for a project, the sales team needs to know which precedents justify it in order to defend it in front of the client. An estimate with no references is unusable.
- **Sensitivity**: high. The transcripts contain confidential commercial information, client names, contractual terms. Access control by project and by team is a requirement.

Three of the four axes push directly toward RAG. The tree leaves no margin: the choice is justified. But there is a nuance worth preserving: **there are parts of the context that are small and stable, and for those traditional CAG is still the best option.** I am thinking, for example, of the company's technology glossary, the standard budget templates, the official rate ranges. Injecting that as static context on every LLM call is simpler, cheaper and more predictable than vectorising it. That is why the tree contemplates the hybrid option: **Project 2's final system is going to have a CAG layer coexisting with the RAG layer, not one replacing the other.**

## Honest trade-offs

I want to close the article with three trade-offs habitually omitted in RAG's promotional literature and worth facing head-on before closing the decision.

**The hidden cost of traceability.** Citing sources is not free. It requires preserving metadata on every chunk (origin, page, date, author), propagating it through the whole pipeline, returning it to the business backend along with the answer, and building a UI that presents it to the user. It is design work many teams underestimate when they estimate the cost of a RAG. If your product can afford not to cite, the whole system simplifies notably. If it cannot, that cost goes in the budget.

**The real cost of operating RAG.** Comparisons that put CAG and RAG in adjacent columns are usually rigged. RAG is not only the cost of inference: it adds the cost of embeddings (one per chunk, once per ingestion), the cost of the vector database (storage and compute), the cost of operating the indexing pipeline, and the cost of re-indexing when the embedding model changes or when the chunking strategy is adjusted. Summed up, **RAG can be more expensive than CAG on corpora that fit in the context window, not cheaper.** The choice is not made on cost; it is made on viability and on functionality.

**CAG does not die, it changes role.** A conclusion that surprises many students is that the programme's final system is not "RAG instead of CAG", but **"RAG in addition to CAG"**. In any serious RAG architecture two types of context coexist: the system's static context (instructions, schemas, glossaries) injected as-is exactly as in Module 2, and the dynamically retrieved context that comes in via RAG. What you built in Module 2 is not thrown away; it is repositioned. That is the reason the pre-session exercise asked you to think about which parts of the corpus could stay in CAG. It was not a rhetorical question.

## Bridge to the next stage

At this point the architectural decision is taken and justified: **RAG with a residual static CAG layer.** The natural temptation is to jump immediately to the implementation phase: install the vector database, choose the embedding model, write the first loader. We are going to resist that temptation.

Before processing any document, before choosing an embedding model, before touching a single line of pipeline, we need to do something many teams skip and later pay dearly for: **audit what we have on the table.** Which sources exist, what state they are in, what quality they have, what is missing. Without that photograph, the rest of the module is built on sand.
