---
title: Professional chunking strategies
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 7
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 32 min
added: 2026-08-13
summary: >
  Twelve chunking strategies in four mental families — mechanical, structural,
  semantic, and advanced/contextual — with measured evidence that chunking moves
  retrieval quality as much as model choice, that sophistication does not
  guarantee improvement, and that recursive 400-512 tokens is hard to beat. Ends
  with seven honest selection criteria.
keywords: [chunking, recursive splitter, semantic chunking, contextual retrieval,
           parent-child, sentence-window, late chunking, agentic chunking,
           metadata enrichment, overlap]
---

# Professional chunking strategies

*Antonio Perez* · 🔴 32 min

The question we still have left is the one with the greatest impact on the final quality of your RAG system: **how do we split documents before generating the embeddings?**

It is not a rhetorical or cosmetic question. If you pass a whole budget to the embedding, the resulting vector is a blurry average mixing authentication with the inventory module and the hosting section, and it will not serve to retrieve anything specific. If you split it into pieces that are too small, you lose the context that makes each piece meaningful. If you split by character, you break words and sentences in the middle of an idea. If you split by separator, you assume the separator always means the same thing (and it almost never does). If you split by meaning — how do you compute meaning before having embeddings?

There are dozens of published strategies, several libraries implementing them, and a growing number of academic papers measuring which works best over which type of corpus. This article is the map of the territory. I cover twelve strategies grouped into four mental families, with criteria for choosing each and honest observations about when they do not work. There is no experimental comparative analysis here: that is exactly the bulk of the live session, where we will measure over your own data which one wins. The goal of this article is that you arrive at the live session with the map, not with the solution.

## Why chunking dominates quality

Before getting into the catalogue, it is worth calibrating how much this decision matters. It is a question most tutorials dodge or answer with vagueness.

Three recent data points to fix the order of magnitude:

- A **Vectara study presented at NAACL 2025** evaluated 25 different chunking configurations over 48 embedding models on standard retrieval tasks. Its most quotable finding: **the variance induced by changing the chunking strategy can be as large as the variance induced by changing model.** In other words, going from naive chunking to well-tuned chunking can give you as much as going from a mediocre model to an excellent one.
- **Chroma's research (2025)** measured five strategies over an internal benchmark. `LLMSemanticChunker` and `ClusterSemanticChunker` reached recall of 0.919 and 0.913 respectively. LangChain's `RecursiveCharacterTextSplitter` well tuned (400 tokens) reached 0.88-0.89. The difference between the best and the worst in the study was 9 percentage points of recall.
- A **Vecta benchmark from February 2026** over 50 academic papers placed the recursive splitter at 512 tokens in first place with 69% accuracy. Semantic chunking, supposedly more sophisticated, came fourth with 54%. **Sophistication does not guarantee better performance**; what guarantees it is the match between the strategy and the type of corpus.

The operational conclusion you will hear repeated several times in this article: **measure over your data before committing architecture.** Generalist benchmarks are indicative, not oracular. What wins in academic papers does not necessarily win on technical software budgets or on requirements-gathering transcripts.

## Four mental families

Before enumerating twelve strategies, it is worth organising the space mentally. The catalogue falls into four groups according to how they decide where to split the text.

**Family 1: mechanical.** They split the text without understanding what it says. They apply rules about the *form* of the text (size, separators, sentences). They are the simplest, the fastest, and surprisingly competitive in many benchmarks.

**Family 2: structural.** They exploit the document's format (HTML, Markdown, JSON, sections of a PDF). The structure already encodes the author's intent, so splitting while respecting it preserves context without needing semantic analysis.

**Family 3: semantic.** They compute where the meaning of the text changes (using embeddings or an LLM) and split there. More expensive, in theory better on documents with no clear structure. In practice they do not always justify the extra cost.

**Family 4: advanced and contextual.** These are not alternatives to the previous ones, they are *complements*: techniques that improve the result of any base strategy by enriching the chunk with additional context before embedding it. They are the most recent (most are from 2024-2025) and where the state of the art currently sits.

Let's look at each family.

> *(Figure in the original: `sesion-07-articulo-03-figura-01-cuatro-familias.jpg` — image not included in this repo.)*

## Family 1 — Mechanical

### Fixed-size

The most basic strategy: split the text into blocks of N tokens or N characters, with an optional overlap parameter between consecutive blocks. It does not look at content, does not respect sentences, does not understand paragraphs.

```python
def fixed_size_chunks(text: str, chunk_size: int, overlap: int) -> list[str]:
    """Split text into fixed-size character chunks with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

**When it works:** very homogeneous corpora where the internal structure of the text does not matter (logs, event streams, flat transcripts with no formatting). Useful as a baseline to compare more sophisticated strategies against.

**When it does not work:** any corpus with internal structure (code, formal documents, conversations). Breaking mid-word or mid-sentence degrades embedding quality measurably.

Typical overlap sits between 10% and 20% of the chunk size. It serves so that an idea falling right on a boundary is not lost: it appears (partially) in both adjacent chunks.

### Recursive character text splitter

The strategy most used in real production, and according to recent benchmarks the one offering the best balance. The idea: instead of splitting by character, you define a hierarchy of separators ordered by preference (by default `["\n\n", "\n", " ", ""]`) and try to split by the strongest separator that produces chunks within the target size.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=80,
    separators=["\n\n", "\n", ". ", " ", ""],
)

chunks = splitter.split_text(document_text)
```

The algorithm first tries to split by paragraphs (`\n\n`). If the paragraphs are still too big, it drops to lines (`\n`). If the lines are still too big, it drops to sentences (`. `), and so on down to characters as a last resort.

**When it works:** practically any natural-prose text. It is the reasonable default, the one you will try first in almost any project.

**When it does not work:** documents with formal hierarchical structure (JSON, code, forms) where generic separators do not capture the real hierarchy. For those cases there are specific structural strategies.

A configuration that performs well according to multiple benchmarks: chunks of 400-512 tokens, overlap of 10-20%. Before inventing anything, try this default against your corpus. **It is surprisingly hard to beat.**

### Sentence-window retrieval

A variant with an elegant idea: index individual sentences (very small chunks) but at retrieval time, return an expanded window around each retrieved sentence. The embeddings compare small, precise units; the LLM receives sufficient context to answer.

```python
# Conceptual sketch
def chunk_to_sentences_with_window(text: str, window: int) -> list[dict]:
    """Index sentences; remember their neighbors for retrieval-time expansion."""
    sentences = split_into_sentences(text)
    chunks = []
    for i, sentence in enumerate(sentences):
        chunks.append({
            "embedding_text": sentence,  # what we embed and index
            "retrieval_text": " ".join(
                sentences[max(0, i - window):i + window + 1]
            ),  # what we return when this sentence matches
            "metadata": {"sentence_idx": i},
        })
    return chunks
```

**When it works:** documents where the concrete information is in specific sentences but the answer needs neighbouring context to make sense (technical manuals, papers, contracts). The retriever finds the "exact point" and the generator receives the "relevant surroundings".

**When it does not work:** documents where information is naturally distributed in blocks (executive summaries, long thematic sections). Here indexing individual sentences fragments more than it helps.

This strategy is one of the oldest patterns still competitive in 2026. It is implemented with LlamaIndex (`SentenceWindowNodeParser`) or by hand relatively easily.

### Sliding window with variable overlap

A variant of fixed-size where the "step" between chunks is not a fixed `chunk_size - overlap` but an independent parameter. It lets you control density: small steps produce many redundant chunks (good coverage, larger index), large steps produce few sparse chunks (smaller index, risk of losing things in the seams).

Useful when working with continuous text without natural separators (raw transcripts without diarisation, temporal event sequences). In practice, sentence-window is usually a better option for cases where sliding window looks attractive.

> *(Figure in the original: `sesion-07-articulo-03-figura-02-tres-estrategias-mismo-texto.jpg` — image not included in this repo.)*

## Family 2 — Structural

### Document-based: Markdown, HTML, JSON

The original document's structure already encodes the author's decisions about where one idea begins and another ends. If the document has Markdown headers, chunking decisions must respect headers. If it has HTML tags, they must respect tags. If it is JSON, they must respect the key hierarchy.

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

markdown_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ],
    return_each_line=False,
)

chunks = markdown_splitter.split_text(markdown_document)
# Each chunk knows under which h1, h2, h3 it lives — that's the "context"
```

For HTML there is `HTMLHeaderTextSplitter` with the same idea applied to `<h1>` to `<h6>` tags. **For JSON, there is no generic splitter that works well**, because the "meaningful structure" depends on the domain: in a budget a component is a logical unit, in an API spec an endpoint is a logical unit, in a dataset a record is a logical unit. The JSON chunker is usually custom — exactly what we are going to do in article 4 for budgets.

**When it works:** documents with explicit, reliable structure. Research from Microsoft Azure Architecture Center (2025) showed that **adding each chunk's header as enriched metadata (changing nothing else) can raise QA accuracy by 15 to 25 points.** Structure is a goldmine of context if you respect it.

**When it does not work:** documents with no structure (flat transcripts, low-quality OCR text, social media content).

### Hierarchical / parent-child chunking

The idea: index small chunks so retrieval is precise, but associate each small chunk with its "parent" chunk (larger), so that on retrieval you can pass the LLM the small chunk that matched *and* the parent's context. The result is a multi-level index.

Conceptually:

```
Document
├── Section 1 (large chunk)
│   ├── Paragraph 1.1 (small chunk)
│   ├── Paragraph 1.2 (small chunk)
│   └── Paragraph 1.3 (small chunk)
└── Section 2 (large chunk)
    ├── Paragraph 2.1 (small chunk)
    └── Paragraph 2.2 (small chunk)
```

You index the paragraphs. When one matches, you return (optionally) the complete section along with the paragraph. LangChain implements this idea with `ParentDocumentRetriever`; LlamaIndex with `HierarchicalNodeParser`.

**When it works:** long documents where a concrete question has its answer in a specific fragment but the question is only understood with broader context. Manuals, technical books, legal documents.

**When it does not work:** corpora of already-small, self-contained chunks (FAQs, short support tickets). Here the hierarchy adds complexity without gain.

Important: parent-child is **a retrieval architecture, not just a chunking strategy**. It implies decisions about what is indexed (the children) and what is returned (the parents or both). It is a major architectural decision.

## Family 3 — Semantic

### Semantic chunking

Instead of cutting by size or by separator, compute embeddings of consecutive sentences and cut where similarity falls below a threshold. The intuition: where the meaning changes, it is convenient to cut.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai.embeddings import OpenAIEmbeddings

semantic_splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-small"),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)

chunks = semantic_splitter.split_text(document_text)
```

The `breakpoint_threshold_type` parameter controls how the cut point is decided: by percentile (cut where the similarity difference is in the top X%), by standard deviation, or by absolute threshold.

**When it works:** multi-topic documents with no explicit structure: research papers with implicit sections, long blog posts, essays.

**When it does not work:** short, focused documents (FAQs, single-topic articles). It also does not work well on text where consecutive sentences have high similarity by design (repetitive jargon, legal templates).

The real cost many tutorials hide: semantic chunking requires embedding **every sentence** of the document during ingestion, which multiplies cost and latency relative to recursive. For 10,000 documents with 100 sentences each, that is a million additional calls to the embeddings API. And recent benchmarks (including NAACL 2025) show the gains relative to well-tuned recursive are often marginal.

### Cluster semantic chunking

An interesting variant: instead of cutting sequentially, group similar sentences even when they are not consecutive. The motivation: in some documents ideas reappear at different moments, and grouping them into the same chunk improves retrieval of that idea.

```python
# Conceptual sketch; production implementations use HDBSCAN or similar.
def cluster_chunks(sentences: list[str], embedder, n_clusters: int):
    embeddings = [embedder(s) for s in sentences]
    cluster_assignments = cluster_algorithm(embeddings, n_clusters)
    chunks = [
        " ".join(s for s, c in zip(sentences, cluster_assignments) if c == k)
        for k in range(n_clusters)
    ]
    return chunks
```

**When it works:** long speeches where the same topic reappears, round-table transcripts with several speakers returning to their points, books with recurring motifs.

**When it does not work:** most technical corpora where ideas are organised linearly. It also **breaks traceability**: a clustered chunk has no "place" in the original document, which complicates citing the source to the user.

### LLM-based / propositional chunking

The most expensive strategy and, in some benchmarks, the one offering the best results. The idea: give the document to an LLM and ask it to extract self-contained propositions — statements that make sense in isolation, without depending on document context.

```python
PROPOSITION_PROMPT = """
Decompose the following text into the smallest set of self-contained propositions.
Each proposition should:
- Express a single atomic fact or claim
- Be understandable without reading the surrounding text
- Resolve all pronouns and references to explicit entities

Output as a JSON list of strings.

Text: {text}
"""

def llm_based_chunks(text: str, client) -> list[str]:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": PROPOSITION_PROMPT.format(text=text)}],
        response_format={"type": "json_object"},
    )
    return parse_json(response.choices[0].message.content)["propositions"]
```

**When it works:** high-value corpora where retrieval quality justifies the extra cost of calling an LLM for each document during ingestion. Critical documentation, support knowledge bases, legal content.

**When it does not work:** large corpora where ingestion cost becomes prohibitive. For a million documents, we are talking hundreds or thousands of dollars in the chunking phase alone. The ROI has to be evaluated explicitly.

In Chroma's benchmarks (2025), `LLMSemanticChunker` reached 0.919 recall, the study's maximum. It is the prototypical case of "more expensive but effective when you can afford it".

## Family 4 — Advanced and contextual

This family is different from the previous ones. They are not alternatives to recursive or semantic; they are **complements** that improve the result of any base strategy.

### Late chunking

A recent concept popularised by Jina AI in late 2024. The idea changes the traditional order of operations. In conventional chunking: split the text first, then embed each chunk in isolation. In late chunking: embed the whole document (or large sections) first, letting the embedding model see all the context, and then "extract" embeddings for each chunk from the global representation.

It requires an embedding model with a long context window (at least 8K-32K tokens) and support for token-level embeddings. Models like `jina-embeddings-v3` and some from OpenAI allow it.

**When it works:** documents where the meaning of each part depends heavily on global context. A paragraph about "the model" in the middle of an ML paper has a very different meaning depending on whether "the model" refers to a transformer network or a Markov model, and the global context resolves it before chunking.

**When it does not work:** when your model does not support large context windows or does not expose token-level embeddings. Most lightweight open-source models (including `all-MiniLM-L6-v2`) do not work here.

It is an emerging technique. If your pipeline already works reasonably, it is probably not worth introducing late chunking right now. Keeping the concept in mind, yes — for when long-context models get cheaper.

### Agentic chunking

An agent with tool calls dynamically decides how to split a document. The agent reads, evaluates the content's complexity, and applies a different strategy for each section (recursive here, structural there, semantic for this other part). The idea is to eliminate the human decision about which strategy to apply, delegating it to the agent.

```python
# Conceptual sketch
CHUNKING_AGENT_PROMPT = """
You are a chunking agent. Given a document, decide for each section
which chunking strategy to apply: recursive, structural, semantic, or sentence-window.
Output a sequence of (section_text, strategy) pairs.
"""
```

**When it works:** heterogeneous corpora where different documents require different treatments and you do not want to maintain N different pipelines. An enterprise system ingesting emails, contracts, papers and code all at once.

**When it does not work:** homogeneous corpora, where it is simpler and more predictable to apply the same strategy to everything. Also: any system with hard ingestion-cost constraints — agents are the most expensive of all.

### Query-dependent chunking

A recent idea from AI21 (2026): instead of static chunking at ingestion, several resolutions of the same document are indexed simultaneously (chunks of 100, 200, 500, 1000 tokens) and at query time the system chooses which resolution to use according to the question. Concrete questions are answered better with small chunks; open questions need large chunks.

**When it works:** corpora where the same document is queried in very different ways. A technical manual where some users ask "what is error code 42?" (a small chunk suffices) and others ask "how does the logging module work?" (you need large chunks).

**When it does not work:** most cases where the query pattern is predictable. And it multiplies storage cost by N resolutions.

It is an emerging pattern; keep an eye on it, but do not invest in it right now unless your case justifies it.

### Contextual Retrieval (Anthropic)

The technique that is probably worth implementing immediately if your RAG pipeline already works but is not quite dialled in. Published by Anthropic in September 2024 and matured through 2025. The idea: before embedding each chunk, enrich it with a short paragraph of LLM-generated context that situates that chunk within the complete document.

Anthropic's canonical prompt:

```
<document>
{whole_document}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{chunk_content}
</chunk>

Please give a short succinct context to situate this chunk within
the overall document for the purposes of improving search retrieval
of the chunk. Answer only with the succinct context and nothing else.
```

> *(Figure in the original: `sesion-07-articulo-03-figura-03-contextual-retrieval.jpg` — image not included in this repo.)*

The generated context is prepended to the chunk before embedding it and before indexing it in BM25:

```
[Generated context: This chunk discusses Q3 2024 revenue figures for the European
market, mentioned in section 4.2 of the annual report.]

[Original chunk: Revenue grew by 3% over the previous quarter...]
```

The numbers Anthropic reports: **35% reduction in retrieval failures** with contextual embeddings alone, **49%** combining it with contextual BM25, up to **67%** adding reranking. Independent benchmarks confirm the improvements (the exact magnitude depends on the corpus, but the direction is robust).

**When it works:** practically any corpus with long documents where individual chunks can lose context when isolated. That is, almost all real cases.

**When it does not work:** corpora of naturally self-contained small chunks (FAQs, short tickets) and any system where ingestion cost is the hard constraint.

On cost: contextualising each chunk requires an LLM call per chunk during ingestion. Anthropic recommends using prompt caching to reduce the cost of passing the whole document each time, which brings the estimated cost down to around $1 per million contextualised tokens. For a small project it is negligible; for large ingestions it has to be measured.

We will see it implemented in the live session and measure whether it pays off for our budgets.

## How to choose: honest criteria

Twelve strategies is a catalogue, not a recipe. Some observations to make the catalogue useful in practice.

**First:** start with `RecursiveCharacterTextSplitter` at 400-512 tokens and 10-20% overlap. Recent benchmarks repeatedly place it among the best options, and it is the cheapest to implement and operate. Only change if you have *measured* evidence that it is not enough.

**Second:** if your corpus has explicit structure (Markdown, HTML, JSON, Word sections), use it. Document-based chunking + enriched metadata is the highest-ROI lever known. Microsoft Azure documented jumps of 15-25 accuracy points from this alone. Do not waste it.

**Third:** if your corpus has chunks that lose meaning in isolation, consider Contextual Retrieval. It is the most mature of the "advanced" techniques and the one that most consistently improves results over any base.

**Fourth:** semantic chunking, LLM-based chunking and related techniques are legitimate but expensive tools. Justify them with data: measure over your corpus that they contribute, do not assume they will because a blog post praises them.

**Fifth:** parent-child / hierarchical chunking is an architecture pattern, not just a chunking one. It implies decisions about what is indexed and what is returned. Do not introduce it if your pipeline does not yet work reasonably with a flat strategy.

**Sixth:** late chunking, agentic chunking and query-dependent chunking are interesting emerging techniques. Keep an eye on them, but do not introduce them into production unless you have a concrete use case justifying them. Novelty is not by itself an advantage.

**Seventh and most important:** the best chunking depends on the document type. A heterogeneous corpus almost always benefits from different strategies for different document types within the same pipeline. That is exactly the project's situation, where we have structured JSON budgets and plain-text meeting transcripts. Treating both with the same strategy is leaving value on the table.
