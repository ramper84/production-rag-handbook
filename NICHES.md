---
title: Five candidate niches for PLAYBOOK.md
doc_type: notes
scope: time-sensitive
added: 2026-09-06
summary: >
  Five project ideas to run through PLAYBOOK.md's architecture decision,
  chosen to be distinct from the repos already built from it (fantasy,
  ortodoncia, the estimator) and to land on different points of the
  decision — not five vector-RAG chatbots wearing different labels.
---

# Five candidate niches for PLAYBOOK.md

Each entry gives a predicted PLAYBOOK.md §2 outcome — a hypothesis to test
against real example queries once `examples/` has content, not a decision
made in advance of them (Axis 2 depends on actual queries, per §1). Treat the
prediction as a reason to pick the niche if you want practice on that part of
the decision tree, not as the answer already given.

## 1. Contract and compliance review for a solo/small legal practice

**What it does**: ingests contracts, clause libraries, and prior case
correspondence; answers questions like "what's our standard limitation-of-
liability clause" or "does this NDA's confidentiality term conflict with our
usual 3-year default," citing the exact source clause.

- **Source material**: PDFs (signed contracts, statutes), Markdown (internal
  clause playbooks, past opinions).
- **Example queries**: free-text, paraphrastic, over prose with no reliable
  ID to filter on — "find contracts where we accepted uncapped liability,"
  "what did we tell this client about indemnification last time."
- **Predicted axis**: Axis 3 — genuine vector RAG. This is the sharpest
  contrast to fantasy's weekly path: no query here names an exact entity, so
  Axis 2's SQL shortcut doesn't apply.
- **What it exercises**: citation-and-verifiable-attribution discipline
  (s11-03) is not optional in this domain — a hallucinated clause has real
  consequences — so this is a good niche for practicing Phase 12's guardrails
  and Phase 14's Critic seriously rather than skipping them.
- **MVP scale**: 1 user (the practicing lawyer) — resist the urge to build
  for a firm nobody has signed yet.

## 2. Rental portfolio management for a landlord or small property manager

**What it does**: answers "what's the maintenance history on unit 4B," "which
tenants are behind on rent as of this month," "show me every complaint filed
against the HVAC contractor" — grounded in leases, maintenance tickets, and
inspection photos.

- **Source material**: PDFs (leases, inspection reports), images (photos of
  reported damage), Markdown (maintenance logs if kept that way).
- **Example queries**: name their entities exactly — a unit number, a tenant,
  a date range. "What's unit 4B's maintenance history" is a `WHERE` clause,
  not a similarity search.
- **Predicted axis**: Axis 2 — SQL-typed retrieval, no embedding index. This
  is the niche to pick if you want deliberate practice *not* reaching for
  pgvector, mirroring fantasy's weekly path exactly.
- **What it exercises**: the intake census's freshness question (Section 1)
  has a real answer here — new maintenance tickets and inspections arrive
  continuously — so this is a natural fit for Phase 9's Scheduled Refresh,
  which most niches on this list will legitimately skip.
- **MVP scale**: 1 landlord or a handful of staff at a small management
  company — tens of units, not thousands.

## 3. Customer support knowledge base for a small e-commerce store

**What it does**: answers shopper and support-agent questions from the
product catalog, return policy, and shipping FAQ — "can I return a used item
after 40 days," "does this product ship to Puerto Rico."

- **Source material**: Markdown (policy pages, FAQ), PDFs (supplier spec
  sheets), possibly product images if visual questions matter ("does this
  come in the blue variant shown in the photo").
- **Example queries**: mixed — some paraphrastic ("can I still return this,"
  many phrasings of the same policy question), some near-static ("what's your
  return window" asked identically hundreds of times a day).
- **Predicted axis**: Axis 1 first — the return/shipping policy corpus is
  small and stable, a CAG candidate on its own. But the catalog is large and
  paraphrastic, which pulls toward Axis 3. Expect to land on **both**,
  composed as independent paths (policy Q&A vs. catalog search) rather than a
  conductor — good practice for Section 2's "Composing more than one answer."
- **What it exercises**: Part 10's advanced retrieval only pays off here if
  volume is real — a good niche for practicing Phase 16's "skip until a
  measured gap" discipline instead of adding reranking on day one because the
  domain sounds like it should need it.
- **MVP scale**: whatever the store's actual support ticket volume is — ask
  before assuming.

## 4. Incident-response and runbook assistant for a small engineering team

**What it does**: on-call engineer asks "what do we do when the payment
queue backs up" or "has this error happened before," and gets the relevant
runbook section plus links to past postmortems that match.

- **Source material**: Markdown (runbooks, postmortems, architecture docs),
  PDFs (any vendor incident reports), possibly screenshots of dashboards
  pasted into past incident channels.
- **Example queries**: a genuine mix — some name an exact service/error code
  (Axis 2 territory), some are open-ended ("why do things break on
  Fridays").
- **Predicted axis**: this is the niche to pick specifically to exercise
  **Axis 5** honestly. It will be tempting to reach for a "supervisor that
  routes between a runbook agent and a postmortem agent" — walk through
  Axis 5's three triggers before building that, and expect most real
  versions of this to resolve to a fixed pipeline (retrieve runbooks +
  retrieve similar postmortems, both, always) rather than genuine
  orchestration. Good practice specifically in *not* over-building.
- **What it exercises**: Phase 9 (Scheduled refresh) again, for a different
  reason than the rental niche — runbooks and postmortems are living
  documents, and a stale runbook served during an incident is actively
  dangerous.
- **MVP scale**: 1 small team, on-call rotation size (~3-8 people).

## 5. Freelancer/small-business finance and tax assistant

**What it does**: ingests bank/transaction exports, scanned receipts, and
tax-rule reference material; answers "how much did I spend on software
subscriptions this quarter" and "is this receipt deductible under my
country's home-office rule."

- **Source material**: images/PDFs (scanned receipts), Markdown or PDF (tax
  rule references — note: `scope: time-sensitive` per this handbook's own
  convention, since tax rules change yearly), plus transaction records
  (CSV — PLAYBOOK.md §1 doesn't cover this format explicitly; extend it the
  same way it handles structured JSON, since a CSV of dated, typed
  transactions is closer to fantasy's structured records than to prose).
- **Example queries**: transaction questions name exact entities (date
  range, category, vendor) — Axis 2 territory; deductibility questions are
  free-text over a small, stable rule set — Axis 1 territory.
- **Predicted axis**: hybrid, same shape as fantasy's parlay path — SQL
  retrieval over transactions feeding a small CAG block of tax rules, one
  path, not two independent ones, because a single question ("is this
  deductible") often needs both at once.
- **What it exercises**: PII/sensitivity handling that isn't quite s06-05's
  PII case (financial data, not personal-identity data) — good prompt to
  extend Section 1's intake checklist with a "financial/regulated data"
  branch if you build this one for real.
- **MVP scale**: 1 user (yourself, or a single freelance client) — this is
  the lowest-risk niche on the list to actually ship, since the data source
  is entirely your own.

## Picking one

If the goal is breadth of practice across the playbook rather than shipping
the highest-value idea first, build them in this order: **2 → 3 → 1 → 5 → 4**.
That order hits SQL-retrieval RAG, then a CAG/RAG composition, then genuine
vector RAG with real citation stakes, then a hybrid single-path composition,
and only then the one niche where seriously considering — and very likely
rejecting — multi-agent orchestration is the actual point.

## 6. Trading and active-investment advisor

Grounded in two local reference repos rather than a hypothesis: a personal
advisor that reads live quotes/positions (Alpaca, paper trading), your own
written thesis and risk rules (CAG), and a persisted decision history (Axis
2 — SQL, no embeddings) to answer "should I buy/sell/hold this" and "how
often was I right." A deterministic risk Critic gates every suggestion; a
human approves before anything executes.

- **Predicted axis**: CAG (thesis/rules) + Axis 2 SQL-retrieval (decision
  history) + a deferred Axis 3 for news + a deterministic Axis-4 Critic — and
  explicitly **no** Axis 5, unlike the reference multi-agent framework this
  niche studied and deliberately didn't copy.
- **What it exercises**: PLAYBOOK.md's live pass-through note (quotes and
  positions never stop changing), and the discipline of rejecting
  orchestration even when a well-known reference implementation uses it.
- **MVP scale**: 1 user (yourself) — this is personal advisory tooling, not
  a product offered to anyone else.

Full phased build guide, including creating the Alpaca account, building its
MCP server, and the exact order-placement safety pattern:
[NICHE-trading-advisor.md](NICHE-trading-advisor.md).
