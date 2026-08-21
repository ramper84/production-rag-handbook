---
title: Data cleaning, normalization and validation
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
series_part: 4
scope: evergreen
source: user-supplied article (follows "Article 3 — multi-format extraction pipeline")
added: 2026-08-11
summary: >
  Distinguishes the *form* contract (Pydantic: the object is well-shaped) from the
  *content* contract (the values are actually usable). Names four recurring
  families of dirty data, argues cleaning must be one auditable layer rather than
  patches spread across chunker/embedder/retriever, and uses Pandera plus an
  explicit repair / quarantine / discard policy to enforce it.
keywords: [data quality, normalization, validation, pandera, deduplication,
           disguised nulls, quarantine, data contract, observability]
---

# Data cleaning, normalization and validation

*Antonio Perez*

In the previous article we closed out the `ingest/` subsystem that produces canonical `Document` objects from any format. Those `Document`s satisfy the Pydantic contract: they have a non-empty `content` and a `metadata` with the required fields. The contract is necessary, but it is only the **contract of form**. It says nothing about the **contract of content**.

Two `Document`s can satisfy the Pydantic contract perfectly and at the same time be radically incompatible for the RAG. A budget with `client_name: "ACME Corp."` and another with `client_name: "Acme Corp"` are individually valid, but when they enter the vector space the embeddings treat them as distinct entities and retrieval fails silently. A field `total_amount: -50000` passes type validation (it is a number) and breaks any downstream arithmetic analysis. A date stored as the string `"15/03/2024"` and another as `"2024-03-15"` are both strings, both valid, and radically different for any temporal filtering in retrieval.

The Pydantic contract is the first line of defence, but it is not the last. This article builds the second: the cleaning and validation layer that guarantees the contract of content before the data reaches the embedding.

## Four families of "dirt" in RAG data

Before choosing tools it's worth having a shared vocabulary for the problems we're going to solve. In enterprise RAG systems there are four recurring families of dirt that deserve their own names, because each damages the system in a specific way.

> *(Figure in the original: `sesion_06_article_4_visual_1_four_families.jpg` — image not included in this repo.)*

**Format heterogeneity.** The same thing written N different ways. Dates as `15/03/2024`, `2024-03-15`, `March 15, 2024` and `15-Mar-24` coexisting in the same corpus. Currencies as `EUR`, `eur`, `€`, `euros`. Client identifiers as `ACME`, `Acme Corp.`, `Acme Corp`, `acme-corp`. For a RAG system this is poison, because embeddings treat each variant as a different token: two fragments about the same client end up in distant regions of the vector space, and retrieval loses precisely the signal the team assumed was obvious.

**Duplicates with divergences.** The same record exists twice with different values in some field. A budget is in the ERP's exported JSON with `total: 80000` and in the manager's manual copy with `total: 82500` (because someone made an adjustment by hand and nobody synchronised). If both enter the RAG, the system will retrieve whichever the chunker indexed first, with no way of knowing which is the correct version. The answer given to the user will depend on a processing order the team does not control. Diagnosing this failure mode in production can take weeks, because the system does not break; it simply gives inconsistent answers that look like "LLM noise".

**Disguised nulls.** Fields that look filled in but contain no information. The string `"N/A"`, `"-"`, `"unknown"`, `"TBD"`, `"pendiente"`, an empty string, a single blank space. These values are technically valid (they are strings), they pass any type validation, and on reaching the embedding pipeline they get vectorised as though they were real content. The result is that the RAG learns to "retrieve" these values as if they were meaningful, and presents them to the user with the same authority as real content.

**Out-of-range values.** Budgets with `total: -50000`, end dates earlier than start dates, percentages above 100, `hours_estimated` fields with values in the millions. Most of these errors originate in manual transcription or in bugs in upstream systems the RAG team does not control. Their impact on the RAG is asymmetric: they are rarely retrieved (embeddings isolate them), but when they are retrieved they generate high-confidence answers about absurd claims — and those are what cost stakeholders most in system credibility.

The four families are fought with different techniques, but all of them require the same architectural decision: **a single point in the pipeline where the rules are applied**, with an explicit contract of what passes and what does not.

## Where to put the cleaning layer

The natural temptation on discovering these problems is to solve them where you find them: the chunker detects an empty field, the embedder finds a duplicate, the retriever filters out-of-range values. Each layer patches as it goes.

That is exactly what not to do.

When cleaning is spread across the chunker, the embedder and the retriever, three predictable things happen. The first is that the rules stop being auditable: there is no single place to say "these are our data invariants" — they are scattered across loose conditionals in three modules. The second is that tests become impossible: testing the chunker requires mocking validations that actually belong to another layer. The third, and the most expensive, is that the system ends up with no single point where a failure can stop it: a malformed record slips into the chunker, survives the embedder, and shows up in retrieval with visible consequences six weeks later.

The cleaning layer has to be a **separate module** in the pipeline, with its own clear location, its own tests, and its own formal guarantees. Recalling the architecture from Article 3 (loaders → parsers → normalizers → `Document`), the natural position is **between the parser and the normalizer**: the parser produces its intermediate representation, that representation passes through the cleaning layer, and only then is it converted into the canonical `Document`. For tabular formats (Project 2's JSON budgets are the paradigmatic example), the intermediate representation is naturally a pandas DataFrame, and the cleaning layer operates on it.

This does not mean non-tabular formats (PDF, DOCX, TXT) escape validation. Their intermediate representations are lists of elements or text, and different techniques apply to them (regex for expected formats, encoding validation, minimum length, placeholder detection). But the case that best illuminates the pattern is the tabular one, and it is where Pandera has its highest value, so that is where I will go deeper.

## Cleaning with pandas over the intermediate representation

Project 2's JSON budget parser produces a DataFrame with columns like `budget_id`, `client_name`, `total_amount`, `currency`, `signed_at`, `status`. Before generating the `Document`s, that DataFrame passes through a sequence of transformations that normalise formats and remove corrupt records:

```python
import pandas as pd
import hashlib
from datetime import datetime

NULL_PLACEHOLDERS = {"", "n/a", "na", "-", "--", "unknown", "tbd", "pendiente"}

def clean_budget_records(df: pd.DataFrame) -> pd.DataFrame:
    """Apply canonical cleaning operations to budget records.

    Each step is intentionally narrow and side-effect free so it can be
    unit-tested in isolation. The order matters: nulls before dedup,
    formatting before validation downstream.
    """
    out = df.copy()

    # 1. Disguised nulls -> real NaN
    out["client_name"] = (
        out["client_name"].astype(str).str.strip().str.lower()
        .where(lambda s: ~s.isin(NULL_PLACEHOLDERS), other=pd.NA)
    )

    # 2. Currency normalization
    currency_map = {"eur": "EUR", "euros": "EUR", "€": "EUR",
                    "usd": "USD", "$": "USD"}
    out["currency"] = (
        out["currency"].astype(str).str.strip().str.lower()
        .map(currency_map).fillna(out["currency"])
    )

    # 3. Date parsing with explicit fallback
    out["signed_at"] = pd.to_datetime(
        out["signed_at"], errors="coerce", utc=True
    )

    # 4. Numeric coercion for amounts (string "80000" -> 80000.0)
    out["total_amount"] = pd.to_numeric(out["total_amount"], errors="coerce")

    # 5. Content-hash dedup: keep the latest version per budget_id
    out["content_hash"] = out.apply(
        lambda r: hashlib.sha256(
            f"{r['budget_id']}|{r['total_amount']}|{r['currency']}".encode()
        ).hexdigest(),
        axis=1,
    )
    out = out.sort_values("signed_at").drop_duplicates(
        subset=["budget_id"], keep="last"
    )

    return out.drop(columns=["content_hash"])
```

Three design details deserve comment. First, each step is narrowly scoped: it does one thing and moves on. This makes each transformation easy to test in isolation and also makes the result easy to reason about: if something goes wrong, the list of suspects is short. Second, step 5 (dedup by content hash) attacks the "duplicates with divergences" family specifically: if the same `budget_id` appears twice with different values, the hashes differ, and the rule "keep the last one by `signed_at`" decides which survives. The rule is debatable and must be a conscious team decision documented in the catalog. Third, the permissive coercions (`errors="coerce"`) turn invalid values into `NaN` instead of raising. This separates cleaning from validation: here we transform what can be transformed; the later validation decides what to do with the resulting `NaN`s.

This function is the first half. It is where the normalization rules are applied. But it decides nothing: it leaves records with empty fields, out-of-range values, and unparseable dates as `NaT`. Deciding what happens to those records belongs to the validation layer.

## Pandera as a data contract

Pandera is a dataframe validation library that plays the same role for pandas (and polars, dask, etc.) that Pydantic plays for individual instances. Where Pydantic says "this object satisfies this schema or raises", Pandera says "this DataFrame satisfies this schema column by column and row by row, or produces a detailed report of which rows fail and why".

For Project 2, the canonical budget schema in Pandera looks like this:

```python
import pandera.pandas as pa
from pandera.pandas import DataFrameModel, Field, Check
from pandera.typing.pandas import Series
import pandas as pd
from datetime import datetime, timezone

class BudgetRecord(DataFrameModel):
    """Canonical contract for budget records before normalization to Document.

    Any DataFrame produced by the budget cleaning pipeline must validate
    against this schema. Records that fail validation are routed to
    quarantine or discarded according to severity.
    """
    budget_id: Series[str] = Field(
        str_matches=r"^BUDGET-\d{4}-\d{4}$",
        description="Stable budget identifier in canonical format",
    )
    client_name: Series[str] = Field(
        nullable=False,
        str_length={"min_value": 2, "max_value": 200},
    )
    total_amount: Series[float] = Field(
        ge=0,
        le=10_000_000,
        nullable=False,
        description="Total in declared currency, non-negative",
    )
    currency: Series[str] = Field(isin=["EUR", "USD", "GBP"])
    signed_at: Series[pd.Timestamp] = Field(
        nullable=False,
        le=datetime.now(timezone.utc),
        description="Sign date must be in the past",
    )
    status: Series[str] = Field(isin=["draft", "signed", "rejected"])

    class Config:
        strict = True            # reject unknown columns
        coerce = False           # cleaning has already coerced types
        ordered = False

    @pa.dataframe_check
    def positive_amount_for_signed(cls, df: pd.DataFrame) -> Series[bool]:
        """Signed budgets must have positive amounts.

        Cross-column rule: status='signed' implies total_amount > 0.
        """
        return ~((df["status"] == "signed") & (df["total_amount"] == 0))
```

The conceptual difference from Pydantic is the dimension they validate over. Pydantic validates one instance and returns success or an exception. Pandera validates an entire DataFrame, and on failure returns a `SchemaErrors` object detailing which rows failed, in which columns, and why. That is exactly the information the next layer needs in order to decide between repairing, quarantining or discarding.

Three elements of the schema above are worth highlighting. The first is the field checks: `ge=0`, `le=10_000_000`, `str_matches=r"^BUDGET-\d{4}-\d{4}$"`. Each expresses a business invariant the team has committed to respecting. The shape of `budget_id` is not an aesthetic detail; it is a contract with the upstream system. The second is the cross-column checks via `@pa.dataframe_check`: rules relating several columns, such as "if the status is signed, the total cannot be zero". These rules catch the subtlest inconsistencies, and they are the ones a Pydantic-only schema cannot express. The third is the schema configuration: `strict=True` rejects undeclared columns (a defence against silent parser changes), and `coerce=False` assumes the prior cleaning already did the coercions (a clean separation of responsibilities).

The Pandera contract is a living piece of the project. When the business decides to accept a new currency, when a new budget status is introduced, when the project maximum changes, the change is made in this file and only in this file, is versioned in git, and the whole downstream pipeline respects it automatically. It is, for data, the equivalent of `data_catalog.yaml` for sources.

## The failure strategy: repair, quarantine, discard

When a row fails validation there are three possible responses, and the decision depends on the type of failure. The policy has to be explicit and documented, not implicit in the code.

**Repair automatically** when the failure is recoverable without semantic loss. A date in the format `"15/03/2024"` that `pd.to_datetime` did not parse with the expected format but does parse with `dayfirst=True`. A `currency` value of `"euros"` that was not in the map but clearly belongs to `"EUR"`. These cases are resolved with an additional cleaning pass specific to the detected errors, with no human intervention.

**Quarantine** when the failure is serious but the record could be useful after review. A `client_name` that is null when the rest of the record is complete; a `total_amount` slightly above the maximum that could be a legitimately exceptional project or a typo. These records do not enter the RAG, but are preserved in a separate table along with their quarantine reason, accessible so a human can decide later. It is the equivalent of limbo: neither in nor out, awaiting arbitration.

**Discard** when the failure indicates clear contamination and adds no value. A record with a `budget_id` that does not match the canonical pattern (probably an artefact of a botched migration); a `total_amount` that is negative or a hundred times the maximum. These records are removed with a detailed log but without reservation: they are beyond rescue and do not deserve to occupy space in quarantine.

Materialising this policy in code needs one more layer, the validation orchestrator:

```python
from dataclasses import dataclass
from typing import Literal
import pandera.pandas as pa
import pandas as pd
import logging

logger = logging.getLogger(__name__)

@dataclass
class ValidationResult:
    valid: pd.DataFrame
    quarantined: pd.DataFrame
    discarded: pd.DataFrame
    report: dict

QUARANTINE_REASONS = {
    "nullable_violation": "missing required field",
    "client_name_str_length": "client name length out of range",
}

def validate_with_policy(
    df: pd.DataFrame, schema: type[pa.DataFrameModel]
) -> ValidationResult:
    """Validate a dataframe and route failures by policy.

    Reparable failures are not handled here — assume the cleaning step
    has already attempted recovery. Anything still failing is either
    quarantined (recoverable with human review) or discarded.
    """
    try:
        valid = schema.validate(df, lazy=True)
        return ValidationResult(
            valid=valid, quarantined=pd.DataFrame(), discarded=pd.DataFrame(),
            report={"total": len(df), "valid": len(df)},
        )
    except pa.errors.SchemaErrors as exc:
        failure_cases = exc.failure_cases
        failed_indices = failure_cases["index"].dropna().unique()
        failed_rows = df.loc[failed_indices].copy()

        # Discard policy: structural failures that signal contamination
        is_discard = failure_cases["check"].isin([
            "str_matches", "ge(0)", "le(10000000)"
        ])
        discard_indices = (
            failure_cases.loc[is_discard, "index"].dropna().unique()
        )
        quarantine_indices = [
            i for i in failed_indices if i not in discard_indices
        ]

        valid_indices = df.index.difference(failed_indices)
        valid = df.loc[valid_indices]
        discarded = df.loc[discard_indices].copy()
        quarantined = df.loc[quarantine_indices].copy()

        logger.warning(
            "validation: valid=%d quarantined=%d discarded=%d",
            len(valid), len(quarantined), len(discarded),
        )

        return ValidationResult(
            valid=valid, quarantined=quarantined, discarded=discarded,
            report={
                "total": len(df),
                "valid": len(valid),
                "quarantined": len(quarantined),
                "discarded": len(discarded),
                "failure_breakdown": failure_cases["check"].value_counts().to_dict(),
            },
        )
```

Two design details deserve underlining. First, `lazy=True` in the call to `schema.validate()`: instead of failing on the first error, Pandera collects every error in the DataFrame and returns them together. This is what makes a policy differentiated by failure type possible; without `lazy=True` we would only know the first error and the policy would be blind. Second, the result always includes the `report`: the orchestrator not only decides what happens to the data, it also leaves metrics for observability. How many valid records passed, how many ended up in quarantine, which failure types predominate. That information is what will alert you when a source starts to degrade, long before the degradation reaches the RAG.

## Honest trade-offs

**Pandera vs Great Expectations.** Both are data validation libraries with different centres of gravity. Pandera is lightweight, embedded in Python code, declarative via `DataFrameModel`: your schema is a Python class that lives with your code and is versioned with it. Great Expectations is more ambitious: it offers data docs (auto-generated HTML documentation of the corpus state), automatic profiling, native integration with orchestrators like Airflow and Dagster, and an expectations model that is more expressive but also heavier to operate. For a project the size of Project 2 (one AI service, a small team, inline validation in the pipeline), Pandera is the right choice: zero infrastructure overhead, schemas that can be understood in five seconds and modified in five minutes. Great Expectations makes sense when the system scales to dozens of pipelines with non-technical stakeholders who need to see the state of data quality, or when there is a dedicated data engineering team to operate it. Before migrating from Pandera to Great Expectations, it's worth having concrete reasons; migrating by default usually means paying the complexity without needing the features.

**Strict mode in production vs permissiveness in development.** In development it's understandable that someone would want to relax the schema so that "everything passes while I iterate". The correct instinct is the opposite: the schema should be strict from day one, and what gets relaxed is the **failure policy**, not the contract. In development, quarantining instead of discarding lets you see the problematic data without the pipeline breaking; in production, discarding stops the corpus being contaminated. But the contract of what is valid and what is not stays the same, and it is exactly what was documented and tested in development. When the contract is lax in development, the problems show up on the day of the production deploy.

**How much to normalise without losing signal.** The temptation to normalise aggressively to reduce variants is real and dangerous. Lowercasing every client name resolves "ACME Corp" vs "acme corp" but deliberately erases the difference between "Apple" (the company) and "apple" (the fruit) if for some reason both appear in the corpus. Replacing all multiple spaces with one makes dedup easier but erases intentional structure in formatted transcripts. The heuristic I apply: **normalise with a scalpel, not a chainsaw.** Normalise first the cases where the heterogeneity is clearly accidental (upper/lowercase variants in currencies, trailing spaces, date separators) and leave for a second iteration (or never) the cases where normalisation could erase semantic signal.
