---
title: Enterprise data audit and inventory
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
series_part: 2
scope: evergreen
source: user-supplied article (prerequisite for articles 3 and 4)
reading_time: 27 min
added: 2026-08-11
summary: >
  Argues the audit *is* the work, not a preliminary to it. Builds a factual
  source census, scores each source on four non-averaging quality dimensions,
  names "context erosion" as the reason lineage matters in RAG specifically, and
  materialises all of it as a versioned, typed data_catalog.yaml that the
  ingestion pipeline reads at runtime.
keywords: [data audit, source inventory, data catalog, lineage, context erosion,
           data quality, deliberate exclusion, provenance, governance]
---

# Enterprise data audit and inventory

*Antonio Perez* · 🔴 27 min

The architectural decision is made: we are going to build RAG with a residual CAG layer. The natural temptation at this point is to write the first loader, install pgvector and start seeing vectors. We are going to resist that temptation for one more article.

Imagine the team hands you what they have. There's a Drive folder with historical budgets in JSON. There's a Dropbox directory with meeting transcripts accumulated over three years, some with descriptive filenames and others called `meeting_final_v3_FINAL.txt`. There's a Git repository with proposal templates in DOCX. There's a master Excel file where someone wrote down the official rates two years ago and nobody is sure whether it's still current. There's an internal API that returns the list of ongoing projects, but it's documented on a Confluence page from a year and a half ago that hasn't been touched since. Where do you start?

This article is the answer to that question. The answer, worth saying up front, is not "start vectorising". It is "build the inventory before touching anything".

## The antipattern: vectorise first, look later

The senior engineer's professional reflex, when handed a pile of data, is to get to work. There is a cultural bias in favour of visible action: a first version that works, however bad, is valued more than a week invested in taking inventory. The problem is that in RAG that reflex has an asymmetric cost. The errors from skipping the audit don't appear on day 1; they appear on day 60, when there is already a pipeline built on assumptions nobody verified.

The antipattern's typical failure modes are three. The first is **silent version mixing**: the corpus contains two contradictory versions of the same budget (the initial proposal and the signed one with changes) and the RAG retrieves the wrong one because there is no metadata distinguishing them. The second is **rotten sources**: documents that look valid but contain obsolete information (policies that changed, rates that were updated, clients who are no longer clients) and which generate confident answers to questions where the truth is the opposite of what the system says. The third is **invisible gaps**: the system answers well for the queries it has data for, but generates plausible and false information for those it doesn't, because nobody was able to tell the user that that category of question isn't covered by the corpus.

All three failure modes have the same root: the team never sat down to look at what was there before processing it. The audit is not a formality preceding the real work; **it is the real work**, and the cost of skipping it shows up amplified in production.

## Source inventory: the census of what you have

The first operational step is building a census. A census is exactly what it sounds like: a list of which sources exist, where they live, who maintains them, what format they have and what volume they occupy. It is factual, verifiable information — not opinion.

The minimum fields the census must capture per source are:

- **Logical name**: a stable identifier you'll use internally (`historical_budgets`, not "the old budgets on Drive").
- **Physical location**: the exact path or URL where the source lives, including the storage system (Drive, Dropbox, S3, database).
- **Technical owner**: the person or team responsible for the source existing and being accessible. This is who you call when the source goes down.
- **Business owner**: the person or team responsible for the content. This is who you ask when the content is ambiguous or needs interpretation.
- **Physical format**: JSON, CSV, PDF, DOCX, TXT, database row, API response.
- **Current volume**: approximate number of records and size on disk.
- **Access method**: FTP, API, manual download, SQL query.
- **Declared periodicity**: how often the source officially changes.
- **Observed periodicity**: how often it actually changes (which does not always match the declared one).

That last pair (declared vs observed) usually reveals problems the team had never looked at. A source that officially updates monthly but whose last modification is seven months old is not a monthly source; it is an abandoned source that someone still thinks is alive. That information is decisive before putting its contents into the RAG.

The census isn't done from memory; it's done by running an inspection script over the sources and manually filling in the fields the script can't deduce. For a filesystem source, for example:

```python
from pathlib import Path
from datetime import datetime
from dataclasses import dataclass

@dataclass
class FilesystemSourceFacts:
    name: str
    path: Path
    file_count: int
    total_size_mb: float
    latest_modified: datetime
    formats_detected: set[str]

def inspect_filesystem_source(name: str, root: Path) -> FilesystemSourceFacts:
    """Collect verifiable facts about a filesystem-based source.

    Subjective fields (owner, sensitivity, reliability) are left to
    the human reviewer; this function only reports what can be
    measured directly from disk.
    """
    files = [f for f in root.rglob("*") if f.is_file()]
    total_bytes = sum(f.stat().st_size for f in files)
    latest_ts = max((f.stat().st_mtime for f in files), default=0)
    formats = {f.suffix.lower().lstrip(".") for f in files if f.suffix}

    return FilesystemSourceFacts(
        name=name,
        path=root,
        file_count=len(files),
        total_size_mb=round(total_bytes / (1024 * 1024), 2),
        latest_modified=datetime.fromtimestamp(latest_ts),
        formats_detected=formats,
    )
```

The script is not sophisticated. Its value lies not in what it does but in the fact that it produces factual, comparable information across sources, without opinion. The rest of the audit is built on that base.

## Quality assessment by dimension

Once you have the census, it's time to assess each source's quality. And here it's worth distinguishing between "abundant data" and "good data", because they are not the same. A source with a hundred thousand records can be useless for RAG if those records are inconsistent with each other. A source with fifty records can be gold if they are well structured, validated and traced.

The assessment is articulated across four dimensions, each scored on a simple scale (I use 1-5 by convention, but any consistent scale works):

**Completeness.** How many records have all the expected fields? If the source declares a schema (explicit or implicit), what percentage satisfies it? In the case of JSON budgets: how many have the `total_amount` field filled in correctly, how many have it as a string instead of a number, how many have it empty?

**Consistency.** Is the same concept represented the same way throughout the source? If the same client appears in thirty documents, is it named identically in all thirty? If there's a `currency` field, does it take the same normalised values (`EUR`, `USD`) or do variants appear (`euros`, `€`, `EUR`, `eur`)? Inconsistencies are poison for chunking and retrieval: two fragments about the same client can end up in different regions of the vector space because the name is spelled differently.

**Actuality.** What date does the last relevant datum have? Is the declared periodicity met in practice? A source with a `last_modified` from two years ago, even if it still exists, is probably not a live source. Deciding whether it's worth vectorising is an architectural decision: there are cases where old data is exactly what the RAG needs (historical precedent), and others where it is pure contamination (obsolete policies).

**Reliability.** Is the source authoritative or derived? A spreadsheet someone fills in by hand each quarter is less reliable than the output of a transactional system with validation. Derived sources (extracts, summaries, copies) have the added problem of falling out of date relative to their origin without anyone noticing.

These four dimensions materialise in a simple structure filled in by manual inspection of each source, aided by the automatic census:

```python
from dataclasses import dataclass
from enum import IntEnum

class QualityScore(IntEnum):
    UNUSABLE = 1
    POOR = 2
    ACCEPTABLE = 3
    GOOD = 4
    EXCELLENT = 5

@dataclass
class QualityAssessment:
    completeness: QualityScore
    consistency: QualityScore
    actuality: QualityScore
    reliability: QualityScore
    notes: str

    @property
    def is_rag_ready(self) -> bool:
        """A source is RAG-ready when no dimension drags below acceptable.

        A single dimension at 1 or 2 typically poisons retrieval even if
        the others are excellent.
        """
        return all(
            score >= QualityScore.ACCEPTABLE
            for score in (
                self.completeness,
                self.consistency,
                self.actuality,
                self.reliability,
            )
        )
```

The `is_rag_ready` rule is deliberately strict. Someone familiar with metrics usually wants to average the dimensions, but the average deceives: a source with `completeness=5` and `reliability=1` is not a "quality 3" source; it is a source whose data is complete but may be a lie, and that is exactly the worst case for RAG. The dimensions do not compensate for each other; **each one is a necessary condition**.

## Lineage and context erosion

There is a fifth criterion that doesn't fit the rubric above because it operates on another plane. It is called **lineage**, and its importance for RAG is hard to overstate.

A datum's lineage is the trail of its origin and its transformations: where it originally came from, which processes modified it, who last touched it and when. In business intelligence systems lineage is a good practice; in RAG it is **a condition of the system's usefulness**.

The reason is direct. When a RAG system retrieves a fragment and presents it as evidence justifying an answer, the user needs to be able to verify the provenance. If the system says "this similar project cost €80,000" and the answer has to defend that figure to a client, three questions need an immediate answer: which specific document does this datum come from? when was that document generated? what level of authority does it have (initial budget, revised proposal, signed contract)? A RAG that cannot answer those three questions for every retrieved chunk is a RAG that is unusable for serious cases.

The opposite phenomenon, which destroys lineage's usefulness, also has a name: **context erosion**. It is the progressive loss of context about a datum as it moves between systems. The budget signed by the commercial director on 15 March 2024 leaves its document management system with all its metadata intact, moves to a Drive folder where it becomes `presupuesto_v3.pdf`, someone downloads it and renames it to `cliente_acme_2024.pdf`, someone else processes it with a script that extracts only the text and produces `cliente_acme.txt`, and by the time it reaches the RAG's ingestion pipeline the system has no way of knowing whether it was an initial proposal or a signed contract. The information is still there, but the context that made it interpretable has evaporated.

> *(Figure in the original: `sesion_06_article_2_visual_1_context_erosion.jpg` — image not included in this repo.)*

Fighting context erosion is the fundamental reason the catalog is a necessary artefact. The catalog is the place where all the context that would otherwise be lost — if you trusted only filenames and each source's loose metadata — is preserved by construction and in versioned form.

## The minimum viable catalog as versioned YAML

All the information we've been collecting (factual census, quality assessment, inclusion decisions, lineage notes) has to live somewhere. My recommendation, for programmes like ours and medium-sized projects, is: in the repository itself, as versioned YAML.

The catalog's structure is simple, and deliberately flat:

```yaml
# data_catalog.yaml
version: 1
last_audited: "2026-05-15"

sources:
  - name: historical_budgets
    description: >
      Closed project budgets since 2020 stored as JSON.
      Source of truth for similar-project retrieval in the estimation system.
    location: drive://AI-Eng/budgets/
    owner_technical: data-platform@company.com
    owner_business: ops-lead@company.com
    format: json
    volume:
      records: 80
      size_mb: 12.4
    refresh:
      declared: monthly
      observed_last_update: "2026-04-15"
      observed_lag_days: 3
    quality:
      completeness: 4
      consistency: 3
      actuality: 5
      reliability: 5
    sensitivity:
      contains_pii: true
      pii_types: [client_names, hourly_rates, internal_margins]
      access_restrictions: internal-only
    lineage:
      upstream: erp-finance-module
      transformations: [export_to_json_quarterly]
    decision: include
    notes: >
      Budgets prior to 2022 use a legacy schema. Filter them out during
      ingestion until the migration script is rerun. Tracked in JIRA-1421.

  - name: meeting_transcripts
    description: >
      Verbatim transcripts of client kickoff and scoping meetings.
    location: dropbox://AI-Eng/transcripts/
    owner_technical: it-ops@company.com
    owner_business: pre-sales@company.com
    format: txt
    volume:
      records: 142
      size_mb: 38.7
    refresh:
      declared: weekly
      observed_last_update: "2026-05-12"
      observed_lag_days: 2
    quality:
      completeness: 5
      consistency: 2
      actuality: 5
      reliability: 4
    sensitivity:
      contains_pii: true
      pii_types: [client_names, personal_names, emails, phone_numbers]
      access_restrictions: internal-only
    lineage:
      upstream: automated-transcription-service
      transformations: [speech_to_text, manual_review]
    decision: review
    notes: >
      Inconsistent format across years: pre-2024 transcripts have no
      speaker tags. Review needed before deciding whether to include
      the legacy block.

  - name: official_rate_card
    description: >
      Official hourly rates per role and seniority.
    location: drive://AI-Eng/rates/rate_card_2024.xlsx
    owner_technical: finance@company.com
    owner_business: cfo-office@company.com
    format: xlsx
    volume:
      records: 1
      size_mb: 0.3
    refresh:
      declared: yearly
      observed_last_update: "2024-01-12"
      observed_lag_days: 480
    quality:
      completeness: 5
      consistency: 5
      actuality: 1
      reliability: 5
    sensitivity:
      contains_pii: false
    lineage:
      upstream: finance-spreadsheet-manual
      transformations: []
    decision: exclude
    notes: >
      Officially the source of truth for rates, but last update is from
      January 2024. CFO office confirms it does not reflect 2025 changes.
      Excluded until a refreshed version is provided.
```

The catalog is more than documentation: it is **a software artefact**. Any code touching the project's sources should read this YAML at startup to know what to process, what to exclude, and what metadata to propagate to each chunk. That means the catalog also needs a typed loader:

```python
from pathlib import Path
from enum import Enum
from typing import Optional
from pydantic import BaseModel, Field
import yaml

class IngestionDecision(str, Enum):
    INCLUDE = "include"
    EXCLUDE = "exclude"
    REVIEW = "review"

class Volume(BaseModel):
    records: int
    size_mb: float

class Refresh(BaseModel):
    declared: str
    observed_last_update: str
    observed_lag_days: int

class Quality(BaseModel):
    completeness: int = Field(ge=1, le=5)
    consistency: int = Field(ge=1, le=5)
    actuality: int = Field(ge=1, le=5)
    reliability: int = Field(ge=1, le=5)

class Sensitivity(BaseModel):
    contains_pii: bool
    pii_types: list[str] = []
    access_restrictions: Optional[str] = None

class Lineage(BaseModel):
    upstream: str
    transformations: list[str] = []

class CatalogSource(BaseModel):
    name: str
    description: str
    location: str
    owner_technical: str
    owner_business: str
    format: str
    volume: Volume
    refresh: Refresh
    quality: Quality
    sensitivity: Sensitivity
    lineage: Lineage
    decision: IngestionDecision
    notes: Optional[str] = None

class DataCatalog(BaseModel):
    version: int
    last_audited: str
    sources: list[CatalogSource]

    def included_sources(self) -> list[CatalogSource]:
        return [s for s in self.sources if s.decision == IngestionDecision.INCLUDE]

def load_catalog(path: Path) -> DataCatalog:
    """Load and validate the data catalog from disk."""
    with open(path, "r", encoding="utf-8") as f:
        raw = yaml.safe_load(f)
    return DataCatalog(**raw)
```

Three advantages of having the catalog as typed code rather than a dead document. First, **automatic validation**: a PR that breaks the catalog's schema doesn't reach production. Second, **explicit coupling**: the ingestion pipeline iterates over `catalog.included_sources()` and propagates `lineage.upstream` and `sensitivity` as metadata on each chunk, with no manual intervention. Third, **change traceability**: the YAML's git log is the history of how Project 2's corpus has evolved, including who decided to include or exclude each source and why.

> *(Figure in the original: `sesion_06_article_2_visual_2_catalog_pipeline.jpg` — image not included in this repo.)*

## The audit report as a deliverable

One more piece sits on top of the catalog, optional but recommended: an audit report generated automatically, which serves to communicate the corpus's state to people who don't read YAML. A product team, an executive committee, a client asking to see which sources feed the system. The report is built in six or seven lines, reading the catalog and formatting it to Markdown:

```python
def generate_audit_report(catalog: DataCatalog) -> str:
    """Generate a human-readable audit report from the catalog."""
    included = catalog.included_sources()
    excluded = [s for s in catalog.sources if s.decision == IngestionDecision.EXCLUDE]
    review = [s for s in catalog.sources if s.decision == IngestionDecision.REVIEW]

    lines = [
        f"# Data audit report — {catalog.last_audited}",
        f"\n**Sources audited:** {len(catalog.sources)} | "
        f"**included:** {len(included)} | "
        f"**excluded:** {len(excluded)} | "
        f"**under review:** {len(review)}\n",
        "## Included sources\n",
    ]
    for s in included:
        lines.append(
            f"- **{s.name}** ({s.format}, {s.volume.records} records) — "
            f"owner: {s.owner_business}, last update: {s.refresh.observed_last_update}"
        )
    if excluded:
        lines.append("\n## Excluded sources\n")
        for s in excluded:
            lines.append(f"- **{s.name}** — reason: {s.notes or 'see catalog'}")
    return "\n".join(lines)
```

Generating this report every time the catalog changes (in CI, ideally) is what turns the audit into a living practice rather than a one-off deliverable. The catalog is not a document written at the beginning and forgotten; it is an artefact that breathes with the project.

## Honest trade-offs

**Formal catalog vs YAML in the repo.** There are professional cataloguing platforms on the market: Atlan, DataHub, Collibra, Microsoft Purview. They do far more than what we've described here: automatic source discovery, column-level lineage, integration with governance systems, business glossaries. They make sense in environments with hundreds of sources and dedicated data governance teams. For a project the size of Project 2 (one or two dozen sources, a small team), the operational overhead of maintaining such a tool far exceeds the value it brings. Versioned YAML in the repo is the right option until the number of sources starts growing seriously, or a regulatory mandate appears requiring a certified tool. When that happens, migrating from YAML to a professional platform is relatively trivial; jumping straight to the platform without having gone through the YAML usually results in an expensive, empty tool.

**Exhaustive audit vs enough to start.** The meticulous engineer's temptation is to document everything perfectly before processing anything. The problem is that sources are moving targets: what you document today will be out of date in two months. The initial audit should cover only the sources you're going to include in the system's first release, not every one that exists in the organisation. Additional sources are incorporated into the catalog at the moment it's decided to include them, not before. **The catalog grows with the project, not ahead of it.**

**Deciding what to leave out deliberately.** There's a common fallacy among people coming from classical data science projects: thinking that every available source must be used. In RAG the fallacy is especially dangerous because bad sources don't manifest as random noise (that would be easy to detect) but as **confident answers containing incorrect information**. Excluding sources deliberately, leaving in the catalog the written record of why they were excluded, is a professional hygiene practice worth normalising from the start. The `rate_card_2024.xlsx` in the earlier example is the typical case: officially it's the source of truth for rates, but it's so out of date that including it would introduce systematic errors. The decision to exclude it isn't laziness; it's architectural discipline.
