---
title: "Agent communication patterns: shared state, handoff, and messages"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 17 min
added: 2026-09-19
summary: >
  A typed shared state that every agent reads and writes is already a
  named, 1970s-AI pattern (blackboard) chosen without naming it the moment
  a project adopts one. Three patterns, a strict ladder, not equally
  ranked alternatives. Shared state (blackboard) is the default: agents
  couple only to the state schema, never to each other, trivially
  traceable, and its one real question is per-field reducer policy
  (accumulate vs. overwrite — a field that should accumulate but
  overwrites is a silent, hard-to-see data loss, not a crash). Direct
  handoff (a "swarm") removes the round trip through a central router —
  an agent hands the baton to the next one itself via a Command-returning
  tool, cutting a two-specialist task from four model calls to two — paid
  for in topological coupling (every agent must know its possible
  neighbors and carry one handoff tool per destination) and scattered
  trace reconstruction; earn it with a measured routing-cost bottleneck,
  never choose it for elegance. Messages (event-driven) break the
  same-process assumption entirely — an agent publishes a fact and moves
  on, indifferent to who's listening or when — trading maximum decoupling
  for eventual consistency, distributed tracing via a correlation id, and
  real operational infrastructure; justified only once agents genuinely
  become separate services with independent deployment lifecycles. The
  three compose: a supervisor over shared state that publishes one event
  when the estimate is ready is a coherent, ordinary architecture. None of
  the three answers what happens when the system shouldn't decide alone —
  that's not a communication pattern, it's persisted state and an outward
  contract.
keywords: [blackboard pattern, shared state, reducers, handoff, swarm,
           Command.PARENT, InjectedToolCallId, event-driven architecture,
           correlation id, eventual consistency, topological coupling,
           routing tax]
---

# Agent communication patterns: shared state, handoff, and messages

*Antonio Perez* · 🔴 17 min

There's a decision in your multi-agent architecture you probably don't
remember making: how the agents communicate with each other.

You don't remember making it because you didn't. It came free with the
framework. When you defined a typed `EstimationState` and had every agent
write its partial result to it, you chose a communication pattern — a good
one, with its own name and fifty years of history — without ever naming
it.

It's worth naming. Because as the system grows, that invisible choice
starts having very visible consequences: on the bill, on the coupling
between agents, and on your ability to understand a trace.

There are three patterns. All three are legitimate. And the order you'll
run into them in production is exactly the order they appear here.

> *(Figure in the original: `fig-01-tres-patrones-comunicacion.png` —
> image not included in this repo. Three panels. "Shared state: nobody
> talks to anybody. Everybody writes" — three teal boxes [`agent A`,
> `agent B`, `agent C`] each with an arrow down into one blue box
> [`EstimationState`: requirements / budget_matches / estimate /
> validation], captioned "Coupling: to the state schema. Trace: one,
> linear, easy to read. Risk: writes clobbering each other. You already
> have this. Start here." "Direct handoff: the agent chooses who gets the
> baton" — `triage` → `pricing` [captioned `Command(goto=..., graph=PARENT)`]
> → branching down to `legacy`, captioned "Coupling: every agent knows its
> neighbors. Trace: jumps with no central point. Risk: routes nobody
> foresaw. When the center is the bottleneck." "Messages: nobody calls
> anybody. Publish and listen" — `producer` → `event bus / queue` →
> fanning out to three `worker` boxes, captioned "Coupling: to the event
> contract. Trace: distributed, needs correlating. When agents cross the
> process boundary.")*

## 1. Shared state: the whiteboard you already have

The pattern is called **blackboard**, and it comes from 1970s AI. The
metaphor is literal: a group of specialists in front of a blackboard. None
talks to another. Each one looks at what's written, sees if it can
contribute something, and writes its contribution. The solution emerges
from successive writes.

Your `EstimationState` is the blackboard:

```python
class EstimationState(TypedDict):
    transcript: str
    requirements: list[str]
    budget_matches: Annotated[list[dict], operator.add]
    estimate: dict | None
    validation: dict | None
    confidence: float | None
```

`budget_searcher` doesn't pass anything to `estimate_generator`. It writes
to `budget_matches` and leaves. The generator, when its turn comes, reads
`budget_matches` off the blackboard. They're completely decoupled from
each other: their only coupling is to the state schema.

And that coupling — to the schema, not between components — is exactly the
kind you already know how to manage. It's the same one you have between
services sharing a database, or between a frontend and an API's contract.
Adding a new agent doesn't require touching any existing agent, only
adding a field. Reordering the flow breaks nobody, because nobody knows
who comes after them.

### The detail that does matter: reducers

The blackboard has exactly one hard question: what happens when two agents
write the same key.

In a strictly sequential flow, it never happens and the problem doesn't
exist. But the moment the supervisor launches two agents in parallel —
perfectly reasonable: searching budgets for three independent components at
once — you have two concurrent writes to `budget_matches`. Without an
explicit policy, the last one wins and the first disappears silently.

That's why `budget_matches` is annotated:

```python
budget_matches: Annotated[list[dict], operator.add]
```

That annotation is the reducer: the function that decides how two writes to
the same key combine. Here, concatenate. Every agent returns a partial
update and the framework merges them according to the policy you declared.

The design rule is simple and will save you strange bugs: for every state
field, consciously decide whether it accumulates or overwrites. Fields a
single agent produces once (`estimate`, `validation`) overwrite, and that's
fine. Fields several agents or several invocations contribute to
(`budget_matches`, `routing_trail`) accumulate. A field that should
accumulate but overwrites is a silent loss of data, and it's one of the
hardest bugs to spot, because the system doesn't fail — it just estimates
with less evidence than it searched for.

### The other side

Blackboard has two real limits.

**The state grows.** Everything an agent might need has to be in the
schema. With six agents and a mature flow, `EstimationState` starts to look
like a God object: one huge type where each agent uses 15% of it. It's the
same smell as an ActiveRecord model with forty columns where each use case
touches four. And the mitigation is the same: projections. Each agent
receives the view of the state that's relevant to it, not the whole thing.

**Everyone can read everything.** The blackboard is a shared read space. If
one agent handles sensitive information and another shouldn't see it,
shared state doesn't protect you on its own. That stops being a
communication question and becomes a privilege one.

**When to use it:** by default. You already have it, it's traceable, it's
cheap, and it decouples agents from each other in the one way that
matters. Most systems stay here, and rightly so.

## 2. Direct handoff: when the center is the bottleneck

In the supervisor pattern, control always returns to the center. Agent →
supervisor → agent → supervisor. Every return to the center is a model call
whose only product is a routing decision.

Handoff eliminates that return trip. The agent that finishes decides,
itself, who's next, and passes the baton directly. In the ecosystem this is
known as a **swarm**, and the mechanism in LangGraph is a tool that, instead
of returning data, returns a `Command`:

```python
from typing import Annotated

from langchain_core.tools import tool, InjectedToolCallId
from langchain_core.messages import ToolMessage
from langgraph.types import Command

def build_handoff_tool(*, target_agent: str, description: str):
    """A tool that transfers control instead of returning data."""

    @tool(f"handoff_to_{target_agent}", description=description)
    def handoff(
        task_brief: Annotated[str, "What the next agent must do, with all relevant context."],
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        return Command(
            goto=target_agent,
            graph=Command.PARENT,
            update={
                "messages": [
                    ToolMessage(
                        content=f"Transferred to {target_agent}.",
                        tool_call_id=tool_call_id,
                    )
                ],
                "task_brief": task_brief,
            },
        )

    return handoff
```

Two things worth understanding well.

**`graph=Command.PARENT`** is what makes the jump escape the agent's own
subgraph and land in the parent graph. Without it, `goto` would look for a
node inside the agent itself and not find it. It's the number-one mistake
when implementing handoff by hand.

**And `task_brief` is the real design decision.** What travels with the
baton? Pass the full message history and the next agent inherits all the
context — and all the cost, and all the noise — of the previous one. Pass
only a brief that the agent releasing the baton writes, and you have a
clean context but you've introduced a semantic bottleneck: whatever isn't
in the brief doesn't exist for whoever's next. A requirement the searcher
considered irrelevant and didn't mention is a requirement the estimator
will never see.

There's no universal answer. There's an explicit decision, and it's yours.
What you can't do is not make it, because the default nearly every
library ships — pass the whole history — is the one that scales worst.

### What you gain and what you pay

> *(Figure in the original: `fig-02-coste-enrutado-supervisor-vs-handoff.png`
> — image not included in this repo. "The routing tax: same task, two
> specialists involved. Each box is a model call." Top panel, "Supervisor,
> 4 calls": `route` [orange] → `budget_searcher` [teal] → `route` [orange]
> → `estimate_generator` [teal]. Bottom panel, "Direct handoff, 2 calls":
> `budget_searcher` → `estimate_generator`, captioned "routing travels
> inside the handoff tool call: it doesn't cost an extra call." Caption:
> what you save in calls, you pay in coupling — every agent has to know
> who it can pass the baton to.)*

For a task touching two specialists, the supervisor spends four model
calls (route, work, route, work). Handoff spends two: the routing decision
travels inside the agent's own call, as one more tool call. At scale, that
difference is money and it's latency.

What you pay is **topological coupling**. In the supervisor pattern, an
agent doesn't know the others exist; it only knows how to do its job. With
handoff, every agent needs to know its neighbors and carry one tool per
possible destination. Adding a new agent means deciding who can jump to it
and touching those agents. You've traded a star for a mesh, and meshes
grow badly.

And you pay in traceability. With a supervisor, every routing decision is
in one place: `routing_trail`. With handoff, the decision is scattered
across the agents, and reconstructing why the system ended up where it did
means walking the jumps.

**When to use it:** when central routing has become a measured bottleneck —
cost or latency — and the transitions between agents are mostly local and
predictable. Not before. Handoff earns its place; it isn't chosen for
elegance.

## 3. Messages: when agents stop sharing a process

The two previous patterns share a premise nobody states: every agent lives
in the same process. They share memory, share a state object, share a
trace. That's why `Command(goto=...)` works: there's a runtime that can
move control from one place to another.

The message pattern breaks that premise. An agent publishes an event; who
receives it, and when, isn't its concern.

```python
# Sketch, not an implementation: the point is the shape of the contract.
await bus.publish(
    "estimation.requirements_extracted",
    {
        "estimation_id": estimation_id,
        "requirements": requirements,
        "correlation_id": correlation_id,
    },
)
```

Nobody calls anybody. Whoever extracts requirements doesn't know a budget
searcher exists. It publishes a fact — requirements have been extracted —
and goes on with its life. Whoever's subscribed to that fact will react. Or
won't. Or three different services will, at once.

This should sound very familiar, because it isn't an AI idea. It's
event-driven architecture, the same one you've been applying between
services for years. And the properties are exactly the ones you know:
maximum decoupling, independent scaling per consumer, retries and
dead-letter queues, resilience if a consumer goes down.

Also the costs you know: eventual consistency (the state is no longer a
coherent snapshot in one object, it's whatever has arrived so far),
distributed traceability (you need a `correlation_id` and a tool that can
stitch it all together), and operational complexity (a bus is
infrastructure you have to deploy, monitor, and maintain).

**When to use it:** when agents stop being functions of one service and
become services. If your agents have different lifecycles, deploy
separately, or are written by another team, message-based communication
stops being an exotic option and becomes the only sensible one. While
they all live inside the AI service — which is where they live now — it's
infrastructure that complicates things without buying anything.

## The position

The three patterns aren't alternatives at the same level. They're three
rungs of a ladder, and climbing a rung you don't need is the most common
way to ruin a multi-agent architecture.

1. **Shared state by default.** It's what you already have, has the best
   traceability-per-unit-of-complexity ratio, and decouples agents from
   each other in the one way that matters. Most systems stay here, and
   do well to.
2. **Handoff when the routing tax is a measured problem, not an imagined
   one.** It's a real savings in calls, bought with topological coupling
   and harder-to-read traces.
3. **Messages when agents cross the process boundary.** Here you're no
   longer choosing a communication pattern between agents — you're
   choosing a distributed-systems architecture, with everything that
   drags along.

And — this is the important part — they're combinable. A supervisor over
shared state inside the AI service that publishes one event when the
estimate is ready, so the business backend can react, is a perfectly
coherent architecture. The question is never which pattern is best, but
which pattern belongs at each boundary of the system.

## What none of the three solves

All three patterns answer the same question: how do agents pass each other
information. None answers this other one:

What happens when the system shouldn't decide alone?

When the estimate comes out wildly off, when the transcript describes
something with no precedent in the history, when confidence is low. At
that point it isn't that one agent needs to talk to another agent. It's
that the system needs to stop, show it to a person, and wait.

And a pause that can last hours or days isn't a communication pattern.
It's a problem of persisted state and an outward-facing contract.

---

> *(Editor's note — `graph=Command.PARENT` presumes a topology `s14-02`
> never builds, worth stating precisely before assuming handoff bolts onto
> the existing flat graph.)* `Command.PARENT` only means something when
> there's a parent/child graph relationship to escape from — i.e., when
> each agent is *itself* a compiled subgraph (LangGraph's actual "swarm"
> shape, typically each agent built as its own tool-calling ReAct loop),
> not a plain async function sitting directly in one flat `StateGraph` the
> way every node in `s14-02` and `s13-02`–`05` is. Adding
> `build_handoff_tool` calls to `s14-02`'s existing `budget_searcher`
> (a bare function, no tool-use loop of its own, no subgraph boundary)
> wouldn't produce this article's handoff behavior — there's no parent to
> escape to. Adopting handoff for real means restructuring each agent as
> its own tool-invoking subgraph first; it is not a drop-in addition to
> the flat supervisor graph these sessions have built so far.

> *(Editor's note — `correlation_id` and `request_id` are the same idea
> under two names, one session apart, worth reconciling rather than
> running both.)* §3's `correlation_id` is functionally identical to
> `request_id` in `app/generation/rag/observability.py`'s `log_stage`
> (Session 9) — a single identifier threading every log line or event of
> one logical operation together for later reconstruction. If both a
> synchronous `request_id`-based trace and an event-driven
> `correlation_id` end up in the same system, propagating one value under
> two field names across the process boundary is a real, avoidable
> footgun for whoever writes the tool that "stitches it all together" this
> article itself says messaging requires.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed.)* No event bus, message queue, or
> `correlation_id` exists — consistent with every article since `s12-01`.
> `EstimationState`-as-blackboard is, as this article itself says, already
> what the codebase's `Estimate`/`WorkModule`/`TaskItem` schema-passing
> shape amounts to conceptually, though none of it runs through LangGraph
> yet. Nothing here changes that finding; this article is the first to
> explicitly name the pattern the reference project's own architecture
> already leans toward by default.

> *(Editor's note — answers the second of `s14-01`'s four open questions.)*
> Routing was `s14-02`; communication is this one. Human intervention and
> tool privilege remain, and this article's own closing section names the
> first of those two directly as unresolved — persisted state and an
> outward contract, not a communication pattern.
