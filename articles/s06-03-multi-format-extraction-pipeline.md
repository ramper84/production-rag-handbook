---
title: Multi-format extraction pipeline
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
series_part: 3
scope: evergreen
source: user-supplied article (follows "Article 2 — the data catalog")
added: 2026-08-11
summary: >
  Argues for a modular ingestion subsystem — loaders, parsers, normalizers —
  converging on a canonical Document (content + metadata), rather than delegating
  everything to unstructured.partition(). Covers per-format parsing strategy for
  JSON/TXT/XLSX/DOCX/PDF, three-source metadata propagation, and why applying
  hi_res PDF parsing indiscriminately is expensive and unnecessary.
keywords: [ingestion, RAG, chunking, canonical document, metadata propagation,
           parsers, unstructured, pdf, docx, xlsx, provenance, citations]
---

# Multi-format extraction pipeline

*Antonio Perez*

With the catalog closed, we have a map of which sources will enter the system. Now comes turning each source's physical content into processable text. For Project 2 that means facing five format families: JSON (historical budgets), TXT (meeting transcripts), XLSX (rate cards and data masters), DOCX (proposal templates) and PDF (signed contracts and laid-out proposals). Each brings its own technical curse.

There is a recognisable temptation here: install `unstructured`, call `partition()`, and declare the problem solved. It is the right answer in the short term and the wrong one in the medium term, for two reasons. First, `unstructured` in its maximum-precision mode is slow and expensive; using it indiscriminately across a corpus that mixes trivial JSON with scanned PDFs squanders resources on the formats that don't need it. Second, and more importantly, delegating everything to a library without thinking about architecture is exactly how you build the pipelines that nobody understands two months later. This article proposes the opposite pattern: a modular architecture where each format is handled with the right tool and everything converges on a common contract.

## The common contract: the canonical Document

Before choosing a single parser, it's worth settling the most important question: what does the `ingest/` subsystem have to produce for the rest of the AI service? If that output isn't defined, any design of the extractors is premature.

The answer is a canonical object that appears throughout the RAG literature under slightly different names (`Document`, `Chunk`, `Passage`) and always has two essential fields: the textual content and the metadata. In Project 2's case:

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field

class DocumentMetadata(BaseModel):
    """Metadata propagated with every document through the pipeline.

    The first three fields come from the data catalog and are mandatory
    for every document, regardless of source format. The rest are
    populated by the parser when the format allows it.
    """
    source_name: str  # matches an entry in data_catalog.yaml
    source_location: str  # original physical path or URL
    ingested_at: datetime

    document_id: str  # stable identifier within the source
    document_title: Optional[str] = None
    document_created_at: Optional[datetime] = None
    document_author: Optional[str] = None

    page_number: Optional[int] = None  # for paginated formats
    section_title: Optional[str] = None  # for structured formats
    contains_pii: bool = False
    extra: dict = Field(default_factory=dict)

class Document(BaseModel):
    """The canonical output of the ingest subsystem.

    Every parser, regardless of input format, must produce instances
    of this class. Downstream chunking, embedding, and retrieval
    operate exclusively on Document objects.
    """
    content: str
    metadata: DocumentMetadata
```

This model is deliberately flat. It has two virtues worth highlighting. First, **homogeneity of the downstream contract**: the chunking module does not know (and does not need to know) whether the `Document` came from a scanned PDF or a structured JSON. It processes `content` and propagates `metadata`, nothing more. Second, **traceability by construction**: every document knows which catalog source it came from (`source_name`), where it originally lived (`source_location`) and, when the format allows, which specific page or section. That information is what reaches the final user-facing answer in the form of a citation.

The `extra` field as an open dictionary is a deliberate escape valve: it lets a specific parser (the DOCX one, for example) propagate metadata that doesn't fit the canonical schema (template fields, comments, revision authorship). That metadata is preserved but not demanded of every parser.

## Modular architecture of the ingest/ subsystem

On top of the canonical contract sits a three-layer structure separating responsibilities by problem type. We'll call them loaders, parsers and normalizers, and it's worth being clear about the boundary between them, because mixing them is the most common error in ingestion pipelines.

```
servicio_ia/
├── ingest/
│   ├── loaders/          # physical access to sources
│   │   ├── filesystem.py
│   │   ├── drive.py
│   │   └── http.py
│   ├── parsers/          # per-format extraction
│   │   ├── json_parser.py
│   │   ├── pdf_parser.py
│   │   ├── docx_parser.py
│   │   ├── xlsx_parser.py
│   │   └── txt_parser.py
│   ├── normalizers/      # homogenisation into Document
│   │   └── canonical.py
│   ├── catalog.py        # loader for data_catalog.yaml (Article 2)
│   └── orchestrator.py   # glues it together and produces Document[]
└── ...
```

**Loaders** answer "how do I reach the file". They know about filesystem paths, HTTP URLs, Drive authentication, S3 keys. They don't know what is inside the file; they just hand it over as bytes or a stream. This separation matters because the same format (PDF, say) can live in Drive, in an S3 bucket or on local disk, and we don't want to triplicate the PDF parser once per place it lives.

**Parsers** answer "what is inside the file". They receive bytes or a local path, choose the right library for the format, and produce an intermediate representation that is already text + document metadata. But that intermediate representation is parser-specific (a pandas dataframe for Excel, a list of elements for a PDF parsed with `unstructured`) — it is not the canonical `Document` yet.

**Normalizers** answer "how do I convert my parser's output into the canonical contract". This is the thin layer that takes the intermediate representation and turns it into `Document` instances, propagating catalog metadata and enriching it with whatever the parser contributes.

The reason for three layers rather than two (a parser returning `Document` directly) is **testability**. Parsers end up being complex logic dependent on heavy external libraries; testing them against the canonical contract means having to fill in catalog metadata in every test. Separating normalisation lets you test parsers against their own intermediate representation (easy to mock) and test normalizers against the canonical contract (also easy). Each test stays focused.

## Parsing strategies by format

Each format calls for a different technical strategy. The choice is not aesthetic: it has consequences for latency, cost and the quality of the information extracted. Here are Project 2's five families.

**JSON** is the "easy" format because it already has structure. For Project 2's historical budgets, there is no text to extract: what has to be decided is which textual representation of the JSON will enter the RAG. Dumping the whole JSON as a string with `json.dumps()` produces noisy embeddings (it mixes technical keys with semantic values). Dumping only the values loses the context of what each one represents. The strategy that works best in my experience is rendering to structured markdown: converting each budget into a readable block with sections, where important keys are promoted to headings and values are written as prose. That is the work of a parser that knows the JSON's schema, not of a generic parser.

**TXT** looks even easier but hides a trap. Project 2's transcripts are not homogeneous plain text: the ones from the automated service from 2024 onwards have `[hh:mm:ss] Speaker: ...`, while earlier ones have heterogeneous formats where sometimes the speaker is omitted and sometimes long turns are grouped into a single block. Treating them all as a bag of text loses an extremely valuable signal: who said what. The correct strategy is a transcript parser that detects the format and produces a representation with identified turns, where each turn later becomes a fragment with `speaker` and `timestamp` metadata. What enters the RAG is not the raw transcript but its enriched turns.

**XLSX** is the most treacherous format because it looks tabular but rarely truly is. An Excel sheet can have merged cells, formulas evaluated at runtime, multiple tables in the same sheet, floating comments, hidden sheets, conditional formatting that encodes information. For Project 2's rate card (excluded from the corpus, but illustrative) the right decision would be to extract the main table with `openpyxl` or `pandas.read_excel()` and convert it to a markdown table, deliberately losing the rest. If the Excel has relevant content in non-tabular cells, it has to be treated as a special case. The practical rule: if the Excel is a pure table, it comes out as a markdown table; if it has complex structure, it either shouldn't be in the corpus or it requires manual conversion first.

**DOCX** is surprisingly friendly. `python-docx` allows walking paragraphs, tables and headings through a clean API, and modern DOCX files have explicit semantic structure (paragraph styles, heading levels) that can be exploited to preserve the document's hierarchy in the output. Project 2's proposal templates are a classic case: a well-built DOCX parser can extract sections by heading (Scope, Deliverables, Timeline) and emit one `Document` per section with the heading as `section_title`. That gives the RAG the ability to retrieve the specific section rather than the entire proposal.

**PDF** is hell, for well-known reasons: PDF is a presentation format, not a content format. The semantic structure is implicit in positions, fonts and sizes, not in an explicit hierarchy. There are three technical options:

- `pypdf` or `pdfplumber` for plain text: fast and cheap, loses tables, columns and structure entirely.
- `pymupdf` (also known as `fitz`): better layout handling, supports image extraction and bounding boxes. A good option when the PDF is clean digital text.
- `unstructured` with strategy `hi_res`: uses computer-vision models to detect tables, headers and sections. It is the right option when the PDF has relevant tables or when there are scans requiring OCR. It is also, by a wide margin, the slowest and most expensive.

The rule I apply for Project 2 (signed contracts and final proposals in PDF): `pypdf` by default for digitally generated documents, `unstructured` with `hi_res` only when the PDF is detected to contain tables or to be scanned. That decision is taken once per source in the catalog, not per document.

## The universal parser: unstructured as a Swiss army knife

There is an alternative to the patchwork above: use `unstructured` for everything. The library exposes a `partition()` function that detects the format and applies the appropriate extractor, returning a list of heterogeneous `Element` objects (`Title`, `NarrativeText`, `Table`, `ListItem`, etc.) with location metadata.

It has real advantages: it unifies the interface, supports more than 20 formats, and has surprisingly good structure-detection models. It also has costs worth looking at squarely. The first is weight: installing `unstructured[all-docs]` puts several hundred megabytes of dependencies into the Docker image (Tesseract, detection models, PyTorch). The second is latency and cost: strategy `hi_res` is an order of magnitude slower than a native parser for simple PDFs, and at large volumes this shows up in indexing time and the compute bill. The third, subtler one, is opacity: when something goes wrong with extraction (a table that isn't detected, a heading categorised as narrative), debugging is hard because much of the work is done by a neural model that doesn't explain its decisions.

My operational recommendation for Project 2: native parsers for the formats whose structure is predictable (JSON, TXT, simple XLSX, DOCX), with `unstructured` reserved for PDF when it's needed (tables, scans) and as an optional fallback for exotic formats that show up later. The pattern is to exploit the library's power without turning it into a single point of dependency that obscures the whole pipeline.

## Metadata propagation through the pipeline

The entire decision to have a canonical `Document` is justified at this point: the metadata travelling with each document is what allows the RAG to produce citations and stakeholders to verify them. The source of this metadata is threefold.

**Catalog metadata.** These are known before touching the specific document: the source's logical name, business owner, PII sensitivity, access restrictions, inclusion decision. They come from the `data_catalog.yaml` we closed in the previous article, and they apply uniformly to every document from that source.

**Parser metadata.** These are known after processing the specific document: title extracted from a header, author read from the file's metadata, creation date, page number, current section. Each parser propagates whichever ones the format allows.

**Pipeline metadata.** These are known at processing time: `ingested_at` (the ingestion run's timestamp), parser version, configuration used (chosen strategy, OCR model). These are useful for debugging and reproducibility.

> *(Figure in the original: `sesion_06_article_3_visual_2_metadata_propagation.jpg` — image not included in this repo.)*

The orchestrator is what combines the three sources. A minimal orchestrator implementation, assuming the `Document` and the modular architecture above:

```python
from datetime import datetime, timezone
from pathlib import Path
from typing import Protocol

class Parser(Protocol):
    """Contract that every format-specific parser must satisfy."""
    supported_formats: set[str]

    def parse(self, content: bytes, source_hint: str) -> list[Document]:
        """Parse raw bytes and return canonical Document instances.

        The parser populates content and the parser-known subset of
        metadata. The orchestrator enriches the result with catalog
        metadata and pipeline metadata before returning.
        """
        ...

def ingest_source(
    source: CatalogSource,
    loader: Loader,
    parsers: dict[str, Parser],
) -> list[Document]:
    """Run the full ingest pipeline for a single catalog source."""
    if source.decision != IngestionDecision.INCLUDE:
        return []

    parser = parsers.get(source.format)
    if parser is None:
        raise ValueError(f"No parser registered for format: {source.format}")

    raw_files = loader.list_files(source.location)
    documents: list[Document] = []
    pipeline_run_ts = datetime.now(timezone.utc)

    for file_ref in raw_files:
        content = loader.read(file_ref)
        parsed_docs = parser.parse(content, source_hint=file_ref.path)

        for doc in parsed_docs:
            # Enrich with catalog metadata (overrides parser defaults
            # where applicable).
            doc.metadata.source_name = source.name
            doc.metadata.source_location = source.location
            doc.metadata.ingested_at = pipeline_run_ts
            doc.metadata.contains_pii = source.sensitivity.contains_pii
            documents.append(doc)

    return documents
```

Three design details deserve comment. First, `Parser` is defined as a `Protocol` (structural typing) rather than an abstract class. This allows any object with a `parse()` method of the right signature to be a valid parser, without forcing inheritance. It is more flexible and tests better. Second, the orchestrator respects the catalog's decision: a source with `decision: exclude` or `decision: review` is not processed, regardless of the file being there. Article 2's architectural discipline is enforced automatically. Third, catalog metadata is applied **after** the parser; this means that if a parser were to falsify `source_name` (deliberately or through a bug), the orchestrator overwrites it with the catalog's canonical value. Defence in depth.

## Honest trade-offs

**Native parsers vs universal unstructured.** Some teams adopt `unstructured` as their single interface and others maintain per-format parsers. The choice is not tribal but contextual. For a corpus with five or six predictable formats (Project 2's case), native parsers are faster, cheaper and easier to debug. For a corpus with dozens of heterogeneous formats where the team is not going to invest in each one, `unstructured` avoids reinventing twenty wheels. The rule I apply is: count the catalog's formats, and if there are fewer than five or six and they're all well understood, go with native parsers; if there are more, or they're unpredictable, go with `unstructured` except where cost/latency force a specific exception.

**Strategy hi_res vs fast for PDF.** The temptation to use `hi_res` across the whole corpus for safety is understandable and dangerous. Over a hundred PDFs, `hi_res` can take twenty times longer than `fast` and cost twenty times more in compute, while over the ninety PDFs that are clean digital text it adds nothing compared with `fast`. The practical rule: classify the PDFs in the catalog (clean digital, digital with tables, scanned) and apply the corresponding strategy. The classification is a one-off manual job; the saving is continuous.

**Acceptable loss of structural information.** There is information in the original formats that will not survive the ingestion pipeline, and it's worth deciding consciously what you lose. Images embedded in DOCX, comments in Word revisions, margin annotations in PDF, conditional formatting in Excel. For a project-estimation RAG like Project 2's, all of that is disposable noise. For a legal contract-review RAG, annotations would be critical information. The decision depends on the use case; what is not acceptable is losing structural information through carelessness rather than by design.

## Bridge to the next stage

At this point we have an `ingest/` subsystem capable of reading any catalog source, processing it with the right parser, and producing a list of canonical `Document` objects with propagated metadata. It is a big advance on the previous state, but one indispensable piece is still missing before anything can be vectorised.

The text the parsers produce is extracted but not clean. The JSON budgets have dates in three different formats (`2024-03-15`, `15/03/2024`, `Mar 15 2024`), currencies appear as `EUR`, `eur`, `€`, client names have spelling variants, there are duplicate records with divergent values in some field, and there are fields the parser filled with `None` when the original document had data but the extractor didn't find it. If we pass this straight to chunking and embedding, the noise propagates into the vector space and retrieval degrades in subtle ways that the team will take months to diagnose.
