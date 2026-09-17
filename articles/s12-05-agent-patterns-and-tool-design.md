---
title: "Agent patterns and designing quality tools"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 21 min
added: 2026-09-16
summary: >
  Two halves that need each other. First, agent shape is three lenses on
  one design decision, not a catalog to pick from: single-step vs.
  iterative (cost vs. need), reactive vs. proactive (robust-but-myopic vs.
  efficient-but-brittle), fixed plan vs. dynamic planning (auditable vs.
  adaptive) — and the axes aren't orthogonal, so state where the agent
  sits on each rather than picking a type. Shape doesn't have to be a
  system constant either: a cheap classifier routing simple inputs to a
  single step and complex ones to the iterative agent gets the best of
  both without paying the worst case on every request. Second, once shape
  is fixed, almost all remaining behavior is governed by tool design —
  the model reads only names, descriptions and schemas, never your code or
  intent, so a wrong tool choice is almost always a vague description, not
  a model failure. Descriptions are prompts: write, test, read the traces,
  adjust. The fix is usually one sentence, not a bigger model.
keywords: [agent patterns, single-step, iterative, reactive, proactive,
           fixed plan, dynamic planning, routing, tool design, tool
           description, tool boundaries, trace-driven optimization,
           Building Effective Agents, writing tools for agents]
---

# Agent patterns and designing quality tools

*Antonio Perez* · 🔴 21 min

"Agent" isn't one thing. Under that word fit very different ways of solving
a problem, and choosing the right shape for yours matters as much as
knowing how to write the loop. An agent that takes a single step and one
that iterates twenty times are different animals, with different costs,
risks, and failure modes.

This article has two halves that need each other. The first is about the
agent's shape: the axes along which one agent differs from another, and how
to choose. The second is about the lever that steers its behavior within
that shape: tool design. Because once the shape is fixed, what the agent
does well or badly depends, to a surprising degree, on how you describe its
tools.

Before starting, the ground. We're working with an estimation agent: it
receives a meeting transcript, and with three tools — `search_budgets`
(retrieves historical budgets), `calculate_estimate` (computes costs),
`validate_estimate` (checks the result) — produces a project estimate.
We'll see how things change over that concrete case.

## 1. Single-step or iterative

The first axis is the most basic: how many turns does the agent take?

A **single-step agent** makes one call, maybe uses a tool, and finishes.
There's no real loop — it's nearly a pipeline with one decision. For a
simple transcript — "a landing page with a contact form" — this is more
than enough: one budget search, one calculation, done. Cheap, fast,
predictable.

An **iterative agent** repeats the decide-act-observe cycle until it
converges. It's what you need when the problem doesn't fit in one step: a
transcript with four components requires searching separately, maybe
reformulating a search that came back thin, computing partials, and
consolidating. Every turn adds adaptive capacity, but also latency, cost,
and a larger failure surface.

The decision isn't philosophical: it's cost against need. If you can solve
the problem in one step, do it in one step. Iteration is a tool for
problems whose shape you don't know ahead of time, not a default. A common
mistake is building an iterative loop for tasks a single step would solve
better and cheaper, and finding out only when the bill arrives.

## 2. Reactive or proactive

The second axis is how the agent relates to the future.

A **reactive agent** decides the next step in light of what it just
observed, with no plan ahead of it. It searches for budgets, sees the
result, and only then decides what to do next. It's simple and surprisingly
robust: because it isn't committed to a plan, it doesn't break when reality
doesn't fit one. Its weakness is that it can be myopic, making locally good
decisions that don't add up to a good whole.

A **proactive agent** anticipates: it forms an idea of the goal and acts
toward it, not just in response to the last stimulus. Faced with the
transcript, a proactive agent might reason "I'm going to need to estimate
four components, so I'll search budgets for all four" before seeing any
result. It's more efficient when the path is predictable, because it isn't
discovering the work as it goes. It's more fragile when it isn't, because a
plan formed too early can be contradicted by the first observation.

For our case, reactivity usually wins. Transcripts bring surprises — a
component that turns out to be two, a migration with no historical
references — and an agent deciding step by step absorbs those better than
one married to a premature plan. But it isn't absolute: a pinch of
proactivity — decomposing the project into components up front — saves
turns without committing to a rigid path.

## 3. Fixed plan or dynamic planning

The third axis refines the second: if there's a plan, when is it decided?

With a **fixed plan**, the agent decomposes the problem at the start and
then executes that plan to the end. Its great virtue is auditability: you
have the plan written down before spending a token, and can justify
afterward why things were done the way they were. In an estimate you're
going to defend to a client, being able to show "the agent decided to
estimate these four components, in this order, for these reasons" has real
value.

With **dynamic planning**, the agent re-plans every turn based on what it
observes. It's what lets it react: if a budget search comes back empty, it
rethinks and searches differently before computing on bad data. It gains in
adaptability what it loses in predictability.

An honest note: these three axes aren't orthogonal. A proactive agent tends
toward a fixed plan; a reactive one, toward dynamic planning. They aren't
three types you pick from a catalog, but three lenses for thinking about
the same design decision, and real agents mix positions on each axis.
What's useful isn't classifying your agent into a box, but being aware of
where you're placing it and why.

To commit to a position for the estimation agent: iterative, mostly
reactive, with light, dynamic planning — a loose initial decomposition into
components, revised on the fly when an observation calls for it. That
combination absorbs transcript variability without paying the cost of an
unnecessary loop or the fragility of a rigid plan. It isn't the only
defensible choice, but it's the one that best fits the nature of the
problem, and being able to articulate why is half the battle.

> *(Figure in the original: `S12-fig-05a-ejes-de-patrones.jpg` — image not
> included in this repo. Three horizontal sliders, each with a purple dot
> marking where the estimation agent sits: "number of steps" [single-step
> ↔ iterative, dot near iterative, captioned "iterates only when the
> problem doesn't fit a single step"], "anticipation" [reactive ↔
> proactive, dot near reactive, captioned "decides step by step; absorbs
> surprises better"], "when the plan happens" [fixed plan ↔ dynamic
> planning, dot toward dynamic, captioned "decomposes at the start,
> revises on the fly"]. Callout: the estimation agent — iterative, mostly
> reactive, light and dynamic planning; the axes aren't orthogonal, they're
> lenses for thinking about the design, not boxes.)*

## 4. Routing the shape per case

There's a decision that precedes all the ones above and is often
overlooked: you don't have to choose one shape for every input. You can
choose the shape per case.

In production, most transcripts are simple and only a few are complex.
Committing to the iterative agent for all of them means paying its cost and
latency even on the cases a single step would solve better. The alternative
is to route: a cheap classification up front decides whether the
transcript is simple — and goes to the single-step path — or complex — and
goes to the iterative agent. That way you pay for iteration only when the
problem demands it, and the system's average cost stays low.

This reframes the pattern choice usefully. The question stops being "what
shape is my agent" and becomes "what shape does each input deserve." The
routing itself is a cheap, deterministic step, and its value is that it
lets you keep the best of each shape without paying the worst case on every
request. It's ordinary engineering, again: you measure the real
distribution of your inputs, design for the common case, and leave a path
for the hard one. The agent's shape doesn't have to be a system constant —
it can be a decision made per request.

## 5. Tools are the interface that steers the agent

With the shape fixed, what determines whether the agent decides well within
it? Almost entirely, the tools: which ones exist and how you describe them.
The model chooses what to do by reading the names, descriptions and schemas
of the tools you give it. It doesn't read your code or your intent; it
reads those sentences. That's why tool design isn't documentation — it's
behavior direction.

This gives you a principle that saves a great deal of debugging time. When
the agent misbehaves — picks the wrong tool, invents odd arguments, stuffs
four components into a search that should have been one, calls things in an
absurd order — the instinct is to blame the model or tweak the loop.
Almost always it's a mistake. The fault is usually in a tool's description,
or in the tool set as a whole, and that's also where the fix is. The model
did what your descriptions told it to; if what it did surprises you, the
descriptions said something different from what you thought.

## 6. The description is a prompt you iterate on

The practical consequence is that tool descriptions get treated like
prompts: written, tested, observed, and adjusted. You don't get it right on
the first try, and that's fine.

Look at `search_budgets`. A naive first version:

```python
{
    "name": "search_budgets",
    "description": "Searches historical budgets.",
    # ...
}
```

With this, the agent has no way to know it should search one component at a
time. Faced with a transcript with an integration and a migration, it will
likely fire a single search mixing both and get back incomparable results.
That's not the model's fault — the description told it nothing else.

The version that fixes the behavior carries the constraint, the
counterexample, and the reason inside the description itself:

```python
{
    "name": "search_budgets",
    "description": (
        "Search historical budgets for ONE software component at a time. "
        "Call this separately for each component in the project; never combine "
        "unrelated components (for example, an ERP integration and a data "
        "migration) in a single query, because mixed results cannot be "
        "compared. Returns comparable historical items with their hours and a "
        "confidence signal."
    ),
    # ...
}
```

The difference between the two isn't cosmetic: it's the difference between
an agent that estimates well and one that produces meaningless numbers, and
it lives entirely in a text field. An effective description tells the model
when to use the tool, when not to, at what granularity, and what it'll get
back. That's the job, and that's where reliability is won — not in a more
expensive model.

> *(Figure in the original: `S12-fig-05b-descripcion-palanca.jpg` — image
> not included in this repo. `search_budgets` branching into two columns —
> "vague description" [`"Searches historical budgets."`] leading to
> "behavior: combines integration and migration in a single search" leading
> to "incomparable results: meaningless estimate", in orange/red; "effective
> description" [`"ONE component at a time; never combine unrelated ones."`]
> leading to "behavior: searches each component separately" leading to
> "comparable references: solid estimate", in green. Callout: same model,
> same loop — change one sentence in the description, change the behavior;
> the fix is almost always in the description, not the model.)*

## 7. The tool set, not just each tool

There's a design layer above each individual description: the set's. The
model doesn't choose a tool in a vacuum, but among all the ones you offer
it, and the relationship between them affects the quality of the choice.

Too many tools with overlapping boundaries confuse: if two tools could
serve the same purpose, the model hesitates and sometimes picks wrong. Too
few and too generic force the model to juggle arguments to express what it
wants. The sweet spot is a small set of tools with sharp boundaries:
`search_budgets` searches, `calculate_estimate` calculates,
`validate_estimate` validates, and none encroaches on another's territory.
As the tool set grows, namespaced names (`budgets_search`,
`estimate_calculate`) help the model group and disambiguate.

A useful warning sign: if you find yourself explaining in a tool's
description when *not* to use it in favor of another, the boundary between
the two is probably drawn wrong. The fix isn't a longer description — it's
a better-delimited tool set.

## 8. Optimizing means reading the traces

How does all this happen in practice, without guessing? Empirically, by
looking at what the agent actually does.

The method is direct. You take a handful of representative transcripts —
simple, complex, with odd edge cases — run the agent over them, and read
the traces: which tool it chose at each step, with what arguments, in what
order, where it got stuck or failed. Every anomalous behavior has a cause
you can trace back to a vague description, a badly drawn boundary between
two tools, a result that returned too much noise, or a mute error that left
the agent blind. You fix the cause, run again, and check that the trace
improves without breaking the other cases.

An example of the kind of finding that turns up. Suppose that, reading
traces, you see the agent calling `calculate_estimate` before having
searched budgets for every component, producing partial estimates over
incomplete data. The instinct is to think the model is "rushing." But the
cause is in `calculate_estimate`'s description, which doesn't state its
precondition: that it expects to receive every component with its
references already retrieved. You add that precondition to the description
— "only call this after budgets have been searched for every component" —
and the behavior corrects itself. You didn't touch the model or the loop;
you adjusted one sentence. That's the improvement loop, and most fixes have
exactly that shape.

This has a design implication worth keeping in mind from the start: the
quality of your traces determines your ability to optimize. An agent that
logs action, arguments, and observation at every step is one you can
improve; one that only returns the final result is a black box you can only
change the model on and pray. And what each tool returns is part of this:
high-value results — just enough to decide the next step, with stable
identifiers — produce good decisions, and errors returned as informative
observations — "1 weak match, low confidence" — let the agent recover
instead of stumbling.

## 9. Closing: shape and interface, neither is magic

Recapping the two halves. Agent patterns are decisions about shape: how
many turns, how much anticipation, when to plan. They're control-flow
architecture decisions, the kind you make every time you design a system,
applied to a case where one of the branches is decided by a model. Tool
design is the lever that steers behavior within that shape. It's interface
and prompt engineering: describe well, delimit well, return well, and
iterate by watching the results.

What unites both halves is what neither of them is. There's no machine
learning here, no secret model to train. There are design choices and an
empirical improvement loop. You don't get a better agent by waiting for a
better model; you get one by choosing the right shape for your problem and
tuning the tools until the traces look the way they should. It's tunable,
it's measurable, and it's your job — the same craft as always, with one
new, bounded piece in the middle.

## Sources

- Anthropic, *Building Effective Agents* — when a problem calls for an
  iterative agent versus a simpler flow, and the discipline of starting
  with the minimum:
  https://www.anthropic.com/research/building-effective-agents
- Anthropic, *Define tools* — effective descriptions, namespaces, and
  high-value results as a behavior lever:
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use
- Anthropic, *Writing effective tools for agents* — tool design treated as a
  surface iterated on empirically:
  https://www.anthropic.com/engineering/writing-tools-for-agents

---

> *(Editor's note — closes session 12 so far by naming what s12-01 and
> s12-04 already did.)* §4's "route the shape per case" is not a new
> proposal — it's s12-01 §7's "two-path architecture with a cheap router up
> front" (simple transcript → pipeline, complex → agent), generalized from
> that one instance into a named, reusable pattern. §§1-3's axes are also a
> retrospective naming of a choice s12-04 already made without arguing for
> it explicitly: that article's `run_agent` is iterative (§1), and its trace
> in §4 — searching, hitting a weak match, reformulating before calculating
> — is this article's §2 reactivity and §3 dynamic planning, demonstrated
> rather than named. Read after s12-04, this article is the argument for
> why that specific implementation's shape was the right call, not an
> alternative to it.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — a real, close precedent, not just an
> absence.)* No pipeline-vs-agent complexity classifier exists — consistent
> with s12-01 through s12-04. But §4's routing pattern has a genuine
> architectural precedent already built in this codebase, for a different
> decision: `app/generation/rag/retrieval/router.py` (Session 10) routes a
> *query* to one or more *collections* through a cascade — explicit target →
> vocabulary rules → LLM classifier with structured output → fallback to
> all — logging which level decided (`level: "explicit" | "rules" |
> "classifier" | "fallback"`) and degrading to fallback on classifier
> failure rather than raising. That's the shape §4's simple/complex router
> would want, reused rather than invented: cascade cheap deterministic
> checks first, fall back to an LLM classifier only when they don't decide
> it, and never let a classifier failure take down the request. Building
> §4's router as a copy of this pattern, not a fresh design, is the
> "smallest thing that works" this playbook's Axis 5 already argues for.

> *(Editor's note — domain transfer, and a correction of an over-eager
> first draft of this note.)* The unpublished trading-advisor project
> referenced in s12-01 and s12-02's notes offers a genuine boundary case for
> §§1-3, but not the one it's tempting to reach for. Its `risk_critic.py`
> is **not a point on these axes at all** — it makes zero LLM calls, so
> there's no `model.decide` for "single-step vs. iterative" or "reactive
> vs. proactive" to describe; it's the same distinction s12-02's note
> already drew for `PLAYBOOK.md`'s Axis 4 Critic, restated here because
> this article's own axes make the same category error easy to fall into a
> second time. The closer fit is trading-advisor's `llm_service.py` — one
> generation call producing a recommendation from retrieved context, no
> tool the model itself chooses to call — which is this article's
> single-step case in its most minimal form: a "single step" doesn't
> require a tool call at all, per §1's own "maybe uses a tool, and
> finishes," and a system that never needs §2/§3's axes to differ from a
> single point is exactly what "if you can solve it in one step, do it in
> one step" predicts for a problem this bounded.
