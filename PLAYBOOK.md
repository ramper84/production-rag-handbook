---
title: Implementation Playbook — from source material to a running CAG/RAG project
doc_type: playbook
scope: evergreen
added: 2026-09-05
summary: >
  The process for turning a folder of context material (Markdown, PDFs,
  images) into a concrete, locally-deployable CAG/RAG/agentic project plan —
  architecture decision, project layout, tech stack, ARCHITECTURE.md,
  docker-compose.yml, and a phased build order, grounded in the articles in
  this handbook and in the two reference implementations that were built from
  them.
---

# Implementation Playbook

This is not another article about how retrieval works. It is the **process**
for going from *"here is a folder of source material, build me something"* to
a project that runs locally under `docker compose up` for manual testing —
using this handbook's articles as the technical justification for every
choice, and the two systems built from them as the concrete pattern to copy
or deviate from with a stated reason.

**Reference implementations**, cited throughout as *the estimator* and
*fantasy*:

| | The estimator (`lidr/ai-engineering`) | fantasy (`ramper/fantasy`) |
|---|---|---|
| Domain | IT project cost estimation from budgets + transcripts | NFL fantasy football decisions |
| Architecture | CAG + RAG + Agentic, composed through one conductor | CAG (draft) + SQL-retrieval RAG (weekly) + hybrid (parlay) — three paths, no conductor |
| Backend | FastAPI, layered `app/{foundation,domain,generation,ingestion,api}/` | FastAPI, flat `app/{services,analysis,guardrails,ingest,routers}/` |
| Store | Postgres + pgvector, real embeddings | Postgres + pgvector installed but mostly unused — SQL over typed columns is the retriever |
| Orchestration | None — a conductor composes fixed methods, no runtime agent routing | None, explicitly (ADR-007) — fixed recorded pipeline, no tool loop, no self-termination |
| Frontend | Rails app (`estimator-web`), separate repo | Streamlit, same repo |
| What it teaches | The full layered pattern when retrieval is genuinely semantic | That RAG's four stages can be real without an embedding index, and that a second architecture (lean, no conductor) is a legitimate choice, not a shortcut |

The single lesson that matters more than either example on its own: **the
right architecture is a property of the data and the queries, not a default.**
The estimator needed embeddings because its retrieval question is "which past
budget resembles this new project" — genuine semantic similarity over prose
and structured records. fantasy's weekly path does not, because "how did this
player do against this opponent" is a `WHERE` clause with an exact answer.
Copying the estimator's `generation/rag/` machinery onto a domain shaped like
fantasy's would be over-engineering; copying fantasy's SQL-only retriever onto
a domain shaped like the estimator's would silently return nothing for a
query with no named entity. **Decide this before writing any code — Section 2
is how.**

---

## 0. How to use this playbook

When the user hands over context material for a new project:

1. **Read everything provided** — Markdown, PDFs, images — before proposing
   anything. Section 1 says how to read each format and what to extract.
2. **Run the architecture decision** in Section 2 and state it explicitly,
   with the axis that decided it. Do not silently default to RAG because it
   is the trendier acronym — the estimator's own s06-01 opens by warning that
   RAG can cost *more* than CAG and the choice is viability, not fashion.
3. **Pick a project tier** (Section 3) — Lean or Layered — based on how many
   of CAG, RAG, Agentic (Axis 4), and orchestration (Axis 5) the project
   actually needs at v1. Do not build the five-layer estimator skeleton for a
   project that only needs CAG, and do not reach for orchestration (Axis 5)
   until a specific, named trigger requires it — the default for almost
   every project is that it does not.
4. **Produce the deliverable the trigger asked for.** Asked for a plan
   in-conversation: hand back one Markdown file, `IMPLEMENTATION_PLAN.md`,
   per Section 10's fixed sections. Told **"create a project based on
   PLAYBOOK.md"**: scaffold an actual repository instead, per Section 11's
   two-checkpoint workflow — an empty project with `examples/`, `README.md`
   and `CLAUDE.md` placeholders, filled in collaboratively, then built out
   only once told **"check all placeholders, if all are filled, then start
   implementation."** After v1 ships, a new capability goes through Section
   11.7's own trigger, **"extend CLAUDE.md to add `<capability>`"** — never a
   silent edit to the frozen file. Either way, `ARCHITECTURE.md` (Section 5's
   template) and `docker-compose.yml` (Section 6's template) are produced as
   real, project-specific content — not references to the templates.
5. **The build targets a fully local v1**: `docker compose up` brings up
   Postgres(+pgvector), Redis, the API, and a thin frontend, with no external
   dependency except the LLM provider's API key. Production concerns
   (auth hardening beyond API keys, autoscaling, managed Postgres, CI/CD) are
   named as follow-ups, not built into v1.

---

## 1. Intake: reading the provided context

Context arrives as Markdown, PDF, or images. Each needs a different
extraction path, and getting this wrong at intake produces exactly the
"garbage in" failure s06-01 and s06-02 spend a whole part warning about — no
amount of downstream chunking or reranking repairs bad extraction.

| Input | Extraction | Notes |
|---|---|---|
| **Markdown** | Read directly; treat headings as chunk-boundary hints (README convention: *"a heading is a claim about what belongs together"*) | Frontmatter (if present) is structured metadata — lift it into the catalog, don't discard it |
| **PDF** | `pypdf` / `unstructured` for text-native PDFs; `hi_res` (layout-aware) only for documents where tables or multi-column layout actually appear — s06-03 argues `hi_res` everywhere is wasted compute | Check for tables before choosing a parser; a wrong parser silently drops or garbles them |
| **Images** | Two paths, never both at once (s05-01): **(a)** multimodal — pass the image straight to the LLM at request time when it is a per-request user upload; **(b)** local extraction (OCR/vision-to-text once at ingest time) when it is reference material that will be queried repeatedly | Picking (a) for a static, reusable corpus re-pays the vision-model cost on every query; picking (b) for a one-off user upload adds an extraction step and a failure mode for no benefit |

**During intake, produce a source census** (s06-02), even an informal one,
before deciding architecture:

- **What is each source?** Format, approximate volume (item count, not just
  file size), and whether it is a one-time snapshot or something that grows.
- **How fresh does an answer need to be?** Static reference vs. daily/weekly
  refresh vs. real-time.
- **What will people actually ask it?** Collect 5-10 concrete example
  queries or interactions directly from the user — not inferred from the
  source documents. Section 2's Axis 2 ("do queries name their entities
  exactly?") is a question about queries, not about the corpus, and it
  cannot be answered from budgets, transcripts, or PDFs alone. If the user
  hasn't given examples yet, ask before running Axis 2 rather than guessing
  from what the data looks like it's for.
- **Do queries name entities exactly**, or are they free-text/ambiguous? This
  single question is the biggest architecture lever (Section 2, Axis 3) —
  judge it against the example queries above, not the source material.
- **Does anything contain personal data?** If yes, a pseudonymization pass
  (s06-05: Presidio + Faker + a mapping table, not blanket redaction) is a
  Section 8 phase, not an afterthought — access control alone does not
  protect a corpus once it is embedded and queryable.
- **Is there already a rulebook or scoring framework the domain follows**
  (compliance policy, house style, a scoring rubric)? That is a candidate for
  the CAG layer or for a deterministic Critic (Section 2, Agentic axis),
  exactly as fantasy's draft framework and board-grading rules are.

---

## 2. The architecture decision

Work through these axes **in order** — each one can end the decision early.

### Axis 1 — Is the corpus small and stable enough to live in the prompt?

CAG (static context in the system prompt, no retrieval) wins when the corpus
is small enough to fit comfortably in a context window and does not change
between requests. `s09-01`, citing Chan et al. 2024: *"when the corpus is
small and stable, CAG is not only simpler but produces better answers ...
the model sees the whole corpus in a single full-attention pass; in RAG the
model sees a subset selected by a retriever that can be wrong."* fantasy's
draft path is the live example: the whole player pool, live projections, and
one graded exemplar travel in the prompt, and that stays true even though the
rest of the system has retrieval elsewhere.

→ **If yes: build CAG.** Still read Section 2's later axes — most real
systems end up composing CAG for one path with something else for another,
as both reference projects do.

→ **If the corpus is large, grows over sessions, or must reflect near-real-time
state**, CAG alone will not hold. Continue to Axis 2.

### Axis 2 — Do the queries name their entities exactly?

This is the axis fantasy's ARCHITECTURE.md §5 turns into the sharpest claim
in either reference project: *"Our queries name their entities... a `WHERE`
clause with an exactly correct answer. Semantic similarity would approximate
what an index lookup gets right."*

- **Entities named, data structured/tabular** (player + week + team, invoice
  + line item + date, ticket + customer + status) → retrieval is **SQL over
  typed columns**, cutoff- and filter-bounded. This is still a genuine RAG
  flow — query, retrieval, augmentation, generation all exist — the retrieval
  stage is just not an ANN index. Cheaper, exact, and trivially debuggable
  with `EXPLAIN ANALYZE` instead of "why did the wrong chunk come back."
- **Queries are free-text, paraphrastic, or over prose that has no reliable
  column to filter on** (meeting transcripts, contracts, knowledge-base
  articles, support tickets read for *content* not for their ticket ID) →
  semantic retrieval is doing real work. Continue to Axis 3.

Do not embed a corpus that Axis 2 already answered with SQL. Standing up
pgvector for a table `WHERE`-clauses already serve exactly is the "no clever
chunking fixes bad architecture" version of s06-01's warning.

### Axis 3 — Vector RAG: pick the store and the pipeline shape

Once semantic retrieval is justified, the defaults from Parts 7-10 are:

- **Vector store**: pgvector, **not** a dedicated vector database, until one
  of s08-02's explicit outgrow signals fires — this handbook's own worked
  decision, and both reference projects standardize on `pgvector/pgvector:pg16`
  even where (fantasy) it ends up barely used. One fewer moving part in
  docker-compose, one fewer service to operate for a v1 that is being
  manually validated.
- **Chunking**: recursive 400–512 tokens as the default (s07-03); a
  structural chunker keyed to the document's own boundaries for anything with
  real structure (JSON records, tables) rather than forcing prose-shaped
  chunking on structured data (s07-04).
- **Index**: **no vector index in v1.** s08-00's own hands-on brief chooses
  sequential scan deliberately, "so sequential scan is the measured
  baseline" — add `hnsw` only once you have a latency number that requires it
  (s08-03, s08-05).
- **Embedding model**: `text-embedding-3-small` (or the provider's
  equivalent small embedding model) by default — both reference projects
  start here. Upgrade only against a measured recall gap (s07-02's decision
  axes), never speculatively.
- **Retrieval**: top-k **and** a distance threshold with a soft-fail path
  (`low_confidence`), never top-k alone (s09-03).
- **Advanced retrieval (reranking, hybrid, query rewriting, routing,
  temporal decay)** are Section 8's Advanced Retrieval phase — bring them in
  only against a measured gap (s10-02), never speculatively. A small gain at
  a small cost that still is not worth the complexity is s10-02's own
  "treacherous quadrant"; naming it in the plan is enough to keep it out of
  v1.

### Axis 4 — Does anything need an agentic (Actor-Critic-Boss) layer?

`s05-05` and both reference projects converge on the same answer: add this
**only** where a domain rulebook already has deterministic pass/fail checks
and the cost of a wrong answer is high enough to justify iteration. In both
reference projects the Critic and Boss are **code, not model calls** — the
rules have correct answers already computable, so paying an LLM to
re-derive them adds cost and variance for nothing. Reserve an actual model
call for the Critic only when the check genuinely requires judgment no rule
can express.

→ **If no rulebook with checkable rules exists yet**, skip the agentic layer
for v1 and note it as a reserved slot (mirroring `ARCHITECTURE.md §12` in
fantasy) rather than building speculative infrastructure.

### Axis 5 — Does this actually need multi-agent orchestration?

**Flagged explicitly: no article in this handbook argues for this.** Parts
5-11 stop at a bounded Actor-Critic-Boss loop (Axis 4), and the README says
so itself — Part 11 hands off to *"session 12: coordinating many specialised
answers instead of generating one big one,"* which is not written yet in
this repo. Everything below is this playbook's own extrapolation, held to
the same evidence standard the articles use, not a summary of one.

**Neither reference project uses agent orchestration, and both say so on
purpose.** The estimator composes CAG/RAG/Agentic through one conductor —
fixed methods, fixed call order, no runtime routing between them. fantasy's
`ARCHITECTURE.md` states it as an ADR (ADR-007): *"There is no agent in this
system... nothing here chooses its own next action, calls tools in a loop,
routes between specialists, or decides when it is finished... naming it
honestly stops someone adding routing this doesn't need."* Every model call
in both systems is a fixed step at a fixed position with a fixed prompt —
including the multi-stage pipelines that look agent-shaped from a distance
(fantasy's 14-stage draft pipeline is a **recorder**, not a scheduler; see
the per-stage run recorder built in Section 8's repo-scaffold phase).

**Default assumption: you don't need this.** The overwhelmingly common
outcome of running this axis honestly is that the steps *can* be enumerated
in advance, in which case the answer is a fixed pipeline (Phase order in
Section 8) with a per-stage recorder for observability — not an agent
framework. Reach for orchestration only when **at least one** of these is
true, and say in the plan which one:

1. **The set or order of steps cannot be known ahead of time** and genuinely
   depends on intermediate results — open-ended tool use where the next tool
   call is chosen by what the previous one returned, not a fixed sequence
   with conditional branches you could draw as a flowchart today.
2. **Multiple independent specialist roles must run over different
   sub-problems and be fused into one answer**, where the number or identity
   of specialists is data-dependent rather than a small fixed set you can
   name now. (A fixed set of 2-3 known specialists called in a fixed order
   is Phase composition, not orchestration — don't let the presence of
   multiple LLM calls alone trigger this axis.)
3. **A supervisor must route a request to one of several heterogeneous
   capabilities at runtime**, and that capability set will keep growing
   after v1. A router choosing among a small, fixed, already-known set of
   collections or paths is s10-05's retrieval-routing pattern, already
   covered by Axis 3 — this axis is for when the things being routed to are
   themselves agents with their own tool access, not retrieval collections.

**What this costs, and why the bar is high.** An orchestration loop
compounds every cost this playbook otherwise fights to bound: `s05-03`'s
pyramidal-test discipline gets harder to hold when the step sequence itself
varies run to run, not just the model's output; spend and latency lose the
fixed ceiling a known-length pipeline gives for free; and the per-stage
observability that makes fantasy's pipeline debuggable (*"a run reporting
$0.00308 while its stages summed to $0.00319 was caught"*) is exactly what a
dynamic step count makes harder to keep. None of that is a reason never to
build it — it is the reason to name the specific trigger before starting,
the same discipline Section 2's "Composing more than one answer" applies to
picking a conductor.

→ **If Axis 5 selects orchestration**, default to the smallest thing that
works for a local v1, in this order of preference:
- A **hand-rolled supervisor loop**: a router function, a small fixed
  registry of specialist functions/agents, a **hard iteration/step cap**, and
  a per-step recorder logging which specialist ran, what it produced, and
  what it cost — the same shape as fantasy's `pipeline/RunRecorder`, just
  with the router's target chosen at runtime instead of fixed. This is
  usually sufficient and stays inside the pytest-testable, deterministic-
  where-possible discipline the rest of this playbook holds to.
- **A graph-based agent framework** (e.g. LangGraph), only once the hand-rolled
  loop has been tried and the branching shape has actually been observed to
  need it — the same "measure before adopting" gate Section 2/Axis 3 applies
  to reranking and hybrid search (s10-02). Never adopt a framework
  speculatively as project scaffolding.
- **The hard cap and full per-step logging are non-negotiable regardless of
  which of the above is chosen** — an unbounded agent loop is an unbounded
  spend loop, and Section 4's "guardrails fail closed" rule applies to the
  step cap exactly as it does to `budget.py`/`rate_limit.py`.

→ **If no trigger above is met**, skip orchestration for v1, build the fixed
pipeline instead, and note the untriggered condition as a reserved slot
(Section 5's ARCHITECTURE.md template, §8) rather than infrastructure.

### Composing more than one answer

Most real projects are not one path. Write down, per user-facing capability
(draft/weekly/parlay in fantasy; estimate/session-chat in the estimator), which
axis answered it. Two valid ways to wire multiple architectures together:

- **A conductor** (the estimator's `EstimationService`): every path funnels
  through one composition point; the layers never call each other directly.
  Pick this when the same request might touch CAG, RAG, and Agentic in
  sequence, or when you can already see a second and third capability coming.
- **Independent paths sharing only plumbing** (fantasy's `services/` and
  `schemas.py`): each capability is its own router-driven flow; nothing
  routes between them. Pick this when the capabilities are genuinely
  separate questions (a draft board vs. a weekly lineup vs. a bet) that would
  gain nothing from being aware of each other. **Decide up front which one a
  third capability would trigger** — fantasy's own ARCHITECTURE.md §11
  records that this exact question was answered too late and named the
  refactor it still owes as a result. State the trigger condition explicitly
  in the plan so it isn't re-litigated when it fires.

---

## 3. Project tier: Lean vs. Layered

Pick based on how many architectures Section 2 actually selected for v1, not
on the biggest system either reference project became after a dozen
sessions.

| | **Lean** | **Layered** |
|---|---|---|
| When | One primary architecture (commonly SQL-retrieval RAG, or CAG alone), a small number of request paths | Two or more of CAG/RAG/Agentic genuinely compose on the same request, or growth to that is a near-term certainty |
| Backend shape | `app/{config.py, schemas.py, services/, analysis/, guardrails/, ingest/, routers/}` | `app/{config.py, dependencies.py, foundation/, domain/, generation/{cag,rag,agentic,conversation}/, ingestion/, api/}` |
| Composition | Routers call `services/`/`analysis/` directly; no conductor | One conductor object (`domain/<name>_service.py`) is the only place layers meet |
| Reference | fantasy | the estimator |
| Growth path | Extract a conductor **the moment** a third path or cross-layer composition appears — do not wait, fantasy's own architecture doc names the cost of waiting | Already structured for it |
| If Axis 5 selected orchestration | Move to Layered — a runtime supervisor routing between specialists needs the conductor's single composition point even more than a second static path does; do not bolt a router onto Lean's direct `router → services` calls | `generation/agentic/orchestrator.py` sits beside `boss.py`/`critic.py`, reached only through the conductor |

Both tiers share everything else in this playbook — tech stack, docker-compose
shape, evals discipline, ARCHITECTURE.md conventions. The difference is
purely how many folders `app/` has on day one.

---

## 4. Tech stack defaults

Defaults below are what both reference projects converged on independently.
Deviate with a stated reason in the plan, not silently.

| Concern | Default | Reasoning / when to deviate |
|---|---|---|
| Language / API framework | Python 3.12, FastAPI + uvicorn | Typed request/response models are the contract; async support for the retrieval path |
| Validation | Pydantic v2 (`pydantic-settings` for config) | One schema serves the wire, the config, and structured LLM output |
| Relational + vector store | PostgreSQL 16 via `pgvector/pgvector:pg16` | One database for both typed columns and embeddings, even when embeddings end up unused (Axis 2). Swap only against an explicit s08-02 outgrow signal |
| Embedding model (only if Axis 3 selected vector RAG) | `text-embedding-3-small` (or the provider's equivalent small model) | s07-02's default; upgrade only against a measured recall gap, never speculatively |
| DB access | SQLAlchemy (async, if RAG/CAG persistence needs an ORM) **or** raw SQL via psycopg 3 (if the domain logic *is* the query, à la fantasy) | Layered/estimator-style projects benefit from the ORM's typed models; Lean/analytics-heavy projects read more honestly as SQL that a person can `EXPLAIN` |
| Schema migrations | Alembic, always | Never `CREATE TABLE IF NOT EXISTS` in app code — one schema owner or you get fantasy's ADR-009 failure mode (two writers silently declaring different schemas) |
| Cache / session / memory | Redis (`redis/redis-stack` if a semantic/vector cache is needed, otherwise plain `redis:alpine`) | Exact-match cache, conversational memory, rate-limit counters, run/audit records |
| LLM access | LiteLLM + Instructor, one wrapper, cross-provider fallback (e.g. `gpt-4o-mini` primary, a Claude Haiku/Sonnet fallback) | One place spend and retries are recorded; never call a provider SDK directly from business logic |
| Prompts | Jinja2 templates, versioned by directory (`prompts/<name>/v1/`, `v2/`, …) | A prompt change is a new version, not a silent edit — both reference projects treat this as load-bearing |
| Structured output | Instructor (Pydantic-validated LLM output, with re-prompt on validation failure) | Avoids hand-rolled JSON parsing and repair hacks |
| Logging | `structlog` | JSON in prod, console in dev; every pipeline stage logs its own name |
| Guardrails | Input: size limits, prompt-injection scan, relevance/topicality check. Output: scope/format filter, PII/disliked-content filter. Spend: a hard cost ceiling that **fails closed** | Caches and memory degrade silently (fail open) on an outage; anything that bounds spend must fail closed (fantasy ADR-008) |
| Testing | pytest, pyramidal (s05-03): mostly structural/deterministic assertions, some statistical, a thin layer of LLM-as-judge | A suite that is all judge-based is slow and brittle on one point of failure |
| Frontend (v1) | **Streamlit** | Python-native, no separate build step, fast enough to stand up in the same repo and container set as the API — the right choice whenever the goal is "locally deployable for manual testing," not a production customer UI. See Section 4a |
| Frontend (post-v1, if productionizing) | A dedicated web app (Rails, Next.js, etc.) calling the API over HTTP, in its own service/repo | Only once the API contract has stabilized under manual testing — mirroring the estimator's later `estimator-web` split |
| Agent orchestration (only if Axis 5 selected it) | A hand-rolled supervisor loop (router function + fixed specialist registry + hard step cap + per-step recorder) | Default even when orchestration is justified — no new dependency, stays inside the pytest-testable discipline the rest of the stack holds to. Reach for a graph framework (e.g. LangGraph) only once this has been tried and measured insufficient — never as day-one scaffolding |
| Local deployment | `docker compose`, Postgres + Redis + API (+frontend), migrations run as a startup command before the app | See Section 6 |

### 4a. Why Streamlit by default, and when not

The ask is explicitly a v1 that is "fully locally deployed with docker-compose
for manual testing and validation." Streamlit is a form over the same API
contract the eventual real frontend will use — it never becomes a second
source of business logic, because it has none. Both `httpx` calls to the
FastAPI service and nothing else live in the Streamlit file. If the plan's
audience is genuinely non-technical end users from day one, or the UI needs
interaction patterns Streamlit cannot express (multi-step wizards with
branching, rich client-side state), name a real frontend framework instead —
but the API-first httpx-only discipline still applies: the frontend calls the
API, it does not reimplement it.

---

## 5. `ARCHITECTURE.md` template

Fill in every bracketed section for the specific project. This is the
document that ships in the project's repo root — not a description of it.

````markdown
# Architecture

This is the architecture contract for [PROJECT NAME]. New code must fit one
of the layers below; if a piece fits nowhere, decide where it lives and
update this document — do not add another folder at the root.

## 0. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.12 | [reason] |
| HTTP | FastAPI + uvicorn | [reason] |
| Store | PostgreSQL 16 (pgvector/pgvector:pg16) | [reason — cite Axis 2/3 outcome] |
| Cache | Redis | [reason] |
| LLM access | LiteLLM ([primary model], fallback [model]) | [reason] |
| Frontend | Streamlit | manual local testing, no separate contract from the API |
| Deploy | docker compose (postgres, redis, api, frontend) | migrations run before the app |

**Not used, deliberately**: [name anything both reference projects use that
this project explicitly does not — e.g. "no vector index in v1", "no agentic
layer", "no ORM" — each is a decision, not an omission].

## 1. Architecture decision

[State the Section 2 outcome explicitly: which axis decided it, what the
alternative would have looked like, and why it was rejected. E.g.: "Axis 2 —
queries name exact entities over structured records, so retrieval is SQL
over typed columns; no embedding index in v1."]

## 2. The layer map

```
app/
├── [fill in from the chosen tier, Section 3]
```

## 3. Dependency rules (MUST / MUST NOT)

| Layer | MAY import | MUST NOT import |
|---|---|---|
| [layer] | [layers above it] | [layers below/beside it] |

## 4. The conductor (Layered tier only)

[Name the single composition object, its entry points, and exactly what each
entry point touches, in order — mirror the estimator's §4 "Camino de la
petición principal."]

## 5. Request paths

For each user-facing capability, the ordered pipeline of stages, marking
which stages call an LLM:

```
POST /api/v1/[capability]
 1. [stage]           [free / LLM]
 2. [stage]           [free / LLM]
 ...
```

## 6. Contracts that do not break

- Routes: [list]
- [Any field/label rendered verbatim, any invariant enforced at multiple layers]
- The schema comes from migrations; nothing creates a table on demand.

## 7. Where does new code go?

| If you add… | It goes in… |
|---|---|
| [row per layer] | |

## 8. Reserved slots

Named so nobody invents a different home for them later: [list anything
Section 2/8 explicitly deferred — a second architecture axis, an agentic
layer, an index].

## Appendix — Architecture Decision Records

One entry per decision that would otherwise be re-litigated: status,
context, decision, evidence (if measured), consequence. Start this file
empty and add an ADR the first time a decision is challenged or reversed —
do not backfill speculative ones.
````

---

## 6. `docker-compose.yml` template

Parameterize `{{project}}` and adjust the frontend service if not Streamlit.
This is deliberately a **development** compose file: bind-mounted source,
`--reload`, migrations run as part of the startup command, no vector index
build step.

```yaml
services:
  api:
    build: .
    container_name: {{project}}-api
    ports:
      - "8000:8000"
    env_file:
      - .env
    environment:
      REDIS_URL: redis://redis:6379
      DATABASE_URL: postgresql+psycopg://{{project}}:{{project}}@postgres:5432/{{project}}
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    volumes:
      - ./app:/app/app
      - ./tests:/app/tests
      - ./data:/app/data
      - ./alembic:/app/alembic
      - ./alembic.ini:/app/alembic.ini
    command: >
      sh -c "alembic upgrade head &&
             uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 5s
      start_period: 30s
      retries: 3

  redis:
    # Use redis/redis-stack instead of redis:alpine ONLY if a semantic cache
    # or Redis-native vector search is part of v1 (RediSearch module).
    image: redis:7-alpine
    container_name: {{project}}-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  postgres:
    image: pgvector/pgvector:pg16
    container_name: {{project}}-postgres
    environment:
      POSTGRES_USER: {{project}}
      POSTGRES_PASSWORD: {{project}}
      POSTGRES_DB: {{project}}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{project}}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  frontend:
    build: .
    container_name: {{project}}-frontend
    env_file:
      - .env
    environment:
      API_BASE_URL: http://api:8000
    command: ["streamlit", "run", "streamlit_app.py", "--server.address=0.0.0.0", "--server.port=8501"]
    ports:
      - "8501:8501"
    depends_on:
      - api
    restart: unless-stopped

volumes:
  redis_data:
  postgres_data:
```

Notes to carry into the plan verbatim:

- **No vector index build step** — Axis 3 says start with sequential scan.
  Add an `hnsw`-building migration only once a measured latency number
  justifies it (s08-05).
- **Migrations run before the app**, in the same container as the code that
  expects them (fantasy ADR-009) — never a runtime `CREATE TABLE`.
- If the project has **no RAG/CAG store need at all** (rare — even a pure
  CAG project usually wants Postgres for job/session state, or SQLite for
  something this small), drop the `postgres` service and say so explicitly,
  rather than including an unused database because the template has one.
- If PII pseudonymization (Section 1) is in scope, add a `PSEUDONYM_HASH_SALT`
  and note in `.env.example` that rotating it invalidates all mappings.
- **`.env` is never committed — only `.env.example` is.** `.env` holds real
  LLM provider keys from the moment it's created; the project's `.gitignore`
  (Section 11.1) excludes it from the first commit, not as a later cleanup.

---

## 7. Directory skeleton

**Lean tier** (fantasy-shaped):

```
{{project}}/
├── app/
│   ├── config.py
│   ├── schemas.py
│   ├── main.py
│   ├── services/        # shared plumbing: redis client, llm client, external APIs
│   ├── analysis/         # deterministic domain logic — NO LLM import, ever, if any
│   │                      #   path must stay hallucination-proof (state this as a rule,
│   │                      #   enforced by a test, if any path applies)
│   ├── guardrails/       # input/output/spend checks
│   ├── ingest/           # offline: source → parse → validate → Postgres
│   │   ├── parsers/
│   │   ├── store/
│   │   └── refresh.py    # ONLY if Phase 9 (Scheduled refresh) applies
│   ├── prompts/          # versioned Jinja2, one dir per prompt family
│   └── routers/          # thin HTTP, one file per capability
├── migrations/           # Alembic; Alembic owns the schema, nothing else creates tables
├── evals/                # golden dataset + metrics + a runnable comparison script
├── tests/
├── data/                 # seed/sample data, .gitignored if it contains real records
├── streamlit_app.py
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── ARCHITECTURE.md
└── pyproject.toml
```

**Layered tier** (estimator-shaped) — use when Section 2 selected two or more
of CAG/RAG/Agentic composing on one request:

```
{{project}}/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── dependencies.py         # composition root — the only place that wires singletons
│   ├── foundation/             # opinion-free plumbing
│   │   ├── llm/                #   LLM wrapper (LiteLLM + Instructor)
│   │   ├── prompts/            #   Jinja2 loader + versioned templates
│   │   ├── guardrails/         #   input / output
│   │   └── persistence/        #   engine + repositories
│   ├── domain/
│   │   ├── schemas/            #   the request/response contract
│   │   └── <name>_service.py   #   THE CONDUCTOR — only place layers meet
│   ├── generation/
│   │   ├── cag/                #   present only if Axis 1 selected CAG
│   │   ├── rag/                #   chunking/ embedding/ store/ retriever.py — only if Axis 3 selected vector RAG
│   │   ├── agentic/             #   boss.py + critic.py — only if Axis 4 selected it
│   │   │   ├── boss.py + critic.py             #   Axis 4: bounded ACB loop, both deterministic code
│   │   │   ├── orchestrator.py                 #   Axis 5 ONLY: router + specialist registry + step cap + recorder
│   │   │   └── specialists/                    #   Axis 5 ONLY: the fixed, named registry the router picks from
│   │   └── conversation/       #   session/memory handling, if multi-turn
│   ├── ingestion/               # offline pipeline feeding generation/rag
│   │   └── catalog/ parsers/ cleaning/ pii/ refresh.py (ONLY if Phase 9 applies)
│   └── api/                    # thin routers, no business logic
├── migrations/ (alembic)
├── evals/
├── tests/
├── data/
├── streamlit_app.py
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── ARCHITECTURE.md
└── pyproject.toml
```

**Only create the `generation/` subfolders Section 2 actually selected.** An
empty `generation/agentic/` waiting for a future need is exactly the
speculative-infrastructure smell this playbook is trying to avoid — use
Section 5's "Reserved slots" table to record the intent instead. This applies
doubly to `orchestrator.py`/`specialists/`: create them only when Axis 5 named
a specific trigger, never alongside `boss.py`/`critic.py` by default just
because the folder is named `agentic/`.

---

## 8. Phased build plan

The order below is the order dependencies actually run in — each phase
produces something the next phase needs. Map every phase in the delivered
plan to the handbook part/article that argues for the technique, not just to
this playbook.

| Phase | What | Handbook reference | Skip if… |
|---|---|---|---|
| **0** | Architecture decision, tier choice, tech stack — write these into the plan explicitly, with the axis that decided each | Part 5 (s06-01) | never |
| **1** | Repo scaffold: chosen directory tree, `pyproject.toml`, `.env.example`, `.gitignore`, `Dockerfile`, `docker-compose.yml`, health endpoint, and a per-stage run recorder (timing, cost, success/failure per stage — fantasy's `pipeline/RunRecorder` shape) for observability from the first request onward | — | never |
| **2** | Source audit and catalog: census every input, format, volume, freshness, PII exposure, **and 5-10 real example queries collected from the user** (Section 1) | s06-02 | never |
| **3** | Extraction: format-specific parsers converging on one canonical internal record/document shape | s06-03 | never |
| **4** | Cleaning + validation: an explicit repair / quarantine / discard policy, not silent coercion | s06-04 | never |
| **5** | PII pass: Presidio-style detection + consistent pseudonymization, if Section 1 flagged personal data | s06-05 | no personal data in any source |
| **6** | CAG layer: build the static context block, delimited and sized against a token budget | Part 5, s09-01 | Axis 1 did not select CAG |
| **7** | Chunking: pick the strategy per document type (structural for records, recursive for prose) | s07-03, s07-04 | Axis 3 did not select vector RAG |
| **8** | Embeddings + persistence: model choice, pgvector schema (typed columns + JSONB split), atomic ingest transaction, **no index yet** | s07-01/02, s08-00/04 | Axis 3 did not select vector RAG |
| **9** | Scheduled refresh: wrap Phases 2-4 (and 7-8 if RAG) as a repeatable job — fingerprint what changed rather than reprocessing everything, if Section 1 found the corpus is not a one-time snapshot | s06-02's freshness axis; the pattern is fantasy's `ingest/refresh.py`, not an s-numbered article | Section 1 found the corpus static/one-time |
| **10** | Retrieval: SQL-typed retriever (Axis 2) or vector retriever with top-k + threshold + soft-fail (Axis 3) | s09-03, or the SQL-retrieval pattern in fantasy §5 | never — every architecture has *a* retrieval stage |
| **11** | Augmentation: structured context assembly (XML-delimited sources with metadata), never `"\n\n".join` | s09-04 | never |
| **12** | Generation: prompt template (versioned), structured output via Instructor, citation/provenance fields carried through | s09-04, s11-03 | never |
| **13** | Guardrails: input (size, injection, relevance), output (scope, dedupe, disliked-content), spend (fails closed) | Part 5 art. 1, fantasy `guardrails/` | never |
| **14** | Agentic layer, if Axis 4 selected it: deterministic Critic against the domain rulebook, deterministic Boss with a bounded iteration count | s05-05 | Axis 4 did not select it |
| **15** | Multi-agent orchestration, if Axis 5 selected it: the hand-rolled supervisor loop (router + fixed specialist registry + hard step cap + per-step recorder) named in Axis 5, built and tested **after** the fixed-pipeline phases above exist, never as a replacement for them | Axis 5 — extrapolation beyond the handbook's articles, not backed by an s-numbered one | Axis 5 selected no trigger — the default for almost every project |
| **16** | Advanced retrieval, only against a measured gap: reranking, hybrid search, query rewriting, routing, temporal decay | Part 10 | no measured gap yet — leave as a reserved slot |
| **17** | Golden dataset + eval harness: pyramidal tests (structural, statistical, judge), a golden set from the start (5-20 cases minimum), built from the example queries collected in Phase 2 | s05-03, s10-02 | never |
| **18** | Frontend: Streamlit calling the API over `httpx`, one screen per capability | Section 4a | never for v1 |
| **19** | Local validation pass: `docker compose up`, run the golden set end-to-end, confirm health checks, record the manual test script in the plan | — | never |
| **20** | ARCHITECTURE.md finalized against what was actually built (not what was planned) — including ADRs for anything that changed from the plan | Section 5 | never |

Not every phase runs for every project. Rather than memorizing a numeric
range, check each phase's own **"Skip if…"** column — several (CAG,
Chunking, Embeddings, Scheduled refresh, PII, Agentic, Multi-agent
orchestration, Advanced retrieval) depend directly on which axis fired in
Section 2 or what Section 1's intake found, and that selection is exactly
what makes the delivered plan project-specific rather than a generic RAG
tutorial. The Multi-agent orchestration phase in particular should stay
unbuilt for most projects — its "Skip if…" is the common case, not the
exception.

---

## 9. Evals and validation checklist

Carry this into every plan's own validation section, filled in for the
project:

- [ ] A golden dataset exists **before** any retrieval or generation code is
      finalized (s05-03) — 5-20 cases minimum, including at least one
      out-of-scope and one adversarial case.
- [ ] At least one case in the golden set has "not enough data" / abstention
      as the correct answer — s11-06's own gap analysis names a golden set
      with no such case as one that rewards always answering.
- [ ] Tests are pyramidal: mostly deterministic/structural, a thin layer of
      LLM-as-judge, none `assert response == expected` on free text.
- [ ] Citation/provenance is checked in code (`verify_citations` /
      equivalent), never trusted to the model (s11-03).
- [ ] If retrieval is SQL, every historical query carries the equivalent of a
      `(season, week)` cutoff — fantasy's ADR: a query that can see the
      answer it's predicting is invisible in testing and wrong in
      production.
- [ ] Spend/budget guardrails fail **closed**; caches/memory fail **open**.
- [ ] `docker compose up` from a clean checkout brings up every service
      healthy and the golden set runs end-to-end with no external
      dependency besides the LLM provider key.
- [ ] The golden-set run makes real, billed LLM calls on every clean-checkout
      validation — keep the set small enough for routine local validation
      (a larger set is a separate, deliberate eval run, not the default
      smoke test), and use the cheapest model in the fallback chain unless
      the check specifically needs the primary model.

---

## 10. The deliverable: `IMPLEMENTATION_PLAN.md`

Every time this playbook is run against a new project's context, hand back
one Markdown file with exactly these sections, in this order:

1. **Context summary** — what was provided, in what formats, and the
   Section 1 source census.
2. **Architecture decision** — the Section 2 axes walked through in order,
   which one(s) decided it, and the composition strategy (conductor vs.
   independent paths) if more than one architecture is in play.
3. **Project tier** — Lean or Layered, with the one-line reason.
4. **Tech stack** — the Section 4 table, filled in, deviations justified.
5. **`ARCHITECTURE.md`** — the full Section 5 template, filled in for this
   project. This is the actual file content, ready to save into the new
   repo, not a summary of it.
6. **`docker-compose.yml`** — the full Section 6 template, filled in and
   parameterized for this project, ready to save and run.
7. **Directory skeleton** — the Section 7 tree, pruned to the folders this
   project actually needs.
8. **Phased build plan** — the Section 8 table, with per-phase notes specific
   to this project's data (which parser for which source, which fields the
   catalog needs, what the golden set's first cases should be).
9. **Evals and local validation plan** — the Section 9 checklist, filled in,
   plus the specific manual test script for this project (what to click,
   query, or `curl` after `docker compose up` to confirm v1 works).
10. **Open questions / assumptions** — anything the context material did not
    settle, stated as an assumption the plan proceeds under, not left
    silent.

Nothing in this list is optional, and nothing outside it belongs in the file
— nothing sold as a plan should still be prose arguing for RAG in general;
by the time this document is written, Section 2 has already decided that.

---

## 11. Bootstrapping a new project (scaffold → fill → implement)

Section 10 is the right delivery when the ask is *"tell me the plan."* This
section is the operational alternative for when the ask is *"build me the
project"*: a two-checkpoint workflow, triggered by two exact phrases, that
scaffolds a real repository, waits for a human to answer what only a human
can answer, and only then starts building.

### 11.1 Trigger: "create a project based on PLAYBOOK.md"

On this instruction — with a project name if one is given, otherwise ask for
one before creating anything:

1. Create the project as a new sibling repository, by default at
   `~/git/ramper/{{project-name}}/` (matching where this handbook and the
   `fantasy` reference project already live), unless the user names a
   different location. `git init` it.
2. Create **only** this minimal scaffold — nothing architecture-specific,
   because Section 2's decision has not been made yet and must not be
   guessed ahead of the example material:

   ```
   {{project-name}}/
   ├── examples/          # empty — the user drops MD/PDF/image context here
   ├── README.md          # Section 11.3 template, placeholders unfilled
   ├── CLAUDE.md          # Section 11.4 template, placeholders unfilled
   └── .gitignore
   ```

   `.gitignore` starts with at least:

   ```gitignore
   .env
   examples/
   __pycache__/
   *.pyc
   .venv/
   ```

   **`examples/` is git-ignored by default.** It commonly holds real client
   budgets, transcripts, or other business-confidential material dropped in
   before anyone has judged whether it's safe to commit. Un-ignore it only
   after the user confirms the contents are safe to check in (synthetic,
   public, or already-cleared data) — until then, describe what's in it in
   `README.md` §4 rather than committing the files themselves. `.env` is
   never committed at any point; only `.env.example` (created in Phase 1) is.
3. Do **not** create `app/`, `docker-compose.yml`, `ARCHITECTURE.md`, or any
   other implementation artifact at this stage. Those are Section 8's
   Architecture-decision and repo-scaffold phases and depend on decisions
   this scaffold deliberately defers — creating them now would be guessing
   the architecture before reading a single example file, exactly what
   Section 2 exists to prevent.
4. Report back which placeholders need a human answer, so the user knows
   what to return with before the second trigger.

### 11.2 Placeholder convention

Every unanswered field uses the literal token `[FILL ME: ...]`, chosen
because it is a single, unambiguous, greppable pattern:

```bash
grep -rn "FILL ME" README.md CLAUDE.md
```

This exact command is what Section 11.6's checkpoint runs. **Never resolve a
placeholder with a guess.** An invented user count or an invented functional
requirement is worse than an open placeholder — it looks decided when it is
not, and nothing downstream will re-check it.

### 11.3 `README.md` template

The product side: what the system must do, who it is for, and at what scale
— the questions Section 1 and Section 2 cannot answer from source material
alone, because they are decisions about the world, not properties of it.

````markdown
# [FILL ME: project name]

## 1. What this is

[FILL ME: one to three sentences — what the system does, for whom, in plain
language]

## 2. Functional requirements

[FILL ME: the concrete capabilities the MVP must deliver, one bullet per
capability — what a user provides, what the system returns. This states what
the system must DO, not how; the how is CLAUDE.md's job.]

- [FILL ME]

## 2a. Example queries

[FILL ME: 5-10 concrete questions or interactions a real user would actually
type or ask, in their own words — not paraphrased from the source material.
This is not optional and not redundant with §2: PLAYBOOK.md §2's Axis 2 ("do
queries name their entities exactly?") decides the retrieval architecture
from these examples, not from the documents in `examples/`, and the golden
eval set is seeded from them too.]

- [FILL ME]

## 3. User profile

- **Who uses this**: [FILL ME: role/persona — internal team, external
  customers, a single user, ...]
- **Expected users at MVP-in-production**: [FILL ME: a number or range — 1,
  ~10, ~100s. This bounds the concurrency/scale decisions in CLAUDE.md; do
  not let the plan over-build for a scale nobody asked for. fantasy is
  single-user and the estimator serves a classroom — neither number
  transfers to a new domain by default.]
- **How they interact with it**: [FILL ME: web UI, API only, chat surface,
  ... — v1 defaults to a local Streamlit UI per PLAYBOOK.md §4a unless
  stated otherwise here]

## 4. Source material

Context files (Markdown, PDFs, images) that ground this project's data and
domain live in [`examples/`](examples/):

- [FILL ME: file/folder — what it is, roughly how much of it there is]

## 5. Other key points

[FILL ME: constraints, deadlines, things this must NOT do, compliance or
privacy requirements — anything a reference project's ADR would have
recorded up front rather than discovered later]

## 6. Status

- [ ] README placeholders filled
- [ ] CLAUDE.md placeholders filled
- [ ] Implementation started — see `CLAUDE.md` for the architecture and
      build strategy
````

### 11.4 `CLAUDE.md` template — and why it freezes

The reference projects' own `CLAUDE.md` files accumulate a running session
log across dozens of sessions (*"Session 14 adds..."*, *"Session 17..."*) —
useful once a project has history, noise before it has any. **Skip that
convention when generating a new project's `CLAUDE.md`.** The template below
keeps only the structural sections and starts with no session narrative.

**Once every `[FILL ME]` in `CLAUDE.md` is resolved, the file is frozen for
the rest of v1.** Implementation follows it; it does not get edited to match
whatever the code ends up doing. A decision that turns out wrong before v1
ships is a deliberate, named exception discussed with the user — and a
candidate ADR in `ARCHITECTURE.md`'s appendix (Section 5) — not a routine
edit to this file. `ARCHITECTURE.md` is the document that is expected to
evolve with the build; `CLAUDE.md` is the contract that does not, until v1
ships and Section 11.7's trigger deliberately reopens it.

````markdown
# CLAUDE.md

Guidance for Claude Code when working in this repository. **This file is
frozen once its placeholders are filled** — implementation follows it and it
is not rewritten as the build proceeds. A decision that changes becomes an
ADR in `ARCHITECTURE.md`, not an edit here. Generated from PLAYBOOK.md in
`production-rag-handbook`.

## 1. Description

[FILL ME: technical restatement of README.md §1-2, for an engineer picking
up the code cold]

## 2. Architecture decision

[FILL ME once `examples/` has real content — run PLAYBOOK.md §2's five axes
against the actual source material and record the outcome: which axis
decided it (CAG / SQL-retrieval RAG / vector RAG / hybrid / multi-agent
orchestration), the composition strategy if more than one path applies, and
the one-line reason. Axis 5 (orchestration) is extrapolation beyond what the
handbook's articles argue for and should stay unselected unless a specific
trigger from §2 is named — do not fill it in as the default. Do not fill any
of this in before the examples exist.]

## 3. Project tier and structure

[FILL ME: Lean or Layered (PLAYBOOK.md §3), and the pruned directory tree
from §7 — only the folders this project actually needs]

## 4. Tech stack

[FILL ME: the PLAYBOOK.md §4 table, filled in for this project; note any
deviation from the defaults and why]

## 5. Common commands

[FILL ME once scaffolded: how to run it locally, run tests, run the golden
eval set — e.g. `docker compose up --build`, `pytest`]

## 6. Configuration

[FILL ME: the environment variables this project needs and what each
controls — mirror `.env.example` once it exists]

## 7. Build strategy

[FILL ME: the PLAYBOOK.md §8 phased build plan, pruned to the phases this
project's architecture decision actually turns on, in build order]
````

### 11.5 Filling the placeholders

This happens collaboratively, over as many turns as it needs, between the
two trigger phrases:

- **README.md's product questions need the human's answer** — functional
  requirements, project name if not already set, user profile, MVP scale,
  other key points. Do not fill these from a guess or from what the
  reference projects happened to need.
- **`examples/` is filled by the user** dropping in the actual source
  material — Section 1's intake process applies once files land there.
- **`CLAUDE.md`'s technical sections can be drafted by running Sections 2-8
  of this playbook** against what is now in `examples/` and `README.md` —
  propose them, but they stay `[FILL ME]` until the user confirms, because
  Axis 2/3's answer depends on the actual shape of the example queries and
  data, not on a template default.

### 11.6 Trigger: "check all placeholders, if all are filled, then start implementation"

1. Run the grep from Section 11.2 against `README.md` and `CLAUDE.md`.
2. **If any `[FILL ME]` remains**, list every one — file, location, what it
   is asking for — and stop. Do not start building around an unanswered
   question.
3. **If none remain**, treat `CLAUDE.md` as the authoritative technical
   strategy and `README.md` as the authoritative product scope, and execute
   Section 8's phased build plan against them, starting with the
   repo-scaffold phase (now properly informed by Section 2's recorded
   decision) through whichever phases that decision turns on. Do not re-open
   the architecture decision at this point — a build-time reason to
   reconsider it is an ADR and a conversation with the user, never a silent
   rewrite of a frozen `CLAUDE.md`.
4. `ARCHITECTURE.md` is written during the repo-scaffold phase and refined
   through the final "ARCHITECTURE.md finalized" phase (Section 5). It is a
   distinct document from `CLAUDE.md` and is expected to change as the build
   proceeds; `CLAUDE.md` is not — see Section 11.7 for the one sanctioned way
   to reopen it.

### 11.7 Extending after v1: the one sanctioned way to reopen `CLAUDE.md`

Section 11.4's freeze is deliberate, not permanent. A shipped v1 is going to
need new capabilities eventually, and without a named way back in, that
pressure resolves one of two bad ways: `CLAUDE.md` gets quietly edited to
match whatever got built (exactly what the freeze exists to prevent), or it
goes stale and stops being read. Neither is acceptable, so there is a third,
explicit trigger for exactly this.

**Trigger: "extend CLAUDE.md to add `<capability>`"**

1. Treat this as a new, scoped pass through Sections 1-2 for the new
   capability only — new example material (more files in `examples/`, more
   example queries in `README.md` §2a) and a fresh run of the architecture
   axes for *this capability*, not a re-litigation of the existing ones.
   Section 2's "Composing more than one answer" governs how it joins what
   already exists (a new arm of the existing conductor, or an independent
   path) — decide that explicitly, the same way a third capability's arrival
   is supposed to be decided up front rather than backed into.
2. Append the new capability to `CLAUDE.md` — new content, under its own
   heading. **Do not rewrite the sections covering already-shipped
   capabilities** to match code that has since drifted from them; a
   divergence discovered in the process is an ADR in `ARCHITECTURE.md`, filed
   separately from this extension.
3. Update `README.md` §2 and §2a with the new capability's requirements and
   example queries, and add a status line for it under §6.
4. The new section is itself subject to the `[FILL ME]` convention (Section
   11.2) and the same freeze once resolved — re-run Section 11.6's
   placeholder check scoped to what's new before building it.
5. This is the only way `CLAUDE.md` changes after v1. A change arriving any
   other way — a passing remark, an implementation detail that turned out
   different — is an `ARCHITECTURE.md` ADR, not a `CLAUDE.md` edit.
