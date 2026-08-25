---
title: Quality evaluation with RAGAS
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 6
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-08-25
summary: >
  Guardrails answer "is this estimate trustworthy?"; nothing in the system
  answers "is the system good, and is it getting better?". Four metrics, two
  for retrieval and two for generation, and the split is what turns four loose
  numbers into a diagnosis — low faithfulness with good context means the
  generator, high faithfulness with low recall means retrieval. The golden set
  is the ceiling on all of it: without a case whose correct answer is "not
  enough data" you are not measuring abstention, you are rewarding answering.
  Offline with ground truth is a regression gate; production has no reference,
  so only the reference-free metrics survive there.
keywords: [RAGAS, faithfulness, answer relevancy, context precision, context
           recall, golden set, ground truth, adversarial cases, abstention
           cases, regression gate, reference-free monitoring, drift, Goodhart's
           law, LLM judge, relative comparison]
---

# Quality evaluation with RAGAS

*Antonio Perez* · 🔴 19 min

The estimation system already does things that a few weeks ago looked hard: it retrieves relevant budgets, synthesises sources that contradict each other, cites verifiably, detects when it is inventing a figure and abstains instead of lying. Every answer it produces passes through guardrails that check whether *that* answer is trustworthy.

And yet, there is a question the system cannot answer about itself: **is it good?** Not "is this estimate good?" — the guardrails answer that — but "is the *system* good, and is it getting better or worse?" Because you have taken dozens of design decisions — the generation prompt, the model, the way the context is assembled, the reranker, the embedding version — and every time you change one, you have no idea whether quality goes up or down. You change the prompt because "it seems better", you deploy it, and you discover weeks later, through a complaint, that you introduced a regression. **Without measurement, every change is a blind bet with a very well-fitted blindfold.**

RAGAS is the framework that removes the blindfold. It turns a RAG system's quality into four numbers you can compute over a test set and compare between versions. It does not tell you whether an answer is true — that is impossible to know with certainty — but it tells you whether today's version is better than yesterday's, and where it is losing quality if it is losing any.

## Four metrics that separate retrieval from generation

RAGAS measures four things, and the key to using them well is understanding that **two measure retrieval and two measure generation.** That division is what turns four loose numbers into a diagnosis.

**Faithfulness** measures whether the answer is grounded in the retrieved context: what proportion of the estimate's claims can be inferred from the budgets that were retrieved. It is the at-scale version of hallucination detection: if the system claims "40h for payments" and that derives from no fragment of the context, faithfulness drops. It measures **generation**.

**Answer Relevancy** measures whether the answer really addresses the question, without wandering off or padding. An estimate answering about components the transcript never asked for, or falling short of what was asked, scores low. It measures **generation**.

**Context Precision** measures how many of the retrieved fragments are genuinely relevant, and whether the relevant ones are well positioned at the top. It is the metric evaluating the reranker's and the search's work: a lot of retrieved noise, or the good stuff buried at the bottom, sinks it. It measures **retrieval**.

**Context Recall** measures whether retrieval brought all the context needed to ground the reference answer. If estimating a module well required three budgets and only two were retrieved, recall drops. It measures **retrieval**, and it needs a reference answer to compare against.

> *(Figure 16 in the original: `art6-fig16-cuatro-metricas.jpg` — image not included in this repo. The flow *query → retrieved context → answer* across the top, with two boxes beneath. **Retrieval** ("evaluate the retrieved context") contains Context Precision and Context Recall, arrowed at the context node; **Generation** ("evaluate the generated answer") contains Faithfulness and Answer Relevancy, arrowed at the answer node. A panel below reads: "Read them together to localise the loss: low faithfulness + good context → generation problem; high faithfulness + low recall → retrieval problem." Caption: "The metrics do not only say quality dropped: they say where to look.")*

The real usefulness appears when you read the four together. **If faithfulness falls but context precision and recall stay high, the problem is generation**: the context was good and the model did not use it well. **If faithfulness is high but context recall is low**, the model behaves well with the little it receives, but retrieval is not giving it what it needs. The metrics do not only tell you quality dropped; they tell you where to look.

> *(Editor's note: the repo has this diagnosis already run. `evals/RAGAS_BASELINE_S11.md` records `context_precision` **1.0** against `context_recall` **0.11**, plus `faithfulness` 0.55 and 0 dangling citations out of 169 — precisely the "high faithfulness, low recall" shape this paragraph reads as a retrieval problem, and the baseline attributes it to a days/hours unit mismatch and one bad query. It also contains a caution this article's trade-offs section earns: `answer_relevancy` comes out ≈ 0 across almost every query, not because the answers are irrelevant but because the "question" is a declarative brief and the "answer" is a structured estimate — a format mismatch the metric cannot see. A concrete case of a number that is worthless in absolute terms and still usable as an A/B comparison.)*

## What RAGAS needs, and why the golden set is the ceiling

To compute those metrics, RAGAS needs, for each test case, four pieces: the question, the answer your system generated, the contexts it retrieved, and a reference answer (the `ground_truth`, the correct estimate according to an expert). The first two metrics, faithfulness and relevancy, are computed without a reference; the two context ones, above all recall, need that ground truth.

And here is what is most underestimated in the whole of evaluation: **the golden set is the ceiling on your metrics' quality.** RAGAS will compute numbers over the set you give it, but those numbers are only worth what the set is worth. A small golden set, or a biased one, or one with badly made reference answers, produces confident and meaningless metrics. Building it well is the hard, human, non-automatable part of all this.

> *(Figure 17 in the original: `art6-fig17-golden-set-techo.jpg` — image not included in this repo. Two columns, both feeding into RAGAS. Left, in green, a **representative golden set** — comparable, ambiguous, contradictory, must-abstain — "covers the real spectrum, adversarial cases included", producing "reliable, comparable metrics". Right, in red, a **poor golden set** — only easy cases, no contradictions, no abstention — "rewards the system for always answering", producing "the same metrics: confident and meaningless". Caption: "RAGAS computes numbers over whatever set you give it; those numbers are worth what the set is worth.")*

For the estimation case, a good golden set is not a list of easy transcripts. It is a representative set covering the real spectrum of what the system will see: cases with comparable budgets and a clear answer, ambiguous cases, cases where the sources contradict each other (to check the system delivers a range and not a false number), and — crucially — **cases where the correct answer is "there is not enough data".** If your golden set does not include a case that must end in abstention, you are not measuring whether your system knows how to abstain; **you are rewarding it for always answering.** Adversarial cases are not optional: they are what distinguishes a robust system from one that only works on the happy path.

> *(Editor's note — checked against the code: this is the gap. `evals/golden_generation_s11.json` holds **five** queries (Q1-Q5: mobile banking, headless e-commerce, telemedicine, industrial IoT, payments gateway), each a well-formed project brief with a `ground_truth` and `relevant_budget_ids`. None is a case whose *correct answer* is "not enough data". Abstention does appear in the baseline — `RAGAS_BASELINE_S11.md` records a "Regulatory reporting" line marked `insufficient` with no invented hours — but as emergent per-line behaviour inside an answer, not as a case the set was built to test. By this paragraph's own standard, the suite is not yet measuring whether the system knows how to abstain. [s10-02](s10-02-artisanal-relevance-measurement.md) also applies here: it puts golden sets at 5-20 queries and warns that ten "has no statistical power", so five is the floor of the floor.)*

## Implementation

With the golden set built, evaluation is a batch process that puts each case through the real pipeline and collects the four pieces RAGAS needs.

```python
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)


def build_eval_dataset(golden: list[GoldenItem], pipeline) -> Dataset:
    rows = {"question": [], "answer": [], "contexts": [], "ground_truth": []}
    for item in golden:
        result = pipeline.run(item.transcript)  # real retrieval + generation
        rows["question"].append(item.question)
        rows["answer"].append(result.answer_text)
        rows["contexts"].append([chunk.content for chunk in result.retrieved_chunks])
        rows["ground_truth"].append(item.ground_truth)
    return Dataset.from_dict(rows)


def run_ragas(golden: list[GoldenItem], pipeline) -> dict[str, float]:
    dataset = build_eval_dataset(golden, pipeline)
    scores = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    )
    log.info("ragas_scores", **scores)
    return scores
```

Two implementation notes. The first, operational: RAGAS internally uses a language model as judge and an embedding model for some metrics. It is configured with your OpenAI key, `text-embedding-3-small` for the embeddings and a chat model as judge; the corpus is in Spanish and the judge evaluates in Spanish without trouble. The second, honest: **RAGAS's API has changed notably between versions** — column names, dataset classes, how the models are passed. The skeleton above reflects the classic form; **pin RAGAS's version in your project and check the exact names against it**, because what is called `ground_truth` and `Dataset` here may be called something else in the version you install.

> *(Editor's note: `scripts/eval_ragas_s11.py` implements this and adds what the prose only mentions — an explicit `LangchainLLMWrapper(ChatOpenAI(..., temperature=0))` judge and `LangchainEmbeddingsWrapper(OpenAIEmbeddings(...))`, passed to `evaluate()`. On the pinning advice, the project does the opposite and then defends itself: `pyproject.toml` declares `ragas>=0.2`, an open floor rather than a pin, `uv.lock` resolves to **0.4.3**, the code still uses the classic `question` / `answer` / `contexts` / `ground_truth` columns with `Dataset.from_list`, and `_per_query_scores` is commented "version-tolerant" and falls back to `NaN` for any metric missing from the result frame. That fallback means a renamed column degrades to silent NaNs rather than an error — which works, and is the kind of quiet failure the rest of this session argues against.)*

## From the one-off measurement to continuous monitoring

Computing the metrics once gives you a snapshot. The real value appears when you use them in two distinct modes, and it is worth not confusing them.

The first is **offline evaluation as a regression gate.** Before deploying a change — a new prompt, another model, an embeddings migration — you run the golden set through the candidate version and compare its four metrics with the current version's. If faithfulness or recall fall, you do not deploy: you have just caught a regression before it reached production, instead of afterwards through a complaint. This needs the ground truth, so it is only possible offline, over your golden set.

The second is **monitoring in production**, and it has a limit that must be respected: **in production you have no reference answer.** For a real user query, nobody has written the correct estimate in advance, so you cannot compute context recall over live traffic. What you can compute are the metrics that need no reference — faithfulness and answer relevancy — over a sample of real traffic.

```python
async def monitor_production_sample(estimates: list[ServedEstimate]) -> dict[str, float]:
    """Reference-free quality monitor on sampled live traffic.

    Faithfulness and answer relevancy need no ground_truth, so they work on
    real queries. Context recall does not: there is no reference answer for a
    live request. Alert on downward drift, not on absolute values.
    """
    dataset = Dataset.from_dict({
        "question": [e.question for e in estimates],
        "answer": [e.answer_text for e in estimates],
        "contexts": [[c.content for c in e.retrieved_chunks] for e in estimates],
    })
    scores = evaluate(dataset, metrics=[faithfulness, answer_relevancy])
    return scores
```

> *(Figure 18 in the original: `art6-fig18-offline-vs-produccion.jpg` — image not included in this repo. Two lanes. **OFFLINE · regression gate (with ground_truth)**: golden set + ground truth → the four metrics → candidate vs current → deploy / block, "catches the regression beforehand". **PRODUCTION · reference-free monitor (live traffic)**: sample of live traffic → faithfulness + relevancy, "need no reference" → alert on drift, "a trend, not a grade" — with context recall boxed in red as "impossible without a reference". Caption: "Offline measures everything and blocks regressions; production only what needs no reference.")*

To that sample you add the operational signals the per-response guardrails already produce: the abstention rate, the rate of dangling citations caught, the rate of degraded lines. Together they give a quality dashboard that moves over time, and an alert when something degrades without anyone having touched anything.

## Honest trade-offs

**RAGAS's judge is a model, with the same circularity as always.** Using an LLM to score another LLM's faithfulness inherits all the limitations of a model-based judge: the metrics are noisy and probabilistic. A faithfulness of 0.82 does not mean "82% true"; it is a comparable number. Use them as trends and as relative comparisons between versions, A against B, not as absolute truths. **The day you treat a 0.82 as an objective grade, the metric starts deceiving you.**

**The golden set is the ceiling, and a low ceiling is invisible.** A small or unrepresentative set gives numbers just as confident as a good one; the difference is that the bad one's mean nothing, and nothing in the output warns you. Investing in a broad, representative golden set with correct reference answers is not preparation for evaluation: **it is the evaluation.** The rest is compute over bad data.

**Optimising one metric alone spoils the others.** It is Goodhart's law in its RAG version: if you chase faithfulness at all costs, the system learns to abstain more and commit less, and relevancy and usefulness sink while faithfulness rises. The tension between faithfulness (not claiming what does not hold up) and relevancy (actually answering) is the same one as abstention, now visible in numbers. **The four metrics are read together, never one in isolation.**

**Drift in production is not always a regression.** Reference-free metrics over live traffic also move when the type of question users ask changes, not only when the system changes. A fall in faithfulness may come from harder queries arriving this week, not from a bug. Do not attribute every dip to a regression: cross the drift with what actually changed before acting.

**Evaluating costs, and it is a batch, not a request.** RAGAS makes many judge calls per case, multiplied by the set's size. It is offline work that is budgeted and scheduled, not something you run on every response's path. Confusing evaluation with an online guardrail is an expensive way of understanding neither.

## What closes, and what opens

With evaluation, the quality arc completes. The retrieved context is prepared so the model does not drown in noise; sources that contradict each other are synthesised into a coherent estimate with its range and its reason; every figure is cited verifiably back to the original budget; every answer is verified against its sources and abstains rather than inventing; the index is kept healthy so nothing degrades in silence; and, finally, four metrics over a golden set make it possible to know whether all that works and whether each change improves or worsens it. **The system no longer merely generates estimates: it generates them cited, verifiable and measurable.**

But all this holds for *one* estimate: a transcript goes in, an estimate comes out. Real projects are not one estimate. A serious engagement decomposes into pieces that barely resemble one another — the frontend, the business logic, the infrastructure, a security audit — and each piece wants its own retrieval, its own reasoning, its own verification. Estimating a complex system well is not generating one big answer: **it is coordinating many specialised answers**, each grounded and measurable, and putting them together without the whole losing the coherence and the traceability that cost so much to build.

Coordinating several specialised reasoning steps — decomposing a large problem into sub-tasks, distributing them, and recomposing the result — is a different architecture from the one we have raised so far. And the discipline of measuring that closes this session does not disappear when the moving parts arrive: **it becomes more necessary**, because the more parts you coordinate, the more places there are where quality can be lost without anyone seeing it.
