---
title: Contextual and temporal filtering
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 6
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-08-21
summary: >
  An embedding encodes what the text says, not when it was written or whether
  it is still true; those dimensions live in the metadata. Two uses, with
  opposite placements — hard filters as early as possible because every
  exclusion is work the later stages skip, soft weights as late as possible
  because that is where being wrong is cheap. Time gets its own treatment:
  hard windows for a categorical reason, exponential decay with a half-life
  for ordinary erosion — though the premise that recency always wins is the
  estimator's, not RAG's, and an editor's note works out how time transfers to
  domains where it does not. Closes with the assembly principle for the whole
  retrieval pipeline — cheap and excluding first, expensive and fine last,
  soft at the end — and the warning that a filter trusted over badly extracted
  metadata is retrieval's most silent failure.
keywords: [metadata filtering, hard filters, soft weighting, temporal decay,
           half-life, hard window, hnsw.iterative_scan, partial index,
           selectivity, dynamic boosting, magic numbers, pipeline ordering,
           metadata extraction at ingest]
---

# Contextual and temporal filtering

*Antonio Perez* · 🔴 18 min

One last scene from the estimation system. A project description arrives: a client portal with a private area, document management and electronic signature. The search runs over the history and finds an almost identical budget — same type of client, same functional scope, same line-item structure. Extremely high semantic similarity, indisputable first place.

The budget is from 2019. The frontend was budgeted in AngularJS, the electronic signature with a provider that no longer exists, the rates are from another era, and the team estimated using practices the studio abandoned three years ago. As a reference for *what line items* a client portal has, it still orients; as a reference for *what it costs today*, it is outright dangerous — and the generating LLM, which will receive that budget as its star context, has no way of knowing.

This is the blind spot none of the similarity techniques can cover, however sophisticated: **the embedding encodes what the text says, not when it was written, nor with what technology, nor for which sector, nor whether it is still true.** Those dimensions exist, but they live outside the text: in the metadata. And metadata, used well, is the retrieval technique with the best cost-benefit ratio in the whole arsenal — the only one whose execution cost is *negative*, because filtering before searching makes everything downstream cheaper.

This article covers the two uses of metadata in retrieval — the hard filter that excludes and the soft weighting that reorders —, the treatment of time as a special case, and the assembly principle that orders all the pieces of a modern retrieval pipeline.

## Hard filters: shrinking the universe before searching

The hard filter is the most direct use: conditions over metadata that exclude documents before similarity gets an opinion. If the project to be estimated is React Native, budgets for unrelated technologies should not even compete; if the studio's policy is not to estimate from references more than four years old, the date cuts them off flat. In SQL, it is the same old `WHERE` cohabiting with vector search:

```sql
SELECT chunk_id, embedding <=> :query_embedding AS distance
FROM budget_chunks
WHERE project_date >= :min_project_date
  AND technology = ANY(:relevant_technologies)
ORDER BY distance
LIMIT 50;
```

The syntax is trivial; the trap is underneath, and it is worth knowing before it bites. **Approximate vector indexes like HNSW do not understand `WHERE`**: the index navigates its graph looking for the nearest neighbours of the *complete* universe, and the filter is applied afterwards over whatever the index returned. If you ask for 50 results with a filter only 5% of the corpus satisfies, the index may return its 50 nearest neighbours, the filter discard 48, and the query deliver 2 results — or zero — with no visible error. Recent versions of pgvector (0.8 onwards) mitigate the problem with the iterative scan (`hnsw.iterative_scan`), which keeps asking the index for candidates until it has gathered the requested number *after* the filter; for very frequent and selective filters, the **partial index** (an HNSW index built only over the rows satisfying the condition) is the structural solution. The underlying lesson is not the specific technique but the habit: **when you combine filters with approximate search, verify the cardinality of what comes back** — and leave it in the logs, because "the filter silently emptied the result" is one of the most bewildering failure modes to debug after the fact.

There is a second precondition that gets taken for granted and should not: **the metadata has to exist, and exist well.** The date comes for free; the technology, the sector or the team size have to be extracted from the document at ingestion time — with rules when the format allows, with an LLM doing structured extraction when it does not — and that extraction is done **once per document, never per query**. The quality of that extraction is the ceiling on everything else: a hard filter over a badly extracted field is worse than no filter, because it excludes the best candidate with total confidence and nobody sees the hole it leaves. Hard filters are reserved for metadata you trust; anything doubtful weights, at most.

## Time: the metadata that is never optional

Of all the metadata, the date deserves its own treatment, because its effect on usefulness is universal and directional: in a history of budgets, **recent is systematically worth more than old** — prices expire, stacks rotate, practices change. The design question is how to materialise that preference, and there are two families of answer.

> *(Editor's note — domain transfer: "Universal" is true of the estimation system and not of RAG in general; the section title should be read as *never optional to think about*, not *always decaying*. A clinical RAG retrieving on symptom similarity is the clean counter-example: a case presentation from 2011 describing the same constellation of symptoms is not less relevant for being old, and applying a half-life to it would suppress exactly the rare presentations the corpus is most valuable for. What survives the transfer is not "recent wins" but the prior question the article's own logic implies: **does this corpus's value erode with time, on what clock, and per which document family?** Three answers recur across domains — value erodes (prices, stacks, market data: decay); value is stable (case presentations, symptom descriptions, mathematical or legal-historical fact: no temporal weighting at all); or validity flips on a date rather than fading (clinical guidelines superseded by a new revision, drug approvals, standards, regulation: a hard window or an explicit `superseded_by` field, never a smooth decay, because a superseded guideline is not 40% true). Note that a single corpus can hold all three at once — a medical system where case notes do not age, guidelines flip, and cost data decays — which is one more argument for the per-family partitioning of [s10-05](s10-05-multi-index-and-routing.md), since half-life is a per-collection parameter, not a global one.)*

**The hard window** is the temporal filter as a condition: only budgets from the last N years. Simple to implement, simple to explain, and with brutal behaviour at the edge: the budget from 3 years and 11 months ago competes on completely equal terms, the one from 4 years and a month does not exist. When the corpus is abundant, the brutality does not matter; when it is scarce — and company histories are scarcer than people admit — the window can leave out the only decent reference for a rare project type.

**Continuous decay** treats age as a progressive penalty rather than a verdict. The usual form is exponential, and its single parameter has a direct business reading — **the half-life**: how many days it takes a budget to lose half its weight.

```python
# app/generation/rag/retrieval/temporal.py

from datetime import date


def temporal_weight(document_date: date, half_life_days: int = 900) -> float:
    """Exponential decay: a document loses half its weight every half_life_days."""
    age_days = (date.today() - document_date).days
    return 0.5 ** (max(age_days, 0) / half_life_days)
```

With a half-life of 900 days (two and a half years), a budget from a year ago keeps 76% of its weight; the 2019 one, around 15%. It still exists — if it is the only reference of its kind, it will appear, correctly degraded — but it can no longer beat a recent equivalent to first place. **The half-life is not optimised with a formula**: it is chosen with domain judgment ("from what point would you stop trusting a budget's figures?") and revised when the domain changes.

> *(Figure in the original: `articulo-06-figura-01-decaimiento-temporal.jpg` — image not included in this repo.)*

The choice between window and decay is not ideological: **the window when there is a categorical reason** (compliance, company policy, an era change that invalidates what came before — "nothing pre-migration counts"); **decay for the normal gradual erosion of value.** And they combine without conflict: a generous window as a safety filter, decay within it as the fine ordering.

## Dynamic weighting: when the query's context changes the weights

The third level of sophistication: letting the importance of each metadata field depend on the query. If the project description revolves around a specific technology, a technology match should weigh heavily; if it describes a project for banking, sector experience rises in value because it drags regulation and timelines with it; if it mentions none of that, those same fields should stay quiet. The idea is attractive and the honest implementation is prosaic: **the contextual adjustment is applied as multipliers over the final ordering** — technology match when the query mentions it, sector match when it applies, the temporal weight always — with the factors defined in configuration, not buried in the code.

And here comes the article's most serious warning, because this is the technique where over-engineering lurks in the best disguise: **every weight is a magic number somebody will have to justify, recalibrate and debug.** A system with seven contextual boosts interacting is a system where nobody knows any more why a document came third — you have swapped the embedding's opacity for an artisanal opacity, which is worse because it additionally *looks* controllable. The sensible progression is conservative: first hard filters and temporal decay, which solve most of the problem with two explainable decisions; dynamic weighting afterwards, only for the fields where there is measured evidence that it contributes, and with the multipliers documented where the next developer will find them. A good exercise in humility: **if you cannot explain in one sentence why a boost is 1.3 and not 1.5, it was not ready for production.**

## Assembling the pipeline: the order is the message

The article closes with the architectural question that gives everything above its point: a modern retrieval pipeline accumulates stages — query reformulation, routing, filters, two-branch search, fusion, reranking, weightings — so in what order do they go? The answer is not arbitrary; it comes from a principle that can be stated in one line and defended in any architecture review:

> **Cheap and excluding at the start; expensive and fine at the end; soft at the close.**

Unfolded over the stages:

1. **Reformulation and routing first**, because they operate on the query and decide *what* is searched and *where* — everything else depends on them.
2. **Hard filters immediately after**, embedded in the search query itself: they shrink the universe before anything expensive traverses it. Every document the filter excludes is work that search, fusion and reranking will not do.
3. **The search** (with its two branches, semantic and lexical) **and its fusion**, over the already-filtered universe, producing the broad candidate set.
4. **Reranking at the end of the expensive stretch**, over the survivors and only over them: it is the costliest stage per document, and every earlier stage exists, in part, so that little and good arrives at it.
5. **The soft weightings** (temporal, contextual) **as the last adjustment**, over the ordering of the finalists — where a correction to the weights is cheap and its effects are visible.

> *(Figure in the original: `articulo-06-figura-02-pipeline-completo.jpg` — image not included in this repo.)*

Note the deliberate asymmetry between the two uses of metadata, which is the article's structural moral: **hard filters go as early as possible; soft weightings as late as possible.** The early filter saves work for everything that follows; the late weighting adjusts over the small set where being wrong is cheap and correcting is quick. Inverting that order produces the two classics of the badly assembled pipeline: reranking documents a filter was going to throw away (money burned), and weighting so early that the soft adjustment expels candidates from the set before the reranker could evaluate them (information destroyed).

And a final note on the complete assembly: **not every query needs every stage.** The pipeline in the figure is the maximum path; each stage must be switchable on and off by configuration, both to measure its contribution and because the simple, sharp query has no reason to pay the toll of the complex one. A pipeline where every piece is optional and observable is not only better engineering — it is the only way to answer, with data, the question every piece has to answer to keep its place: *what exactly do you contribute, and how much do you cost?*

## Relevance does not live in the text alone

The idea to take away: semantic similarity answers *"what is this document about?"*, and real relevance almost always also needs *"how old is it, what technology, what sector, and how much do I trust it?"*. Those answers live in the metadata. Used as a hard filter, they make the whole pipeline cheaper and eliminate what no later stage could have fixed; used as soft weighting, they refine the final order without destroying candidates; and time — the universal metadata — is treated with windows when there is a categorical reason and with decay when value simply erodes. All of it under the condition that holds the building up: **metadata extracted with quality at ingestion, because a confident filter over a bad field is the most silent error in all of retrieval.**
