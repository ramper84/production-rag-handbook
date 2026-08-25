---
title: "Embedding model selection: production trade-offs"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 7
series_part: 2
scope: time-sensitive
source: user-supplied article (landscape stated as of May 2026)
reading_time: 28 min
added: 2026-08-13
summary: >
  The 2026 model landscape, why MTEB is a coarse filter rather than an oracle,
  Matryoshka Representation Learning and its renormalisation trap, and five
  decision axes (dimensionality, language, domain, hosting/cost, licence) — with
  a worked decision for text-embedding-3-small and an explicit account of when it
  would be the wrong call.
keywords: [embedding models, MTEB, matryoshka, dimensionality, multilingual,
           self-hosted, licensing, text-embedding-3-small, bge-m3, cost]
---

# Embedding model selection: production trade-offs

*Antonio Perez* · 🔴 28 min

In the previous article we nailed down the theory: an embedding is a vector, the geometry induced by training is what makes nearby vectors correspond to nearby texts, and the similarity metric is practically irrelevant as long as you use the one the model recommends. Good.

Now comes the decision that really moves the dial on your system's quality: which model you choose to vectorise your data. And there is no universal answer. For the project's AI service we have at least five reasonable candidates in front of us, with prices ranging from zero to several dollars per million tokens, dimensionalities between 384 and 3072, licences from MIT to closed proprietary, and benchmarks that will tell you different things depending on who you ask.

This article exists so that you arrive at the live session with your decision made and argued, not copied from a blog. I cover the 2026 landscape, why MTEB is not the final word, what Matryoshka Representation Learning is, the five real decision criteria and how they land on the project, and I finish with the model we are going to use (`text-embedding-3-small`) and why — including when it would not be the right choice.

## The landscape: who is out there in 2026

The embedding-model space has moved a lot in the last six months. The picture you paint today is different from a quarter ago. I am going to focus on the six names any Spanish-speaking AI engineer should know as of May 2026, separated into two blocks: commercial API and self-hosted open source.

> *(Figure in the original: `sesion-07-articulo-02-figura-01-comparativa-modelos.jpg` — image not included in this repo.)*

**Commercial (API access).**

- **OpenAI `text-embedding-3-small`.** 1536 dimensions by default, Matryoshka support down to 256, $0.02 per million tokens. The pragmatic workhorse: cheap, fast, decently multilingual, trivial integration with the SDK you have already been using since Session 01. MTEB around 62.
- **OpenAI `text-embedding-3-large`.** 3072 dimensions by default, Matryoshka down to 256, $0.13 per million tokens. Better quality than the small but 6.5 times more expensive. Justified when the domain is complex and the volume is not enormous.
- **Cohere `embed-v3`.** Especially strong multilingually (100+ languages with balanced quality), reranking built into its API, $0.10 per million. If your corpus mixes several non-Anglo languages, Cohere usually beats OpenAI on cross-lingual retrieval benchmarks.
- **Voyage AI `voyage-3-large`.** Optimised specifically for retrieval, not for classification or STS. Leads recent retrieval-focused MTEB benchmarks. Apache 2.0 on some of its lite models. Its pricing is similar to Cohere's.

**Open source (self-hosted).**

- **`BAAI/bge-m3`.** Robust multilingual, supports 100+ languages, 1024 dimensions, MIT. Three modes in a single model: dense, sparse and multi-vector. It is the serious option when data privacy or cost at scale are hard constraints. Requires a GPU for decent production latencies.
- **`sentence-transformers/all-MiniLM-L6-v2`.** 384 dimensions, Apache 2.0, mainly trained on English. The "small fast cheap" of the open-source ecosystem: runs reasonably on CPU, occupies a few hundred MB, and is excellent for a prototype. MTEB around 56 — clearly below modern models, but not insignificant.

There are other relevant names (Google Gemini Embedding 2, Jina v5, Qwen3-Embedding, Nomic) that I will mention when they contribute something specific. The list above is not exhaustive; it is the minimum set of names you should be familiar with.

A calibration warning before continuing: the prices, dimensionalities and scores I quote are valid at the moment I write this article (May 2026). Commercial providers cut prices every few months, open-source models improve every release. Verify before committing to an important architectural decision.

## MTEB is not what it seems

If you have searched "best embedding model" in any online ranking, you have probably ended up at Hugging Face's MTEB Leaderboard. MTEB is the Massive Text Embeddings Benchmark: a set of standardised tasks (retrieval, classification, clustering, reranking, STS) over public datasets in several languages, and an aggregate score per model. It is the sector's de facto reference and, in general, it is worth understanding what it says.

But there are three things MTEB does not tell you that are worth being clear about before using it as your only decision criterion.

**First: MTEB measures average performance on generalist public datasets.** Your domain is not generic. A model scoring 65 on MTEB over English news and public books may score 40 over your specific corpus of technical budgets in Spanish with financial jargon. And conversely: a model that is mediocre on MTEB may shine unexpectedly in your niche. Rankings are a starting point, not an oracle.

**Second: MTEB has become an optimisation target in itself.** Many models are tuned specifically to raise their MTEB score, which is not the same as being tuned to produce better results in real applications. This is the same dynamic that affected SAT scores in the US, or ImageNet in computer vision a decade ago: when a metric becomes a target, it stops being a good metric.

**Third: recent research (Vectara, NAACL 2025)** measuring 25 different chunking configurations over 48 embedding models found that **the variation introduced by the chunking strategy can be as large as or larger than the variation between models.** In other words: moving from naive chunking to well-tuned chunking can give you as much as moving from a mediocre model to an excellent one. Operational conclusion: investing time in optimising your chunker pays off more than obsessing over which model is 1-2 points better on MTEB. Article 3 is about exactly this; it is not a coincidence.

The honest way to use MTEB: as a coarse filter to discard clearly weak models, and as a secondary reference. The correct way to choose a model: benchmark on your own data with your real queries. That is exactly what we are going to do in the live session.

## Matryoshka: the trick that changes the maths

There is one concept that deserves its own section because it changes how you think about dimensionality: **Matryoshka Representation Learning (MRL)**.

The idea, simplified: during training, the model is optimised simultaneously to produce good embeddings at several nested dimensionalities (typically 256, 512, 1024, 1536, 3072). The magic consequence is that **the first dimensions of the embedding carry more information than the last ones**. You can truncate the vector to any length among the supported dimensions and keep most of the semantic quality.

How much quality do you keep? For `text-embedding-3-large`, OpenAI reported that the vector truncated to 256 dimensions beats the full `text-embedding-ada-002` at 1536. That is a six-to-one size reduction with a quality gain over the previous model.

There are two ways to apply it in production.

**Via the API parameter.** If you use OpenAI, you pass the `dimensions` argument in the call and the server returns the embedding already truncated and renormalised. This is the correct way.

```python
from openai import OpenAI

client = OpenAI()

def embed(text: str, dimensions: int = 1536) -> list[float]:
    """Embed text with explicit dimensionality control via Matryoshka."""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
        dimensions=dimensions,
    )
    return response.data[0].embedding

# Full embedding for highest quality
full = embed("OAuth 2.0 authentication backend", dimensions=1536)

# Compressed embedding for storage-constrained scenarios
compressed = embed("OAuth 2.0 authentication backend", dimensions=256)
```

**Via manual truncation of the vector.** If for whatever reason you have the full vector and want to reduce it afterwards (for example, you stored 1536d back in the day and now want a 256d version without re-calling the API), you can truncate it yourself. But there is a gotcha: **on truncating you lose the unit-norm property**, and many metrics and vector databases assume normalised vectors. You have to renormalise by hand.

```python
import math

def renormalize(vec: list[float]) -> list[float]:
    """Renormalize a vector to unit L2 norm.

    Required after manual truncation of Matryoshka embeddings.
    """
    norm = math.sqrt(sum(x * x for x in vec))
    if norm == 0:
        raise ValueError("Cannot normalize a zero vector")
    return [x / norm for x in vec]

# Correct manual truncation
full = embed("OAuth 2.0 authentication backend", dimensions=1536)
truncated_256 = full[:256]
truncated_256_normalized = renormalize(truncated_256)

# Wrong: skipping renormalization
# truncated_256 still has values, but ||truncated_256|| < 1, which
# breaks downstream cosine similarity calculations that assume unit norm.
```

The practical rule: when you can request the dimensionality you want directly via the API (`dimensions=768`), do it that way. Only fall back to manual truncation when you are retrieving already-stored vectors and want a shorter version.

When is truncating worth it? When you are going to have millions of vectors in production and storage cost or search latency start to matter. For the project's 15 budgets, the maths gives a few hundred chunks at most — the difference between 1536 and 256 dimensions is measured in megabytes, it is not the priority. What matters is that you know the lever exists and how to pull it when its moment comes.

## Five decision criteria

Boiled right down, the decision of which model to use reduces to balancing five axes. There is no single answer, there is an explicit trade-off you accept on each axis.

**Axis 1 — Dimensionality.** More dimensions, in general, more representational capacity. But also more bytes per vector, more compute latency in batch operations, more search time in the vector database. For a small project (thousands of vectors) dimensionality is irrelevant. For a system with hundreds of millions of vectors, going from 1536 to 768 can be the difference between fits in RAM and does not fit.

**Axis 2 — Corpus language.** If all your data is in English, that opens the door to English-centric models that are fast and cheap (like `all-MiniLM-L6-v2`). If you have a mix of languages or a dominant non-Anglo language, you need a multilingually trained model (`text-embedding-3-*`, `bge-m3`, `embed-multilingual-v3`). For the project, the component descriptions are in English (programme convention) but the briefs arriving from future clients will probably come in Spanish. This excludes English-only models and forces a decent multilingual one.

**Axis 3 — Domain.** A model trained mostly on news and general prose can be surprisingly mediocre on specialised technical jargon. For medical corpora there are medical models. For code there are models like `voyage-code-2`. For the "software budgets" domain there is no public specialised model, so we are going to have to use a generalist model and validate that it works reasonably over our data before building architecture on top.

**Axis 4 — Hosting and cost.** The classic trade-off is API vs self-hosted. The API gives you managed operations, automatic scaling and zero infra, in exchange for variable cost per token, network latency, dependence on an external provider, and data leaving your perimeter. Self-hosting gives you zero cost per token (only compute), predictable local latency, data that never leaves, and operational dependence on maintaining a GPU inference service. For low volumes (<20M tokens/month) the API is almost always cheaper in total. For high volumes with sensitive data, self-hosted wins.

**Axis 5 — Licence.** This matters more than most tutorials mention. OpenAI, Cohere and Voyage models are proprietary: you pay to use the API, you cannot self-host them, and the terms of service apply to your data. Models like `bge-m3` (MIT), `all-MiniLM-L6-v2` (Apache 2.0) or the `voyage-3-*` lite ones are self-hostable with permissive licences. If your end client contractually requires that no data leaves their infrastructure (banking, healthcare, defence), the options reduce to self-hosted.

These five axes are not prioritised equally in every project. For a startup at prototype stage, hosting and cost dominate: cheap and fast to integrate wins. For an enterprise healthcare system, licence and privacy dominate: nothing that cannot be self-hosted. For a system with a very specialised corpus, domain dominates: you will probably end up fine-tuning. Identify first which axis weighs most in your context and the decision simplifies.

> *(Figure in the original: `sesion-07-articulo-02-figura-02-ejes-decision.jpg` — image not included in this repo.)*

## The project's decision and why

For the project's AI service, the locked model is **OpenAI `text-embedding-3-small` with default dimensions (1536)**. The reasoning, axis by axis:

- **Dimensionality:** 1536 is excessive for the size of the project's corpus, but the extra storage cost is negligible at these scales (kilobytes, not gigabytes). Keeping the default simplifies things and leaves Matryoshka as a future optimisation lever.
- **Language:** decently multilingual. Component descriptions in English, briefs probably in Spanish. `text-embedding-3-small` is not the best multilingual model on the market (`bge-m3` or Cohere are), but it is clearly sufficient.
- **Domain:** no model is specialised in "software budgets", so any generalist will do. The live session will validate with real data that it works reasonably.
- **Hosting and cost:** API already configured since Session 01, zero additional friction. Ingesting 15 budgets with 5-10 chunks each and about 100 tokens per chunk is around 15,000 tokens total: $0.0003. Negligible.
- **Licence:** proprietary, but the project is academic and there is no sensitive data. Acceptable.

Why not the reasonable alternatives:

- **`text-embedding-3-large`:** 6.5x more expensive for +2 MTEB points. The improvement is not justified at this volume nor for this use case. If poor retrieval were seen in production, it would be the most obvious upgrade.
- **`bge-m3` self-hosted:** technically superior multilingually and free per token, but it introduces operational dependence (an inference server, ideally with a GPU for decent latency) that adds no pedagogical value to the programme. We leave it as a supplementary resource for the student who wants to explore.
- **`voyage-3-large`:** probably the best on recall over retrieval-focused benchmarks, but it introduces one more provider with its own API key, its own billing, and a higher cost. For a student already battling with OpenAI, adding Voyage is not the right pedagogy.
- **`all-MiniLM-L6-v2` local:** tempting for simplicity but clearly inferior multilingually and on MTEB. We will use it in the live session as a quick contrast so the student sees with their own eyes the difference between a lightweight model and a modern one.

The choice is not "it is the best possible model". The choice is "it is the best pedagogy/quality/cost/operations balance for this context". If tomorrow we change the context (scaling to millions of budgets, sensitive healthcare data, a very specific domain) the decision would change. Keep that mental flexibility.

## Practical comparison with code

To settle the decision you need to have made it with data, not with prose. What follows is the minimum measurement pattern you will run and refine in the live session. It works over any subset of texts from your own corpus.

```python
import time
from openai import OpenAI
from sentence_transformers import SentenceTransformer

# --- Setup ---

client = OpenAI()
local_model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

def embed_openai_small(text: str, dimensions: int = 1536) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
        dimensions=dimensions,
    )
    return response.data[0].embedding

def embed_local_minilm(text: str) -> list[float]:
    embedding = local_model.encode(text, normalize_embeddings=True)
    return embedding.tolist()

# --- Measurement harness ---

def benchmark(name: str, embed_fn, texts: list[str]) -> dict:
    """Run an embedding function over a list of texts and return basic metrics."""
    start = time.perf_counter()
    embeddings = [embed_fn(t) for t in texts]
    elapsed = time.perf_counter() - start

    return {
        "model": name,
        "n_texts": len(texts),
        "total_seconds": round(elapsed, 3),
        "per_text_ms": round((elapsed / len(texts)) * 1000, 1),
        "dimensions": len(embeddings[0]),
        "first_embedding_norm": round(
            sum(x * x for x in embeddings[0]) ** 0.5, 4
        ),
    }

# --- Run on a sample of your project's data ---

sample_texts = [
    "OAuth 2.0 authentication backend with JWT tokens for fintech mobile app",
    "Product catalog service with full-text search and category filtering",
    "GDPR consent management module with audit log",
    "Kubernetes deployment pipeline with blue-green release strategy",
]

print(benchmark("openai-3-small-1536d", embed_openai_small, sample_texts))
print(
    benchmark(
        "openai-3-small-256d",
        lambda t: embed_openai_small(t, dimensions=256),
        sample_texts,
    )
)
print(benchmark("local-minilm-l6-v2", embed_local_minilm, sample_texts))
```

The qualitative observations you will draw from running this over your own dataset:

- **Latency per text:** OpenAI hovers around 200-400 ms per serial call (network + server processing). Local MiniLM will be around 10-30 ms per text on CPU. In batch the figures change: OpenAI's API accepts batches and throughput rises a lot. We cover batching in the pre-session exercise.
- **Dimensionality:** 1536 vs 256 vs 384. MiniLM's 384 are not linearly comparable to OpenAI's truncated 256; they are different spaces trained with different techniques.
- **Norm:** check that it is 1.0 (within floating-point error) in all three cases. If it is not, the model does not deliver normalised vectors and you need to normalise them yourself before using cosine in databases that assume norm 1.

On real cost: with OpenAI you can calculate it approximately as `n_tokens × $0.02 / 1,000,000`. For your 4 example texts, that is about 40 tokens total, rounding: $0.0000008. For the project's full corpus, it is not even worth tracking. When volumes grow it is worth it, and then OpenAI's Batch API comes into play, applying a 50% discount in exchange for asynchronous processing of up to 24 hours. We will see it in the pre-session exercise.
