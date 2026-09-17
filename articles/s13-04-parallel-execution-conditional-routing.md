---
title: "Parallel execution and conditional routing"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 12 min
added: 2026-09-16
summary: >
  Budget search drags the whole estimation flow because it runs one
  component at a time when the components don't depend on each other.
  Fan-out with LangGraph's Send API splits it into a dispatch function
  (one Send per component) and a worker node (one component each), run in
  parallel — and the reducer from the previous article is what makes
  fan-in possible: without an accumulator field, concurrent branches would
  clobber each other instead of concatenating. A conditional edge is a
  function that reads state and returns the next node's name — the same
  mechanism dispatching fan-out, now used to branch — and belongs only at
  real decision points. Cycles (retrying by routing back to an earlier
  node) are normal in agentic systems but need an explicit bound in your
  own logic — a retry counter in state and a routing function that gives
  up cleanly after N attempts — not just LangGraph's global
  recursion_limit, which is a safety net, not a strategy. Parallelism's
  cost is the state merge: concurrent fields must be accumulators, and a
  worker's output should stay minimal and confined to that field, or a
  race condition appears where there wasn't one.
keywords: [LangGraph, Send API, fan-out, fan-in, map-reduce, conditional
           edges, routing function, cycles, recursion_limit, retry_count,
           bounded retry, race condition, parallel execution]
---

# Parallel execution and conditional routing

*Antonio Perez* · 🔴 12 min

Of the estimation flow's five steps, one drags the rest: budget search. It
goes through the components one at a time and, for each, runs a retrieval
against the vector store. If a project has eight components, that's eight
retrievals in a row, each waiting for the previous one to finish. Every
other node is fast; this is the one that sets the total time.

And the interesting part is that this work doesn't have to be sequential.
Searching for component A's budget doesn't depend on the result of
searching for component B's — they're independent tasks. When work is
independent, running it in series is a decision you pay for in latency
without getting anything back. This article covers the two tools the graph
provides for this: running what's independent in parallel, and routing on
what the state says.

## 1. Fan-out with the Send API

The pattern for parallelizing per-element work is fan-out: instead of one
node looping over the list, you dispatch one execution branch per element,
all at once. LangGraph expresses this with the **Send API**.

The idea is to split `search_budgets` into two pieces. A dispatch function
that, given the list of components, emits one `Send` per component toward a
worker node. And a worker node that processes a single component. LangGraph
runs every `Send` in parallel.

```python
from langgraph.types import Send

def fan_out_budget_search(state: EstimationState) -> list[Send]:
    # One parallel branch per component.
    return [Send("search_one_budget", {"component": c}) for c in state["components"]]

def search_one_budget(payload: dict) -> dict:
    match = retrieve_reference_budget(payload["component"])
    # Return only the accumulator field; the reducer concatenates all branches.
    return {"budget_matches": [match]}
```

The dispatch is wired in as a conditional edge coming out of the
classification node, and the worker connects to the estimate node:

```python
builder.add_conditional_edges("classify_components", fan_out_budget_search, ["search_one_budget"])
builder.add_edge("search_one_budget", "generate_estimate")
```

This is where the previous article's reducer stops being a detail and
becomes what makes the parallelism possible. Every branch returns
`{"budget_matches": [match]}`. If `budget_matches` were an overwrite field,
the branches would clobber each other and only the last result would
survive. Because it's annotated with `operator.add`, LangGraph concatenates
every branch's output into a single list. That's the fan-in: the branches
converge and the reducer fuses them. The estimate node runs exactly once,
once every branch has finished, with every budget already gathered.

The impact is direct: eight retrievals that used to run in a row now happen
at once, and that step's time goes from the sum of all of them to,
roughly, the slowest one. In flows with independent retrievals, this is
among the optimizations with the best effort-to-result ratio.

## 2. Conditional edges: routing on state

Parallelism solves "do several things at once." Conditional routing solves
"decide what comes next." A conditional edge is a function that inspects
the state and returns the next node's name. It's the same mechanism that
dispatches the fan-out, now used to branch.

In the estimation flow, the natural decision point is validation. If the
estimate passes, the flow ends; if not, it diverts to a node that flags it
for review:

```python
from langgraph.graph import END

def route_after_validation(state: EstimationState) -> str:
    return END if state["status"] == "validated" else "flag_for_review"

builder.add_conditional_edges("validate_and_consolidate", route_after_validation)
```

The routing function doesn't call the model or do work: it only reads the
state and decides. That separation — work in the nodes, decision in the
edges — is what keeps the graph readable. A practical consequence: put
conditional edges only at real decision points. If every transition turns
into a routing function, the graph loses exactly the clarity it was
adopted for.

## 3. Cycles, but bounded

Conditional edges allow something a linear pipeline didn't: going back. If
validation fails, instead of finishing you could reroute to an earlier step
to retry with different parameters. Cycles are normal and useful in
agentic systems.

The danger is obvious: an unbraked cycle is an infinite loop, and in a flow
that calls a model, an infinite loop is an infinite bill. That's why every
cycle needs an explicit cap. LangGraph ships a global recursion limit that
cuts the execution if it's tripped — a safety net against a runaway loop:

```python
config = {"configurable": {"thread_id": estimation_id}, "recursion_limit": 25}
result = await graph.ainvoke({"transcript": transcript}, config)
```

That limit is the last line of defense, not the strategy. The strategy is
bounding the cycle in your own logic: a retry counter in the state, and a
routing function that, once the cap is exceeded, stops retrying and
diverts to review.

```python
def route_after_validation(state: EstimationState) -> str:
    if state["status"] == "validated":
        return END
    if state["retry_count"] >= 2:
        return "flag_for_review"  # give up cleanly after two attempts
    return "generate_estimate"  # bounded retry
```

That way the cycle has a guaranteed exit by design, not only thanks to the
framework's safety net.

> *(Figures in the original: `S13-fig-04b-condicional-ciclos_1.jpg` and,
> later, `S13-fig-04b-condicional-ciclos.jpg` — images not included in
> this repo. Both instances show the same diagram: `generate_estimate` →
> `validate_and_consolidate` [orange-bordered, captioned
> `route_after_validation(state)`: reads `status` and `retry_count`],
> branching to `END`/`status: validated` [green] or, when `retry_count >=
> 2`, to `flag_for_review`/`status: needs_review` [orange] — plus a dashed
> orange loop back from `validate_and_consolidate` to `generate_estimate`
> captioned "bounded retry: retry_count < 2". Caption: the edge reads
> state and decides; the cycle has a guaranteed exit by design. Callout,
> in a red-bordered box: the framework's safety net —
> `recursion_limit` cuts the execution if the cycle runs away; the
> strategy is the explicit cap, not the net.)*

## 4. The cost of parallelism

Parallelizing isn't free in complexity, even though the code looks simple.
The cost is in the state merge. As soon as several branches write at once,
you have to guarantee their outputs combine predictably, and that imposes
two disciplines.

First: fields receiving concurrent writes must be accumulators. An
overwrite field under parallelism is a bug waiting to happen, because the
result depends on which branch finished last. Second: every branch must
consume a compatible input and return the same shape the aggregator
expects. If one branch returns something with a different structure, the
fan-in breaks or, worse, combines silently and incorrectly. The rule that
summarizes both: keep the worker's output minimal and confined to the
accumulator field. A worker that only returns `{"budget_matches": [match]}`
is trivial to combine; one that also touches `status` or `estimate`
introduces race conditions where there weren't any.

It's the same old principle — minimal, well-typed state — seen now through
the lens of concurrency. Parallelism rewards graphs with clean state and
punishes ones that drag along extra fields.

## 5. What's next

With fan-out, the flow's slowest step no longer sets the total time; with
conditional edges, the graph decides its path based on what the state
says; and with bounded cycles, it can retry without running away. The flow
is now fast and flexible. What it still isn't is robust: it's still
missing what happens when one of those parallel branches fails, when a
retrieval times out, or when a node raises an exception halfway through.
Routing the happy path is one thing; holding the flow together when
something breaks is another, and that's the next ground.

## Summary

- Fan-out parallelizes independent work. The Send API dispatches one
  branch per element; the worker node processes just one. Per-component
  retrievals go from running in a row to happening at once.
- The reducer makes fan-in possible. `operator.add` concatenates every
  branch's output into a list; the next node runs once, with everything
  gathered. Without an accumulator, branches clobber each other.
- A conditional edge reads the state and returns the next node. Work
  lives in nodes, decisions live in edges. Use them only at real decision
  points.
- Cycles must be bounded. LangGraph's recursion limit is the safety net;
  the strategy is a counter in the state and a routing function that
  gives up cleanly after N attempts.
- Parallelism's cost is the state merge. Concurrent fields must be
  accumulators, outputs must have a compatible shape, workers should stay
  minimal. Clean state parallelizes well; fat state introduces race
  conditions.

## Sources

- LangGraph — Send API and parallel execution (map/fan-out):
  https://docs.langchain.com/oss/python/langgraph/graph-api
- LangGraph — conditional edges and the recursion limit:
  https://docs.langchain.com/oss/python/langgraph/graph-api
- Fan-out, edge, and bounded-cycle best practices:
  https://www.swarnendu.de/blog/langgraph-best-practices/

---

> *(Editor's note — a figure placement/duplication worth flagging, not
> silently fixing.)* The two figure references in the original point to
> the *same* diagram — conditional routing plus the bounded-retry loop —
> under two filenames differing only by a `_1` suffix. The first occurrence
> sits right after §1 (fan-out), a section that diagram doesn't illustrate
> at all; the second sits after §3 (cycles), where it belongs. No figure in
> the original actually diagrams the Send-based fan-out/fan-in pattern §1
> is about, despite §1 being the section with genuinely new visual content
> to show (branches forking and converging). Read as: the cycles diagram
> got duplicated into an earlier slot, not that fan-out had its own figure
> that went missing without a trace.

> *(Editor's note — a state-schema gap this article introduces silently.)*
> `retry_count` is read in §3's bounded routing function but was never
> added to `EstimationState` in s13-02 or s13-03 — the same class of gap as
> `AgentResult`/`Step` in s12-02/04. A reader implementing this needs to add
> `retry_count: int` to the state schema themselves (overwrite semantics
> are correct here, not accumulate — you want the current count, not a
> running list of every count ever written).

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed.)* No `Send`, fan-out, or `retry_count`
> exists yet, consistent with every article since s12-01. But §3's
> bounded-retry *principle* — cap it in your own logic, don't rely on a
> framework's global safety net alone — already has a working precedent
> in this codebase, via a different mechanism: `app/generation/conversation/
> compression/policy.py` re-prompts the LLM on a failed Pydantic validator
> up to `max_retries=6` times (Session 4/9), an explicit, counted retry cap
> with the same shape as this article's `retry_count >= 2`, years before
> LangGraph enters the picture. Building §3's cap as a straightforward
> `int` counter checked in a routing function is consistent with how this
> codebase already bounds retries elsewhere — not a new pattern to learn,
> the same one relocated into a graph.
