---
title: Integrating dynamic context from external sources
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 5
series_part: 1
scope: time-sensitive
source: user-supplied article
reading_time: 27 min
added: 2026-08-25
summary: >
  How to enrich a CAG system's context at request time without jumping to RAG
  yet. Static context lives in code and is paid for once through caching;
  dynamic context is fetched per request, costs tokens every turn, adds latency
  the user feels, and is input rather than program — treat it as injectable and
  delimit it. Three mechanisms: attachments (multimodal upload versus local
  extraction, one or the other, never both), web search (provider tool,
  LLM-oriented service, or raw SERP), and business-database queries, which must
  go through an HTTP contract and never a shared database.
keywords: [dynamic context, static context, CAG, attachments, PDF extraction,
           pypdf, Docling, multimodal, Files API, web search, Tavily, SERP API,
           function calling, layer isolation, prompt injection, token budget,
           latency, agentic loop]
---

# Integrating dynamic context from external sources

*Antonio Perez* · 🔴 27 min

Until now, the estimator has worked with a closed contract: the client sends a transcript, the AI service assembles the system prompt with the static CAG context, and the LLM produces the estimate. All the knowledge entering the decision lives in two places: the transcript the user sends, and the context loaded by the product team into the Jinja2 templates.

That model breaks as soon as the system leaves the laboratory. Nobody estimates a project from an isolated transcript — there is always a PDF of the technical specification, a previous commercial proposal, a recent benchmark of the database the team is considering, or a history of similar projects in the internal database. The user expects the estimate to incorporate all that material, and the only alternative to integrating it is asking them to copy and paste its content into the transcript, which is exactly the bad experience we have spent the whole of session 04 avoiding.

This article covers the three canonical mechanisms for enriching a CAG system's context at run time without jumping to a RAG architecture yet: **file attachments, web search, and queries to the business backend's database.** Each solves a different kind of need. Knowing when to choose each — and when to combine them — is what separates a prototype from a product.

## 1. The critical distinction: static context vs dynamic context

Up to session 04, all the context travelling to the LLM has been **static**: it lives in code (Jinja2 templates, examples hardcoded in the system prompt) or in typed parameters the form produces. It is predictable, versionable and testable.

**Dynamic context** is what the system obtains at run time, in response to a concrete request. It does not live in code; it lives in external systems: the user's filesystem, the web, a database, a ticketing system. And there are three operational rules to internalise before going on:

**Rule 1 — dynamic context is input, not program.** Treating it as code (concatenating it blindly into the prompt, letting the user inject instructions disguised as an attachment) is the recipe for classic prompt injection. Any content entering from outside your system must be clearly delimited in the prompt, and the LLM is never given the ability to interpret it as instructions.

**Rule 2 — dynamic context has a real cost per request.** While static context is paid for once in token caching, dynamic context is re-included in every call and consumes new tokens. Attaching a 30-page PDF to every turn of a conversation easily doubles the session's cost.

**Rule 3 — dynamic context introduces latency your user notices.** Processing a PDF can take 1-3 seconds before it reaches the LLM. Doing a web search adds another 2-5 seconds. And querying the business backend's database, another round trip. The difference between a product that feels alive and one that feels broken is right here.

With those three rules assumed, on to the three mechanisms.

## 2. File attachments

The user uploads a PDF with the project's technical specification. Your AI service has to incorporate that content into the LLM's context. There are two canonical ways to do it, and the choice is not trivial.

> *(Figure in the original: `002-caminos-a-b-adjuntos.jpg` — image not included in this repo. Two parallel columns. **Path A — Direct multimodal**: original PDF → provider's Files API (binary upload) → multimodal LLM (text + diagrams), listed as zero extraction code, interprets diagrams and images, coupled to the multimodal provider, more tokens consumed. **Path B — Local extraction**: original PDF → local extraction (pypdf, PyMuPDF, Docling) → any LLM (text only), listed as provider-independent, fine control over the content, loses visual information, prepares the ground for RAG. Caption: "The choice is not trivial: both paths are defendable depending on context.")*

### Path A — Direct multimodal: the PDF travels to the LLM

The large providers have added native support for PDFs in recent months. You pass the file to the API and the model extracts text, interprets diagrams and reasons over the visual content without you having to do any preprocessing.

At Anthropic, the canonical pattern is uploading the file through the Files API and then referencing it in the message's content block:

```python
import anthropic

client = anthropic.Anthropic()

with open("specification.pdf", "rb") as f:
    uploaded = client.beta.files.upload(
        file=("specification.pdf", f, "application/pdf"),
    )

response = client.beta.messages.create(
    model="claude-opus-4-7",
    max_tokens=2048,
    betas=["files-api-2025-04-14"],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {"type": "file", "file_id": uploaded.id},
                },
                {
                    "type": "text",
                    "text": "Use this technical specification as additional context when producing the estimate.",
                },
            ],
        }
    ],
)
```

OpenAI offers the same pattern in the Responses API, with the file uploaded via the Files API and referenced in the message's input. The conceptual difference with Anthropic is minimal; the SDK difference is encapsulated by your provider wrapper from session 03.

> *(Editor's note: `claude-opus-4-7` and the `files-api-2025-04-14` beta are both current and correctly used, and the `document` → `source: {type: "file", file_id}` block shape is the documented one. One omission worth knowing when adapting this: **the beta flag is required on the upload call too**, not only on `messages.create` — the snippet passes `betas=` only to the second call.)*

The advantages are clear. Zero extraction code. The model sees the diagrams and interprets them — an ASCII architecture diagram, a Gantt chart, a screenshot of a wireframe — and incorporates it into its reasoning. The file's upload latency is paid once (the Files API keeps the file for the conversation, you upload only once) and the remaining turns reference the `file_id`.

The disadvantages are less obvious but important. You are coupled to the multimodal provider: switching from OpenAI to a local open-source model with no native PDF support forces an architecture change. Token consumption is higher than for an equivalent extracted text, because the model internally tokenises both the textual content and a visual representation of each page. And you have less control over which parts of the document enter the context: all or nothing.

### Path B — Local extraction: only the text travels to the LLM

The other path is extracting the document's content in your AI service, before the LLM call, and sending only text. For text-native PDFs, `pypdf` or `PyMuPDF` resolve the simple case in a few lines; for scanned PDFs or complex layouts, libraries like Docling or MarkItDown produce structured markdown preserving tables, headings and hierarchy. For Word there is `python-docx`.

```python
from pypdf import PdfReader
from io import BytesIO


def extract_text_from_pdf(file_bytes: bytes) -> str:
    reader = PdfReader(BytesIO(file_bytes))
    parts = []
    for index, page in enumerate(reader.pages, start=1):
        text = page.extract_text() or ""
        parts.append(f"--- Page {index} ---\n{text}")
    return "\n\n".join(parts)
```

Once you have the text, you concatenate it into the prompt with a clear delimiter the LLM can recognise:

```python
attachments_block = "\n\n".join(
    f"<attachment filename='{att.filename}'>\n{extract_text_from_pdf(att.bytes)}\n</attachment>"
    for att in attachments
)

user_prompt = f"""
<transcript>
{transcript}
</transcript>

<attachments>
{attachments_block}
</attachments>

Produce a software estimate based on the transcript. Use the attachments as additional context.
"""
```

The advantages invert relative to path A. You are provider-independent: your wrapper keeps working with any model, multimodal or not. You have fine control over what passes into the context: you can filter pages, redact sensitive sections, trim to a token budget. And, crucially, **you are preparing the ground for RAG**: the text-extraction logic you write today is exactly the first piece of the chunking pipeline you are going to assemble in module 3.

The disadvantages are the code you have to maintain (extraction for PDF, for Word, for images…) and the loss of visual information: if the document contains a critical architecture diagram, a plain-text extraction does not capture it. There is a middle ground — Docling, MarkItDown and LlamaParse internally use multimodal models to produce enriched markdown — but then you depend on external services again.

### How to choose

Either path is defendable inside a mature CAG architecture. The operational choice:

- If what matters is development speed and you do not mind lock-in with a multimodal provider: **path A**.
- If you want to understand the full document-processing flow better and prepare conceptually for the RAG module: **path B**.

**What you do not do is implement both in parallel.** It is an architectural decision, not a feature that accumulates.

> *(Editor's note — checked against the code: the estimator took **path B**, and its own docstring gives the article's reason for it. `app/attachments/extractor.py` dispatches by extension to `pypdf` for PDF and `python-docx` for DOCX, truncates each extraction to `max_chars` "to protect the prompt budget", and notes that "real chunking and retrieval enter in module 3" — which is precisely path B's argument that the extraction logic is the first piece of the RAG pipeline. Mechanisms 2 and 3 are **not implemented**: no Tavily, no SERP client, no `find_similar_projects` tool and no internal endpoint, at this branch. The Rails controller in this article is a pattern sketch, not a transcription of shipped code.)*

## 3. Web search

The second dynamic-context mechanism is web search. The need arises when the estimate involves technologies, prices or benchmarks the model does not have in its knowledge cutoff. Imagine the transcript mentions "we want to use Bun instead of Node" and you need to estimate the learning curve for a team coming from Node — the relevant data is recent and the model, with no web access, is going to hallucinate or answer with exaggerated caution.

There are three approaches, and again the choice depends on how much control you want and how much coupling your architecture tolerates.

### Approach 1 — The provider's native tool

OpenAI and Anthropic expose web search as a first-class tool inside the Responses API and the Messages API respectively. You enable it in the tools list and the model decides when to use it:

```python
response = client.responses.create(
    model="gpt-4.1",
    input=user_prompt,
    tools=[{"type": "web_search"}],
)
```

It is the simplest option and the one with the best integration with the model's reasoning: the LLM can decide it needs to search, formulate the query, receive the results and cite them in its answer as part of a single flow. The cost is billed on the same provider account.

The catch: total lock-in. If tomorrow you change provider, you lose the tool. And the quality of the results depends on the provider's index (Bing in OpenAI/Azure's case, a mixture at Anthropic), which you do not control.

### Approach 2 — An independent search service

The alternative is delegating the search to a service specialised for LLMs, such as Tavily, Exa or Firecrawl. These services return results optimised for model consumption: longer snippets, content extracted as clean markdown, semantic ranking, domain filters.

```python
from tavily import TavilyClient

tavily = TavilyClient(api_key=settings.tavily_api_key)


def web_search(query: str, max_results: int = 5) -> list[dict]:
    results = tavily.search(query=query, max_results=max_results)
    return [
        {"title": r["title"], "url": r["url"], "snippet": r["content"]}
        for r in results["results"]
    ]
```

Then you expose the function to the LLM as a tool you define yourself, like any other function calling. The LLM decides when to invoke it, your AI service executes it against Tavily, and returns the results to the model on the next turn.

The advantage: provider independence, AI-oriented result quality, the same wrapper serves any LLM. The catch: you have to wire the function calling yourself, maintain one more key, and pay another bill.

### Approach 3 — A traditional SERP API

SerpAPI or Serper are the rawest option: they return Google's or Bing's structured results as they are. You have to do the content fetching, the cleaning and the summarising yourself before passing it to the LLM. It is the option of maximum control and maximum maintenance load. For the estimator it rarely makes sense — the investment does not pay off unless you have very specific search requirements.

### When to enable search

Web search is not free: it adds latency (2-5 seconds) and tokens in the LLM's context. The practical rule is to enable it **only when the system prompt cannot answer with the model's own information and the question is time-sensitive.** For the estimator, that translates to:

- Recent technologies (framework versions released in the last 6 months).
- SaaS price comparisons.
- Recent hardware or cloud service benchmarks.
- Availability or stability of specific libraries.

For everything else — architectural patterns, team practices, typical risks by project type — the model already has the information in its training cutoff, and enabling web search only adds noise.

## 4. Queries to the business backend's database

The third mechanism is the most interesting from an architecture point of view because it exposes the programme's layer separation clearly. The need: when the system estimates a project, you want it to consider the similar projects your company has done before — their actual hours, their deviations, their materialised risks. That data lives in the business backend's database (PostgreSQL, MySQL, whatever), not in the AI service.

### What NOT to do: let the AI service access the database directly

The temptation is giving the AI service direct access to the business backend's database. It is an architectural error that is paid for dearly:

- **Schema coupling**: the AI service ends up knowing the internal structure of the business backend's data model, and any schema change breaks both.
- **Permissions**: the AI service ends up with database credentials that outlive their purpose and have overly broad permissions.
- **Duplicated logic**: the business rules about which projects to count, how to aggregate hours or how to filter end up implemented in two places.

When we reach modules 3-4 with RAG, the vector database *will* live close to the AI service — but that is the AI service's *knowledge* database, not the business backend's *operational* database. The distinction matters.

### What TO do: function calling against the business backend

The correct pattern is expressing the query as a tool the LLM can invoke, where the tool's implementation makes an HTTP call to the business backend, which is the one resolving the query against its own database and returning a clean payload.

As a diagram:

```
LLM
 │  (decides to invoke tool)
 ▼
AI service (Python)
 │  (authenticated HTTP call)
 ▼
Business backend (Rails or another stack)
 │  (queries its database applying business rules)
 ▼
Business backend's PostgreSQL
```

The AI service defines the tool:

```python
similar_projects_tool = {
    "type": "function",
    "function": {
        "name": "find_similar_projects",
        "description": (
            "Find historical projects with similar scope, technologies and team size. "
            "Returns aggregated metrics on actual hours, deviations and materialized risks."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "technologies": {"type": "array", "items": {"type": "string"}},
                "team_size": {"type": "integer"},
                "scope_summary": {"type": "string"},
            },
            "required": ["technologies", "scope_summary"],
        },
    },
}
```

And the implementation is an HTTP client to the business backend:

```python
async def find_similar_projects(
    technologies: list[str],
    scope_summary: str,
    team_size: int | None = None,
) -> dict:
    response = await http_client.post(
        f"{settings.business_backend_url}/api/internal/similar_projects",
        json={
            "technologies": technologies,
            "scope_summary": scope_summary,
            "team_size": team_size,
        },
        headers={"Authorization": f"Bearer {settings.internal_api_token}"},
    )
    response.raise_for_status()
    return response.json()
```

For its part, the business backend exposes the internal endpoint (in Ruby on Rails, aligned with the programme's reference implementation):

```ruby
# app/controllers/internal/similar_projects_controller.rb
module Internal
  class SimilarProjectsController < InternalApiController
    def create
      similar = Project.completed
        .with_any_technology(params[:technologies])
        .with_scope_similar_to(params[:scope_summary])
        .limit(5)

      render json: {
        projects: similar.map { |p| ProjectMetricsSerializer.new(p).as_json }
      }
    end
  end
end
```

Remember: **the pattern is independent of the business backend's stack.** The AI service's HTTP client talks to any backend exposing an authenticated REST endpoint, be it Rails, NestJS, Spring, Django or a Go service.

### Why this pattern is the correct one

This architecture preserves the programme's three layers cleanly. The AI service knows nothing about the projects schema: it only knows there is a `find_similar_projects` tool that takes certain parameters and returns certain data. The business backend keeps its authority over the rules of what counts as a "similar project" and which metrics to return. And the operational database is still accessed only from where it should be.

When we build RAG in module 3, the pattern will get more complicated — the vector database will live in the AI service and will query by semantic similarity — but the isolation rule between layers holds: **the AI service and the business backend talk to each other by HTTP contract, never by a shared database.**

## 5. Combining the three mechanisms

In a real estimator case, the three mechanisms can coexist in the same request. The user uploads a transcript and a specification PDF (mechanism 1), the LLM decides it needs current AWS prices for certain services mentioned (mechanism 2), and it also queries the similar projects from the history (mechanism 3) before producing the final estimate.

> *(Figure in the original: `001-tres-mecanismos-contexto-dinamico.jpg` — image not included in this repo. Three external sources on the left — **attachments** (PDF, Word, images), **web search** (recent data), **business database** (history, catalogue) — all converging on **AI service** (Python + FastAPI) in the middle, which connects to **LLM** (reasoning) on the right. Caption: "Every source adds tokens and latency. Enable them with judgment, not by default.")*

The general pattern orchestrating this is the **agentic loop** the Responses API already implements by default: the LLM reasons, decides which tool to invoke, receives the results, reasons again, and either calls another tool or produces the final answer. You expose the tools and let the model decide the sequence.

There are two disciplines any CAG system enriched with dynamic context needs to internalise to work well in production:

1. **Token budget.** The sum of system prompt + transcript + attachments + search results + database results can blow up the context window fast. Define a maximum budget per turn and apply truncation or summarisation when it is exceeded.
2. **Traceability.** Every tool invoked is a new observable span. Connect this with the structured observability you set up in session 03 (structlog + Logfire/Langfuse). A turn of the estimator stops being one LLM call and becomes a graph of invocations that needs end-to-end visibility.

## 6. Summary: when each mechanism

The three mechanisms cover three different kinds of need, and you should have them mentally classified in order to choose well:

| Mechanism | When to use it | Main cost | Latency added |
|---|---|---|---|
| **Attachments** | The user supplies documentation specific to this request | Tokens of the processed document | 1-3 s per document |
| **Web search** | The question is time-sensitive or requires data after the training cutoff | Tokens of the snippets + the search provider's bill | 2-5 s per query |
| **Business backend's database** | The answer must be based on the company's own data (history, catalogue, clients) | HTTP round trip + tokens of the returned payload | 100-500 ms per query |

The two questions you must ask yourself before adding any dynamic-context mechanism to the system are always the same: **could the model answer well without this information, and it is only going to answer better with it — or without this information is it going to fail systematically?** And, **is the added latency going to degrade the experience more than the value the extra context adds?**

The instinct in LLM systems tends to be "add more context, it cannot hurt". **It can.** More context means more tokens per turn, more latency, more surface for prompt injection, more debugging complexity when something goes wrong. The operational rule is the opposite: start with the minimum necessary context, measure the quality of the answers, and only add dynamic context when you have evidence the system needs it.

The three mechanisms you have seen here are the tools. The architectural discipline to use them with judgment is what separates a CAG system that scales from one that drowns.
