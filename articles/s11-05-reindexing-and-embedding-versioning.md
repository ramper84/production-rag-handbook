---
title: Reindexing and embedding versioning
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 17 min
added: 2026-08-24
summary: >
  The index rots in two ways and neither raises an exception. Content drift —
  the document changed, the vector did not — makes retrieval cite a source that
  no longer says what it said. Version mixing is worse: cosine between vectors
  from two different models is a plausible number with no meaning, and the
  database returns it happily. The defence is that every vector records how it
  was made — model, dimensions, normalisation, preprocessing — and that queries
  never cross versions. Incremental reindexing is the daily tool and a trap
  outside its version; a model change is a blue/green migration, verified
  before promotion, never half-done.
keywords: [embedding versioning, reindexing, content drift, version mixing,
           silent failure, embedding_version column, source hash, staleness,
           incremental reindex, blue/green migration, shadow index, atomic
           cutover, verification before promotion, event-driven reindexing]
---

# Reindexing and embedding versioning

*Antonio Perez* · 🔴 17 min

The estimation system has a corpus of historical budgets already vectorised and persisted. Retrieval works, generation leans on real sources, and all the work of the previous sections — citing, verifying, abstaining — rests on a premise that is rarely questioned: **that the index's vectors still faithfully represent what the documents say, and that they all live in the same space.**

That premise erodes on its own. New budgets arrive that have to be added. An old budget is corrected and its text changes, but its vector is still the old one. A better embedding model appears and you want to migrate. Each of these events, badly managed, degrades retrieval. And it does so in the worst possible way: **without throwing a single error.**

## The failure that raises no error

Most bugs shout: an exception, a stack trace, a red test. Vector index drift does not shout. It whispers, and sometimes not even that.

There are two ways for the index to rot. The first is **content drift**: a budget was indexed, then somebody corrected a figure in the original document, and the stored vector still represents the old text. The search retrieves that fragment believing it says 40h when the current document says 60. Generation cites a source that no longer matches its content — a false attribution that is not the model's fault, but the index's.

The second is more insidious: **version mixing**. Imagine you re-embed half the corpus with a new model and leave the other half with the old one. Now you have vectors from two different models in the same table. The cosine similarity between a model-A vector and a model-B one **means nothing** — they are two different geometric spaces — but the database compares them anyway and returns you a number. A plausible number. A "close" neighbour with no semantic sense whatsoever. There is no error, no exception: only silently broken retrieval, and a generation grounded on irrelevant fragments retrieved with a meaningless metric.

> *(Figure 13 in the original: `art5-fig13-fallo-silencioso.jpg` — image not included in this repo. Two panels. **Content drift**: a green box "Current document — Payments module 60h" above a red box "Indexed vector — Payments module 40h (old text)", separated by a ≠, noted "retrieval brings 40h: it cites a source that no longer says what it said. No error." **Version mixing**: two vector boxes, model A and model B, with `cos(A, B) = 0.83` struck through and labelled "meaningless number", noted "two different geometric spaces; the database compares them anyway and returns an absurd neighbour." Caption: "Neither one throws an exception: just silently broken retrieval.")*

This is what makes the index's life cycle deserve care: **the cost of getting it wrong is not a visible outage, it is an invisible degradation** you discover months later, when somebody complains that "the estimates are not as good any more" and you have no idea why.

## Versioning the index: every vector knows how it was made

The defence against version mixing is conceptually simple: **every vector carries a record of how it was produced, and queries never cross versions.** A vector is only ever compared with others born of exactly the same process.

"The same process" is more than "the same model". It is the model, its dimension, whether it was normalised, and the preprocessing configuration (chunking and cleaning) with which the embedded text was generated. Any of those pieces changing produces vectors that are not comparable with the previous ones.

```python
class EmbeddingVersion(BaseModel):
    model: str                 # "text-embedding-3-small"
    dimensions: int            # 1536
    normalized: bool
    preprocessing_id: str      # id/hash of the chunking + cleaning config

    @property
    def key(self) -> str:
        return f"{self.model}:{self.dimensions}:{self.normalized}:{self.preprocessing_id}"
```

That `key` is stored alongside every chunk, in an `embedding_version` column, and becomes a mandatory part of every retrieval query:

```sql
-- Retrieval always scopes to the single active embedding version.
-- Comparing vectors across versions is meaningless, so we never do it.
SELECT chunk_id, document_id, content
FROM chunks
WHERE embedding_version = :current_version
ORDER BY embedding <=> :query_vector
LIMIT :k;
```

> *(Figure 14 in the original: `art5-fig14-versionado-vector.jpg` — image not included in this repo. On the left, a row in the chunks table: `chunk_id "fin-2024-07#c3"`, `document_id "fin-2024-07"`, `content "Payments module..."`, `embedding [0.013, -0.21, ...]`, with `embedding_version "…3-small:1536:true:cfg-a7"` and `source_hash "a7f3…e1"` highlighted. On the right, the retrieval query with `WHERE embedding_version = :current` highlighted, annotated "vectors from another version are excluded: two different spaces are never compared". Caption: "The version filter is not an optimisation: it is the guarantee that cosine means something.")*

The `WHERE embedding_version = :current_version` is not an optimisation: **it is a correctness guarantee.** Without it, a half-finished migration silently contaminates every search. With it, the old version's vectors simply stop participating, even though they remain physically in the table during the transition.

## When and how to reindex

The principle that decides when reindexing is needed is the same one that justifies versioning: anything that changes how a vector is produced invalidates its comparison with vectors produced another way. The triggers follow from that.

A new or corrected document affects only that document: that is **incremental reindexing**, the common and cheap case. A change of embedding model, of dimension, or of chunking strategy affects the whole corpus: that is a **version migration**, expensive and infrequent. The practical rule: **if the change touches a document, reindex that document; if it touches the process, reindex everything.**

For incremental reindexing, the key piece is knowing what has changed without re-embedding what has not. A hash of the source's content, stored alongside the chunk, resolves this: if the document's current hash does not match the one that was embedded, the vector is stale.

```python
def is_stale(chunk: StoredChunk, source_hash: str, current: EmbeddingVersion) -> bool:
    """A chunk is stale if its source text changed or it belongs to an old version."""
    return chunk.source_hash != source_hash or chunk.embedding_version != current.key


async def reindex_incremental(documents: list[Document], current: EmbeddingVersion) -> None:
    for document in documents:
        source_hash = content_hash(document.text)
        existing = await get_chunks(document.id)
        if existing and not any(is_stale(c, source_hash, current) for c in existing):
            continue  # up to date, skip

        await delete_chunks(document.id)
        chunks = chunk_and_embed(document, current)  # reuses the existing ingestion pipeline
        await insert_chunks(chunks)
        log.info("document_reindexed", document_id=document.id, version=current.key)
```

Two things worth noticing. The first: `chunk_and_embed` is not reimplemented here; it is the same ingestion pipeline that already vectorises documents, invoked with the current version. **Reindexing does not invent a new path, it reuses the existing one.** The second: **incremental reindexing is only valid within a version.** As soon as the active version changes, inserting new chunks alongside the old ones is precisely the version mixing we want to avoid. Incremental is the day-to-day tool; it is not the tool for a migration.

## Migrating version: never half-way

When the model changes, no incremental reindexing will do. The whole corpus has to be re-embedded with the new model, and while that happens, search has to keep working with the old one. The safe way is **building the new index alongside the old, verifying it, and switching from one to the other in one go.**

```python
async def migrate_embedding_version(new: EmbeddingVersion) -> None:
    """Re-embed the whole corpus into a new version, then cut over atomically.

    Never mix versions in the live query space: build alongside, verify, switch.
    The old vectors keep serving queries until the switch; if verification
    fails, nothing changes for the user.
    """
    await build_shadow_index(new)          # embed every document with the new model
    if await verify_shadow_index(new):     # counts match, dimensions correct, sample queries sane
        await promote_active_version(new)  # atomic switch of the active version pointer
        await drop_old_version_vectors()   # only after a successful, verified switch
    else:
        await discard_shadow_index(new)
        log.error("embedding_migration_aborted", version=new.key)
```

> *(Figure 15 in the original: `art5-fig15-migracion-blue-green.jpg` — image not included in this repo. Three stages left to right. **1 · Build in parallel**: "queries → v1, active, serving" in green above a dashed "v2 (shadow), embedding the whole corpus", noted "the user notices nothing". **2 · Verify v2**: ticks for document count, correct dimensions, sample queries — "only if everything passes is it promoted". **3 · Atomic switch**: "queries → v2, new active" with "v1 discarded" struck through, noted "a single step: no mixed instant". A red dashed arrow returns from stage 2 to stage 1: "if verification fails → v2 is discarded, v1 stays intact". Caption: "Never half-way: while v2 is built and verified, v1 keeps serving; the switch is a single step.")*

The pattern is the classic **blue/green** applied to vectors. While the shadow index is built, queries keep being served from the active version — the `WHERE embedding_version` points at the old one and the user notices nothing. The switch of active version is a single atomic step: before, everybody searches the old one; after, everybody the new one. **There is no instant at which the two mix in one query.** And if the shadow index's verification fails — because documents are missing, because the dimensions do not match, because some test queries return absurd neighbours — it is discarded and nothing happens: the user carries on with the version that worked.

**Verification before promoting is not optional.** A migration that switches the active version without checking that the new index is complete and healthy can replace a good index with a broken one in a single atomic step, which is exactly the scenario blue/green was meant to avoid.

## Honest trade-offs

**Incremental is cheap and is a trap outside its version.** For new or corrected documents, re-embedding only what changed is the right option and almost free. But using incremental when what changed is the model is how you arrive at version mixing. The question before every reindexing is always the same: **did the document change, or did the process change?** The answer decides the tool.

**Migrating costs money and time, and has to be budgeted.** Re-embedding a whole corpus is as many calls to the embedding model as you have chunks, plus the temporary storage of the shadow index, which during the transition doubles the vector space. It is not a routine operation: it is a migration, it is planned as such, and the cost is accepted in exchange for the safety of not breaking search in production.

**Staleness detection is only as good as your change capture.** The content hash works if you have the source's current text to hash. If a budget is modified in an external system and nobody tells you, your index never learns it is stale. Detecting content drift demands, at some point, a synchronisation mechanism or a periodic rehash; **the hash alone does not discover changes you never hear about.**

**Skipping versioning looks free until it is not.** "We are never going to change model" is one of the most expensive sentences in a RAG system, because you will end up changing it — a better one will appear, or a cheaper one — and then the absence of versioning turns a clean migration into a silent contamination impossible to diagnose. **The `embedding_version` column is dirt-cheap insurance against an invisible and very expensive failure. Put it in from the start, even if today you only have one version.**

> *(Editor's note — checked against the code, and this is the one recommendation the implementation has not taken. There is no `embedding_version` column and no `source_hash` column: the chunk tables carry `id`, `document_id`, `chunk_type`, `content`, `embedding`, a generated `content_tsv`, `metadata` and `created_at`, and dimension is fixed corpus-wide by an `EMBEDDING_DIMENSIONS` constant rather than recorded per row. Nothing named `EmbeddingVersion`, `is_stale`, `reindex_incremental` or a shadow index exists in `app/` or in the Alembic migrations. So the system is currently in exactly the state this section describes as one model change away from silent contamination, and it cannot detect content drift at all. Two practical notes for whoever closes the gap: the `metadata` JSONB column can hold a version key without a migration, as a stopgap; and the SQL above says `FROM chunks`, which predates the split into `budget_chunks`, `transcript_chunks` and `technical_doc_chunks` — the version filter has to go on all three, and a migration has to re-embed all three.)*

**Reindexing by calendar either wastes or falls short.** A cron that reindexes everything nightly burns compute re-embedding what has not changed; one that reindexes monthly lets staleness accumulate in between. Where you can, tie reindexing to change events — a budget approved, a document corrected — rather than to a clock. **The clock is the last resort, not the first.**

## What this leaves unresolved

With versioning, incremental reindexing for content drift and blue/green migrations, the index stops rotting in silence: every vector knows how it was made, queries never cross versions, and a model change does not break search half-way through.

But a question remains that none of these guarantees answers, and it is the same one everything in this subject returns to. You have migrated to a new embedding model, with all the care in the world. Search did not break. **But did it improve?** Does the new model retrieve more relevant budgets than the old one, or have you spent an expensive migration to stay the same, or even to get slightly worse without noticing? Blue/green guarantees you break nothing visible; it does not guarantee the change was an improvement.

Keeping the index healthy prevents quality degrading by accident. But knowing whether a design decision — a new model, a different prompt, another way of assembling the context — genuinely improves or worsens the system is something you cannot answer by inspecting one response or verifying an index. **It is answered by measuring**, over a representative set, with numbers comparable between versions. Without that measure, every migration and every adjustment is a blind bet with a very well-fitted blindfold.
