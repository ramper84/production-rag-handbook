---
title: "Observability: LangSmith and Logfire for the AI service"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 6
scope: evergreen
source: user-supplied article
reading_time: 10 min
added: 2026-09-16
summary: >
  Every question this session asked — does the framework earn its place,
  does parallelizing pay off, what to make robust — assumed you could see
  the execution from the inside. The span is the unit: a named stretch of
  execution with a start, an end, and attributes, organized into a
  parent/child tree (a trace) that mirrors request → graph run → node →
  model call / DB query. Two reference tools, different in scope, not
  quality: LangSmith (LangChain) traces and evaluates agents natively,
  most natural inside the LangChain ecosystem, activated mostly by env
  vars. Logfire (Pydantic), built on OpenTelemetry, instruments the whole
  application — FastAPI, asyncpg, httpx — with one line each, and exposes
  spans over SQL, so cost-per-estimate is a query, not a dashboard. The
  deciding trade-off for a service with its own database: LLM-only tools
  see the model call and its result but not what happened in between; for
  a service where the retrieval-store round trip is often the actual
  problem, that seam is exactly where Logfire's full-stack tracing looks
  and LLM-only tracing doesn't.
keywords: [observability, span, trace, LangSmith, Logfire, OpenTelemetry,
           instrument_fastapi, instrument_asyncpg, instrument_httpx,
           cost per estimation, waterfall, full-stack tracing]
---

# Observability: LangSmith and Logfire for the AI service

*Antonio Perez* · 🔴 10 min

Throughout everything so far, the same requirement has kept showing up:
measure before deciding. Does the framework earn its place? Measure the
baseline. Is parallelizing worth it? Compare before and after. What should
be made robust? Look at where it actually fails. All of those questions
share an assumption we've left unexamined until now: that you can see the
execution from the inside. You can't decide what to optimize if you don't
know how long each node takes, you can't know if a recovery strategy works
if you can't see what's failing, and you can't talk about cost if you don't
measure it per estimate.

That's the graph's last stretch: observability. It isn't an extra bolted on
at the end — it's the instrumentation that turns a graph that works into a
graph you know things about.

## 1. What observing an agentic flow means

The basic unit is the **span**: a stretch of execution with a name, a
start, an end, and attributes. A span measures how long an operation took
and what happened inside it. Spans are organized into a tree — a **trace**
— that reflects parent/child relationships: the request contains the
graph's execution, which contains each node, which contains the model call
or the database query. On top of that structure you compute the metrics
that matter for this flow: latency per node, success rate per node, and
cost per estimate.

A graph lends itself especially well to this because nodes are already the
natural unit of measurement. One span per node tells you, at a glance,
where the time goes.

> *(Figure in the original: `S13-fig-06a-traza-waterfall.jpg` — image not
> included in this repo. A waterfall/Gantt chart, "a trace of the
> estimation, span by span": `POST /estimate` as the outermost bar (0-1000
> ms), containing `extract_requirements` and `classify_components`
> sequentially early on, then four parallel `search_one_budget (c1-c4)`
> bars starting together and ending at different widths — the longest
> (c4, orange-bordered) marked with a dashed line captioned "the slowest
> branch marks the step's end" — followed sequentially by
> `generate_estimate` and `validate_and_consolidate`. Caption: each bar is
> a span, its width is the node's latency; parallelism is visible at a
> glance. Callout: cost per estimate is the sum of the model spans; success
> rate is read per node across many runs.)*

## 2. Two tools, two philosophies

There are two reference tools, and the difference between them isn't
quality — it's scope.

**LangSmith**, from LangChain, is a tracing, evaluation, and debugging
platform built for agents. It traces a graph's execution natively — the
run shows up as a navigable tree — and adds evaluation as a first-class
citizen. It's framework-agnostic, but its most natural fit is when you're
already inside the LangChain ecosystem. Turning it on is mostly
environment configuration:

```bash
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=...
LANGSMITH_PROJECT=estimation-service
```

With that, the graph's execution traces itself and can be inspected step
by step.

**Logfire**, from Pydantic, is an observability platform built on
OpenTelemetry. Its distinguishing trait is that it doesn't observe only the
model layer — it observes the whole application. And it fits our stack
almost literally, since it instruments each of its pieces with one line:

```python
import logfire

logfire.configure()
logfire.instrument_fastapi(app)      # the HTTP layer
logfire.instrument_asyncpg()         # DB queries: retrieval and the checkpointer
logfire.instrument_httpx()           # the OpenAI Responses API calls (the SDK uses httpx)
```

For per-node latency, it's enough to wrap each node's body in a span:

```python
async def search_one_budget(payload: dict) -> dict:
    with logfire.span("node.search_one_budget", component=payload["component"]["name"]):
        match = await retrieve_reference_budget(payload["component"])
    return {"budget_matches": [match]}
```

And since Logfire exposes spans over SQL, cost per estimate is a query, not
a closed dashboard:

```sql
SELECT
    attributes->>'thread_id' AS estimation_id,
    SUM((attributes->>'llm_cost_usd')::float) AS cost,
    MAX(duration) AS wall_time
FROM records
WHERE service_name = 'ai-service'
GROUP BY estimation_id;
```

## 3. Full-stack vs. LLM-only

Here's the trade-off that decides which to use for this project, and it
isn't a matter of taste. LLM-centric tools — LangSmith, and also Langfuse
or Arize — see the model layer very well: the prompt, the call, the tool
invoked, the result. But when a node calls a tool that queries the vector
store, those tools see the call and see the result; what happened in
between is a black box.

And in an AI service built on FastAPI, asyncpg, and Postgres, a good share
of the problems live exactly there, in the seams. A retrieval that's slow
because a database query takes too long. A timeout that's actually a
connection problem. An estimate that's expensive not because of the model,
but because work got repeated. An LLM-only tool would show you "the node
called the search and got results" without telling you the query took
three seconds. Logfire, by tracing the whole request over OpenTelemetry,
lets you see whether the problem is in the AI or in the infrastructure —
exactly what you can't distinguish by looking only at the model layer.

That's why, for this stack, the reference choice is Logfire: it
instruments the whole service in three lines, rides on the same Postgres,
and makes visible the seam where problems usually are. LangSmith is the
natural choice when the project leans heavily on LangChain and you want its
agent evaluation and debugging as central pieces. It isn't that one is
better — they see different things, and for a full-stack service, seeing
the whole stack is what matters.

> *(Figure in the original: `S13-fig-06b-llm-vs-fullstack.jpg` — image not
> included in this repo. A horizontal chain: "HTTP request" [FastAPI] →
> "graph node" [AI service] → "Postgres query" [asyncpg — the seam,
> orange-bordered] → "model call" [httpx] → "response". Above the first,
> second and fourth boxes: green "visible" pills, labeled "solo-LLM
> (LangSmith / Langfuse / Arize): sees the call and the result, not the
> query in between." Above the third box (the Postgres query): a red
> "black box" pill. Below the whole chain, a green bracket spanning all
> five boxes labeled "Full-stack (Logfire, on OpenTelemetry): sees the
> whole request, including the Postgres query." Caption: the problem
> usually lives in the seam between the call and the result — for a
> service with a database, seeing the whole stack is what decides it.)*

## 4. Closing

With observability, the estimation graph stops being a box that produces
estimates and becomes a system you know things about: how long each node
takes, which ones fail, what each run costs. And that closes the circle
this whole arc opened with. The question of whether formal orchestration
earned its place no longer gets answered by faith in the abstraction or by
rejecting it outright — it gets answered with the trace in front of you.
The flow went from being an imperative loop that had to be read carefully
to being an explicit, persistent structure, capable of recovering from its
own failures, and now, measurable at every one of its steps. A system you
can see is a system you can improve; the rest is iterating with data.

## Summary

- The unit is the span; the tree of spans is the trace. That structure is
  what the flow's metrics get read from: latency per node, success rate,
  cost per estimate. In a graph, each node is a natural unit of
  measurement.
- LangSmith traces and evaluates agents natively; its most natural fit is
  inside the LangChain ecosystem. Turned on almost entirely with
  environment variables.
- Logfire observes the whole application over OpenTelemetry and
  instruments FastAPI, asyncpg, and the HTTP client with one line each.
  It exposes spans over SQL, so metrics are queries.
- LLM-only vs. full-stack is the deciding criterion. LLM-only tools don't
  see the seam between the call and the result, which is usually where a
  database-backed service's problems are. Logfire does.
- For this stack, Logfire is the reference; LangSmith is the natural
  choice if the project lives inside LangChain. They see different
  things.

## Sources

- Logfire — AI and full-stack observability (Pydantic):
  https://pydantic.dev/docs/logfire/get-started/ai-observability/
- LangSmith — agent tracing and evaluation (LangChain):
  https://docs.smith.langchain.com/
- OpenTelemetry — semantic conventions for GenAI applications:
  https://opentelemetry.io/docs/specs/semconv/gen-ai/

---

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — not a green field this time either.)*
> No Logfire, LangSmith, or OpenTelemetry dependency exists yet. But this
> article's core idea — a per-stage span-like wrapper, measuring duration,
> correlated by a request-scoped id — already has a real, working
> implementation: `app/generation/rag/observability.py`'s `log_stage`
> (Session 9), a `structlog`-based context manager emitting a
> `stage.started`/`stage.completed`/`stage.failed` trio with `duration_ms`,
> correlated by `request_id` — used across `estimation_service.py` and
> every RAG API router. Its own docstring states almost exactly this
> article's §1 argument: it exists "to make a single request's journey...
> trivially greppable in the JSON logs." Adopting Logfire is not
> introducing observability where none existed — it's an upgrade from
> structured JSON logs to a real span/trace data model with a waterfall
> view and SQL queryability, and a real design question this article
> doesn't address: replace `log_stage` outright, keep both running in
> parallel (redundant, two sources of truth for the same durations), or
> have `log_stage` emit into a Logfire/OpenTelemetry span rather than (or
> in addition to) a log line. Worth deciding deliberately, not by whichever
> lands in the codebase first.

> *(Editor's note — a snippet that illustrates one thing and quietly
> narrows another.)* §2's `search_one_budget` span-wrapping example is the
> same node `s13-04` introduced and `s13-05` hardened with a `RetryPolicy`
> and an internal timeout/graceful-degradation `try`/`except`. This
> article's version shows neither — a bare `await retrieve_reference_budget(...)`
> inside the span, nothing else. Read as illustrating span placement in
> isolation, which is reasonable; read as the node's full body, it would be
> a silent regression, dropping error handling two articles established
> for the exact same function. Wiring this in for real means adding the
> span *around* s13-05's existing try/except, not replacing it.
