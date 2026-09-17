---
title: "State and persistence: reducers, checkpointers, and memory"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 13 min
added: 2026-09-16
summary: >
  A typed, well-delimited graph already reads at a glance — but state
  living only in memory is a blind spot: a restart mid-estimation loses
  everything done so far. Reducers decide how each node's update combines
  into state (overwrite by default, accumulate with
  Annotated[list[X], operator.add]) — and resuming with an initial state
  that includes accumulator fields duplicates them, since the reducer
  merges rather than replaces; pass only new entries on resume, never
  accumulated fields. A checkpointer persists state after every node —
  InMemorySaver for dev, SqliteSaver for one server, AsyncPostgresSaver for
  production — and can reuse the project's existing Postgres (creates its
  own tables, coexists with pgvector) without new infrastructure. thread_id
  ties an execution to its history: same thread_id resumes, different
  thread_id starts clean. Short-term memory (execution state, ephemeral,
  lives in the checkpointer, for resume/inspect/human-approval) and
  long-term memory (estimation history, durable business data, lives where
  business data lives) are different problems — the checkpointer is not
  your product database. State size is a direct performance cost: it
  serializes on every transition, and a bloated state turns checkpoint
  writes into the real bottleneck.
keywords: [LangGraph, reducers, operator.add, checkpointer, AsyncPostgresSaver,
           thread_id, short-term memory, long-term memory, state size,
           serialization cost, persistence, resume]
---

# State and persistence: reducers, checkpointers, and memory

*Antonio Perez* · 🔴 13 min

A graph with typed state and well-delimited nodes already reads at a
glance. But if that state lives only in memory, the system has a blind
spot: all it takes is the process restarting mid-estimation — a
deployment, a crash, a timeout — for all the work done so far to be lost.
In a short flow that might not hurt; in one that retrieves budgets for
several components and consolidates an estimate, starting from zero every
time something falls over isn't acceptable.

Persisting state is what turns the graph into something that holds up in
production. And there's a data point that orders the priorities: according
to LangChain's 2026 agent engineering report, over 60% of production agent
incidents originate in state management. Not the model, not the prompt —
state. This article is about exactly that: how state gets combined
(reducers), how it gets persisted (checkpointers), and what kind of memory
each thing is.

## 1. Reducers: how each update gets combined

When a node returns a partial update, LangGraph has to decide how to
integrate it into the state. That decision is governed by the field's
reducer.

By default, a field is overwritten: the last value written wins. That's
right for `status` or `estimate`, where only the final value matters. For
other fields you want to accumulate, and that's where the reducer changes
the behavior:

```python
from typing import Annotated, TypedDict
import operator

class EstimationState(TypedDict):
    budget_matches: Annotated[list[BudgetMatch], operator.add]  # accumulates
    status: str  # overwrites
```

The distinction isn't cosmetic: an accumulator field survives restarts by
combining with what was already there, while an overwrite field takes its
last written value. And there's a detail that bites in production. When you
resume an execution from a checkpoint and pass it an initial state that
includes accumulator fields, the reducer doesn't replace — it combines. The
result is you can duplicate data without noticing: the budgets show up
twice, because `operator.add` concatenates what you pass with what was
already saved. The rule is simple: on resume, pass only the new entries,
never the accumulated fields.

## 2. Checkpointers: persisting without writing a database layer

A checkpointer persists the state after every node's execution. That's what
makes it possible to pause, resume, inspect the execution step by step,
and, later on, stop at a point for a human to approve. And it gives you
that without you having to write a persistence layer yourself.

There are several backends. `InMemorySaver` for development and tests.
`SqliteSaver` for a single server. And `PostgresSaver` (with its async
variant, `AsyncPostgresSaver`) for production with several instances. Since
the AI service is async — FastAPI over `asyncpg` — the variant that fits is
the async one.

What matters for the project: the checkpointer rides on the same
PostgreSQL the system already uses — the one with the `pgvector` extension
and the embeddings. The checkpointer creates its own tables and coexists
without friction with the vector store's. There's no new infrastructure to
stand up.

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from psycopg_pool import AsyncConnectionPool

# At service startup: one pool over the project's Postgres (the pgvector one).
pool = AsyncConnectionPool(conninfo=DATABASE_URL, max_size=10, open=False)
await pool.open()

checkpointer = AsyncPostgresSaver(pool)
await checkpointer.setup()  # run once: creates the checkpoint tables

graph = builder.compile(checkpointer=checkpointer)
```

The piece that ties each execution to its history is `thread_id`. It's
passed in the invocation's config, and it's the key checkpoints for that
execution are saved under:

```python
config = {"configurable": {"thread_id": estimation_id}}
result = await graph.ainvoke({"transcript": transcript}, config)
# Same thread_id resumes from the last checkpoint instead of starting over.
snapshot = await graph.aget_state(config)
```

Same `thread_id`, same history: the execution resumes from the last
checkpoint. Different `thread_id`, a new, clean execution. Use the
estimate's own identifier as `thread_id`, and every estimate gets its own
persistent, resumable trail.

> *(Figure in the original: `S13-fig-03a-checkpointer-postgres.jpg` — image
> not included in this repo. Five nodes [`extract`, `classify`, `search`,
> `generate`, `validate`] each with a dashed line down to "writes state
> after every node," feeding into a "PostgreSQL (project)" cylinder holding
> two boxes: "langgraph checkpoints" [key: thread_id (execution state)] and
> "pgvector embeddings" [budget corpus, already existing]. Callout:
> `agent_state(config)`: resumes from the last checkpoint. Caption: the
> checkpointer creates its own tables and coexists with pgvector — no new
> infrastructure.)*

## 3. Short-term memory and long-term memory are not the same thing

This is where it pays to be precise, because "memory" gets used for two
different things, and conflating them leads to bad architecture decisions.

**Short-term memory** is one execution's state: what one specific estimate
lasts, tied to its `thread_id`. It lives in the checkpointer. Its purpose
is operational — resume, inspect, allow a human approval — and it is, by
nature, ephemeral: once the estimate closes, that state carries no more
product value.

**Long-term memory** is something else: the history of estimates over
time, which serves as context for future estimates. That's business data,
durable, spanning sessions and executions. And it isn't the checkpointer's
job. The checkpointer is built for one execution's state, not to be your
memory store across sessions.

That gives an architectural position that fits the system's layering: **the
checkpointer is not your product database.** The estimate history — what
the system learns from past projects — lives where business data and the
corpus feeding retrieval already live, not in the checkpoint tables. If all
you need is an execution's final result, to save as a historical estimate,
save it in the business store; it's simpler and it's where it belongs. The
checkpointer solves "resume this execution where it left off"; the history
solves "what do we know about past estimates." They're two problems, and
mixing them up dirties both.

> *(Figure in the original: `S13-fig-03b-memoria-corta-larga.jpg` — image
> not included in this repo. Two cards side by side. "Short-term memory"
> [execution state]: Scope — one estimate (thread_id); Lives in — the
> checkpointer; Purpose — resume, inspect, approve (HITL); Nature —
> ephemeral. "Long-term memory" [estimate history]: Scope — across sessions
> and executions; Lives in — business store + RAG corpus; Purpose — context
> for future estimates; Nature — durable, business data. Callout: the
> checkpointer is not your product database — resuming an execution and
> remembering the history are two different problems.)*

## 4. The cost of a fat state

Everything you put in the state gets serialized to the checkpoint store on
every transition between nodes. That sentence has a direct production
consequence: state size is a performance decision, not only a design one.

A light state — identifiers, already-distilled findings, routing fields —
serializes in milliseconds. A state dragging along raw model responses with
all their metadata grows fast: states of hundreds of kilobytes, even
megabytes, have been seen, where the checkpoint write goes from a few
milliseconds to several hundred and becomes the execution's real
bottleneck. The agent isn't slow because of the model — it's slow because
at every step it's serializing a huge object.

The discipline is the same one that keeps the graph readable: store in the
state the minimum needed to reason and route — IDs, distilled results,
flags — and leave the transient, bulky stuff out, in function scope. A
minimal, typed state persists cheaply, inspects easily, and resumes
quickly. It's the same decision as the state schema's, seen now from the
serialization-cost side.

## 5. What's next

With reducers that combine correctly, a checkpointer over the Postgres we
already have, and a clear idea of what's execution memory and what's
business data, the graph no longer loses the thread: it survives a
restart, can be inspected step by step, and distinguishes the ephemeral
from the durable. On top of that persistent-state foundation, things a flat
loop didn't allow become possible: running what doesn't depend on each
other in parallel — the per-component budget search — and routing
conditionally on what the state says as it goes. That's the next ground.

## Summary

- The reducer decides how each update combines. Overwrite by default
  (`status`, `estimate`); accumulate with `operator.add`
  (`budget_matches`). Careful on resume: passing accumulator fields in the
  initial state duplicates them.
- The checkpointer persists state after every node and gives pause,
  resume, and inspection without writing a database layer.
  `AsyncPostgresSaver` is the variant that fits the AI service's async
  stack.
- Reuse the project's Postgres. The checkpointer creates its own tables
  and coexists with pgvector; no new infrastructure. `thread_id` ties each
  execution to its history and makes it resumable.
- Short-term and long-term memory are different problems. Short (execution
  state) lives in the checkpointer and is ephemeral; long (estimate
  history) is durable business data and lives where business data lives.
  The checkpointer is not your product database.
- State size is performance. Everything serializes on every transition:
  light state, millisecond writes; fat state, the checkpoint becomes the
  bottleneck.

## Sources

- LangGraph — persistence and checkpointers:
  https://docs.langchain.com/oss/python/langgraph/persistence
- LangGraph — short-term and long-term memory:
  https://docs.langchain.com/oss/python/langgraph/memory
- State schema and checkpointer design in production:
  https://www.kalviumlabs.ai/blog/langgraph-in-production-stateful-multi-step-agents/

---

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — "reuses the same Postgres" is true at the
> database level and misleading at the connection-pool level.)* No
> checkpointer code exists, consistent with s13-01/02. But the codebase
> already runs **two** separate connection stacks against
> `Settings.DATABASE_URL` — `app/foundation/persistence/database.py`
> documents this split deliberately: a **sync** SQLAlchemy engine
> (`create_engine`, driver `postgresql+psycopg://`, used by Session 6's
> ingestion paths) and a **async** SQLAlchemy engine (`create_async_engine`,
> driver swapped to `+asyncpg`, used by the Session 8 RAG store so it never
> blocks the event loop). This article's `AsyncConnectionPool` is a
> **third** pool: raw `psycopg_pool`, not SQLAlchemy, and not `asyncpg`
> either — it shares a driver family with the *sync* engine (`psycopg`,
> v3) while the codebase's async path uses `asyncpg`. "The same Postgres" is
> accurate (one database, one `DATABASE_URL`); "the same pool" or "the same
> driver as your async path" would not be. Wiring this in adds a third
> pool, not zero — worth stating precisely in a plan rather than assuming
> "reuses existing infrastructure" covers the connection layer, not just
> the database.

> *(Editor's note — a gap in the shown code, not the concept.)* The
> checkpointer setup block's `await pool.open()` / `await checkpointer.setup()`
> calls are shown at what reads as module level, which is not valid Python
> — `await` requires an `async def` around it (a FastAPI `lifespan` handler
> is the natural place, given the codebase's existing async-engine pattern
> already runs its own startup wiring in `dependencies.py`). Almost
> certainly an artifact of the article showing the calls in the order
> they happen without the wrapping boilerplate, not a claim that this runs
> unwrapped — but worth flagging as written, since a reader copying it
> verbatim gets a `SyntaxError`, not a working checkpointer.

> *(Editor's note — this is s13-01's second justification, arriving on
> schedule.)* s13-01 named two things a framework has to get right to earn
> its complexity tax over a hand-rolled loop: state management (per the
> same 60%-of-incidents figure this article opens with) and, separately,
> parallelism. This article is the first of those two delivered; s13-02
> already delivered the routing/conditional-edges piece. Parallel budget
> search — flagged in both s13-01's and this article's own closing section
> as still to come — is the one piece of s13-01's justification this
> session hasn't yet built.
