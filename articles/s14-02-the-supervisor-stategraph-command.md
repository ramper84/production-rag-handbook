---
title: "The supervisor: building routing by hand with StateGraph and Command"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-09-19
summary: >
  A supervisor does exactly three things — decompose, delegate, consolidate
  — and, if well designed, zero domain work; the moment it holds a business
  tool it's just another overloaded agent wearing a router's name. Build it
  by hand, not through a library abstraction, because routing is precisely
  the part you don't want to delegate to a layer you don't control. Three
  decisions carry the design: what the supervisor sees (a deliberately
  built digest — a five-line compact projection of state, constant cost
  per decision regardless of transcript length or iteration count, never
  the full message history); how it decides (a closed, validated Literal
  of destinations plus a one-sentence reason field nobody reads except you,
  at 3am, in the trace); and how it moves control (Command, which updates
  state and redirects in one return value, typed against the same Literal
  LangGraph uses to infer the graph's possible edges). A hard routing-step
  cap is non-negotiable from line one. The contrarian, usually-correct
  default: most routing decisions are deterministic preconditions, not
  judgment calls — resolve those in code, call the model only for genuine
  ambiguity, and if writing that split reveals there's no real ambiguity
  at all, that's the discovery that you never needed a model-driven
  supervisor to begin with. Flat topology (one supervisor, N specialists)
  is the default; hierarchical (team sub-supervisors) only once a flat
  router's own option count degrades its accuracy the same way an
  overloaded tool set would.
keywords: [supervisor pattern, Command, StateGraph, state digest, routing
           decision, Literal, hybrid routing, deterministic preconditions,
           routing_steps, routing_trail, flat vs hierarchical, create_supervisor]
---

# The supervisor: building routing by hand with StateGraph and Command

*Antonio Perez* · 🔴 19 min

In the estimation graph you already have, there's a question that never
gets asked at runtime: what's next? It never gets asked because the answer
is written in the code. Components come after extracting requirements,
budgets after that, the estimate after that. The order is a property of
the file, not a decision.

A supervisor is the component that turns that question into something that
happens at runtime. It receives the current state, decides which
specialist acts next, and does it again and again until the work is done.
It's the architecture's heart, and also the easiest place to hurt yourself:
an opaque router turns the whole system into a black box.

That's why we're going to build it by hand. Not because abstractions don't
exist — they do, and we'll get to them — but because routing is precisely
the part you don't want to delegate to a layer you don't control.

## 1. What a supervisor actually does

Three things, and they're worth separating because the literature blends
them together:

1. **Decompose**: look at the task and understand what pieces of work are
   missing.
2. **Delegate**: choose the specialist that produces the next piece and
   hand it control.
3. **Consolidate**: when nothing is missing anymore, close out and produce
   the result.

What it doesn't do, if well designed: domain work. The supervisor doesn't
estimate hours, doesn't read budgets, doesn't validate coherence. If it has
business tools in hand, stop calling it a supervisor — it's just another
agent that also happens to route, and you've reintroduced the overloaded
node you were trying to eliminate.

In our estimation flow, the supervisor is a node with no tools at all. Its
only output is a decision.

## 2. The state, and the digest

The first thing to decide isn't how it routes, but what it sees in order
to route.

The temptation is to pass it the full message history from every agent.
That's what several ecosystem abstractions do by default, and it's a bad
idea the moment the flow has more than two hops: context grows without
bound, cost per decision rises with every iteration, and the relevant
signal gets buried under intermediate transcripts the supervisor doesn't
care about.

The supervisor doesn't need to know what the budget searcher said. It
needs to know whether it already searched.

That's a **digest**: a compact projection of the state, built by you, that
answers the one question the supervisor actually has to resolve.

```python
from typing import Annotated, Literal
import operator

from typing_extensions import TypedDict

class EstimationState(TypedDict):
    transcript: str
    requirements: list[str]
    budget_matches: Annotated[list[dict], operator.add]
    estimate: dict | None
    validation: dict | None
    confidence: float | None
    routing_steps: int
    routing_trail: Annotated[list[dict], operator.add]
    status: str

def build_state_digest(state: EstimationState) -> str:
    """Compact projection of the state. This is all the supervisor gets to see."""
    return (
        f"requirements_extracted: {len(state['requirements'])} items\n"
        f"budget_matches_found: {len(state['budget_matches'])}\n"
        f"estimate_produced: {state['estimate'] is not None}\n"
        f"validation_done: {state['validation'] is not None}\n"
        f"routing_steps_so_far: {state['routing_steps']}"
    )
```

Five lines. Constant cost per decision, regardless of how long the
transcript is or how many iterations the flow has gone through. If the
supervisor later needs more information to decide well, you add it to the
digest explicitly, and you know exactly what you're paying for.

The two fields `routing_steps` and `routing_trail` aren't decorative. The
first is the brake; the second is the memory of what it did. We'll see
shortly why both are mandatory.

## 3. Routing as a typed decision

A supervisor whose output is free text is a bug waiting to happen. You need
the decision to be a value from a closed, validated set, and you need it
to fail loudly if the model steps outside the script.

```python
from pydantic import BaseModel, Field

AgentName = Literal[
    "requirements_extractor",
    "budget_searcher",
    "estimate_generator",
    "coherence_validator",
    "finalize",
]

class SupervisorDecision(BaseModel):
    next_agent: AgentName = Field(description="The specialist that must act next.")
    reason: str = Field(description="Why this specialist, in one sentence.")
```

No component in the system reads the `reason` field. You read it, in the
trace, at three in the morning, when the supervisor has routed to
`finalize` without having estimated anything and you need to understand
what it thought it was doing. It's cheap and it's going to save you more
than once.

With that, the node:

```python
import logfire
from langgraph.types import Command

MAX_ROUTING_STEPS = 12

SUPERVISOR_INSTRUCTIONS = """You coordinate a software estimation pipeline.
Given the current progress digest, choose the single specialist that must act next.

Rules:
- Requirements must be extracted before budgets are searched.
- Budgets must be searched before an estimate is produced.
- An estimate must exist before it can be validated.
- Choose "finalize" only when the estimate has been produced and validated.
Never choose a specialist whose work is already done.
"""

async def supervisor(state: EstimationState) -> Command[AgentName]:
    with logfire.span("supervisor.route") as span:
        if state["routing_steps"] >= MAX_ROUTING_STEPS:
            span.set_attribute("routing_budget_exhausted", True)
            return Command(
                goto="finalize",
                update={"status": "routing_budget_exhausted"},
            )

        response = await client.responses.parse(
            model="gpt-5",
            input=[
                {"role": "system", "content": SUPERVISOR_INSTRUCTIONS},
                {"role": "user", "content": build_state_digest(state)},
            ],
            text_format=SupervisorDecision,
        )
        decision = response.output_parsed

        span.set_attribute("next_agent", decision.next_agent)
        span.set_attribute("reason", decision.reason)

        return Command(
            goto=decision.next_agent,
            update={
                "routing_steps": state["routing_steps"] + 1,
                "routing_trail": [decision.model_dump()],
            },
        )
```

Three details worth attention.

**`Command` does two things at once.** It updates the state (`update`) and
moves control (`goto`). That's what lets routing be a node's own decision
rather than a conditional edge declared outside it. The `Command[AgentName]`
return type isn't cosmetic: LangGraph uses it to infer the graph's possible
destinations when building it, so the set of destinations and the set of
values the model can return are, literally, the same `Literal`. A new
destination that isn't added to the `Literal` doesn't mentally compile, and
doesn't work at runtime either.

**The routing budget is non-negotiable.** `MAX_ROUTING_STEPS` is the only
thing stopping a confused supervisor from bouncing between two agents until
it exhausts your OpenAI bill. An infinite loop in a deterministic graph is
an obvious bug; in a graph routed by a model, it's the default behavior in
the face of an ambiguous instruction. Cap it from the first line, not after
the first scare.

**The span carries the decision, not the response.** `next_agent` and
`reason` as span attributes mean your trace is a navigable record of every
fork the system took. This is exactly what gets lost when routing happens
inside a library abstraction, and it's the main reason to build it by
hand.

## 4. Workers return control

Every specialist does its work, writes its partial result to the state,
and hands the baton back to the supervisor. Nothing more.

```python
async def budget_searcher(state: EstimationState) -> Command[Literal["supervisor"]]:
    with logfire.span("agent.budget_searcher"):
        matches = await search_budgets(requirements=state["requirements"])
        return Command(
            goto="supervisor",
            update={"budget_matches": matches},
        )
```

Notice the reducer. `budget_matches` is annotated with `operator.add`, so
this agent accumulates instead of overwriting. If the supervisor decides to
invoke it twice with different requirements — which it can do, because the
route is its own — the results add up instead of clobbering each other.
The accumulation semantics are a state design decision, not an accident.

> *(Figure in the original: `fig-01-ciclo-enrutado-supervisor.png` — image
> not included in this repo. "The routing cycle": `EstimationState`
> [requirements: 8 items, budget_matches: [], estimate: None, validation:
> None, routing_steps: 1, routing_trail: [...], captioned "shared
> whiteboard (typed state)"] → `digest` → `supervisor` [tools: -] →
> `SupervisorDecision` [next_agent: "budget_searcher", reason: "..."] →
> `Command(goto="budget_searcher")` → `budget_searcher` [tools:
> search_budgets] → `Command(goto="supervisor", update=("budget_matches":
> [...]))` back to the shared state. Captioned: 1) the supervisor reads a
> state summary, not the message history; 2) it returns a typed decision —
> every jump lands in the trace and in routing_trail; 3) the worker writes
> its partial result and returns control. The cycle repeats until
> finalize.)*

## 5. Assembling the graph

```python
from langgraph.graph import StateGraph, START

builder = StateGraph(EstimationState)

builder.add_node("supervisor", supervisor)
builder.add_node("requirements_extractor", requirements_extractor)
builder.add_node("budget_searcher", budget_searcher)
builder.add_node("estimate_generator", estimate_generator)
builder.add_node("coherence_validator", coherence_validator)
builder.add_node("finalize", finalize)

builder.add_edge(START, "supervisor")

graph = builder.compile(checkpointer=checkpointer)
```

A single declared edge. Every other transition lives inside the nodes, in
the `Command`s. This is intentional: the graph no longer describes a flow,
it describes a set of capabilities and a router. The shape of the run
emerges at execution time.

And the checkpointer is the same one as always, over the same Postgres. No
new infrastructure. This matters more than it looks: every intermediate
state of the routing cycle gets persisted, which means a run can be
stopped, inspected, and resumed at any of its hops.

## 6. Trade-offs, and an alternative that often wins

### The routing tax

Every supervisor decision is a model call that doesn't produce a single
hour of estimate. In a flat topology, a task touching four specialists
costs eight calls: four of work and four of routing. The linear graph made
four. You're paying a 100% overcost for the flexibility of letting the
route decide itself.

That tax is justified if the route genuinely varies. If 95% of transcripts
end up with the supervisor choosing the same sequence, you're paying a
model to reinvent a `for` loop every time.

### The hybrid supervisor: the boring option that's usually right

Here's a position that runs against the ecosystem's current: most routing
decisions don't need an LLM. That requirements have to be extracted before
budgets get searched isn't a nuanced judgment call — it's a precondition.
Encoding it as an instruction in a system prompt and praying the model
respects it is trading a guarantee for a probability, and paying for the
privilege.

The pattern that's going to serve you in production is a router that
resolves what's deterministic with rules and only calls the model when
there's genuine ambiguity:

```python
async def supervisor(state: EstimationState) -> Command[AgentName]:
    if state["routing_steps"] >= MAX_ROUTING_STEPS:
        return Command(goto="finalize", update={"status": "routing_budget_exhausted"})

    # Deterministic preconditions: no model call needed, no way to get them wrong.
    if not state["requirements"]:
        return Command(goto="requirements_extractor", update=_bump(state))
    if not state["budget_matches"]:
        return Command(goto="budget_searcher", update=_bump(state))

    # Genuine ambiguity: the estimate exists but validation flagged concerns.
    # Re-estimate with more context, search for further analogues, or accept?
    # This is a judgement call. Here the model earns its keep.
    return await route_with_model(state)
```

The result is a system that's cheaper, faster, more predictable, and
easier to test, that keeps intelligence exactly where it earns its keep.
And it has a pedagogical virtue: it forces you to name which decisions in
your domain are genuinely difficult. If writing this reveals there's no
real ambiguity at all — that every fork is a precondition — you've
discovered something important: you don't need a model-driven supervisor.
You need the graph you already had.

## 7. Non-determinism and testing

When the route is decided by a model, two runs over the same transcript
can take different paths. Practical consequence: don't test the path, test
the result and the invariants. That an estimate got produced. That no
agent acted without its preconditions. That `routing_steps` never exceeded
the cap. `routing_trail` gives you all of that in a state field,
inspectable from a test with no extra instrumentation needed.

## 8. The bigger picture: flat, hierarchical, and the abstractions

**Flat vs. hierarchical.** With four specialists, a flat supervisor is
plenty. The problem shows up with fifteen: the router has to discriminate
among fifteen options on every decision, and its accuracy degrades exactly
the way an agent's would with fifteen tools. The answer then is to group by
team, with a higher-level supervisor routing to teams and sub-supervisors
routing within them. It's the same hierarchy as an organization's, and for
the same reason.

> *(Figure in the original: `fig-02-supervisor-plano-vs-jerarquico.png` —
> image not included in this repo. Two panels. "Flat: one supervisor, N
> specialists. Start here" — one orange `supervisor` box fanning out to
> four teal boxes [`extractor`, `searcher`, `generator`, `validator`],
> captioned "2 model calls per specialist (one to route, one to work).
> Degrades when the supervisor has too many options to discriminate
> between." "Hierarchical: teams with their own supervisor. Only if
> needed" — an orange `supervisor` fanning out to two orange team-
> supervisor boxes [`analysis_team`, `pricing_team`], each fanning out to
> two teal specialists [`extractor`/`searcher`; `generator`/`validator`],
> captioned "narrows each supervisor's decision space, but adds one more
> level of routing: more cost, more latency, deeper traces to read.")*

The cost is real: one more level of hierarchy is one more routing call per
hop and one deeper trace to read. Don't start here. Get here when the flat
supervisor starts failing you.

**The ecosystem's abstractions.** There are libraries that assemble this
topology for you — `create_supervisor` and company. It's worth knowing they
exist, and worth knowing this too: LangChain's own current recommendation,
for most cases, is to implement the supervisor pattern directly, with
tools and `Command`, rather than use the abstraction — precisely because
that's what keeps control over what context each agent receives, and keeps
every routing decision visible in the traces.

This isn't an ecosystem anecdote. It's confirmation of something you
already knew how to apply in any other domain: the abstraction that hides
the decision you need to inspect isn't saving you work — it's deferring
it. Here, the decision you need to inspect is exactly the routing, so the
abstraction that hides it is the one that doesn't serve you.

## What the supervisor still doesn't solve

With this you have an architecture that routes and consolidates, with
every fork logged and a brake that keeps it from running away.

But we've taken a big decision for granted without discussing it: agents
communicate by writing to shared state. None talks to another directly.
It's a choice, not the only one, and it has consequences for cost,
coupling, and how much context the system drags along. There are
topologies where the baton passes from agent to agent without returning to
the center, and there are ones where communication is an event stream
nobody orchestrates.

Which one suits you depends on where the bottleneck actually is. And
that's the next conversation.

---

> *(Editor's note — no Sources section in the source text, unlike every
> other article in this series.)* Every `s12`/`s13`/`s14-01` article closed
> with a `Fuentes` list; this one ends at "that's the next conversation"
> with no citations block. Left out rather than invented — if a Sources
> section exists in the original and was simply not included when this was
> provided, it should be added here rather than reconstructed from
> guesses about what it probably cites.

> *(Editor's note — `_bump` is used but never defined, same class of gap
> as `AgentResult`/`Step`/`retry_count` before it.)* §6's hybrid supervisor
> calls `_bump(state)` twice without showing its body. Given what it needs
> to produce — an `update` dict matching the deterministic branches'
> `Command(goto=..., update=_bump(state))` shape — it almost certainly
> mirrors the model-driven version's inline
> `{"routing_steps": state["routing_steps"] + 1, "routing_trail": [...]}`,
> just without a `SupervisorDecision` to log a `reason` from (or with a
> synthetic one, e.g. `"deterministic precondition"`). Worth writing out
> explicitly before this becomes real code — a helper this central to every
> deterministic branch is exactly the kind of thing worth not leaving
> implicit.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — two precedents, one closer than the
> other.)* No `supervisor`, `Command`, `SupervisorDecision`, or
> `MAX_ROUTING_STEPS` exists yet. But two of this article's core ideas have
> real ancestors. First, close: `app/generation/rag/retrieval/router.py`'s
> `RoutingDecision` (Session 10) already carries a `reason: str` field with
> the description *"One sentence justifying the routing choice"* — nearly
> verbatim what `SupervisorDecision.reason` argues for here, four sessions
> earlier and for a different kind of routing (collections, not agents).
> Second, looser: `app/generation/conversation/compression/` (Session 5:
> `anchors.py`, `policy.py`, `summarizer.py`) already solves "don't hand
> the model the full history, build a bounded projection" — but for
> conversational memory, condensing free-text turns into a summary, not
> for typed pipeline state condensed into a five-line digest. Different
> technique, same underlying principle, two sessions before this article
> needed to argue for it from scratch.

> *(Editor's note — directly strengthens `PLAYBOOK.md`'s Axis 5
> implementation guidance, not just its trigger list this time.)* The
> digest principle (constant per-decision cost, never full history) and
> the deterministic-preconditions-first hybrid supervisor are concrete
> answers to a question `PLAYBOOK.md`'s existing "hand-rolled supervisor
> loop" guidance left open: what does the router function actually look
> at, and how much of it is a model call at all? `PLAYBOOK.md` updated to
> cite both directly.
