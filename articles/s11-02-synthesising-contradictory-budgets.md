---
title: "Synthesising multiple budgets: combining sources that contradict"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-08-24
summary: >
  Three comparable budgets price the same module at 40h, 90h and 55h, and the
  naive generator averages them into an undefendable 62. The disagreement is
  not noise — it is the data telling you the estimate depends on a variable the
  client never mentioned. Weight sources on three signals, not seven; compute a
  deterministic anchor in code with a weighted median so one outlier cannot
  drag it; then let the model reason over the arithmetic rather than invent it.
  Contradiction is declared only between strong sources, ranges are returned
  where disagreement is real, and collapsing one into a clean single number is
  lying with the appearance of rigour.
keywords: [source synthesis, contradictory evidence, weighted median, source
           weighting, deterministic anchor, two-stage synthesis, contradiction
           threshold, ranges vs point estimates, false precision, structured
           output, source_chunk_ids, small-N statistics]
---

# Synthesising multiple budgets: combining sources that contradict

*Antonio Perez* · 🔴 19 min

The estimation system rarely leans on a single historical budget. To estimate a new project's payments module, it retrieves three or four budgets from similar projects, each with its own figure. And here appears the problem no context-preparation technique resolves: **those figures do not agree.** A 2024 fintech project priced the payments module at 40 hours. Another, e-commerce from 2023, at 90. A third, fintech and recent, at 55. All three are relevant. All three are well retrieved and well distilled. And they contradict each other.

Synthesis is the step that turns that set of sources into a single coherent estimate. It is, by a distance, where the difference shows most between a system that looks serious and one that is. A naive generator does one of three things with those figures, and all three are bad: it silently averages and returns "62 hours" explaining nothing; it keeps the first one it finds; or, worse, it invents an intermediate number that sounds reasonable and corresponds to no source. All three produce a plausible figure. **None produces a defendable one.**

Synthesising well is not choosing a number. It is understanding *why* the sources disagree, deciding how much each weighs, and producing an estimate that is honest about what it knows and what it does not.

## The problem is not combining, it is that they contradict

If the three budgets all agreed on 50 hours, there would be nothing to synthesise: any average gives the right result. The work appears precisely when they disagree, and the first question is not "what number do I put?" but **"why do they disagree?"**

There is almost always a reason, and it is almost always in the data if you look. The 40-hour project had the payment gateway already integrated from a previous engagement: it priced only the connection, not the infrastructure. The 90-hour one built it from scratch and is also two years old, when the tooling was less mature. The 55-hour one is recent and of comparable scope. **The discrepancy is not noise: it is information.** It is telling you the estimate depends on a variable — whether the gateway is already integrated — that the client probably never mentioned in the transcript.

> *(Figure 4 in the original: `art2-fig4-tres-fuentes-contradiccion.jpg` — image not included in this repo. Three source cards side by side for "payments module": *Fintech · 2024* at **40h**, "gateway already integrated", weight 0.70; *E-commerce · 2023* at **90h**, "built from scratch", weight 0.50; *Fintech · 2025* at **55h**, "comparable scope", weight 0.80. Below them a number line from 40h to 90h with the span shaded and marked "open range: 40–90h", and a diamond at 55h labelled "weighted anchor". Caption: "The discrepancy is not noise: the weighted median fixes the anchor, but the range says what it depends on.")*

A synthesis that flattens that signal into "62 hours" throws away the most valuable thing the data had. A good synthesis preserves it: *"between 55 and 90 hours if built from scratch; around 40 if the gateway is already integrated"*. That is an estimate a project manager can use, because it tells them **which question to ask before committing.**

The objective, then, is twofold: produce a defendable number (or range), and make the reason for the discrepancy explicit when there is one.

## Weighting the sources with judgment

Not all sources deserve the same vote. The retrieval pipeline already computed, for each fragment, three signals that matter here: **relevance** (how applicable the reranker judged it to the query), **recency** (a weight that decays with the budget's age) and **semantic similarity** with the query. Those signals are the basis for weighting.

```python
class SourceWeight(BaseModel):
    relevance: float   # reranker score, normalized to 0..1
    recency: float     # temporal decay weight, 0..1
    similarity: float  # vector similarity, 0..1


def combined_weight(w: SourceWeight) -> float:
    """Single auditable weight per source.

    The coefficients are a policy decision, documented and versioned.
    Three signals, not seven: the goal is a number you can defend in
    one sentence, not a pile of multipliers nobody can justify.
    """
    return 0.5 * w.relevance + 0.3 * w.recency + 0.2 * w.similarity
```

The comment in the code is not decorative. The temptation when weighting is to add factors: a boost for sector, another for team size, another for client type. **Every factor you add is a magic number you will have to defend**, and seven factors interacting with one another replace the model's opacity with an artisanal opacity that only looks under control. The practical rule is harsh but healthy: if you cannot explain in one sentence why relevance weighs 0.5 and not 0.4, that coefficient was not ready. Start with the few signals that really move the result and add more only with evidence that they help.

## Aggregate before generating

There are two ways to do the synthesis. You can hand the model all the sources with their weights and ask it to reason and produce the estimate in one shot. It is simple, but the final number comes out of a black box: you do not know whether the model respected your weights, nor why it landed on that figure, and auditing it afterwards is hard.

The alternative, more robust for a system where the figure has to be defendable, is **separating the arithmetic from the judgment**. First you compute in code a deterministic anchor per component — a weighted central figure, its dispersion, and a signal for whether there is a real contradiction — and then you give the model the sources and those aggregates, so it reasons over them instead of inventing the arithmetic. The number has an auditable skeleton; the model contributes the judgment the arithmetic cannot give (understanding that 40 and 90 differ by scope, not by error).

```python
class ComponentAggregate(BaseModel):
    component: str
    weighted_estimate: float   # central anchor, in hours
    low: float
    high: float
    contradiction: bool
    sources: list[str]         # chunk_ids contributing to this component


def weighted_median(values_weights: list[tuple[float, float]]) -> float:
    """Robust central tendency: a single outlier budget cannot drag it."""
    items = sorted(values_weights, key=lambda vw: vw[0])
    half = sum(weight for _, weight in items) / 2
    acc = 0.0
    for value, weight in items:
        acc += weight
        if acc >= half:
            return value
    return items[-1][0]


CONTRADICTION_REL_SPREAD = 0.5  # >50% spread between strong sources -> surface it
STRONG_WEIGHT_FLOOR = 0.4


def aggregate_components(
    evidence: list[BudgetEvidence],
    weights: dict[str, SourceWeight],
) -> list[ComponentAggregate]:
    by_component: dict[str, list[BudgetEvidence]] = defaultdict(list)
    for item in evidence:
        if item.hours is not None:
            by_component[normalize_label(item.component)].append(item)

    aggregates: list[ComponentAggregate] = []
    for component, items in by_component.items():
        vw = [(it.hours, combined_weight(weights[it.chunk_id])) for it in items]
        anchor = weighted_median(vw)
        hours = [it.hours for it in items]
        low, high = min(hours), max(hours)
        rel_spread = (high - low) / anchor if anchor else 0.0
        strong_values = {value for value, weight in vw if weight >= STRONG_WEIGHT_FLOOR}
        contradiction = rel_spread > CONTRADICTION_REL_SPREAD and len(strong_values) > 1
        aggregates.append(
            ComponentAggregate(
                component=component,
                weighted_estimate=round(anchor, 1),
                low=low,
                high=high,
                contradiction=contradiction,
                sources=[it.chunk_id for it in items],
            )
        )
        log.info(
            "component_aggregated",
            component=component,
            anchor=anchor,
            low=low,
            high=high,
            contradiction=contradiction,
        )
    return aggregates
```

Two decisions deserve comment. **The first**: I use a weighted median and not a mean. The median is robust to outliers, and in estimation that matters, because a budget with a very different scope should not drag the anchor toward its anomalous figure merely by being in the set. **The second**: contradiction is not declared by high dispersion flat out, but by **high dispersion among strong sources**. If the only discordant figure comes from a low-weight source, it is an outlier you can ignore; if two high-weight sources disagree, it is a real contradiction that has to be brought to light. That distinction avoids flooding the user with false alerts.

> *(Editor's note: the code does not implement that second decision. `rel_spread` is computed from `min(hours)` and `max(hours)` over **all** sources, so a low-weight outlier still inflates it; `strong_values` only gates on there being more than one distinct strong figure. Two strong sources at 50h and 52h — agreement by any reading — plus one weak outlier at 200h yields `rel_spread = 2.88` and `contradiction = True`, exactly the false alert the paragraph says the design avoids. The same `low`/`high` are handed to the model as the range to stay within, so the weak outlier widens the bounds too. Measuring the spread over the strong sources only, and deriving `low`/`high` from them, makes the code match the prose. The article's own worked example is unaffected: 40h/0.70, 90h/0.50 and 55h/0.80 give an anchor of exactly 55h and a genuine contradiction, matching Figure 4.)*

## Generating the synthesis: range, reason and sources

With the aggregates computed, generation stops being "invent a number" and becomes "reason over this data". The output schema forces every component to be a **range, not a point**: when there is confidence, the range collapses (`low == high`); when there is contradiction, the range reflects it. And every component drags along the `chunk_id`s of the sources backing it, because an estimate that does not know where each number comes from is not defendable.

> *(Figure 5 in the original: `art2-fig5-sintesis-dos-etapas.jpg` — image not included in this repo. Two panels showing the split. Left, **1 · Arithmetic (deterministic)**: weighted evidence (`SourceWeight`) → `aggregate_components()` → anchor + low/high + contradiction flag. Right, **2 · Judgment (model)**: aggregates + evidence → `responses.parse` + instructions → `Estimate` carrying range · reason · sources. The connector between them is labelled "the model reasons, it does not invent the arithmetic". Caption: "The aggregate gives the auditable anchor; the model contributes the judgment the arithmetic does not have.")*

```python
class SynthesizedComponent(BaseModel):
    component: str
    low_hours: float
    high_hours: float            # equals low_hours when it is a confident point
    rationale: str               # why this number; explains any disagreement
    source_chunk_ids: list[str]
    contested: bool              # sources disagreed for a substantive reason


class Estimate(BaseModel):
    components: list[SynthesizedComponent]
    total_low: float
    total_high: float
    summary: str
```

The instructions to the model are where the synthesis's honesty is won or lost. Asking for "an estimate" is not enough; you have to explicitly forbid blind averaging and require the discrepancy to be explained:

```python
SYNTHESIS_INSTRUCTIONS = """\
You synthesize a software estimate from multiple historical budgets.

Rules:
- Never blindly average conflicting figures. If sources disagree, find the
  reason in the evidence (scope, recency, team, client complexity) and state
  it in `rationale`.
- Stay within the [low, high] range of the precomputed aggregate for each
  component unless you can name an explicit reason to go outside it.
- When `contradiction` is true for a component, return a range (low_hours <
  high_hours) and explain the dependency that drives it. Do not collapse a
  real disagreement into a false-precise single number.
- Every component must list the source_chunk_ids it is grounded on. Do not
  cite a source that was not provided.
- If the evidence for a component is thin or only weakly relevant, say so in
  the rationale rather than projecting false confidence.
"""


def synthesize_estimate(
    evidence: list[BudgetEvidence],
    aggregates: list[ComponentAggregate],
) -> Estimate:
    response = client.responses.parse(
        model=settings.generation_model,  # gpt-5, reasoning effort medium
        input=[
            {"role": "system", "content": SYNTHESIS_INSTRUCTIONS},
            {"role": "user", "content": render_synthesis_input(evidence, aggregates)},
        ],
        text_format=Estimate,
    )
    return response.output_parsed
```

The deterministic anchor and the instructions work together: the aggregate tells the model where the centre is and where the reasonable edges are, and the instructions stop it leaving that without justifying itself. **The model does not invent the arithmetic; it explains it and adjusts it with the judgment the arithmetic lacks.** For the example's payments module, that produces something like: *"55–90h if built from scratch (2023 and 2025 projects); ~40h if the gateway is already integrated, as in the 2024 fintech project"*, with the three `chunk_id`s in `source_chunk_ids` and `contested=True`.

> *(Figure 6 in the original: `art2-fig6-anatomia-componente.jpg` — image not included in this repo. Anatomy of one `SynthesizedComponent`: `component "Payments module"`, `low_hours 55` and `high_hours 90` boxed together and tagged "range · contested", `contested true`, `rationale "55–90h if built from scratch; ~40h if the gateway is already integrated"`, and `source_chunk_ids ["fin-2024-07#c3", "ecom-2023-02#c1", "fin-2025-04#c2"]`. Arrows run from that list to three source cards — 40h "integrated gateway", 90h "from scratch", 55h "comparable scope". Caption: "Each component points, by id, at the budgets it comes from: the foundation of a verifiable citation.")*

## Honest trade-offs

**One-pass synthesis versus two stages.** The single pass is simpler and makes better use of the model's reasoning, but the number comes out opaque and is costly to audit. Two stages give a deterministic, traceable anchor at the price of more machinery and a new risk: that the model departs from the anchor without justifying it, which is precisely what the instructions have to police. For an estimation system somebody will use to commit money, auditability usually wins.

**The weighting is only as good as the signals feeding it.** If the reranker's relevance is badly calibrated or the temporal decay uses a half-life that does not match your domain, the weighted anchor will be confidently wrong. Weighting sources does not fix bad signals; it amplifies them. Before tuning coefficients, check that the signals you are weighting measure what you think.

**Median versus mean, and the small-N problem.** The median is robust, but with two or three sources it is a blunt tool: the "median" of two values does not contribute much. With small sets, the real value is less in the central statistic than in the range and in the explanation of why it opens. Do not over-interpret an anchor computed over three points.

**The contradiction threshold is a policy, not a truth.** Too strict and you will mark any normal variation as a contradiction; too lax and you will hide real disagreements. The example's 50% is a reasonable starting point, not a sacred value: adjust it by observing how many alerts turn out actionable versus how many are noise.

**The range makes the product uncomfortable, and you have to defend it anyway.** A "55–90h" range is less comfortable to show than a clean "70h", and there will be pressure to collapse it. But collapsing a real disagreement into a point of false precision is **lying with the appearance of rigour**. Honest synthesis sometimes has to deliver a range, and the interface has to be ready to show it and to explain what it depends on.

**Synthesising does not manufacture information.** If the four sources are of low relevance, no weighting and no reasoning turns them into a good estimate. A confident synthesis over poor evidence is worse than one admitting the data does not stretch that far. **The output's confidence has to be a reflection of the input's quality, not a default value.**

## What this leaves unresolved

Synthesis now produces an estimate that is coherent, that reasons about its own contradictions and that delivers honest ranges instead of filler numbers. But in combining several sources into each component it has done something with an uncomfortable consequence: **it has interwoven the threads.** The payments figure no longer comes "from a budget"; it comes from three, weighted, with reasoning on top.

And that makes harder, and more urgent, a question anyone receiving the estimate is going to ask: **this 55 — where exactly does it come from?** Having the `chunk_id`s in the output is a start; it is not yet a verifiable answer. That every claim in the estimate can point, checkably, at the historical budget and the concrete line it derives from is a problem in itself, and it is the foundation on which trust in everything else is built. **An estimate that cannot show its origin is not an estimate: it is a well-formatted opinion.**
