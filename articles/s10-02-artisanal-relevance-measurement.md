---
title: "Knowing whether reranking pays: artisanal relevance measurement"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 10
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 23 min
added: 2026-08-18
summary: >
  Turning "it seems better" into "precision went from 0.48 to 0.80 for 250 ms".
  Why intuition fails at judging relevance, how to build a golden set of 5-20
  hand-annotated queries with a written criterion, precision@k at the k your
  system actually uses, latency measured warm and by median, and a gain-versus-
  cost table — including the treacherous quadrant where a small gain at a small
  cost still is not worth the complexity toll.
keywords: [golden set, precision@k, recall@k, annotation criterion, binary relevance,
           latency budget, median latency, decision table, measurement harness,
           statistical power, over-engineering]
---

# Knowing whether reranking pays: artisanal relevance measurement

*Antonio Perez* · 🔴 23 min

## "It seems better" is not an argument

Picture the situation. You have added a reranking stage to the estimation system's retrieval pipeline: a second model now reorders the candidate budgets before passing them to the LLM. You run three test queries, look at the results, and indeed the budgets appearing at the top look better. The temptation is to close the ticket and move on.

Now picture the conversation two weeks later. Someone on your team asks why every estimate takes half a second longer than it used to. Your answer is that retrieval improved. The next question is inevitable: **by how much?** And "it seems better" does not survive that question. Nor does it survive a serious code review, an architecture committee, or the client paying the infrastructure bill.

This article is about turning "it seems better" into "retrieval precision rose from 0.48 to 0.80 in exchange for 250 milliseconds per query". The good news: you need no evaluation framework, no data team, no weeks of work. You need an afternoon, domain judgment and a spreadsheet. We will call that practice **artisanal measurement**: deliberately small, deliberately manual, and sufficient for the decision in front of you.

## Why intuition deceives when measuring relevance

Before building the solution it helps to understand why eyeballing fails precisely here, because it does not fail through carelessness: it fails by design of how our attention works.

**We test with the wrong queries.** When we evaluate by hand we choose queries that occur to us in the moment — and the ones that occur to us are the easy ones, the ones we ourselves would phrase well. Real users write vague queries, mix topics, use their sector's terminology rather than ours. A system can shine on our three test queries and stumble on half the real ones.

**We remember the memorable, not the representative.** If reranking spectacularly rescues a budget that used to be buried, that anecdote dominates our perception even if nothing changed on the rest of the queries. Human memory weights by emotional impact; a metric weights by frequency. To decide about a system, the second is what counts.

**We compare against a moving reference.** Eyeballing the new configuration on Tuesday and the old one on Thursday introduces every kind of noise: different mood, different memory of what counted as "acceptable". Without a fixed reference, every comparison is against a different yardstick.

The solution to all three problems is the same: **fix in advance a set of representative queries with their known correct answers, and measure every configuration against that same set.** That set has a name.

## The golden set: your reference truth in miniature

A **golden set** is a small collection of real domain queries, each annotated by hand with the documents that are genuinely relevant to it. It is the reference truth against which any retrieval configuration is measured: if the system returns the annotated documents it is right; if it returns others it is wrong. No ambiguity, and no dependence on anyone's memory.

For the estimation system, a golden-set entry looks like this: the query is the description of a project to estimate ("e-commerce platform with catalogue, cart and admin panel") and the annotation is the list of historical budgets an experienced estimator would consider useful as a reference for that project. Not the semantically similar ones, nor the ones using the same technology: **the ones you would actually use to estimate.**

Three decisions define a golden set's quality, and none is technical:

**Which queries go in.** The selection must cover real use, not comfortable use. A healthy mix to start: two or three frequent, direct queries (the typical case the system sees daily), a couple of known hard queries (the ones where you have already seen the system get confused — adjacent domains like e-commerce and payments, for instance), and at least one query with exact terms that must be respected (technology names, acronyms, specific products). If the system processes meeting transcripts, some query should be long and disordered the way transcripts are, not a laboratory sentence.

**How many queries.** Fewer than you think. Between five and twenty well-chosen queries suffice to decide whether a technique pays; **the error that really invalidates the measurement is not sample size, it is the sample not resembling real use.** A golden set of five representative queries informs more than one of fifty invented in ten minutes. Start small: growing a living golden set is trivial; throwing away a large, badly built one is painful.

**Who annotates, and by what criterion.** Annotation is a domain judgment: deciding which budgets are relevant to a query requires knowing how to estimate projects, not knowing Python. It should be done by whoever would use the result — the estimator, the technical lead — and before annotating it is worth writing the criterion in one sentence ("relevant if it would serve as a direct effort reference for this project"). **The written criterion prevents silent drift**: the third query being annotated by a different yardstick than the first. Relevance is annotated **in binary** — relevant or not — resisting the temptation of graded scales: binary is less expressive, but it is consistent across annotations and sufficient for the decision we are after.

## Precision over the first k: the metric that fits on a napkin

With the golden set fixed, the metric almost defines itself. What matters to the pipeline is the quality of the documents that finally reach the LLM — the first k of the ranking, where k is the size of the context we pass (five budgets, say). The natural metric is **precision at the first k** (precision@k): of the k documents returned, what fraction is relevant according to the golden set.

The calculation, with an example. For the e-commerce query the golden set marks four budgets as relevant. The system returns its top-5; we check each position against the annotation.

> *(Table image in the original — not included in this repo.)*

Three hits out of five returned: **precision@5 = 3/5 = 0.60**. Repeat the calculation for each golden-set query and average: that average is the number describing the configuration. An alternative configuration — with reranking, with another search strategy — is measured against the same golden set and the two averages are compared on equal terms.

> *(Figure in the original: `articulo-02-figura-01-golden-set.jpg` — image not included in this repo.)*

Two nuances that raise the measurement without complicating it:

**Choose k deliberately.** The metric's k must be the system's k: if you pass five budgets to the LLM, measure precision@5. Measuring precision@10 when you only use five documents answers a question nobody asked. And if you are undecided between passing three or five documents to the generator, measuring both (precision@3 and precision@5) turns that doubt into a decision with data too.

**Recall as a complement.** Precision asks "of what you returned, how much is worth it?"; its mirror metric, **recall**, asks "of what was worth it, how much did you return?". If while annotating the golden set you have identified *all* the relevant budgets for each query — viable in a company corpus, unviable in one of millions of documents — computing recall@k is free and catches a failure precision cannot see: **the valuable document that appears nowhere at all.** More sophisticated metrics exist that also reward the order within the top-k, but to decide whether a technique enters the pipeline, precision and recall over your real k are more than enough. Metric sophistication has its moment; this is not it.

## The table's other column: latency

Relevance is half the decision. The other half is what it costs to obtain, and in retrieval the dominant cost is latency: every technique that adds quality — a second model, an extra search, a fusion — adds time on the critical path of every query.

Measuring it artisanally requires only two precautions. **First: measure warm.** The first query after starting the service pays fixed costs (loading models, connections, cold caches) that do not represent normal operation; discard it and measure from the second onward. **Second: keep the median of several runs, not the mean.** Three to five runs per query suffice; the median resists the outlier of a momentary machine spike, which with samples this small would drag the mean mercilessly.

The result is the complete table supporting the decision: one row per configuration, one column of precision, one column of median latency. Everything that follows is reading that table with judgment.

## The measurement harness in the AI service

Taking this to code is deliberately unglamorous. The golden set is a data file versioned alongside the code — changing it must go through review, **because changing the yardstick changes the meaning of every previous measurement**:

```json
{
  "annotation_criterion": "Relevant if it would serve as a direct effort reference for estimating this project",
  "queries": [
    {
      "id": "q01",
      "query": "E-commerce platform with product catalog, cart and admin panel",
      "relevant_budget_ids": ["budget-2023-014", "budget-2024-002", "budget-2022-031", "budget-2023-027"]
    },
    {
      "id": "q02",
      "query": "Mobile app with Stripe payment integration and push notifications",
      "relevant_budget_ids": ["budget-2024-011", "budget-2023-019"]
    }
  ]
}
```

And the harness is a script that walks the golden set, runs the retrieval pipeline and computes the two columns:

```python
# scripts/measure_retrieval.py
"""Artisanal retrieval measurement against a hand-annotated golden set."""

import json
import time
from pathlib import Path
from statistics import median

GOLDEN_SET_PATH = Path(__file__).parent / "golden_set.json"
TOP_K = 5
RUNS_PER_QUERY = 3

def precision_at_k(retrieved_ids: list[str], relevant_ids: set[str], k: int) -> float:
    """Fraction of the top-k retrieved documents that are relevant."""
    top = retrieved_ids[:k]
    if not top:
        return 0.0
    hits = sum(1 for budget_id in top if budget_id in relevant_ids)
    return hits / len(top)

async def measure(pipeline) -> None:
    golden_set = json.loads(GOLDEN_SET_PATH.read_text())
    precisions: list[float] = []
    latencies_ms: list[float] = []

    for entry in golden_set["queries"]:
        relevant_ids = set(entry["relevant_budget_ids"])

        for _ in range(RUNS_PER_QUERY):
            start = time.perf_counter()
            results = await pipeline.retrieve(entry["query"])
            latencies_ms.append((time.perf_counter() - start) * 1000)

        retrieved_ids = [chunk.budget_id for chunk in results]
        precision = precision_at_k(retrieved_ids, relevant_ids, TOP_K)
        precisions.append(precision)
        print(f"{entry['id']}: precision@{TOP_K} = {precision:.2f}")

    print(f"mean precision@{TOP_K}: {sum(precisions) / len(precisions):.2f}")
    print(f"median latency: {median(latencies_ms):.0f} ms")
```

One design decision deserves an explicit defence: **this lives in `scripts/`, not in the application's layers.** That is deliberate. An artisanal harness is a one-off decision tool, not infrastructure: it needs no endpoint, no tests of its own, no abstraction for future use cases. Prematurely turning it into an "evaluation module" of the service is the classic over-engineering nobody wants to maintain afterwards. When the system genuinely needs continuous evaluation — automated, in CI, with history — that will be another piece with another design; this script will have served its purpose: **answering one concrete question, today.**

## The decision framework: gain against cost

With the table in front of you, the decision reduces to placing each technique on two axes: how much relevance it gains and how much latency it costs. Suppose these results in the estimation system:

> *(Table image in the original — not included in this repo.)*

The naive reading is "reranking multiplies latency by eight", which is arithmetically true and irrelevant. **The correct reading uses the right denominator: the latency budget of the complete experience.** In this system, retrieval is followed by the LLM generating the estimate, which takes several seconds. The added 255 ms are less than 5% of the total time the user perceives — and in exchange, two of every five documents in the context go from being noise to being signal. The estimate is generated over correct references. The decision is obvious in this direction.

Change the scenario and the same table decides the opposite. In an interactive autocomplete with a total budget of 300 ms, those same 255 ms are 85% of the budget: unaffordable even if the relevance gain were double. **The technique is neither good nor bad; it is expensive or cheap relative to a budget, and the budget is set by the product, not by the pipeline.**

> *(Figure in the original: `articulo-02-figura-02-cuadrante-decision.jpg` — a decision quadrant: relevance gain on the vertical axis against added latency as a fraction of total budget on the horizontal, with four zones — enable without hesitation, evaluate against budget, discard, and the marginal-gain zone where the added complexity does not pay.)*

The quadrant leaves a treacherous zone worth naming: **small gain at small cost.** The temptation is to enable the technique "because it adds something and barely costs anything". But a technique's cost is never only its latency: it is also the extra model to operate, the dependency to update, the new failure mode to diagnose at three in the morning. **A 0.02 improvement in precision rarely pays that complexity toll.** If the table does not show a gain you can notice, the senior answer is not to add the piece — and the table is precisely what lets you say "no" with grounds, which is what it was built for.

## What this measurement does not give you (and why that is fine)

Let us be honest about the tool's limits, because using it beyond them would indeed be a mistake.

A golden set of ten queries **has no statistical power**: a difference of 0.05 between two configurations may be annotation noise. The differences that justify decisions with this tool are the large, consistent ones — 0.48 to 0.80 — not tenths. The annotation carries the annotator's bias: a single annotator with a written criterion is enough to decide, but their judgment is not the domain's universal truth. And **the measurement stops at retrieval**: it says which documents reach the LLM, not what the LLM does with them; perfect retrieval does not guarantee a correct estimate, it only makes one possible.

None of this invalidates the practice, because the practice claims no more than it gives. Artisanal measurement answers a design question — *does this technique enter the pipeline?* — with exactly the rigour that question needs. Systematic evaluation of a complete RAG system, with dedicated frameworks, metrics over generation and continuous execution, is a discipline in itself and has its own place later in the programme. Arriving there having internalised the artisanal version means arriving understanding what the frameworks measure inside, instead of consuming their numbers on faith.

## The number that changes the conversation

The idea to take away: the difference between a retrieval system that evolves with judgment and one that accumulates fashionable techniques is not in the techniques — it is that every addition was decided against a table with two columns: **how much it gains, how much it costs.** Building that table costs an afternoon: a handful of real queries, manual annotation with a written criterion, a metric that fits on a napkin and a median of latencies. It is the most profitable tool in the whole RAG arsenal, and the only one in this module that requires writing no production code.

In the live session that table will be the protagonist: every retrieval technique we incorporate into the estimation system will pass through it, and we will see live how the numbers — and not impressions — decide what stays in the pipeline and what is discarded.
