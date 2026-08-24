---
title: "Content augmentation: preparing retrieved context before generating"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-08-24
summary: >
  The layer between retrieval and generation, where raw chunks become distilled
  evidence. Noise in the context costs three measurable things — tokens,
  attention, and hallucination risk, since irrelevant figures are fuel for
  grabbing the wrong one. Extractive compression cannot invent and wins for a
  corpus of figures; abstractive compression adds a second generation point and
  must be fenced by a strict schema where a missing figure stays None. Every
  stage preserves chunk_id or traceability dies before it is built, and every
  stage that can discard must log what it discarded. Closes on the problem
  distillation cannot touch: two clean, comparable sources that disagree.
keywords: [content augmentation, context distillation, extractive compression,
           abstractive compression, structured extraction, BudgetEvidence,
           Pydantic, chunk_id provenance, lost in the middle, edge loading,
           token budget, dropped chunks, silent recall loss, over-compression]
---

# Content augmentation: preparing retrieved context before generating

*Antonio Perez* · 🔴 19 min

When the estimation system receives a transcript and turns it into a query, the retrieval pipeline returns a handful of fragments of historical budgets: the most relevant ones, already filtered, reordered by a cross-encoder and weighted by recency. It is a good candidate set. The problem starts immediately afterwards, in the step almost nobody looks at: **what exactly do we hand the model that is going to generate the estimate?**

The honest answer, in most implementations, is "the fragments as they are". They are wrapped in delimiters, ordered, whatever does not fit is truncated, and into the prompt. And there is the hole. A fragment of a real budget is not a clean record card with the figure you are looking for. It is a piece of a client document: headers, payment terms, an authentication module that is beside the point, the payments one that is not, totals, VAT, a note about the milestone calendar. When you are estimating a payments module and you retrieve that fragment, **80% of what you give the model is noise for this particular query.**

Content augmentation is the layer that lives between retrieval and generation, and its job is to turn raw fragments into **distilled context**: just enough, in the right order, without losing traceability. It does not improve retrieval, that is already done, nor does it improve the generation prompt. It improves the material the model works with. And it turns out to be one of the best cost-benefit quality levers in the whole system.

## Why noise in the context is expensive

Passing raw fragments has three costs, and all three are measurable.

**The first is tokens.** You pay for every input token, and noise is tokens. If half of each fragment is budget boilerplate, you are paying double the context for the same signal. At volume, that is real money and real latency.

**The second is attention.** Models do not distribute their attention uniformly over a long context: what is at the beginning and at the end weighs more than what is left in the middle. If the line that really backs your estimate, *"payments module with Stripe gateway, 40h"*, ends up buried in the middle of a long fragment full of irrelevant lines, the model may not lean on it even though it is there. Retrieval brought it; generation ignores it.

**The third, and the most serious for an estimation system, is hallucination risk.** The greater the density of irrelevant figures you give the model, the easier it is for it to grab the wrong one. If the context contains a "120h" belonging to a reporting module from another project, and the model drags it over to the payments line, you have generated a plausible and false figure. **Numeric noise is not neutral: it is fuel for hallucination.** Distilling the context is not only efficiency, it is a first line of defence.

## The augmentation layer as a composable pipeline

Augmentation is not one technique, it is several, and they are best treated as composable stages with a uniform contract — fragments in, distilled evidence out — like the rest of the pipeline. **Each stage must be independently switchable, disableable and measurable**, because each one can help or do harm depending on the case.

```python
class AugmentedContext(BaseModel):
    evidence: list[BudgetEvidence]
    dropped_chunk_ids: list[str]
    token_estimate: int


def augment_context(
    chunks: list[RetrievedChunk],
    target_components: list[str],
    token_budget: int,
) -> AugmentedContext:
    """Turn raw retrieved chunks into distilled, ordered evidence.

    Stages: compress -> extract key points -> order -> fit budget.
    Every stage is independently toggleable and logged.
    """
    compressed = [compress_chunk(c, target_components) for c in chunks]
    evidence = [extract_key_points(c) for c in compressed]
    ordered = order_by_relevance(evidence)
    return fit_to_budget(ordered, token_budget)
```

> *(Figure 1 in the original: `art1-fig1-augmentation-pipeline.jpg` — image not included in this repo. It shows the augmentation layer as a box between **Retrieval** (raw chunks) and **Generation** (distilled context), containing four stages in sequence: *Compress* — removes the noise, *Extract key points* — into structure, *Order* — edge-loading, *Fit* — within budget. Caption: "Augmentation improves neither retrieval nor the prompt: it improves the material the model works with.")*

The signature already carries an important decision: **augmentation needs to know what you are estimating** (`target_components`). Retrieval works at the level of the global query; distillation is specific to what you are about to generate. It is what lets you discard a fragment's authentication module when you are estimating payments.

One detail that runs through every stage and is easy to break: **preserving the source `chunk_id`.** When you compress a fragment, it is still evidence derived from a concrete historical budget, and later on every line of the estimate will have to be able to point at which source it came from. If your compressor produces orphan text with no id, you have destroyed traceability before generating it. Every piece of distilled evidence drags its chunk's id along with it.

## Compression: extractive versus abstractive

Compressing a fragment means keeping what matters and throwing away the rest. There are two families, and the choice is not aesthetic.

**Extractive compression** selects fragments of the original text without rewriting them. For budgets, which are semi-structured, it is usually enough to keep the lines mentioning the component you are estimating. It is cheap, there is no model call, it is fast, and it has a property that for estimates is worth gold: **it cannot invent anything, because it only copies.**

```python
def compress_chunk(
    chunk: RetrievedChunk,
    target_components: list[str],
) -> CompressedChunk:
    """Extractive compression: keep only lines relevant to the target.

    No model call, no rewriting -> nothing can be hallucinated here.
    """
    targets = [t.lower() for t in target_components]
    kept = [
        line
        for line in chunk.text.splitlines()
        if any(t in line.lower() for t in targets) or _looks_like_figure(line)
    ]
    return CompressedChunk(
        chunk_id=chunk.chunk_id,
        document_id=chunk.document_id,
        text="\n".join(kept) if kept else chunk.text,
        was_compressed=bool(kept),
    )
```

The `if kept else chunk.text` is not an oversight: if the extractive filter is left with nothing, returning the whole fragment is preferable to returning empty. **A compression that silently empties the context is exactly the same error as an over-aggressive metadata filter in retrieval**: you degrade recall and do not find out. That is why `was_compressed` travels in the output and is logged.

**Abstractive compression** uses a model to summarise the fragment focusing on the query. It is more flexible, it understands synonyms, it reorganises, it condenses long prose, and it is the natural option for narrative fragments like transcripts, where there are no "lines" to filter. But it introduces something worth facing head-on: **a second generation point.** If the summary hallucinates — and summarising is generating — that hallucination enters the context with the appearance of a source, and the final generation cites it as though it were real data. You have moved the hallucination problem upstream, where it is harder to detect.

When abstractive augmentation is necessary, the way to contain it is not to let the model write free prose, but to force it to extract structure:

```python
class BudgetEvidence(BaseModel):
    chunk_id: str
    document_id: str
    component: str
    hours: float | None
    cost_eur: float | None
    sector: str | None
    project_year: int | None
    note: str  # short, grounded justification


def extract_key_points(chunk: CompressedChunk) -> BudgetEvidence:
    """Abstractive extraction constrained to a strict schema.

    The model fills fields, it does not write free prose. A missing
    figure stays None instead of being invented.
    """
    response = client.responses.parse(
        model=settings.augmentation_model,  # cheap tier, e.g. gpt-5-mini
        input=[
            {"role": "system", "content": KEY_POINT_EXTRACTION_INSTRUCTIONS},
            {"role": "user", "content": chunk.text},
        ],
        text_format=BudgetEvidence,
    )
    evidence = response.output_parsed
    evidence.chunk_id = chunk.chunk_id
    evidence.document_id = chunk.document_id
    log.info(
        "key_points_extracted",
        chunk_id=chunk.chunk_id,
        has_hours=evidence.hours is not None,
    )
    return evidence
```

The strict schema does two things. It restricts the output to concrete fields, so the model cannot ramble; and it allows an explicit `None`, which is the difference between "this budget did not give the hours figure" and "the model invented a number to fill the gap". In estimation, **that honest `None` is worth more than a filler figure.**

> *(Figure 2 in the original: `art1-fig2-crudo-a-evidencia.jpg` — image not included in this repo. It sets a raw retrieved fragment against the `BudgetEvidence` object it distils to. On the left, a budget document with header, payment terms, an OAuth2 line at 60h, the payments module at 40h highlighted, reporting at 120h, AWS infrastructure at 35h, VAT and total, and a trailing `chunk_id: fin-2024-07#c3`. On the right, the parsed fields: the same `chunk_id`, `component "Payments module"`, `hours 40` highlighted, `cost_eur null`, `sector "fintech"`, `project_year 2024`. The arrow between them is labelled "compress + extract" with a badge reading "preserves chunk_id", and the panel notes that an absent figure stays null rather than being invented. Caption: "Distillation turns noisy text into comparable structure, without losing where the data came from.")*

Key-point extraction is also where augmentation contributes its greatest value for this use case. A raw fragment is prose or a disordered table; `BudgetEvidence` is a clean record card with component, hours, cost, sector and year. The generation model receives **comparable structure** instead of text it has to parse mentally, and that reduces noise and risk at the same time.

## Ordering and prioritising: put the good stuff where it will be seen

Once you have distilled evidence, the order matters, and for the same reason it mattered inside each fragment: the model's attention is not flat. The most robust strategy is **edge loading** — placing the strongest evidence at the beginning and at the end of the context block, and leaving the weakest in the middle, which is where the model looks least.

```python
def order_by_relevance(evidence: list[BudgetEvidence]) -> list[BudgetEvidence]:
    """Edge-load the context: strongest evidence first and last.

    Assumes `evidence` arrives sorted by descending relevance score.
    """
    ordered: list[BudgetEvidence] = []
    front = True
    for item in evidence:
        if front:
            ordered.insert(0, item)
        else:
            ordered.append(item)
        front = not front
    return ordered
```

> *(Editor's note: this function does not do what the docstring and Figure 3 describe — it inverts the intent. `insert(0, item)` pushes everything already at the front one place inward, so with evidence sorted strongest-first, `[e0, e1, e2, e3]` comes out as `[e2, e0, e1, e3]`: the two **strongest** items land in the middle, precisely where lost-in-the-middle costs the most. It degrades as the list grows — at six items the output is `[e4, e2, e0, e1, e3, e5]`, putting the strongest evidence dead centre and the two **weakest** on the high-attention edges, which is the exact inversion of what Figure 3 prescribes. The handbook already has a correct implementation of this manoeuvre — `reorder_u_pattern` in [s09-04](s09-04-augmentation-assembling-context.md) — which builds the two halves separately and concatenates `front + list(reversed(back))`, yielding `[e0, e2, e3, e1]`. Use that shape; the prose here is right and the code is not.)*

> *(Figure 3 in the original: `art1-fig3-carga-extremos.jpg` — image not included in this repo. A U-shaped curve of model attention (vertical axis, low to high) against position in the context (horizontal axis, start / middle / end). The curve is high at both edges, labelled "strong" in green, and sags through the middle, labelled "weak", with the trough marked "lost in the middle" and the edges marked "edges = high attention". Caption: "Attention is not flat: the strongest evidence goes at the edges; the centre is the worst place for a critical figure.")*

The relevance signal for ordering does not have to be the reranker's alone. Retrieval already computed a temporal weight by recency; a budget from six months ago is usually a better cost reference than one from four years ago. Combining relevance and recency in the ordering is legitimate, **as long as you do it explicitly and measurably, not with a pile of magic multipliers impossible to justify.**

And in the end, the token budget rules. If the distilled evidence still does not fit, you do not truncate blindly through the middle: you discard the weakest pieces first **and record which**, so that a generation failure can be traced back to "I left out the source that backed it".

```python
def fit_to_budget(
    evidence: list[BudgetEvidence],
    token_budget: int,
) -> AugmentedContext:
    kept, dropped, used = [], [], 0
    for item in evidence:
        cost = estimate_tokens(item)
        if used + cost <= token_budget:
            kept.append(item)
            used += cost
        else:
            dropped.append(item.chunk_id)
    log.info("context_fitted", kept=len(kept), dropped=len(dropped), tokens=used)
    return AugmentedContext(evidence=kept, dropped_chunk_ids=dropped, token_estimate=used)
```

## Honest trade-offs

**Extractive versus abstractive is not a preference, it is a risk decision.** For a corpus of figures, extractive wins almost every time: cheap, fast, incapable of inventing. Abstractive is for when the source is narrative and there is no structure to filter. If you are going to use it, restrict it to a schema and assume you have added a generation point that also has to be watched.

**Compressing to save money can come out more expensive.** An LLM summariser per fragment multiplies by the number of retrieved fragments. If you retrieve eight and summarise each with a call, those eight calls can cost more, in money and in latency, than passing the raw fragments to generation. Abstractive compression only pays off when the saving in generation tokens exceeds the cost of the compression calls, and that depends on your volume. **Measure it before assuming it.**

**Every distillation is a decision about what to throw away, and throwing away what you should not is a silent failure.** If your extractive compressor is based on word matching and the budget called "collections module" what you are searching for as "payments", you discard it. You have degraded recall in the augmentation layer, not in the retrieval one, and that is harder to diagnose because retrieval looked correct. The lesson is the usual one: **every stage that can throw information away has to log what it threw away.**

**Over-compression erases the nuance that justified the analogy.** Sometimes a historical budget's value is not in the figure, but in a marginal note — *"40h but with the gateway already integrated from the previous project"* — which explains why that number is not transferable without adjustment. If you distil down to just "payments: 40h", the model loses the context that would have made it estimate well. **The optimal compression point is not guessed, it is measured against the quality of the final estimate.**

**Enriching is also generating.** A common technique is adding a synthetic header to each fragment that situates it — *"fintech sector budget, 2024, project with a similar payments module"* — so the model understands at a glance what it is reading. It helps, but that header is written by a model, so the same caution applies: cheap model, short format, and never a claim that is not in the fragment.

## What this leaves unresolved

Augmentation prepares the material: it cleans it, structures it, orders it, trims it to what fits. It makes the model work with signal instead of noise. But there is a problem no amount of distillation resolves, and it is the one that really separates a mediocre estimate from a good one.

**When two historical budgets, both relevant, both recent, both well distilled, say different things about the same thing, augmentation decides nothing.** One project priced the payments module at 40h; another, equally comparable, at 90h. Both enter the context clean. Compressing and ordering does not resolve which weighs more, nor how to combine contradictory evidence into a single coherent estimate that can also explain where each number comes from. That is no longer preparing the context: it is **synthesising sources.** And that is where generation quality really starts to be won.
