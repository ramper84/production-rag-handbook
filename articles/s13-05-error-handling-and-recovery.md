---
title: "Error handling and recovery in complex flows"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 12 min
added: 2026-09-16
summary: >
  Not every failure is the same, and treating them alike means retrying
  what won't fix itself or giving up on what a simple retry would have
  solved. Four failure types, four distinct strategies, all but one fully
  automatic: transient failure → retry with exponential backoff
  (LangGraph's per-node RetryPolicy, no hand-written retry loop); a
  persistently down dependency → fallback plus a circuit breaker, since
  retrying only prolongs the agony; an exception mid-node → resume from
  the last checkpoint, since state persisted through the last completed
  node means no work is lost; low confidence or ambiguity → interrupt()
  pauses the graph, persists state, and waits — indefinitely if
  needed — for a human decision via Command(resume=...). The one gotcha
  worth internalizing: on resume, the node re-executes from its start, and
  interrupt() returns the resume value instead of pausing again — so the
  work before the interrupt call repeats and must stay cheap and
  idempotent. Neither automating everything nor gating every step works;
  the human gate belongs only where the cost of a wrong answer is high and
  the system genuinely isn't sure — one gate at the critical point, not
  ten scattered through the flow.
keywords: [error handling, RetryPolicy, exponential backoff, timeout,
           graceful degradation, circuit breaker, interrupt, Command,
           human-in-the-loop, checkpointer, resume, idempotent]
---

# Error handling and recovery in complex flows

*Antonio Perez* · 🔴 12 min

The estimation flow is already fast and flexible: it parallelizes what's
independent and decides its path based on what the state says. But all of
that describes the happy path. In production, the happy path is only one
of the things that happens. One of the parallel search branches can fail.
A retrieval can time out against a slow store. A node can raise an
exception mid-execution because the model returned something unexpected.
Routing when everything goes well is one thing; holding the flow together
when something breaks is another, and it's what separates a prototype from
a system.

The good news is that a graph with persistent state starts from a strong
position for this. The bad news is that "handle errors" isn't one thing:
it's several distinct strategies, and applying the wrong one to the wrong
kind of failure is its own source of problems. This article separates
them.

## 1. Every failure type asks for a different response

Before the code, the distinction that orders everything: not every failure
is the same, and lumping them together leads to retrying what won't fix
itself, or giving up on what a simple retry would have solved.

A **transient failure** — a latency spike, a momentary network blip — fixes
itself if you try again. The response is retry with backoff. A
**persistently down dependency** doesn't get fixed by retrying: retrying
only prolongs the agony and multiplies the cost. The response is a
fallback path and, where it pays off, a circuit breaker that stops hammering
the broken dependency. An **exception in a node** stops the execution, but
because state is persisted through the last node that did finish, no work
is lost — it resumes from there. And a case that isn't a technical failure
but is still a stop: **low confidence or ambiguity**, where the right move
isn't for the system to decide alone, but to stop and ask a human.

> *(Figure in the original: `S13-fig-05a-estrategias-fallo.jpg` — image not
> included in this repo. A table, "recovery strategies by failure type":
> Transient failure [timeout, network spike] → Retry with exponential
> backoff → automatic; Dependency down [persistent failure] → Fallback +
> circuit breaker → automatic; Exception in a node → Resume from the last
> checkpoint → automatic; Low confidence / ambiguous [highlighted row] →
> `interrupt`: a human gate at the critical point → human. Caption: the
> right response depends on the failure — retry the transient, not the
> persistent; stop only for the critical.)*

## 2. Retries, timeouts, and fallback

For transient failures, LangGraph lets you attach a retry policy to a node.
The framework re-executes the node on failure, with exponential backoff,
without you writing the retry loop yourself:

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "search_one_budget",
    search_one_budget,
    retry_policy=RetryPolicy(max_attempts=3, backoff_factor=2.0),
)
```

Timeouts are the node's own responsibility, because the node is the one
that knows which operation can hang. The key is degrading gracefully: a
search that times out shouldn't take down the whole estimate — it should
record the gap and let the flow continue:

```python
import asyncio

async def search_one_budget(payload: dict) -> dict:
    try:
        match = await asyncio.wait_for(
            retrieve_reference_budget(payload["component"]), timeout=5.0
        )
        return {"budget_matches": [match]}
    except asyncio.TimeoutError:
        # Degrade gracefully: record the gap, do not kill the whole estimate.
        return {"errors": [f"budget search timed out for component {payload['component']['name']}"]}
```

Notice the failure gets written to the `errors` field, which is an
accumulator, not an exception propagating upward. The branch that failed
contributes its gap to the state, the others contribute their budgets, and
the estimate node receives the set along with information about what's
missing. That's the fallback pattern: when a piece isn't available, the
flow takes an alternative path instead of dying. At the external-dependency
level, the natural extension is the circuit breaker: if the retrieval store
fails repeatedly, you stop calling it for a cooldown period and go straight
to the degraded path, instead of every estimate paying the cost of
rediscovering that it's still down.

## 3. The human gate: `interrupt`

There's a kind of stop that more automation doesn't fix. When the estimate
comes out with low confidence — scarce budgets, components that don't match
the history well — the right response isn't for the system to decide on
its own and continue, but to stop and ask a person. That's what `interrupt`
is for.

`interrupt` pauses the graph mid-node, persists the state, and exposes a
value to whoever invoked it. The execution sits waiting — indefinitely, if
needed — until someone resumes it with a decision. It needs a checkpointer,
because without persistence there's nowhere to save the pause point.

```python
from langgraph.types import interrupt

def validate_and_consolidate(state: EstimationState) -> dict:
    estimate = consolidate(state)
    if estimate["confidence"] >= CONFIDENCE_THRESHOLD:
        return {"estimate": estimate, "status": "validated"}

    # Low confidence: pause and ask a human before continuing.
    review = interrupt({"reason": "low_confidence_estimate", "estimate": estimate})
    if review["action"] == "reject":
        return {"status": "needs_review"}
    return {"estimate": review.get("estimate", estimate), "status": "validated"}
```

The execution resumes by passing the person's decision, which becomes the
value `interrupt` returns:

```python
from langgraph.types import Command

await graph.ainvoke(Command(resume={"action": "approve"}), config)
```

One detail that avoids surprises: on resume, the node re-executes from the
beginning, and `interrupt` then returns the resume value instead of
pausing again. That is: the work before `interrupt` repeats. Keep that work
cheap and idempotent, and let anything expensive live after the pause.

> *(Figure in the original: `S13-fig-05b-puerta-humana.jpg` — image not
> included in this repo. `validate_and_consolidate` [orange-bordered,
> "confidence < threshold"] → `interrupt()` — PAUSE [state persisted via
> checkpointer, value exposed to the caller] → "a person reviews"
> [approve/edit/reject] → branching to `status: validated` [green,
> approve/edit] or `status: needs_review` [orange, reject]. Caption: on
> resume, the node re-executes; `interrupt()` no longer pauses, it returns
> the decision — keep the prior work cheap and idempotent.)*

## 4. Automate everything, or put up a gate

This article's honest trade-off isn't technical, it's a matter of
judgment: what resolves itself, and what deserves stopping for a human? And
the default answer a lot of people reach for — automate everything — is as
wrong as its opposite.

Automating everything fails in the cases where the system doesn't have the
information to decide well: a weak estimate sent to a client as if it were
solid is a business problem, not a technical one another retry fixes.
Putting a human gate at every step fails the other way: it turns a flow
that was supposed to be fast into an approval queue, and burns out the
person who has to approve things the system was resolving perfectly well
on its own.

The rule that works is proportional to the cost of the error. Transient
failures and degradations, whose worst case is an estimate with a noted
gap, resolve themselves: retry, fallback, circuit breaker. The human gate
is reserved for the point where the cost of being wrong is high and the
system genuinely isn't sure: the low-confidence estimate that's about to go
out under the company's name. One gate, at the critical point — not ten
scattered through the flow.

## 5. What's next

With retries for the transient, fallback and circuit breakers for the
persistent, checkpoint-resume for exceptions, and a human gate at the
low-confidence point, the flow no longer falls over at the first problem:
it absorbs the failures it can and stops where it should. But there's a
question all of this leaves open that can't be answered from the code
alone: where is it actually failing, how long does each node take, what
does each estimate cost? Deciding what to make robust, and checking the
strategies actually work, requires seeing the execution from the inside.
That's the last stretch: observability.

## Summary

- Every failure type asks for a different response. Transient → retry
  with backoff. Dependency down → fallback and circuit breaker. Exception
  → resume from the last checkpoint. Low confidence → human gate.
- Retries with backoff are declared per node with a retry policy; the
  framework re-executes without you writing the loop.
- Timeouts live in the node and degrade gracefully: a search that expires
  records the gap in the errors accumulator instead of taking down the
  whole estimate.
- `interrupt` pauses, persists, and waits for a human decision; it resumes
  with `Command(resume=...)`. It needs a checkpointer; the node
  re-executes on resume, so the work before the pause must be cheap and
  idempotent.
- Neither automate everything nor gate every step. The human gate is
  reserved for the critical, high-cost, low-certainty point; everything
  else resolves itself.

## Sources

- LangGraph — retry policies and fault tolerance:
  https://docs.langchain.com/oss/python/langgraph/graph-api
- LangGraph — interrupts and human-in-the-loop intervention:
  https://docs.langchain.com/oss/python/langgraph/interrupts
- Error, cycle, and human-approval best practices in LangGraph:
  https://www.swarnendu.de/blog/langgraph-best-practices/

---

> *(Editor's note — a real composition bug across two articles' shown
> code, not a flaw in either one alone.)* Stack this article's §3 directly
> onto s13-04's `route_after_validation`, as both articles' own continuity
> invites, and a human's explicit rejection can get silently overridden.
> When a reviewer picks `"reject"`, this article's node returns
> `{"status": "needs_review"}` and nothing else. s13-04's routing function
> only checks two things: `status == "validated"` → `END`, else
> `retry_count >= 2` → `flag_for_review`, else → `generate_estimate` (retry
> again). A `"needs_review"` status with `retry_count` still under 2 falls
> into the *last* branch — back to `generate_estimate` for another
> automated attempt, not to review — because the routing function has no
> way to distinguish "the deterministic check says not good enough yet"
> from "a person already looked at this and said no." Both statuses read
> identically as `!= "validated"`. Flagged, not fixed: the two articles are
> each internally consistent, and only conflict when combined, which is
> exactly the failure mode worth catching before combining code from
> different sessions on the assumption that matching field names mean
> matching semantics.

> *(Editor's note — two techniques on the same node that don't compose the
> way they might look like they do.)* `§2` attaches a `RetryPolicy` to
> `search_one_budget` and, separately, wraps its body in a timeout that
> catches its own `TimeoutError` and returns gracefully rather than
> raising. Because the timeout is caught inside the function, it never
> reaches LangGraph's retry machinery — `RetryPolicy` only re-executes a
> node on an exception that escapes it, and this node's timeout path
> returns normally. That's very likely the right design (retrying a 5s
> timeout three times via the framework would cost 15s+ before even
> reaching the graceful path shown), but the article presents both as if
> they were general node hardening rather than two handlers for two
> disjoint failure classes on the same node — worth stating as a deliberate
> split in a real implementation, not something a reader should assume
> composes automatically.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — the codebase already resolves low
> confidence, just never by asking a human.)* No `interrupt`, `RetryPolicy`,
> or `CONFIDENCE_THRESHOLD` exists yet. But confidence-gating itself is not
> new here: `app/domain/schemas/estimation.py` defines
> `LOW_CONFIDENCE_THRESHOLD` and a validator
> (`low_confidence_requires_out_of_scope_prefix`) forcing any answer under
> it to self-declare; `app/foundation/guardrails/output.py` rewrites a
> summary that's low-confidence but doesn't say so; and
> `app/generation/agentic/boss.py` (s05-05's Boss) halves `confidence_pct`
> when it gives up after exhausting its own iteration budget. All three are
> **fully automated** — filter and label, never pause and ask — the same
> Axis 4-vs-general-agent distinction s12-02's note already drew, now
> showing up as an actual gap between what exists and what this article
> proposes. This article's `interrupt()` gate would be a genuinely new
> escalation *tier* above that automated handling, not a replacement for
> it — and its `CONFIDENCE_THRESHOLD` is a different name and, almost
> certainly, a different value than the real `LOW_CONFIDENCE_THRESHOLD`,
> worth reconciling deliberately (one threshold for auto-label, a lower one
> for stopping entirely?) rather than by accident if this gets built.
