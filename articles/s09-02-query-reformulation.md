---
title: Query reformulation
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 9
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 25 min
added: 2026-08-14
summary: >
  Why embedding a raw transcript fails — length dissolves the signal,
  conversational noise drowns the keywords, anaphora encodes nothing — and the
  five families of reformulation that fix it. Argues for structured extraction
  over HyDE and sub-query decomposition on cost predictability, debuggability
  and downstream utility, with a validated Pydantic schema, a composed search
  text, and a monitored fallback.
keywords: [query reformulation, HyDE, sub-query decomposition, step-back
           prompting, structured extraction, metadata filters, json schema,
           fallback rate, recall]
---

# Query reformulation

*Antonio Perez* · 🔴 25 min

You already have an AI service that can embed text and search for similar chunks. The temptation, when the first real meeting transcript lands, is the simplest in the world: embed that transcript and pass it to the Session 08 search endpoint. If we have taken such care to chunk the budgets well and to maintain the vector space's dimensionality, shouldn't that be enough?

> *(Figure in the original: `art_2_figura-6-impacto-espacio-vectorial.jpg` — image not included in this repo.)*

It is not enough. And the reason is no deep mystery of high-dimensional algebra; it is something far more mundane. The transcript the client just gave you over the video call looks like this:

> So look, what we were talking about the other day. We need something that brings suppliers together with buyers in our thing, right? It's not an Amazon or anything, it's more like Stripe Connect but adapted. Each supplier with their panel, their commissions, KYC because we move money between accounts... the sector is health, we're going to start with two pilot clinics in Munich so BaFin comes into it. And reconciliation with SAP is non-negotiable, they have everything set up there. How much does this come to and where do we start?

This is not a query. It is the condensed transcript of five minutes of conversation that in a real meeting would sit inside a torrent of three thousand tokens where the client also talks about their previous CTO, the delay on the last project, a management course they took last summer, and their dog. And even in this fragment, without conversational noise, three things happen that break vector search.

**First, length dissolves the signal.** Your vector database contains chunks of about three hundred tokens, each describing a concrete component of a historical budget. The embedding of a 300-token chunk lives in a very specific region of the space: this component is about "OAuth integration for B2B fintech" and its vector points toward that region. When you put two thousand tokens with five mixed topics into the same embedder, the resulting vector is something like the centroid of five different regions: a point that is "in the middle of all of them" and near none. Cosine distances to your historical chunks compress around a mediocre mean and the system cannot tell the critical from the peripheral.

**Second, conversational noise drowns the technical keywords.** In the example above, the operative words for discriminating the project — "marketplace", "Stripe Connect", "KYC", "BaFin", "SAP", "health", "pilot" — are buried among connectors and colloquialisms ("so look", "our thing", "it's not an Amazon", "non-negotiable"). The embedding model does not know those seven words are the ones that matter; it treats them as more tokens in the conversational flow. The result is a vector that resembles the average of any commercial meeting more than the specific region of "B2B fintech with KYC for health".

**Third, anaphora means nothing to the embedding model.** When the client says "what we were talking about the other day", "our thing" or "as you were telling me", they are leaning on context that is not in the transcript. Your system, which did not attend the previous conversation, has no way to resolve those references. The embedder simply encodes the words as if they were informative content, contaminating the vector with signal that refers to nothing.

The operational conclusion is direct: **you need a layer that converts the transcript into something the retriever can use.** That layer is called query reformulation, and it is the piece with the greatest impact on the system's recall because, without it, no adjustment of top-K, no threshold, no reranker will save you from the degraded input you are feeding the retriever.

## The five families of reformulation

The industrial literature groups reformulation techniques into five reasonably well-delimited families. Each solves a subset of the problems above and each has its price. It is worth knowing all of them before choosing.

**Query rewriting** is the simplest. You pass the input to the LLM with the instruction "reformulate this as a concise, specific technical query for searching a database of software budgets". The model produces something like "B2B marketplace with KYC payments for the health sector in Germany, SAP integration". You embed that output and search. The technique works reasonably well when the input is short but badly formulated, and reasonably badly when the input is long and multi-topic, because you lose information in the arbitrary compression the model performs.

**Sub-query decomposition** attacks the mixed-topics problem by fanning out into several searches. The LLM analyses the input and produces a list of sub-queries, each oriented to a sub-topic. For the transcript above they would be something like "B2B marketplace with supplier onboarding and commissions", "payment integration with KYC and fintech compliance", "accounting reconciliation with SAP", "BaFin regulation for the health sector in Germany". Each sub-query is run independently against the retriever, recovering its own chunks; at the end the results are fused — typically with Reciprocal Rank Fusion — before passing to augmentation. Recall improves notably because each sub-topic has its own semantic "beam". So does the cost: four searches instead of one, four embeddings, and the added complexity of fusing results that may be contradictory.

**Step-back prompting** goes up a level of abstraction before searching. Instead of embedding the specific, the LLM generates a more general question — "what type of fintech platform projects have enterprise clients in regulated sectors historically had" — and you search with that. The idea is that overly specific questions sometimes retrieve badly because no individual chunk covers the exact combination of requirements; going up a level finds chunks that do cover the relevant concepts, though at another granularity. In practice, step-back is a technique that shines in QA over structured knowledge (Wikipedia, technical texts) and performs worse in narrow domains like software estimation where the "general concept" to start from is diffuse. The technique exists and is valid in other contexts; in this programme we will barely touch it.

**HyDE (Hypothetical Document Embeddings)** inverts the problem elegantly. Instead of embedding the question, it asks the LLM to generate a hypothetical answer — a fictional document that would resemble the correct answer if it existed — and embeds that answer. For our transcript, the LLM would produce something like "This project consists of a marketplace payments platform oriented to German healthcare providers. It includes authentication with reinforced KYC, payment orchestration via Stripe Connect, bidirectional accounting reconciliation with SAP S/4HANA, and BaFin compliance for fintech". You embed that synthetic document and search. **HyDE's critical insight is that your vector database contains descriptive documents** (descriptions of past budget components), and a synthetic document resembles other documents in the vector space more than a short question does. It works very well when the domain is stable and known to the model — and worse when the model hallucinates technologies your company has never used.

**Structured extraction** is the fifth way and the one the programme will choose by default. Instead of asking the LLM for reformulated text or a synthetic document, you ask it for a structured object: a JSON validatable against an explicit schema, with fields for primary function, technologies mentioned, sector, scale, geography, regulations and constraints. For the transcript above the output would be an object with `function = "B2B payments marketplace platform"`, `technologies = ["Stripe Connect", "KYC", "SAP"]`, `sector = "healthcare"`, `scale = "pilot"`, `country = "Germany"`, `regulations = ["BaFin"]`. That object is used in two ways: its textual fields are composed into a synthetic text that gets embedded (similar to HyDE but more controlled), and **its categorical fields are used as metadata filters in pgvector** — something the other four techniques do not allow because they produce no structure usable downstream.

## The programme's choice: structured extraction

The reason the programme chooses structured extraction is not that it is the most sophisticated — HyDE produces slightly better vectors in pure academic benchmarks — but that it offers the best balance across four dimensions that matter in production: **cost predictability, latency predictability, debuggability, and downstream utility.**

On cost and latency, structured extraction and HyDE are comparable: both require a single LLM call before the search. HyDE generates a document of 200-400 tokens, extraction generates a JSON of 50-100 tokens; the difference in output tokens is small but consistently in extraction's favour. In query rewriting the output is shorter still but the information loss is greater. Sub-query is the most expensive option: one call to decompose plus N calls to the retriever, where N can be 3-5.

On **debuggability**, structured extraction is the only one that produces an inspectable artefact. When an estimate is bad, you can look at the intermediate JSON and see whether the reformulator misunderstood the sector, missed a key technology, or read "pilot" as scale `"small"` instead of `"pilot"`. With query rewriting you have to guess what the model understood from the reformulated text; with HyDE you have to read a synthetic document that mixes extraction with generation.

On **downstream utility**, structured extraction wins without argument. The JSON not only serves to compose the embedded query, it also feeds the retriever's metadata filters. If the client mentions "health sector", the search is not only concentrated semantically on similar budgets; it is also restricted structurally to chunks marked `sector = "healthcare"`. That is information lost by every other technique.

In the AI service, the component running this reformulation is `retrieval/query_reformulator.py`. The project's canonical schema is defined in Pydantic:

```python
from pydantic import BaseModel, Field
from typing import Literal

class EstimationQuery(BaseModel):
    function: str = Field(description="Primary product function in 3-7 words")
    technologies: list[str] = Field(
        default_factory=list,
        description="Specific technologies, services, or integrations mentioned"
    )
    sector: str | None = Field(
        default=None,
        description="Industry or vertical if explicitly mentioned"
    )
    scale: Literal["pilot", "small", "medium", "large"] | None = Field(
        default=None,
        description="Project scale if inferable from the conversation"
    )
    country: str | None = Field(
        default=None,
        description="Geographic scope if mentioned"
    )
    regulations: list[str] = Field(
        default_factory=list,
        description="Regulatory frameworks mentioned (GDPR, BaFin, HIPAA, etc.)"
    )
    constraints: list[str] = Field(
        default_factory=list,
        description="Non-negotiable requirements or hard constraints"
    )
```

The model call uses OpenAI's Responses API with a strict `text.format` so the model does not deviate from the schema:

```python
async def reformulate(transcript: str) -> EstimationQuery:
    response = await client.responses.create(
        model="gpt-5-mini",
        input=[
            {
                "role": "system",
                "content": REFORMULATION_SYSTEM_PROMPT,
            },
            {"role": "user", "content": transcript},
        ],
        text={
            "format": {
                "type": "json_schema",
                "name": "EstimationQuery",
                "schema": EstimationQuery.model_json_schema(),
                "strict": True,
            }
        },
    )
    return EstimationQuery.model_validate_json(response.output_text)
```

The system prompt instructs the model to extract only what is explicitly mentioned or unambiguously inferable, and to leave optional fields `null` when there is not enough evidence. **The temptation to "fill in" with common sense** — inferring GDPR because there is personal data, inferring Stripe because there are payments — **is the main source of production errors**, and it is worth repressing explicitly in the prompt.

The validated object is then composed into a compact synthetic text, which is what actually gets embedded and passed to the retriever:

> *(Figure in the original: `art_2_figura-5-flujo-reformulacion-fallback.jpg` — image not included in this repo.)*

```python
def compose_search_text(q: EstimationQuery) -> str:
    parts = [q.function]
    if q.technologies:
        parts.append(f"with {', '.join(q.technologies)}")
    if q.sector:
        parts.append(f"for the {q.sector} sector")
    if q.country:
        parts.append(f"in {q.country}")
    if q.regulations:
        parts.append(f"compliant with {', '.join(q.regulations)}")
    if q.constraints:
        parts.append(f"requiring {', '.join(q.constraints)}")
    return ". ".join(parts) + "."
```

For the transcript above, this produces:

> "B2B payments marketplace platform. with Stripe Connect, KYC, SAP. for the healthcare sector. in Germany. compliant with BaFin. requiring SAP reconciliation."

Compare that text with the raw transcript. The vector embedded from it lives near the region where the historical budget chunks for B2B fintech platforms in regulated sectors are, instead of in the diffuse centroid of the whole meeting. **The difference in the retriever's recall over the same vector database is typically between two and five times**, depending on the corpus.

There is a fallback pattern worth mentioning and leaving implemented. When JSON validation fails — because the model produced something that does not fit the schema, or because the transcript is genuinely ambiguous — the system falls back to a simpler version: pure query rewriting, returning free reformulated text. It is worse than structured extraction but better than the raw transcript. **The fallback must be activated and logged so you can iterate on the cases where extraction does not work; if it never fires, the reformulator is being too tolerant of its outputs; if it fires more than 5%, there is a systematic problem with the prompt or the schema.**

> *(Figure in the original: `art_2_figura-4-matriz-tecnicas-reformulacion.jpg` — image not included in this repo.)*

## Honest trade-offs: when to move up to something more sophisticated

The default decision is not the universal decision. There are conditions under which it is worth raising the architectural cost to another technique, and it is worth recognising them.

**Moving up to HyDE** makes sense when the corpus is very descriptive (each chunk is an elaborated paragraph, not a structured entry) and the model knows the domain well. For a RAG system over medical or legal documentation where chunks are narrative text, an embedded 300-token synthetic document produces better vectors than a composed 50-token text. For your software estimation system, where chunks are descriptions of budget components with semi-formal structure, the benefit is marginal.

**Moving up to sub-query decomposition** makes sense when transcripts are consistently multi-topic and the topics are orthogonal. If half your transcripts cover two or three distinct projects in the same conversation, a single structured query will lose information. In those cases, decomposition followed by RRF improves recall at the cost of doubling latency and cost. For the typical pattern — a client describing one project with several dimensions — structured extraction captures the dimensions in JSON fields and performs better than decomposing into sub-queries.

**Moving up to a hybrid technique** — structured extraction for filtering, HyDE for semantic search — is the natural next step if retrieval quality falls short after a few weeks of use. The two run in parallel: one call extracts the JSON for filters, another generates the hypothetical document for embedding. Cost doubles but recall can rise another step. It is the reasonable evolution trajectory of the system, not the initial choice.

**What does not pay off is complicating the reformulator before measuring.** The temptation to start with HyDE "because it sounds better in the papers" leads to the same place as the temptation to start with a hybrid retriever or a reranker: complex systems that are hard to debug when they fail and whose marginal improvements are not even measurable because there was no simple baseline to compare against. The module's operating rule is to build the simplest version of the component, instrument it, measure its behaviour over real transcripts, and only then decide whether the additional complexity is justified.

## Connection with the live session

The session's central block is exactly this: iterating on reformulation. We are going to take the same ambiguous transcript some of you will have worked on in the pre-session exercise and contrast three paths over it: direct embedding of the raw transcript as the naive baseline, the structured extraction you have just read about here, and HyDE as a contrast. Over each we will measure how many of the retrieved chunks actually belong to budgets from the correct sector and geography, and we will see how the retriever's quality depends almost proportionally on the quality of the query we hand it.

There is also a productive debate worth anticipating. When the model produces the JSON, there are prompt design decisions that change behaviour substantially: do we let it infer technologies not explicitly mentioned, or force it to stick to what was verbalised? Do we mark the sector as `null` when there is ambiguity, or risk inferring it? Do we accept extracting `scale = "pilot"` from a mention of "two pilot clinics", or leave that `null` too? These are the decisions that separate a reformulator that works reasonably well in a demo from one that survives day-to-day contact with real-world transcripts, and they are the material of the live session's second and third blocks.
