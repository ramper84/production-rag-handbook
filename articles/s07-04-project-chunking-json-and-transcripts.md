---
title: "Project chunking: JSON budgets and transcripts"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 7
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 26 min
added: 2026-08-13
summary: >
  Lands article 3's catalogue on two concrete document types: a structural JSON
  chunker keyed on the business unit (one component = one chunk) with parent
  context prepended, and topic-segmentation for unstructured transcripts. Gives
  an explicit rule for what belongs in the embedded text versus in filterable
  metadata, and names the parent-context trick as the static, LLM-free version
  of Contextual Retrieval.
keywords: [chunking, JSON chunker, structural chunking, topic segmentation,
           contextual headers, metadata enrichment, traceable ids, ingest router,
           pgvector filtering]
---

# Project chunking: JSON budgets and transcripts

*Antonio Perez* · 🔴 26 min

Article 3 made clear a principle that some tutorials treat as a detail and recent benchmarks confirm as decisive: **the best chunking depends on the document type.** Treating the whole corpus with the same strategy is a decision that, in heterogeneous projects, leaves quality on the table.

The programme's project is precisely heterogeneous. We have two types of data with radically different structural properties:

- **Historical budgets in JSON**: explicit hierarchical structure, predictable schema, each budget component is a logical business unit. When a future client sends a brief, we want to retrieve similar historical components to help estimate.
- **Requirements-gathering meeting transcripts**: 45 minutes of transcribed plain text, with no structure beyond optional speaker marks, with topics that alternate, are revisited, and get interrupted. When someone searches for what was discussed about authentication in the 3 March meeting, we want to retrieve the exact segment.

This article lands article 3's catalogue on these two concrete document types. For budgets we design a specific structural chunker that respects the JSON hierarchy. For transcripts we design a topic segmenter. And we connect both to the AI service's pipeline. It is the first real landing, and it is exactly the first step of the pre-session exercise.

## Two document types, two chunkers

Before getting into each design, it is worth fixing the architecture. The AI service is going to have **two specialised chunkers sharing a common interface**, not a generic chunker trying to handle both cases. The reason is the same one you would apply to any system with different input types: if two inputs require radically different treatment, two explicit implementations are cleaner than one with internal conditionals.

In Python (skeleton, no implementation):

```python
from abc import ABC, abstractmethod

class Chunker(ABC):
    """Common interface for any chunking strategy in the pipeline."""

    @abstractmethod
    def chunk(self, document: dict | str) -> list[Chunk]:
        """Split a document into a list of Chunk objects."""

class JSONStructuralChunker(Chunker):
    """Chunks budget JSON documents by business unit (component)."""

    def chunk(self, document: dict) -> list[Chunk]:
        ...

class TopicSegmentationChunker(Chunker):
    """Chunks long meeting transcripts by topic shifts."""

    def chunk(self, document: str) -> list[Chunk]:
        ...
```

The common interface serves two purposes: making each chunker testable separately, and leaving the door open to adding more strategies within the same pipeline during the live session (which is exactly what will happen in the hands-on blocks). The pre-session exercise asks you to implement `JSONStructuralChunker`; the `TopicSegmentationChunker` and others are introduced live on this same base.

A note on where this lives in the architecture: both chunkers are the AI service's responsibility (Python + FastAPI). The business backend (Rails in the reference implementation, or whichever stack the student chooses) simply calls the ingestion endpoint sending the documents as JSON. Who decides which chunker applies to which document is an internal decision of the AI service, not a contract with the business backend.

## JSON budgets: why generic splitters fail

If you take article 3's `RecursiveCharacterTextSplitter` and pass it a whole budget JSON, three things happen, all bad.

**First:** the splitter treats the JSON's braces, quotes and commas as normal characters. It is going to cut in the middle of a key, leaving `"client_metadata": {"sector":` in one chunk and `"finance", "country":` in the next. The resulting chunk is neither valid JSON nor readable prose. The embedding that comes out is noise.

**Second:** even if the cut falls in a syntactically fortunate place, **the parent-child hierarchy is lost.** A chunk containing the component "OAuth 2.0 authentication backend" without the parent budget's information (which client, which sector, which year) is semantically poor. When a query like "authentication for fintech" arrives, that chunk will compete with hundreds of other authentication components from irrelevant sectors.

**Third:** if you serialise the JSON to plain text before chunking (a common and tempting pattern), the embedding model sees something like `OAuth 2.0 authentication backend Implementation of OAuth 2.0 flows...` and treats each component as equivalent prose. You lose the explicit hierarchy the JSON format captured.

The correct strategy is the one article 3 catalogued as **structural**: respect the document's logical unit. For budgets, that unit is **the component**. A budget component is a self-contained piece that makes sense on its own: it has its name, its description, its stack, its estimated hours, its complexity. Each component is a natural chunk.

## The structural JSON chunker

The `JSONStructuralChunker` design starts from three explicit decisions worth being clear about before typing code.

**Decision 1 — Granularity: one component = one chunk.** We do not split the whole budget into one (we would lose specificity), nor split each individual field into a chunk (we would lose coherence). The component is the correct granularity because **it coincides with the domain's unit of reasoning**: when a client asks for an OAuth backend, that is what we want to retrieve, not "the complete budget where OAuth was once mentioned".

**Decision 2 — Chunk content: readable text enriched with parent context.** The chunk's `text` field (what is going to be embedded) is not the component's raw JSON. It is a readable textual representation combining the component's details with relevant context from the parent budget. Something like:

```
[Project: Mobile banking API with OAuth 2.0 authentication and PSD2 compliance]
[Client sector: finance | Year: 2024 | Main tech: ruby_on_rails]

Component: OAuth 2.0 authentication backend
Description: Implementation of OAuth 2.0 flows with JWT-based session management,
multi-tenant token isolation, and rate limiting per client.
Tech stack: ruby_on_rails, postgresql, redis
Complexity: high
Estimated hours: 120
```

The first two bracketed lines are what article 3 called **contextual chunk headers**: parent-document information prepended to the chunk. Microsoft Azure documented that this simple technique raises QA accuracy by 15 to 25 points without touching anything else in the pipeline. It is the highest-ROI lever known in RAG. We take advantage of it here.

An interesting variant of this decision is the connection with article 3's **Contextual Retrieval**. What we are doing is **the static, cheap version of Anthropic's technique**: we use parent information we already have in the JSON, without calling an LLM. In the live session we will introduce the full version with an LLM contextualiser and measure whether the difference justifies the cost.

**Decision 3 — Metadata: filterable fields that are not embedded.** There are two types of information about a component: the semantic (which we want the embedding to capture for search) and the structural (which we want as filters and as return information). The client's sector, the year, the main technology, the complexity and the estimated hours go in metadata. This serves two purposes: filtering results (`sector = 'finance' AND year >= 2023`) and returning structured information to the client without having to parse the chunk's text.

> *(Figure in the original: `sesion-07-articulo-04-figura-01-json-a-chunks.jpg` — image not included in this repo.)*

The class skeleton looks like this:

```python
from dataclasses import dataclass
from typing import Any

import tiktoken

@dataclass
class Chunk:
    chunk_id: str
    text: str
    metadata: dict[str, Any]
    token_count: int

class JSONStructuralChunker:
    """Chunks a budget JSON document at the component level.

    Each budget component becomes one chunk. The chunk text combines
    the component's own fields with contextual headers from the parent
    budget (client sector, year, main technology, project summary).
    """

    def __init__(self, model_for_token_count: str = "text-embedding-3-small"):
        self._tokenizer = tiktoken.encoding_for_model(model_for_token_count)

    def chunk(self, budgets: list[dict]) -> list[Chunk]:
        chunks: list[Chunk] = []
        for budget in budgets:
            chunks.extend(self._chunk_one_budget(budget))
        return chunks

    def _chunk_one_budget(self, budget: dict) -> list[Chunk]:
        parent_context = self._build_parent_context(budget)
        return [
            self._build_chunk(component, budget, parent_context)
            for component in budget["components"]
        ]

    def _build_parent_context(self, budget: dict) -> str:
        client = budget["client_metadata"]
        return (
            f"[Project: {budget['project_summary']}]\n"
            f"[Client sector: {client['sector']} | "
            f"Year: {budget['year']} | "
            f"Main tech: {budget['main_technology']}]"
        )

    def _build_chunk(
        self, component: dict, budget: dict, parent_context: str
    ) -> Chunk:
        text = self._render_component_text(component, parent_context)
        return Chunk(
            chunk_id=f"{budget['budget_id']}::{component['component_id']}",
            text=text,
            metadata=self._build_metadata(component, budget),
            token_count=len(self._tokenizer.encode(text)),
        )

    def _render_component_text(
        self, component: dict, parent_context: str
    ) -> str:
        return (
            f"{parent_context}\n\n"
            f"Component: {component['name']}\n"
            f"Description: {component['description']}\n"
            f"Tech stack: {', '.join(component['tech_stack'])}\n"
            f"Complexity: {component['complexity']}\n"
            f"Estimated hours: {component['estimated_hours']}"
        )

    def _build_metadata(self, component: dict, budget: dict) -> dict[str, Any]:
        return {
            "budget_id": budget["budget_id"],
            "component_id": component["component_id"],
            "client_sector": budget["client_metadata"]["sector"],
            "main_technology": budget["main_technology"],
            "year": budget["year"],
            "complexity": component["complexity"],
            "estimated_hours": component["estimated_hours"],
        }
```

What is missing for this to be production-ready, and is left as your work in the exercise: Pydantic validation of the input schema (a JSON with a missing field should not throw an opaque `KeyError`), structured logging of how many chunks each budget produces, handling the edge case "component with an abnormally long description" (is it split or accepted?), and integration with the `POST /embeddings/ingest` endpoint. Those are decisions you are going to make; the skeleton is the chassis.

## Transcripts: why character splitters destroy the content

Let's change document type. A typical requirements-gathering meeting transcript has these properties:

- Between 6,000 and 12,000 tokens. Too much to embed in a single piece.
- Zero formal structure: no headers, no sections, no topic markers.
- Alternating speakers, marked as `Speaker A:`, `Speaker B:` or similar.
- Topics that alternate, get interrupted, return later. A discussion about authentication can jump to hosting and come back to authentication 10 minutes later.
- A lot of conversational content of low information density: confirmations, repetitions, digressions.

If you apply `RecursiveCharacterTextSplitter` with 512 tokens and 15% overlap, the result is functional but poor. The cuts fall in arbitrary positions, sometimes mid-way through a speaker's turn. Two consecutive chunks can each contain half an idea, with the other half of the first left hanging. When someone searches "what requirements did we discuss about authentication", the retriever may return three different chunks where none has the complete discussion.

The correct strategy for transcripts is the one article 3 catalogued as **semantic**: topic-based segmentation. Embed consecutive sentences (or consecutive turns), detect where the similarity between neighbouring embeddings falls below a threshold, and split there. The result is chunks that coincide with coherent topic blocks, not with arbitrary cuts.

## Topic-based segmentation for transcripts

The `TopicSegmentationChunker` design starts from three decisions, parallel to the JSON chunker's but with different reasoning.

**Decision 1 — Granularity: each topic block = one chunk.** We do not segment by sentence (chunks too small) nor by minute of duration (does not respect content). We segment where the topic changes. The resulting duration varies: sometimes a topic lasts 30 seconds, sometimes 8 minutes. **The granularity adjusts to the content, not to the clock.**

**Decision 2 — Chunk content: the topic block with its meeting context.** Just as with budgets, we prepend parent-document information. In this case, meeting metadata: client, date, main participants, project phase. The chunk's structure ends up as:

```
[Meeting: Requirements gathering · 2024-03-15]
[Client: FintechCorp · Phase: discovery]
[Speakers: Antonio (consultant), Maria (CTO), Pedro (lead dev)]

Topic block: Authentication and security
Maria: We need OAuth 2.0 with refresh tokens, and we need it to work with our existing
SAML provider for enterprise clients.
Antonio: Can we assume the SAML side is already configured?
Maria: Yes, the SAML metadata exchange is done. We just need to integrate.
Pedro: For the mobile flows we want PKCE, and we'd like the access tokens to be short-lived,
maybe 15 minutes max.
```

**Decision 3 — Metadata: temporal and speaker information.** For transcripts the useful metadata is the position in the meeting (early/mid/late discussion, optionally a timestamp if the transcript has one), the block's dominant speaker, and the detected topic. Typical filters will be by meeting date, by client, by phase.

> *(Figure in the original: `sesion-07-articulo-04-figura-02-segmentacion-transcripcion.jpg` — image not included in this repo.)*

The segmenter's skeleton:

```python
from dataclasses import dataclass

from sentence_transformers import SentenceTransformer

@dataclass
class Utterance:
    speaker: str
    text: str
    position: int  # ordinal index in the transcript

class TopicSegmentationChunker:
    """Segments meeting transcripts by detecting topic shifts.

    Uses sentence-level embeddings and a similarity threshold to find
    boundaries where consecutive utterances diverge semantically.
    """

    def __init__(
        self,
        embedding_model_name: str = "sentence-transformers/all-MiniLM-L6-v2",
        similarity_threshold: float = 0.55,
    ):
        self._model = SentenceTransformer(embedding_model_name)
        self._threshold = similarity_threshold

    def chunk(self, transcript: dict) -> list[Chunk]:
        utterances = self._parse_utterances(transcript["raw_text"])
        boundaries = self._detect_topic_boundaries(utterances)
        return self._build_chunks_from_boundaries(
            utterances, boundaries, transcript["meta"]
        )

    def _parse_utterances(self, raw_text: str) -> list[Utterance]:
        """Split raw transcript into Utterance objects by speaker turn."""
        ...  # implementation depends on the transcript format

    def _detect_topic_boundaries(
        self, utterances: list[Utterance]
    ) -> list[int]:
        """Return indices where a new topic block begins."""
        embeddings = self._model.encode(
            [u.text for u in utterances],
            normalize_embeddings=True,
        )

        boundaries = [0]  # first block always starts at 0
        for i in range(1, len(embeddings)):
            similarity = float(embeddings[i] @ embeddings[i - 1])
            if similarity < self._threshold:
                boundaries.append(i)
        return boundaries

    def _build_chunks_from_boundaries(
        self,
        utterances: list[Utterance],
        boundaries: list[int],
        meeting_meta: dict,
    ) -> list[Chunk]:
        """Group utterances between consecutive boundaries into chunks."""
        ...
```

On the parameters: `similarity_threshold=0.55` is a starting point, not a universal value. Too high and you produce too many blocks (any variation is read as a topic change); too low and you produce enormous blocks (only drastic changes are detected). The exact value depends on the embedding model and the transcript's style. In the live session we will calibrate it over the project's real transcripts.

An observation on the choice of embedding model here: for the internal segmentation we use `all-MiniLM-L6-v2` even though the rest of the system uses `text-embedding-3-small`. The reason is that segmentation needs to embed many individual sentences fast and cheap; a local 384-dimension model is perfect for this, whereas calling the API for each sentence would be unnecessarily expensive and slow. The embeddings for the final search index remain `text-embedding-3-small`. **Different pieces of the pipeline can legitimately use different models.**

## Metadata enrichment: the underestimated lever

I return to the Microsoft Azure finding I mentioned several times: enriching each chunk with structural metadata raises QA accuracy by 15 to 25 points, without changing anything else in the pipeline. It is among the findings with the best effort/impact ratio in all the recent RAG literature.

In our project, both chunkers are already doing this enrichment in three ways:

**First:** prepending contextual headers to the chunk's text (the bracketed block you see at the start of the JSON chunk and at the start of the transcript chunk). These headers go *inside* the text that will be embedded, so the vector incorporates that information into the semantic geometry.

**Second:** storing structured metadata outside the text, in the `Chunk`'s `metadata` dictionary. This metadata will not be embedded but will travel with the chunk to the vector database. In Session 08 we will see that pgvector allows filtering by this metadata, combining vector search with classic SQL filters. A query like "auth components for fintech from the last year" is resolved by doing vector search for the semantic part + an SQL filter for `client_sector = 'finance' AND year >= 2024`.

**Third:** using traceable IDs (`{budget_id}::{component_id}` or `{meeting_id}::{block_index}`). This is not retrieval-enriching metadata, but it is operationally critical: it lets you know which original document each chunk comes from, which you will need to cite sources to the user, audit results, or invalidate specific chunks when the parent document is updated.

One decision worth making explicit: **what information goes in the chunk's text and what goes in the metadata.** The practical rule is this. If the information changes the chunk's semantic meaning for a natural query ("authentication for fintech" → the sector matters for distinguishing that search from "authentication for e-commerce"), it goes in the text. If the information is discrete and will be used to filter results (`year`, `complexity`, `estimated_hours`), it goes in metadata.

Sometimes it goes in both places. The client's sector, for example, we can put both in the text header (so it weighs semantically) and in metadata (to filter). **This is not unjustified redundancy: each copy fulfils a different role.**

## Composition in the AI service

Once you have the two chunkers, the question remains of how the AI service decides which to apply to an incoming document. For this session the answer is simple because we only have two types: the `POST /embeddings/ingest` can receive a `document_type` field in the body and route internally:

> *(Figure in the original: `sesion-07-articulo-04-figura-03-arquitectura-servicio-ia.jpg` — image not included in this repo.)*

```python
class IngestRouter:
    """Routes documents to the appropriate chunker based on type."""

    def __init__(
        self,
        json_chunker: JSONStructuralChunker,
        transcript_chunker: TopicSegmentationChunker,
    ):
        self._chunkers = {
            "budget": json_chunker,
            "transcript": transcript_chunker,
        }

    def chunk(self, document: dict, document_type: str) -> list[Chunk]:
        if document_type not in self._chunkers:
            raise ValueError(f"Unknown document type: {document_type}")
        return self._chunkers[document_type].chunk(document)
```

An architectural decision worth mentioning even though it is not addressed until later sessions: the AI service could detect the document type automatically (look at whether it is structured JSON or plain text, read a `kind` field from the payload, inspect the file's URL). For this point in the programme the "explicit by payload" decision is simpler and more auditable, so we go with it. Automatic detection is article 3's agentic chunking — a legitimate option but with an extra cost that is not justified here.

The business backend, when it calls the ingestion endpoint, sends the document with its type:

```ruby
# Client-side example in Ruby/Rails. Any HTTP client works the same way.
class AIServiceClient
  def ingest_budget(budget_payload)
    HTTP.post(
      "#{ai_service_url}/embeddings/ingest",
      json: {
        document_type: "budget",
        budget: budget_payload,
      },
    )
  end

  def ingest_transcript(transcript_payload)
    HTTP.post(
      "#{ai_service_url}/embeddings/ingest",
      json: {
        document_type: "transcript",
        transcript: transcript_payload,
      },
    )
  end
end
```

That the business backend is Rails is accidental to the architecture. Any HTTP client fulfils the same function: the contract between the two layers is simple REST, independent of the stack.
