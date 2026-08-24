---
title: Citation and verifiable attribution
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-08-24
summary: >
  An identifier is not a citation. A verifiable one resolves to a source that
  exists, locates the concrete line backing the claim, and is traceable to the
  original — and line-level citation is not a generation decision but an
  ingestion one, enabled long before by how the sources were stored. Referential
  integrity is checked in code after generating, never trusted to the model,
  because a dangling citation looks exactly like a legitimate one. Structured
  citation is the source of truth and inline versus footnote is presentation
  over it. The AI service emits document_id and locator, never URLs: the
  business layer knows who the user is and what they may open.
keywords: [verifiable citation, source attribution, locator, char span,
           dangling citation, referential integrity, structural vs semantic
           verification, citation formats, inline markers, footnotes,
           permission-checked links, layer responsibility, access control]
---

# Citation and verifiable attribution

*Antonio Perez* · 🔴 19 min

The estimate the system produces is no longer a loose figure. Each component arrives with the list of identifiers of the fragments it derived from: `["fin-2024-07#c3", "ecom-2023-02#c1"]`. It is a good start. But **an identifier is not a citation.** The project manager who receives "payments module: 40h" and, next to it, `fin-2024-07#c3` knows nothing they did not know before: that code does not tell them which budget it comes from, from what year, nor which exact line backs it. And the system consuming the estimate cannot check anything with it either if it does not resolve to something real.

A citation a human cannot resolve and a system cannot verify is not a citation: **it is decoration.** It gives a sensation of rigour — "look, it cites sources" — without contributing the only thing that matters, which is being able to go to the source and check that the number is real. In an estimation system this is not an academic ornament. It is the difference between "trust me" and "check it yourself", and the second is what makes somebody commit hours and money leaning on the output.

This article is about turning those identifiers into verifiable citations: ones that resolve to a real source, that point at the concrete line backing each claim, and where the system guarantees that no citation points into the void.

## What makes a citation verifiable

A decorative citation says "based on historical data". A verifiable citation satisfies three properties, and all three are checkable.

**The first is that it resolves**: the identifier points at a source that really exists and that the system can retrieve. If the model cites `fin-2024-07#c3` but that fragment was never in the retrieved context, the citation is dangling in the void. Resolving is not optional: it is the line between attribution and fiction.

**The second is that it locates**: the citation points not merely at a forty-page document, but at the concrete line or fragment backing the claim. "Based on project X (2024)" is better than nothing, but "project X (2024), line: Payments module (Stripe), 40h" is what lets a human open the document and find the datum in ten seconds. The locator is what separates a vague attribution from a verifiable one.

**The third is that it is traceable to the origin**: there exists a path, from the number in the estimate to the original budget, that anyone with permission can walk. It is not enough for the system to know where the datum comes from; the estimate's consumer has to be able to reach the source, ideally with one click.

> *(Figure 7 in the original: `art3-fig7-escala-verificabilidad.jpg` — image not included in this repo. Three rows scored against the columns *resolves* / *locates* / *traceable*. Top, in red and tagged **decorative**: `fin-2024-07#c3`, "opaque id, meaningless to the consumer" — three crosses. Middle, amber and tagged **presentable**: "Project X (2024)", "resolves to a document, but not to the line" — resolves only. Bottom, green and tagged **verifiable**: "Project X (2024) — 'Payments module (Stripe) 40h'" with an "[open original budget]" link — all three ticks. Caption: "Verifiability rises as the citation resolves to a real source, locates the line and is traceable to the origin.")*

The three together turn "40h for payments" into an **auditable claim**. Let us build them.

## From id to resolvable citation

The first step is resolving the identifier to an object with meaning. The retrieved fragment already drags the metadata we need — which document it is from, from what year, and, if it was captured at ingestion, which original line it represents. The citation is the projection of that metadata into a form designed to be read and checked by a human.

```python
class Citation(BaseModel):
    chunk_id: str
    document_id: str
    document_title: str            # human-meaningful: "Presupuesto Fintech App — Cliente X"
    project_year: int
    locator: str                   # the exact source line backing the claim
    char_span: tuple[int, int] | None  # offsets into the source document, if captured


def resolve_citation(chunk_id: str, retrieved: dict[str, RetrievedChunk]) -> Citation:
    """Project a retrieved chunk into a human-meaningful, verifiable citation.

    A KeyError here means the id was never in the retrieved context: a
    dangling citation, handled by the integrity check, not silently.
    """
    chunk = retrieved[chunk_id]
    return Citation(
        chunk_id=chunk.chunk_id,
        document_id=chunk.document_id,
        document_title=chunk.document_title,
        project_year=chunk.project_year,
        locator=chunk.source_line,
        char_span=chunk.char_span,
    )
```

The `locator` is the field that decides whether the citation is genuinely verifiable or merely presentable. And here there is an honest dependency worth facing head-on: **you can only cite at line level if you captured the locator back then.** If the fragments were indexed storing the original line or the character range they came from, the citation can be exact. If it was not captured, the most you can aspire to is citing at document level — "based on project X (2024)" — and verification becomes cruder, because the reader has to hunt for the datum by hand through the whole document. **Verifiable line-level citation is not something you decide at generation: it is something you enable much earlier, in how you stored your sources.**

## Referential integrity: no dangling citations

Resolving a citation assumes the identifier exists in the retrieved context. But models, even instructed to cite only provided sources, sometimes invent a good-looking identifier that was never there. **A dangling citation** — an id that does not belong to the context handed to the model — **is the most dangerous citation failure, because it looks exactly like a legitimate one.**

That is why referential integrity is not trusted to the model: it is verified in code, after generating.

```python
class CitationIntegrityReport(BaseModel):
    resolved: list[str]
    dangling: list[str]   # cited ids that were never in the retrieved context


def check_citation_integrity(
    estimate: Estimate,
    retrieved_ids: set[str],
) -> CitationIntegrityReport:
    resolved, dangling = [], []
    for component in estimate.components:
        for cid in component.source_chunk_ids:
            (resolved if cid in retrieved_ids else dangling).append(cid)
    if dangling:
        log.warning("dangling_citations", ids=dangling)
    return CitationIntegrityReport(resolved=resolved, dangling=dangling)
```

> *(Figure 8 in the original: `art3-fig8-cita-colgante.jpg` — image not included in this repo. On the left, an estimate citing four sources; on the right, the retrieved context holding three. Green arrows connect `fin-2024-07#c3`, `ecom-2023-02#c1` and `fin-2025-04#c2` to their matches; the fourth, `fin-2099-01#c9`, is red and its dashed arrow runs off to a cross marked "does not exist in the context — dangling → reject". Caption: "It is verified in code, not trusted to the model: a dangling citation looks exactly like a legitimate one.")*

Detecting the dangling citation is half the work; the other half is the policy for what to do with it. The reasonable options, from most to least strict: **reject the whole estimate and retry** with a harder instruction; **degrade the affected component** to "no verifiable source" and lower its confidence; or, at minimum, **do not let an estimate with a non-resolving citation leave the service.** The one that is never acceptable is ignoring it, because then you are handing the user a false attribution with a verified stamp.

It is worth being precise about what this check guarantees and what it does not. **Referential integrity confirms the cited source exists and was in the context. It does not confirm the source says what the citation claims it says.** It is a structural verification, not a semantic one: necessary, but not sufficient. We will come back to this at the end, because it is exactly where the harder problem begins.

> *(Editor's note — checked against the code, and this article is implemented. `check_citation_integrity` ships as `verify_citations` in `rag/validation.py`, with `CitationReport` in place of `CitationIntegrityReport`. Three differences matter. **The real report is per line, not per component**, and carries a third status this article does not mention: `LineCitationStatus = Literal["grounded", "dangling", "insufficient"]`, where *insufficient* is a line the model declined to ground rather than one it grounded badly — an honest refusal, which the binary resolved/dangling split here cannot express. **The policy composes two of the three options** rather than choosing one: `estimator.py` retries once with feedback naming the offending ids, then, if they are still dangling, downgrades the estimate's `confidence` to low instead of rejecting. And **the locator is not what this article describes**: there is no `char_span` or `document_title` captured at ingest; `SourceReference.evidence` is a verbatim span the *model* copies from the chunk. That is cheaper and strictly weaker — a stored offset cannot be misquoted and a model-copied span can — which is precisely the crack the closing section names. The Rails `CitationLinkResolver` is not implemented.)*

## Formats: structure first, presentation afterwards

The question "inline, footnotes or links?" is usually posed badly, as though they were mutually exclusive alternatives. They are not. The estimate is a data structure — JSON with components and, now, citations per component — and the formats are different ways of rendering the same structure. **The source of truth is the structured citation**; inline and footnotes are presentation decisions taken on top.

This separation matters because different parts of the output ask for different formats. The estimate's components are data: their citations travel as structured data and the frontend decides whether to show them as chips, as a column or as a side panel. The prose summary field, by contrast, does benefit from inline markers resolving to a list of sources at the foot, as in an article.

```python
def render_sources_block(citations: list[Citation]) -> str:
    """Render structured citations as a numbered sources block.

    The same Citation objects can feed inline markers in the prose summary;
    this is presentation over a single structured source of truth.
    """
    lines = []
    for index, citation in enumerate(citations, start=1):
        lines.append(
            f"[{index}] {citation.document_title} ({citation.project_year}) — {citation.locator}"
        )
    return "\n".join(lines)
```

Each format has its compromise. **Inline markers** (`40h [1]`) are compact and natural in prose, but they clutter if every number carries its own and they become ambiguous when a claim leans on several sources at once — which, after synthesising, is the norm. **Footnotes** separate the claim from its backing cleanly and scale well to many sources, at the cost of forcing the reader to jump. **Links to the original document** give maximum verifiability — one click and you are at the source — but they open a question that is not cosmetic: historical budgets are usually confidential, and not everyone who sees an estimate has permission to open the budget it came from.

And there a responsibility boundary appears that is worth respecting. **The AI service should not emit URLs nor take for granted that the consumer can see every document**: it emits `document_id` and `locator`, neutral data. It is the business layer that knows who the user is and what they have permission to see, and that resolves that `document_id` into a real, authorised link. The pattern is independent of the stack — any HTTP backend can do this resolution — but in the reference implementation, in Rails, it looks like this:

```ruby
# Business backend (Rails): resolve a citation's document to a link the
# current user is actually allowed to open. The AI service emits document_id
# and locator; it never emits URLs or assumes visibility. Stack-agnostic:
# any HTTP backend can perform the same permission-checked lookup.
class CitationLinkResolver
  def initialize(user)
    @user = user
  end

  def link_for(document_id)
    document = HistoricalBudget.find(document_id)
    return nil unless @user.can_view?(document)

    Rails.application.routes.url_helpers.historical_budget_path(document)
  end
end
```

> *(Figure 9 in the original: `art3-fig9-contrato-capas-enlace.jpg` — image not included in this repo. Three boxes left to right. **AI service**: "emits, neutral: `document_id`, `locator`", annotated "no URL, no assumption about permissions". **Business backend**: `HistoricalBudget.find` → `user.can_view?(doc)` → "link or nil", annotated "knows the user and what they may see". **Frontend**: two outcomes — "citation with link (with permission)" and "text-only citation (without permission)". Caption: "The AI service emits no URLs and assumes no visibility; each layer does its part and access control is respected.")*

If the user does not have permission, the link is simply not offered, and the citation stays in its verifiable textual form — document, year, line — without exposing a resource they should not see. **Verifiability and access control do not contradict each other; each layer does its part.**

## Honest trade-offs

**Citing at line level is a promise paid for at ingestion.** All the elegance of the `locator` depends on somebody, earlier, having stored the exact provenance of each fragment. If your corpus does not have it, do not improvise line citations over data that does not support them: cite at document level and be honest about the granularity. **An invented line citation is worse than a sincere document citation.**

**Structured citation is extra work the model sometimes skips.** Forcing the model to emit, for each component, the correct identifiers from the context is a constraint it satisfies worse the longer the generation is. The integrity check is the net that catches those failures, but it is worth assuming they will exist and designing the response policy before seeing them in production, not after.

**The link to the original is the best verification and the most fragile.** One click to the source is what really closes the trust loop, but it depends on the documents still existing, on the routes being stable, and on access control working. A broken link — or worse, a link that exposes a confidential budget to the wrong person — does more damage than having no link. If you cannot guarantee persistence and permissions, the verifiable textual citation is a dignified option.

**Too much citation tires the reader and stops being read.** If every number in the estimate drags three markers, the reader stops looking at them, and a citation nobody reads verifies nothing. Citing well also means deciding the grain: **component level is usually the right balance** between rigour and legibility, with line-by-line detail reserved for when somebody wants to audit.

## What this leaves unresolved

With all this, the estimate has citations that resolve, that locate the line and that are traceable to the original budget with each layer's access control in its place. Referential integrity guarantees that no claim cites a source that does not exist.

And yet, there is a crack none of these checks closes. Referential integrity confirms that `fin-2024-07#c3` was in the context and resolves to a real budget. **It does not confirm that budget says "40h for payments".** The model could have cited, with a perfectly valid identifier, a fragment that actually talks about something else, or attributed to it a figure that does not appear in it. The citation would be impeccable — well formed, resolvable, with its link — and the attribution would be false.

**A citation that points at a real source which does not back it is a hallucination with an alibi.** Detecting it is no longer checking identifiers: it is checking that the source's content genuinely sustains what the estimate claims. That is a different, deeper problem, and it is where the system's last layer of trust is played out.
