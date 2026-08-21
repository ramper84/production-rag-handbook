---
title: "Embeddings: from text to semantic geometry"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 7
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 21 min
added: 2026-08-13
summary: >
  The minimum theory behind semantic search: what an embedding is, why
  contrastive training makes distance mean similarity, and the three metrics
  (cosine, dot product, Euclidean) — with the argument that metric choice is a
  property of the model, not an architectural decision. Closes with four honest
  limits, including that embeddings cannot reason about numbers, dates or ids.
keywords: [embeddings, semantic search, cosine similarity, dot product,
           euclidean distance, contrastive learning, curse of dimensionality,
           hybrid search, BM25, normalization]
---

# Embeddings: from text to semantic geometry

*Antonio Perez* · 🔴 21 min

You have just finished Session 06 with real projects from several sectors. Each one with its components broken down, estimated hours, technology stack and descriptions.

What you want to do now is this: when a new client arrives with a brief along the lines of *"we need an authentication service with OAuth flows for a mobile app in the financial sector"*, the system should automatically find which components of your historical budgets are relevant for building the estimate. And it should find them even if the brief shares no words literally with your documents. "Authentication service" should find "OAuth 2.0 backend", "JWT authorization module", "single sign-on integration", and all the rest even though none of them uses the exact word "authentication".

This is semantic search. The problem is not new; what is new is the tool we are going to use to solve it: embeddings. This article establishes the minimum theoretical basis for understanding what they are, why they work, and how two texts are compared via their embeddings. We are not yet discussing which model to choose (that is the next article) nor how to split long documents (articles 3 and 4). Just the first brick.

## What an embedding is

An embedding is, mechanically, a function that takes any text and returns a fixed-dimension vector of real numbers. For OpenAI's `text-embedding-3-small`, that dimension is 1536. For Sentence Transformers' `all-MiniLM-L6-v2`, it is 384. For `text-embedding-3-large`, 3072 dimensions. But the important thing is not the number, it is the property that holds over those vectors: **semantically similar texts produce nearby vectors in the space R^n**.

Let's see it with code. We assume you already have `OPENAI_API_KEY` configured and the OpenAI client installed, exactly as we left it in Session 01.

```python
from openai import OpenAI

client = OpenAI()

text = "OAuth 2.0 authentication backend with JWT tokens for fintech mobile app"

response = client.embeddings.create(
    model="text-embedding-3-small",
    input=text,
)

embedding = response.data[0].embedding

print(f"Dimensions: {len(embedding)}")
print(f"First 5 values: {embedding[:5]}")
print(f"Last 5 values: {embedding[-5:]}")
print(f"Value type: {type(embedding[0]).__name__}")
```

The output is a list of 1536 floats. If you print it in full, you will see absolutely nothing interpretable: a sequence of small numbers around zero. No individual dimension means anything legible to a human. Dimension 142 is not "fintech-ness" and 803 is not "project complexity". **The individual dimensions have no direct interpretation** — they are the result of a blind optimisation process during training.

What does have meaning is the directions and the relative distances between vectors. That is what we are going to exploit.

## How they learn the geometry

Modern embedding models are trained with a technique called **contrastive learning**. Simplifying considerably: during training the model is shown millions of triplets made of an *anchor* (a text), a *positive* (a text that should be semantically close to the anchor) and one or more *negatives* (unrelated texts). The training loss punishes the model when the anchor ends up closer to the negative than to the positive, and rewards it when the anchor ends up closer to the positive. After millions of iterations, the model finds a geometric representation of the space where proximity measures something resembling semantic similarity.

What counts as "the positive" depends on what is being trained. For general-purpose embeddings, positives are usually paraphrases, translations, consecutive sentences from the same document, or question-answer pairs. For embeddings specialised in code, positives are code fragments that solve similar problems. For multilingual embeddings, positives are translations of the same text into different languages. **The nature of the positives determines what the model understands by "similar".**

This has a direct practical consequence: if you train a model on general-English paraphrases and then feed it technical budgets in Spanish with specific financial jargon, do not expect the same level of discrimination as in the domain it was trained for. We address this in the next article when we discuss model selection.

An epistemological note worth stating clearly: nobody explicitly programmed the dimensions of the vector space. They emerged from training. There are research lines (mechanistic interpretability) attempting to decipher what each direction encodes, but at production level we treat the model as a black box whose distances work. They work empirically, we measure it with benchmarks, and that is enough to build systems on them.

The classic example always cited is `vector("king") - vector("man") + vector("woman") ≈ vector("queen")`. It is a real result from word embeddings of a decade ago (Word2Vec), but be warned: with modern sentence embeddings those arithmetic games almost never work that cleanly. What does remain true is that nearby vectors still correspond to nearby texts, and that is the only thing you need to build semantic search.

> *(Figure in the original: `sesion-07-articulo-01-figura-01-espacio-semantico.jpg` — image not included in this repo.)*

## Similarity metrics

To talk about "nearby vectors" you need a metric. There are three you will encounter in every vector-search API and library. Those three are the only ones you need to know.

> *(Figure in the original: `sesion-07-articulo-01-figura-02-metricas-similitud.jpg` — image not included in this repo.)*

**Cosine similarity.** Measures the angle between two vectors. Returns a value between -1 and 1, where 1 means they point in the same direction, 0 that they are perpendicular, and -1 that they point in opposite directions. For text, the usual practice is that values fall between 0 and 1 (modern embeddings rarely produce vectors pointing in opposite directions, because the contrastive loss does not explicitly reward negatives ending up on the opposite side of the space).

The formula:

```
cosine(A, B) = (A · B) / (||A|| × ||B||)
```

where `A · B` is the dot product and `||A||` is the Euclidean norm of vector A.

Its defining trait: **it is insensitive to the magnitude of the vectors.** It only looks at direction. Two vectors pointing at exactly the same place give `cosine = 1` even if one is twice as long as the other. This is desirable for text, because we do not want a long document (which may end up producing a vector with greater magnitude) to look less similar to a short query merely because of its size.

**Dot product.** It is the numerator of cosine similarity. The formula:

```
dot(A, B) = sum(A[i] × B[i] for i in range(n))
```

Returns an unbounded value, sensitive to both direction and magnitude. Computationally it is cheaper than cosine because it does not require computing the norms.

There is an important detail: **most modern models, including `text-embedding-3-small`, produce vectors already normalised to length 1** (Euclidean norm = 1). When vectors are normalised, dot product and cosine similarity give exactly the same result. That is why many production vector databases use dot product internally: same quality as cosine, fewer operations per query.

**Euclidean distance.** The straight-line distance between the two points in R^n. The formula:

```
euclidean(A, B) = sqrt(sum((A[i] - B[i])^2 for i in range(n)))
```

Returns a value between 0 (identical) and infinity (very far). Sensitive to magnitude.

For normalised vectors (length 1), Euclidean distance and cosine similarity are related by a simple formula: `euclidean(A, B)^2 = 2 - 2 × cosine(A, B)`. They order results the same way, so any top-k search using Euclidean over normalised vectors returns the same results in the same order as using cosine.

**The practical rule for choosing a metric: use the one used during the model's training.** The model card states it. If the model was trained with cosine as the similarity function in the loss, use cosine in production. If it was trained with dot product (and produces normalised vectors), use dot product. For OpenAI's `text-embedding-3-small` the vectors come normalised, so cosine and dot product are interchangeable. For `sentence-transformers/all-MiniLM-L6-v2`, the model card recommends cosine.

There is no mystery. There is no universally better metric. **It is a property of the model, not an architectural decision.**

Let's implement all three with Python's standard library, without numpy. This is exactly what you will use in the `compare.py` of the pre-session exercise:

```python
import math

def cosine_similarity(vec_a: list[float], vec_b: list[float]) -> float:
    """Compute cosine similarity between two equal-length vectors.

    Returns a value in [-1, 1]. For text embeddings from modern models,
    values typically fall in [0, 1].
    """
    if len(vec_a) != len(vec_b):
        raise ValueError("Vectors must have the same dimensionality")

    dot = sum(a * b for a, b in zip(vec_a, vec_b))
    norm_a = math.sqrt(sum(a * a for a in vec_a))
    norm_b = math.sqrt(sum(b * b for b in vec_b))

    if norm_a == 0 or norm_b == 0:
        raise ValueError("Cannot compute similarity for zero-norm vectors")

    return dot / (norm_a * norm_b)

def dot_product(vec_a: list[float], vec_b: list[float]) -> float:
    """Compute the dot product. For normalized vectors, equivalent to cosine."""
    if len(vec_a) != len(vec_b):
        raise ValueError("Vectors must have the same dimensionality")
    return sum(a * b for a, b in zip(vec_a, vec_b))

def euclidean_distance(vec_a: list[float], vec_b: list[float]) -> float:
    """Compute the Euclidean distance. Lower means more similar."""
    if len(vec_a) != len(vec_b):
        raise ValueError("Vectors must have the same dimensionality")
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(vec_a, vec_b)))
```

Three functions, thirty lines, no external dependency. That is everything you need to build comparison between embeddings.

## Applied to the project

Let's see the geometry in action over texts from the project's domain. Embed these three pairs and observe the results:

```python
from openai import OpenAI

client = OpenAI()

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return response.data[0].embedding

pairs = [
    # Pair 1: technically close, different wording
    (
        "User authentication API with role-based access control",
        "Login service backend with permission management",
    ),
    # Pair 2: unrelated, same domain (web backend)
    (
        "User authentication API with role-based access control",
        "Real-time WebSocket chat module with message persistence",
    ),
    # Pair 3: generic, ambiguous overlap
    (
        "Performance optimization for high-traffic endpoints",
        "Caching strategy for database-heavy queries",
    ),
]

for text_a, text_b in pairs:
    vec_a = embed(text_a)
    vec_b = embed(text_b)
    sim = cosine_similarity(vec_a, vec_b)
    print(f"Similarity: {sim:.4f}")
    print(f"  A: {text_a}")
    print(f"  B: {text_b}")
    print()
```

Without running it, you can already anticipate the qualitative behaviour. For pair 1 you would expect high similarity: both texts describe essentially the same thing (user authentication with access control) with different vocabulary. For pair 2, substantially lower similarity: both are web backend but they are different features. For pair 3, the similarity is going to be interesting: both texts talk about performance optimisation but from different angles (endpoints vs queries), so the result will depend on how far the model associates "performance" with "caching" in its training.

**What matters is the structure of the result, not the absolute numbers.** If pair 1 comes out clearly above pair 2, the model is discriminating well. If all three gave similar values, you would have a problem: the model does not separate near from far in your domain, and you would need to change it.

This is exactly what you measure in the `SANITY_CHECK.md` of the pre-session exercise, though with three different pairs. It is not formal validation of retrieval quality (that comes in Session 11 with metrics like recall@k and NDCG), but it is the minimum acceptable for confirming the pipeline works end to end before investing time in optimising it.

## What an embedding does not solve

A couple of honesties before closing the article, because the promotional narrative around embeddings tends to present them as a silver bullet and they are not.

**Embeddings do not understand numbers, dates or identifier codes.** If your query is "budgets from 2024" and the budgets carry the year in metadata, do not expect the embedding to learn to filter by year on its own. For that you use structured filters over the metadata, not vector similarity. Numbers appear in embeddings as any other token and the model does not apply arithmetic to them.

**Embeddings are weak at exact matches of rare words.** If you search for a proper noun that appears verbatim in a document, BM25 (the classic term-frequency / inverse-document-frequency metric) will probably retrieve it better than any embedding. That is why many serious production searches combine both (*hybrid search*), a topic we will see in Session 10.

**Embeddings suffer the curse of dimensionality.** In high-dimensional spaces, distances between random pairs of points tend to concentrate in a narrow range. This is why seeing similarity values between 0.2 and 0.5 for unrelated texts is completely normal: the "zero" of non-relatedness does not appear in practice. **Calibrate your thresholds on your own dataset**; do not assume `sim > 0.7` means "very similar" in absolute terms. It only means "more similar than the unrelated pairs I measured".

**The choice of model matters far more than the choice of metric.** Switching from cosine to dot product when the model is normalised changes nothing. Switching from an English-only model to a multilingual one when your data is half Spanish and half English does change the results drastically.
