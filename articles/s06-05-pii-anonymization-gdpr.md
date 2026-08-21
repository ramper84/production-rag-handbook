---
title: PII, anonymization and GDPR in the ingest pipeline
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
series_part: 5
scope: evergreen
source: user-supplied article (follows "Article 4 — cleaning, normalization and validation")
reading_time: 28 min
added: 2026-08-11
summary: >
  Access control does not protect a RAG corpus: once a sensitive value is in the
  vector space, any semantically close question reaches it. Names three leakage
  modes (direct, aggregation, inference), the four GDPR concepts that drive
  technical decisions, and argues for consistent reversible pseudonymization
  (Presidio + Faker + a mapping table) over generic redaction, which measurably
  degrades retrieval.
keywords: [PII, GDPR, anonymization, pseudonymization, presidio, faker,
           right to be forgotten, mapping table, semantic leakage, data minimization]
---

# PII, anonymization and GDPR in the ingest pipeline

*Antonio Perez* · 🔴 28 min

The corpus we now have has been through inventory, extraction and validation. The records are consistent, clean, and respect the business invariants. But they contain, without exception, sensitive personal and commercial information. Client names in the transcripts, email addresses in budgets, phone numbers of contacts in proposals, internal project identifiers that reveal organisational structure, contractual terms that the legal department asked not to be shared.

There is a common intuition worth dismantling up front: many teams assume that access control at the system level (authentication, authorisation, application ACLs) is enough to protect that data. The intuition works in traditional databases. It does not work in RAG, and the reason is structural, not an implementation detail. Once a sensitive datum is in the vector space, it is available to any semantic query that comes close to it, regardless of how the original question was phrased. **Protection has to happen before the embedding**, and this article builds that layer.

## The real problem: semantic leakage via RAG

The difference between a PII leak in a relational database and a leak in a RAG system is the following. In the relational database, the attacker needs to formulate a specific query pointing at the sensitive table or column. If the `email` column of the `clients` table is protected by permissions, there is no SQL query that returns it, no matter how skilled the attacker.

In RAG the attack is indirect. The user does not query tables; they ask questions in natural language. The system searches semantically, retrieves the most relevant chunks, and presents them to the generative model as context. If the chunks contain the sensitive datum (literally, in the text), the model is going to use it in its answer. There is no permission level inside the vector that can hide it, because the vector does not know what is sensitive.

It's worth distinguishing three leakage modes in order to design the right defences.

> *(Figure in the original: `sesion_06_article_5_visual_1_pii_leakage_modes.jpg` — image not included in this repo.)*

**Direct leakage** is the most obvious. A user asks "which clients have hired us for cloud migration projects?" and the RAG returns an answer listing "Banco Sabadell, Inditex and Repsol" because those names are literally in the retrieved chunks. It is trivial to exploit and trivial to prevent if anonymization is in place.

**Leakage by aggregation** is subtler. Each individual query looks innocuous, but the attacker combines several to reconstruct sensitive information. "Which projects did we complete in 2024?", "Which was the most expensive?", "In which sector?", "Which technologies did we use?". Each question returns partial data; the attacker joins them into a complete picture. Defending against this requires thinking in terms of **aggregate information surface**, not individual chunks.

**Leakage by inference** is the most dangerous because it happens even after naive anonymization. If you replace the name "Juan García, CEO of Acme Corp" with "[PERSON], CEO of [ORG]", the information appears to have been protected. But the context surrounding the token is still there: the sector, the dates, the amounts, the geography. Combined with catalog and parser metadata, that can be enough for an attacker with domain knowledge to identify the individual. The defence against this mode is not just to anonymise, but to **reduce the combinatorics of clues surrounding the individual**.

The three attack families share one characteristic: none requires administrative access to the system. Legitimate user credentials and natural-language questions are enough. That is why anonymization has to happen before the embedding, not as a filter on the response.

## The minimum GDPR framework applied to the pipeline

GDPR is the European regulation governing the processing of personal data. Its full text covers a great deal, but there are four concepts any AI Engineer working with enterprise data in the EU needs internalised, because they condition concrete technical decisions in the pipeline.

**Personal data.** GDPR's definition is deliberately broad: any information that can identify, directly or indirectly, a natural person. Names, emails and phone numbers are obvious. Less obvious are the indirect identifiers: an IP address, a cookie, an employee number, and even combinations of data that individually do not identify but together do. For Project 2, that means meeting transcripts (with participants' names) are trivially personal data, but so can budgets be when they combine sector, amount, date and geography, if the combination narrows the candidate population to a single identifiable client.

**Anonymization vs pseudonymization.** The distinction is technical and legal at once. *Irreversible anonymization* means not even the system's operator can recover the original datum; the anonymised datum stops being "personal data" in the GDPR sense, and the regulation stops applying to it (with conditions). *Pseudonymization* means the real datum is substituted with a fictitious one via a reversible mapping kept separately; it remains personal data for GDPR purposes (the mapping table is personal information), but it is operationally simpler to manage. For RAG, pseudonymization usually wins because it preserves the corpus's semantic coherence: "Juan García" is not replaced with `<PERSON>` (which destroys structure), but with "Carlos Martínez" — always, across the whole corpus.

**Right to be forgotten.** Article 17 of GDPR gives any person the right to request that their personal data be deleted from a system. For a traditional database this is a `DELETE`. For RAG it is an architectural problem: the chunks mentioning the individual are vectorised and scattered across the index. Without an explicit mapping saying "these chunks contain information about Juan García", deletion is impossible. This is one of the reasons (not the only one) why the pseudonymization mapping table is an architectural piece, not an implementation detail.

**Minimization.** The principle says only the data strictly necessary for the declared purpose should be processed. For Project 2, this translates into an operational question: does the estimation system need the real names of clients to do its job? The reasonable answer is no: the system needs the patterns of past projects (sector, scope, technologies, complexity), not the client's specific identity. That observation is what justifies aggressive pseudonymization from the start: we are not losing anything useful, we are removing a risk.

## Microsoft Presidio: detection and anonymization in the pipeline

Presidio is Microsoft's library for PII detection and anonymization. Alternatives exist (pure spaCy, nltk, commercial solutions like AWS Comprehend), but Presidio has three characteristics that make it the practical choice for a pipeline like Project 2's: modular architecture (analyzer + anonymizer are interchangeable), pre-built recognizers for common PII (email, phone, IBAN, IPs, card numbers, dates, locations, people, organisations), and explicit support for custom recognizers that extend the catalog of detectable entities.

Basic usage has two steps. The analyzer detects PII entities in a text and returns their positions; the anonymizer applies a transformation operation over those positions.

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig
from presidio_analyzer.nlp_engine import NlpEngineProvider

# Spanish-aware NLP engine (default ships with English only)
nlp_config = {
    "nlp_engine_name": "spacy",
    "models": [{"lang_code": "es", "model_name": "es_core_news_md"}],
}
nlp_engine = NlpEngineProvider(nlp_configuration=nlp_config).create_engine()

analyzer = AnalyzerEngine(nlp_engine=nlp_engine, supported_languages=["es"])
anonymizer = AnonymizerEngine()

text = (
    "El cliente Juan García (CEO de Acme Corp) confirmó el presupuesto "
    "BUDGET-2024-0315 por correo a contacto@acme.com el 12 de marzo."
)

results = analyzer.analyze(text=text, language="es")
# results contains a list of RecognizerResult with start, end, entity_type, score

anonymized = anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators={"DEFAULT": OperatorConfig("replace", {"new_value": "[REDACTED]"})},
)
print(anonymized.text)
# "El cliente [REDACTED] (CEO de [REDACTED]) confirmó el presupuesto
#  [REDACTED] por correo a [REDACTED] el [REDACTED]."
```

Two details of the setup deserve attention. First, the explicit configuration of the Spanish spaCy model. By default, Presidio loads `en_core_web_lg` (English) and the recognizers are tuned for that language. Without this configuration, over Spanish text the false-negative rate on `PERSON` and `LOCATION` entities shoots up: the system fails to detect names that are obvious to a Spanish speaker. Second, `supported_languages=["es"]` is necessary so the analyzer does not try to load the English model as a fallback.

The `replace` operation with `[REDACTED]` is the simplest and the worst for RAG, for the reasons already discussed. We use it here only to demonstrate the flow. The strategy we apply to Project 2 is different and appears further on.

## Custom recognizers for Project 2's domain

The default recognizers cover the universal categories well: emails, phones, IBANs, generic names. They do not know the project domain's specific identifiers. For Project 2 there are at least two categories of proprietary identifiers worth detecting and handling:

- **Budget IDs**: the pattern `BUDGET-YYYY-NNNN` already documented as an invariant in Article 4's Pandera schema. These identifiers are not PII in the strict sense, but they reveal sensitive commercial information (volume of closed projects, internal numbering structure).
- **Internal client codes**: identifiers like `CLI-1042` or `CLT-INT-A047` that appear in transcripts and budgets, and that map one-to-one to real clients.

Presidio allows adding these recognizers with `PatternRecognizer`, which wraps one or more regular expressions:

```python
from presidio_analyzer import PatternRecognizer, Pattern

budget_id_pattern = Pattern(
    name="budget_id_canonical",
    regex=r"\bBUDGET-\d{4}-\d{4}\b",
    score=0.95,
)
budget_id_recognizer = PatternRecognizer(
    supported_entity="BUDGET_ID",
    name="budget_id_recognizer",
    patterns=[budget_id_pattern],
    supported_language="es",
)

client_code_pattern = Pattern(
    name="client_code_internal",
    regex=r"\b(?:CLI|CLT-INT)-[A-Z0-9]{3,8}\b",
    score=0.9,
)
client_code_recognizer = PatternRecognizer(
    supported_entity="CLIENT_CODE",
    name="client_code_recognizer",
    patterns=[client_code_pattern],
    supported_language="es",
)

analyzer.registry.add_recognizer(budget_id_recognizer)
analyzer.registry.add_recognizer(client_code_recognizer)
```

Two things to note. First, `score` is a decisive parameter: it indicates the confidence with which the recognizer claims to have detected the entity. When several recognizers overlap (for example, a generic "alphanumeric code" pattern and our specific `BUDGET-` pattern), Presidio keeps the one with the higher score. Starting with high scores (0.9-0.95) on very specific patterns and low ones (0.4-0.6) on generic patterns is good operational hygiene. Second, the `supported_entity` you declare (`BUDGET_ID`, `CLIENT_CODE`) is the semantic label you will use later in the pseudonymization phase to apply the right transformation. A budget ID is replaced with another fake but coherent budget ID; an email with another email; a name with another name. **The label is what decides which generator to use.**

For specific client names that follow no regular pattern (like "Banco Sabadell" or "Inditex"), there is a second, complementary tool: `RecognizerResult` loaded from an explicit dictionary. Maintaining that dictionary as part of the project (in the catalog, in a referenced table) is operationally trivial and catches the cases that neither the NLP nor the regexes detect.

## Reversible pseudonymization with Faker and a mapping table

Here comes the key architectural piece. Instead of replacing the detected entities with generic tokens (`[PERSON]`, `[EMAIL]`), we replace them with **consistent fictitious values** generated by Faker, maintaining in parallel a mapping table that records each substitution so it can be reversed when necessary.

> *(Figure in the original: `sesion_06_article_5_visual_2_pseudonymization_flow.jpg` — image not included in this repo.)*

```python
from dataclasses import dataclass
from typing import Optional
from faker import Faker
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

@dataclass
class PseudonymMapping:
    """Single mapping entry to be persisted to a secure store."""
    original_value: str
    pseudonym: str
    entity_type: str
    first_seen_at: str  # ISO timestamp
    source_name: str    # catalog source where it appeared first

class ConsistentPseudonymizer:
    """Replaces detected entities with stable fake values.

    The same input value always maps to the same fake output across
    the entire corpus, so semantic relations are preserved. The mapping
    is persisted to allow reverse lookup and right-to-be-forgotten.
    """

    def __init__(self, mapping_store, locale: str = "es_ES"):
        self.faker = Faker(locale)
        self.store = mapping_store  # backed by an encrypted store
        self.generators = {
            "PERSON": self.faker.name,
            "EMAIL_ADDRESS": self.faker.email,
            "PHONE_NUMBER": self.faker.phone_number,
            "LOCATION": self.faker.city,
            "ORGANIZATION": self.faker.company,
            "BUDGET_ID": lambda: f"BUDGET-{self.faker.year()}-{self.faker.random_number(digits=4, fix_len=True)}",
            "CLIENT_CODE": lambda: f"CLI-{self.faker.random_number(digits=4, fix_len=True)}",
        }

    def get_or_create_pseudonym(
        self, original: str, entity_type: str, source_name: str
    ) -> str:
        existing = self.store.lookup(original, entity_type)
        if existing:
            return existing.pseudonym

        generator = self.generators.get(entity_type, self.faker.word)
        pseudonym = generator()
        self.store.save(PseudonymMapping(
            original_value=original,
            pseudonym=pseudonym,
            entity_type=entity_type,
            first_seen_at=self._now_iso(),
            source_name=source_name,
        ))
        return pseudonym
```

Four elements of the design deserve comment. First, **consistency is per original value, not per chunk**. The same string "Juan García" is always pseudonymised to the same "Carlos Martínez" even if it appears in hundreds of different chunks. Without this, two chunks about the same client would end up in distant regions of the vector space and retrieval would break again for exactly the same reason "format heterogeneity" broke it in Article 4. Second, the generators are **specific per entity type**: a name is replaced with another name, not with an email or a date. The semantic signal of the field type is preserved. Third, the mapping store is a **separate component** of the pipeline, encrypted, with its own access control. If you have to demonstrate to a GDPR auditor that you know whose data lives in your system, the query goes against that store, not against the vector corpus. Fourth, `source_name` is persisted with the mapping, which makes it possible to answer "which catalog sources mention this person" without traversing the vector index.

Integration with the ingest pipeline happens at the end, after Article 4's validation and before Session 07's chunking. The orchestrator takes each validated `Document`, looks at `metadata.contains_pii` (propagated from the catalog in Article 3), and if it is `True` applies pseudonymization before handing it to the next stage. The `Document` that comes out has the same `content` as the one that went in except for the replaced tokens, and the rest of the downstream pipeline needs to know nothing about Presidio or Faker; it consumes `Document`s whose `content` is already safe.

## The right to be forgotten in RAG: a practical case

Anyone who reaches production with this system will sooner or later receive a right-to-be-forgotten request. A client or an employee says "I want my data no longer in your AI system". The operational question is: what steps does the team execute?

With the architecture we have described, the steps are five. First, query the mapping store with the requester's name to identify all associated pseudonyms. There may be several if the person appears in variants (full name, first name and surname, alias). Second, search the vector index for the chunks containing those pseudonyms, or associated with documents carrying those pseudonyms in metadata. Third, delete those chunks from the vector index. Fourth, delete the corresponding entries from the mapping store: the mapping ceases to exist, and if the person appears again in a new document they will receive a new pseudonym unrelated to the previous one. Fifth, record the operation in an audit log demonstrating that the request was handled within the required time and manner.

Each of these steps is operationally trivial thanks to the mapping table. Without it, steps one, two and four are impossible, and the system is in permanent breach of GDPR Article 17. That is why the mapping table is not a detail; it is the piece that holds compliance up.

## Honest trade-offs

**Irreversible anonymization vs reversible pseudonymization.** Some teams defend irreversible anonymization for simplicity: you substitute with `<PERSON>`, there is no mapping store, there is no risk of the mapping leaking, and it is no longer "personal data" under GDPR. The problem is that embeddings of a corpus with `<PERSON>` degrade significantly compared with a corpus with consistent pseudonyms. Internal tests (not publishable, but replicable) show drops of 15-25% in retrieval metrics when generic substitution is used. For production systems with a quality commitment, reversible pseudonymization is almost always the right answer. Irreversible anonymization is reserved for corpora the team understands as "public by default" (internal publications anonymised for external distribution, for example) or for cases where a legal contract with a provider explicitly requires it.

**Presidio false positives in Spanish.** Presidio performs considerably worse in Spanish than in English. The base NLP model (`es_core_news_md`) labels common words as `PERSON` entities with irritating frequency: words like "Mar", "Sol", "Cruz", "Alba" are detected as proper names even when they appear as common nouns. spaCy's Spanish training datasets have less volume and less diversity than the English ones, and it shows. Three strategies mitigate the problem: raise the score threshold for confirming entities (going from 0.5 to 0.7 reduces false positives at the cost of some false negatives), add a blacklist of known words that must not be treated as PII even when the model detects them, and train a lightly customised NER model with domain data if the corpus volume justifies it. The choice depends on the project's cost-benefit; for Project 2, the first two strategies are sufficient.

**Impact on embedding quality.** Although consistent pseudonymization preserves most of the semantic signal, it inevitably introduces some noise. A real name carries micro-information that a fake name does not replicate (geographic origin, perceived gender, frequency in the corpus). For a project-estimation RAG like Project 2's, that noise is negligible: retrieval works in terms of project patterns, not nominal identity. For systems where identity does matter (personal assistants, individualised client-relationship systems), the cost of pseudonymization is higher and it is worth quantifying with specific benchmarks before adopting it. The practical rule for Project 2: pseudonymise first, measure afterwards with a representative query set, and decide case by case whether some entity type (rarely) deserves to be left untreated.
