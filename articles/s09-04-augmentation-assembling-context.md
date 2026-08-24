---
title: "Augmentation: assembling context so the LLM uses it well"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 9
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 32 min
added: 2026-08-14
summary: >
  Why `"\n\n".join(chunks)` produces invented citations, cross-contamination
  between documents, and answers that silently ignore the retrieved context.
  The five assembly decisions that fix it — XML `<source>` delimiters carrying
  metadata, chunk order against lost-in-the-middle, whole-chunk truncation that
  counts the wrapper, a generation prompt with explicit grounding and a licence
  to refuse, and a structured schema with nullable numerics — closed by
  post-generation citation validation.
keywords: [augmentation, xml delimiters, source citation, lost in the middle,
           grounding, insufficient context, structured output, json schema,
           citation validation, reasoning effort, token budget, truncation]
---

# Augmentation: assembling context so the LLM uses it well

*Antonio Perez* · 🔴 32 min

## The temptation of `"\n\n".join`, and why it fails

Article 3's retriever has returned five chunks of historical budgets, properly filtered by sector and year, all with cosine distance below the threshold. You have Article 2's reformulated query with its structured fields (function, technologies, scale, regulatory constraints). All that is left is to pass it all to the model and obtain an estimate. The immediate temptation — the one half the RAG tutorials on the internet teach — looks like this:

```python
context = "\n\n".join([chunk.content for chunk in retrieved_chunks])
prompt = f"Context:\n{context}\n\nGenerate an estimate for: {query_text}"
response = client.responses.create(model="gpt-5", input=[{"role": "user", "content": prompt}])
return response.output_text
```

The code works, in the sense that it throws no exception and returns something that looks like an estimate. What it produces with uncomfortable regularity are three classes of failure that appear without warning in production.

**The first failure is invented citations.** The model, without clear instructions on how to reference sources, fabricates identifiers that look plausible ("according to project number 312...") when that project was not among the retrieved chunks. **The second is cross-mixing**: the model combines information from distinct chunks as if it came from a single project, generating an estimate that corresponds to none of the real historical budgets. **The third, and the most subtle, is the answer without context**: the model silently ignores the chunks you passed it and produces an estimate based on its general trained knowledge, leaving you with the false impression that retrieval worked when in fact it had no influence on the answer.

All three pathologies have a common cause: the model received no instructions on how to treat the block of text we handed it. It does not know whether the chunks are authoritative context from which it must extract facts, inspirational suggestions it may ignore, or documentation it must cite. It does not know whether it should refuse to generate when the context is insufficient or always force an answer. It does not know what source to attribute to each claim. **This stage of the RAG flow — augmentation — is not "putting chunks in the prompt"; it is constructing an input to the model that tells it all those things explicitly and makes doing the right thing easy.** This article walks the five decisions that assembly implies.

## XML delimiters: letting the model tell context from instruction

Modern models are trained on massive quantities of XML — specifically on the conventions Anthropic and OpenAI have popularised for marking structural parts of a prompt — and the practical consequence is that they recognise tags like `<source>`, `<context>` or `<document>` as semantically significant boundaries. When a chunk arrives wrapped in `<source id="142">...</source>`, the model understands two things that raw concatenation leaves unclear: where that unit of information begins and ends, and what structural metadata accompanies it.

In your project's AI service, the function that assembles the context block lives in `generation/context_assembler.py` and has this shape:

```python
def build_context_block(chunks: list[RetrievedChunk]) -> str:
    blocks = []
    for chunk in chunks:
        meta = [
            f'id="{chunk.id}"',
            f'sector="{chunk.sector}"',
            f'project_year="{chunk.project_year}"',
            f'chunk_type="{chunk.chunk_type}"',
            f'distance="{chunk.distance:.3f}"',
        ]
        attrs = " ".join(meta)
        blocks.append(f"<source {attrs}>\n{chunk.content.strip()}\n</source>")
    return "\n\n".join(blocks)
```

Three deliberate details in this format. **First**, each chunk carries structured metadata as XML attributes, not embedded in the text. The metadata is what lets the model cite precisely ("according to source id=142") and mentally filter by sector when relevant; putting sector and year in prose inside the chunk would force the model to parse it every time. **Second**, `distance` is included among the attributes. This is debatable: some systems prefer to hide the distance from the model so it does not overfit to the nearest chunks; the programme takes the opposite position because exposing the distance gives the model an explicit relevance signal it can use when weighing evidence (more on this in the ordering section). **Third**, the delimiter is `<source>`, singular, not `<context>` or `<document>`: the tag is chosen for its connotation — the model tends to treat each `<source>` as an attributable unit of information, exactly what we want in order to force citations.

A frequent alternative is **JSON delimited context**, where the block is serialised as an array of objects with `id`, `content` and `metadata`. It works, but it has a silent failure: models tend to "read" JSON structures inside the prompt as data to interpret rather than as authoritative instructions. A JSON with a `"content": "..."` field is sometimes treated as description, not as reference context. The XML delimiter carries the correct connotation of "this is reference content you must consult".

> *(Figure in the original: `art_4_figura-10-anatomia-contexto.jpg` — image not included in this repo.)*

## Chunk order: lost in the middle is real and predictable

Once the assembler builds the context block, it remains to decide in what order the chunks go inside it. The naive intuition is that it does not matter: the model reads the whole context and weights every part equally. The empirical evidence says otherwise, and measurably so.

The reference paper is **"Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., 2023)**. The experiment is direct: the authors build prompts with a context composed of N documents of which only one is relevant to the question, and systematically move the relevant document's position. The resulting accuracy curve is **U-shaped**: the model recovers the information when the relevant document is at the beginning of the context (high accuracy) or at the end (high accuracy), but loses it regularly when it is in the middle (accuracy falling by up to twenty percentage points). The effect has been replicated on GPT-4, on Claude, and on later models; it is not an artefact of a specific architecture.

> *(Figure in the original: `art_4_figura-11-lost-in-the-middle.jpg` — image not included in this repo.)*

The operational implication is direct. If your retriever returns ten chunks ordered by ascending distance, and you place them in the context in that same order, the chunk at rank=1 gets privileged attention (it is at the start), the chunk at rank=10 also gets it (it is at the end), and chunks rank=4-7 sit in the penalty zone. Your retrieval is doing the right work but the model is degrading half your chunks through prompt geometry.

There are two mitigation strategies, and the programme adopts the first for simplicity. **Most-relevant-first with no further artifice**: leave the chunks in ascending-distance order. The reasoning is that at K=5-10 (the programme's typical range) the lost-in-the-middle effect is modest and the most relevant chunks occupy the privileged opening positions. At K=15-20 the effect becomes more severe and then the **U-pattern** strategy is worth it, where chunks are reordered so rank=1 goes at the start, rank=2 at the end, rank=3 second from the start, and so on. It is simple code:

```python
def reorder_u_pattern(chunks: list[RetrievedChunk]) -> list[RetrievedChunk]:
    front, back = [], []
    for i, chunk in enumerate(chunks):
        (front if i % 2 == 0 else back).append(chunk)
    return front + list(reversed(back))
```

The programme leaves this function as a configurable option in `context_assembler.py` but does not enable it by default. The decision of when to enable it belongs to the system's operator and is based on observable metrics: if after deploying to production the estimates' quality is noticeably worse when K rises, that is a sign lost-in-the-middle is penalising and the reorder is worth it. In the live session we will see a concrete demo of the effect by deliberately placing the critical chunk in different positions and observing the resulting estimate.

> *(Editor's note — checked against the code: `reorder_u_pattern` is **not** in `context_assembler.py`, which holds only `_wrap_chunk`, `build_context_block` and `truncate_to_token_budget`. The function exists in this article and nowhere in the reference implementation, so "left as a configurable option" describes an intention rather than a state. It is still the correct implementation of the manoeuvre — see the next note — but it has to be written before it can be enabled.)*

> *(Editor's note: [s11-01](s11-01-content-augmentation-preparing-context.md) revisits this and edge-loads unconditionally rather than leaving it off by default — reasonable, since by then the context holds distilled evidence rather than raw chunks and the ordering decision carries more weight. Its own implementation is broken, though, and `reorder_u_pattern` above is the version to use: see the note at that function.)*

## Defensive truncation: cut the whole chunk, not the content

When the context block exceeds the token budget the system allows itself to spend per request, content has to be discarded. The classic antipattern is truncating by characters or by words on reaching the limit: the last chunk is cut in half, ends up incomplete, and the model tries to interpret it anyway with three predictable consequences. The first is that the truncated chunk loses semantic coherence and contributes noise instead of signal. The second is that any citation to that chunk's id will be structurally invalid — the model cites a project of which it only received half the evidence. The third is that all the truncated chunk's tokens are wasted: it contributes less than zero to the answer.

The programme's rule is **to truncate at whole-chunk granularity**: if the chunk does not fit entire, it does not enter the context. The function looks like this:

```python
def truncate_to_token_budget(
    chunks: list[RetrievedChunk],
    max_context_tokens: int,
    encoder,
) -> list[RetrievedChunk]:
    selected = []
    used_tokens = 0
    for chunk in chunks:  # already sorted by relevance
        wrapped_size = len(encoder.encode(_wrap_chunk(chunk)))
        if used_tokens + wrapped_size > max_context_tokens:
            break
        selected.append(chunk)
        used_tokens += wrapped_size
    return selected
```

The key detail is `_wrap_chunk(chunk)`: the function counts the tokens of the chunk **already wrapped** with its XML delimiters and metadata, not just the content. The delimiters cost between 30 and 50 tokens per chunk; ignoring them in the calculation leaves the system with a budget consistently more optimistic than reality. If your theoretical limit is 8,000 context tokens and you forget to count the wrappers, you will start trimming at the end of the prompt when you have already exceeded the real limit.

The second point about truncation is **leaving room for the output**. The model's token budget is total: input plus output. If the model has a 200k-token window and the expected output is 1,500 tokens (a structured estimate with citations), the context should not occupy 199,000 tokens but leave a clear buffer. The programme adopts as a heuristic rule reserving 15% of the total budget for the output and another 5% for prompt overhead (system message, instructions, structured query). On a model with a 200k window, that leaves 160k for retrieved context — which in practice is more than any reasonable retrieval is going to need.

## The generation prompt: explicit grounding and an "I don't know" policy

The prompt is what turns a block of chunks into an actionable instruction. The difference between a mediocre prompt and a disciplined one for RAG lies in four concrete elements: **the source restriction, the obligation to cite, the insufficiency policy, and the evidence/assumption distinction.** The estimator's system prompt looks like this:

```python
ESTIMATOR_SYSTEM_PROMPT = """You are a senior software estimation assistant.
Your job is to produce structured budget estimates for new software projects
based on historical reference projects.

Rules you must follow:

1. Base every estimate ONLY on the information contained in <source> blocks.
   Do not rely on general knowledge or training data to set numbers.

2. Cite every quantitative claim with the source id it comes from. Example:
   "Backend implementation: 45 engineer-days (source 142, source 387)".

3. Never invent source ids. If no source supports a claim, surface it as an
   assumption with explicit impact level instead.

4. If the provided context is insufficient to estimate the new project,
   set confidence to "insufficient" and list what additional information
   would be needed. Do not force an estimate.

5. Distinguish evidence-backed components from assumptions you must make
   to bridge gaps in the historical data.

Output must conform to the provided JSON schema."""
```

Each rule is there for a concrete reason. **Rule 1** contains explicit grounding with `ONLY` in capitals; though it seems trivial, the typographic contrast is a signal models recognise as emphasis and it improves adherence. **Rule 2** forces attribution: without this line the model cites "sometimes" and inconsistently; with it, citing is part of the contract. **Rule 3** is what reduces invented citations — the model has an explicit path ("surface as assumption") for saying "I have no evidence", which lowers the pressure to invent. **Rule 4** is the most important contract of all: the model may refuse to estimate, and the `confidence="insufficient"` output activates the downstream path the orchestrator handles with judgment. **Rule 5** numerically separates what the corpus supports from what requires extrapolation.

The user prompt is shorter and combines the context block with the reformulator's structured query:

```python
def build_user_prompt(context_block: str, structured_query: EstimationQuery) -> str:
    return f"""Historical reference projects:

{context_block}

New project to estimate:

{structured_query.model_dump_json(indent=2)}

Generate a structured estimate. Cite sources for every quantitative component.
If the historical context does not cover this kind of project sufficiently,
return confidence="insufficient" and explain what is missing."""
```

**The repetition of the instruction at the end of the user prompt is deliberate.** The system prompt defines the rules; the user prompt reactivates them right before the moment the model is going to generate. Models attend especially strongly to the end of the prompt, and putting the critical reminder there improves the rate of honest answers when the context is weak.

## Output schema: structured output as a contract

The system uses the Responses API with `text.format` and a strict JSON schema, exactly the same mechanics as Article 2's reformulator. The difference is the schema's complexity, which here captures the estimate's whole structure plus traceability metadata:

```python
from typing import Literal
from pydantic import BaseModel, Field

class SourceCitation(BaseModel):
    source_id: int
    relevance: Literal["primary", "supporting", "tangential"]
    used_for: str = Field(description="Which component this source informed")

class Assumption(BaseModel):
    description: str
    impact: Literal["high", "medium", "low"]
    rationale: str

class CostComponent(BaseModel):
    name: str
    engineer_days: int
    sources: list[int] = Field(description="Source ids that support this component")

class Estimate(BaseModel):
    total_engineer_days: int | None
    cost_breakdown: list[CostComponent]
    duration_weeks: int | None
    sources: list[SourceCitation]
    assumptions: list[Assumption]
    confidence: Literal["high", "medium", "low", "insufficient"]
    reasoning: str
    insufficient_context_explanation: str | None = Field(
        default=None,
        description="If confidence is 'insufficient', explain what is missing"
    )
```

The schema encodes several architectural decisions. `total_engineer_days` and `duration_weeks` are `int | None`: when `confidence == "insufficient"`, the model must return them as `None` instead of inventing a number. Each `CostComponent` carries its own list of `sources`, which enables fine-grained traceability per component, not only globally. `Assumption` explicitly separates "what is assumed" from "why" so later human review can evaluate the assumption. And `insufficient_context_explanation` activates the soft-fail path symmetric to the retriever's: when the model cannot estimate, the system captures the reason in a dedicated field rather than in an ad-hoc output.

The API call fits without surprises:

```python
response = client.responses.create(
    model="gpt-5",
    input=[
        {"role": "system", "content": ESTIMATOR_SYSTEM_PROMPT},
        {"role": "user", "content": user_prompt},
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "Estimate",
            "schema": Estimate.model_json_schema(),
            "strict": True,
        }
    },
    reasoning={"effort": "medium"},
)
estimate = Estimate.model_validate_json(response.output_text)
```

Two parameters deserve comment. The model chosen for generation is `gpt-5` (not `gpt-5-mini` like the reformulator): the task here — synthesising evidence from multiple sources, reasoning about components, deciding when not to estimate — is genuinely complex, and the capable model's extra cost is justified. The `reasoning.effort="medium"` parameter is what, in OpenAI's reasoning models, replaces the `temperature` earlier versions accepted: **`temperature` is no longer valid on gpt-5** — it is in the "Deprecated parameters in reasoning models" guide we covered in Session 01 — and the operational lever for controlling the balance between speed and depth is now `reasoning.effort` with values `low`, `medium`, `high`. For estimation, `medium` is the reasonable point: `low` produces superficial estimates without going deep into the reasoning, `high` increases latency and cost with no measurable improvement for this case.

## Post-generation validation: closing the loop

Structured output guarantees the output has the right *shape* — all the fields are present, the types are as expected, the `Literal`s are within their valid values. What it does not guarantee is **semantic coherence between the output and the retrieved chunks**. Post-generation validation covers exactly that.

The critical check is the citations. The model, even with clear instructions, occasionally cites a `source_id` that was not among the retrieved chunks. The cause may be an attention failure, a confusion between similar IDs, or an outright hallucination. Whichever it is, the validation is trivial and is not optional:

```python
def validate_citations(estimate: Estimate, retrieved_chunks: list[RetrievedChunk]) -> list[int]:
    valid_ids = {c.id for c in retrieved_chunks}
    cited_ids = set()
    cited_ids.update(c.source_id for c in estimate.sources)
    for component in estimate.cost_breakdown:
        cited_ids.update(component.sources)
    return sorted(cited_ids - valid_ids)
```

If `validate_citations` returns a non-empty list, the orchestrator has three operational options. The first is to retry the generation with an extra message along the lines of *"your previous response cited invalid source ids: ..."*; it usually works and is the default option. The second is to downgrade confidence automatically (if the model said `confidence="high"` but invented a citation, drop to `medium` and record the incident). The third is to reject the response and return to the business backend a message of "unreliable estimate, requires manual review". The programme adopts the first by default with a maximum of one retry; if the second attempt also cites invalid IDs, it falls to the third option.

The second validation is **confidence coherence**: if the model said `confidence="insufficient"`, the `insufficient_context_explanation` field must be present and non-empty; the numeric fields (`total_engineer_days`, `duration_weeks`) must be `None`. Any inconsistency (saying "insufficient" but filling in numbers, or saying "high" without citing sources) is treated as a malformed response and retried. The third is **numeric sanity**: an estimate of a hundred thousand engineer-days, or of three weeks for a complex B2B project, is probably a failure, and the system flags those cases for review without blocking the response — sanity helps the human who reviews, it is not an absolute guardrail.

> *(Figure in the original: `art_4_figura-12-pipeline-validacion.jpg` — image not included in this repo.)*

## Honest trade-offs

Controlling the model's "creativity" deserves a dedicated paragraph because many engineers' intuition still asks for lowering `temperature` to zero and the parameter no longer exists on gpt-5. In reasoning models, the equivalent effect — more deterministic answers, less variation between calls — is obtained with a low `reasoning.effort` combined with very restrictive prompts. The operational truth is that with a well-structured prompt and a strict structured-output schema, inter-call variability is already very low without touching parameters: the model is constrained by the output shape and by the system prompt's rules, and that removes most of the degrees of freedom that in free chat would produce different answers.

The question of **strict vs flexible instruction** returns to the asymmetry of errors that structured Article 3. The current system prompt is severe: "ONLY from context", "never invent", "return insufficient if needed". That severity has a cost — the model sometimes refuses to estimate where a human could reasonably have extrapolated — but the saving in hallucinations compensates. The flexible alternative ("use the context as primary reference but you may extrapolate when reasonable") produces more apparent coverage and much less real reliability. **For a system whose output will influence project budgets, severe is better than complicit.**

The **cost of mandatory citations** is measurable and worth naming. Forcing the model to cite every quantitative component increases output tokens by between 10% and 20% — each `SourceCitation` is between five and ten tokens, and a breakdown of ten components with citations easily doubles the response's size versus one without. Over thousands of requests a month, the cost is not negligible. But the traceability the citations enable is what distinguishes an estimate "the system produced" from one "the system can defend", and for financial estimation that distinction is operationally critical.

## Connection with the live session

The session's fifth block is an iteration over the generation prompt. We will start from the minimal prompt (raw concatenation, no rules) and add constraints one at a time, observing how the output changes over the same transcript and the same chunks. First only XML delimiters with no special instructions: the output starts to take shape. Then explicit grounding ("ONLY"): the general-knowledge hallucinations disappear. Then the obligation to cite: attributions appear but some are invented. Then the insufficiency policy: the model stops forcing estimates when the context does not support one. Each step of the experiment is observable in the output and takes advantage of the student having the end-to-end flow already assembled.

There is also a deliberate demo of the lost-in-the-middle phenomenon. We will take the same set of five chunks and generate the estimate with two different orders: most-relevant-first and an adversarial order where the critical chunk sits in position three. The difference in the resulting estimate is visible — sometimes the model simply does not use the critical chunk — and the exercise serves to internalise that context order is not neutral, but part of the prompt's design.

What closes this article and leads to the next is an architectural observation. Up to here, everything we have built in S09 — reformulator, retriever, assembler, generator — lives in a single Python process. That is what is asked for to reach the MVP, but it leaves the system with an operational problem: the retriever (which S10 will evolve with reranking) and the generator (which the business will hit thousands of times a day) live behind the same endpoint, with the same authentication, with the same rate limit, in the same process. Article 5 separates those two layers into distinct routers of the AI service, gives them different security and rate-limiting regimes, and connects the pattern with how the Rails business backend will invoke the AI service in production.
