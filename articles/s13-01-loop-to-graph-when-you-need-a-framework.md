---
title: "From hand-rolled loop to graph: when you need a framework"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 16 min
added: 2026-09-16
summary: >
  A hand-rolled agentic loop works and works well while the flow is short;
  it degenerates into nested ifs, state flags and order-explaining comments
  once steps gain dependencies, conditional branches, parallelizable work,
  or backtracking. The 2026 landscape has three orchestration models —
  graph-based (LangGraph, stable 1.0 since October 2025), role-based
  (CrewAI), conversational (the AutoGen lineage, now unified into
  Microsoft Agent Framework) — plus Google ADK and Pydantic AI on a Python
  team's radar. The right question isn't framework-or-not, it's what shape
  the flow actually has: a single call needs no orchestration, a single
  reasoning-and-tool loop needs no graph, only real graph shape —
  dependencies, conditional routing, parallelism, persist/resume, a human
  checkpoint — earns a framework's complexity tax. LangGraph is four ideas
  (typed shared state, node functions, conditional edges, a checkpointer),
  with create_agent covering the single ReAct loop and StateGraph covering
  the rest. Over 60% of production agent incidents trace to state
  management, not model quality — which is exactly what a checkpointer is
  for. Measure the existing hand-rolled loop as the baseline before
  deciding; if reexpressing it as a graph gains nothing measurable, the
  loop was already the right answer.
keywords: [LangGraph, CrewAI, AutoGen, Microsoft Agent Framework, Google
           ADK, Pydantic AI, StateGraph, create_agent, checkpointer, state
           management, orchestration frameworks, complexity tax, build vs
           buy, graph orchestration]
---

# From hand-rolled loop to graph: when you need a framework

*Antonio Perez* · 🔴 16 min

The estimation system today has a hand-built agentic layer: a loop over the
RAG pipeline that, given a meeting's text, decides which operation to run
next — retrieve budgets, calculate, validate — and repeats until it closes
an estimate. It's a `while` with function calling and a handful of
branches. It works, and it works well while the flow is short.

The problem shows up when the flow grows. As soon as there are several
steps with dependencies, conditional decisions that depend on what the
previous step returned, work that could run in parallel, or situations that
need backtracking, the imperative loop starts accumulating nested `if`s,
state flags, and comments explaining why the order is what it is. The code
stays correct, but it stops being readable, and above all it stops being
easy to reason about when something fails in production.

That's where formal orchestration comes in. And with it, the question this
article wants to answer with judgment rather than fashion: when does an
orchestration framework earn its keep, and when was the hand-rolled loop
already the right answer?

## 1. The framework landscape in 2026

The ecosystem has settled down considerably compared to previous years.
Today, broadly, three orchestration models coexist.

**Graph-based orchestration.** You model the system as a directed graph:
nodes that do work, edges that decide the transition. The standard-bearer
is **LangGraph**, from LangChain, which reached a stable 1.0 in October
2025 alongside LangChain 1.0. In this model the flow is defined ahead of
time and is explicit, which gives deterministic control and simpler
debugging at the cost of more upfront design.

**Role-based orchestration.** You define agents as team members, each with
their role, their tools, and their task. The standard-bearer is **CrewAI**:
a "manager" agent delegates to specialists. The mental model is intuitive
and prototypes fast, at the cost of less fine-grained control in flows with
complex branching.

**Conversational orchestration.** Agents coordinate by exchanging messages
in turns. This is AutoGen's lineage. Worth knowing: Microsoft merged
AutoGen and Semantic Kernel into the **Microsoft Agent Framework**, which
reached 1.0 in April 2026; the two original projects are now in maintenance
mode. It's the natural choice in Azure/.NET ecosystems.

Outside those three, two more names belong on a Python team's radar.
**Google ADK** (Agent Development Kit), the code-first, model-agnostic kit
inside Google Cloud's agent platform, with sequential, parallel, and loop-
flow agents. And **Pydantic AI**, Python-native, typed, with FastAPI-style
dependency injection, which fits especially comfortably with a service
built on that stack. On top of these, the model providers' own agent SDKs
(OpenAI, Anthropic) are built for the single-agent loop.

The important picture isn't the list — it's the underlying observation:
these frameworks' novelty is bounded. All of them solve variations of the
same problem — coordinating steps, keeping state, recovering from errors —
with different abstractions. None of them is magic. They're systems
engineering applied to a new kind of service.

## 2. Frameworks vs. internal orchestration: the right question

The temptation, when a mature framework shows up, is to adopt it by
default. The opposite temptation, on a team that values control, is to
build everything by hand. Both are shortcuts that skip the only question
that matters: what shape does my flow actually have?

It's worth being honest about what each path costs. LinkedIn's engineering
team summarized their production experience with a blunt recommendation:
try to buy before you build, and build only if what you need doesn't exist,
because the space moves very fast. The pattern that dominates in practice
is hybrid: a framework or platform for the standard 80% of the flow, and
custom code only for the percentage that's your domain differentiator.

The operational advice most repeated by teams who've taken agents to
production is just as sober: start simple, instrument heavily, and add
complexity only where the data demands it. Most teams overshoot a rung of
sophistication — they build a multi-agent system where one well-
instrumented single agent would have been enough.

One data point orders the priorities. According to LangChain's 2026 agent
engineering report, **over 60% of production agent incidents originate in
state management**: agents that lose the thread, repeat work, or crash
halfway through because state wasn't persisted properly. Not model quality,
not the prompt — state. That says a great deal about where the real work
is, and about what a framework needs to get right to earn its place.

## 3. Where LangGraph fits

For a flow like the estimation one — a sequence of steps with their own
responsibility, some conditional branching, and parallelizable work — the
graph model is the best fit. And its model is deliberately small.

A graph in LangGraph is four ideas:

1. A **shared, typed state** every node reads and updates.
2. **Nodes**, which are functions that receive the state and return a
   partial update.
3. **Edges**, which connect nodes; conditional edges inspect the state and
   decide where to go.
4. A **checkpointer**, which persists state after every step and makes
   pausing, resuming, and recovering possible.

Everything else — parallel execution, branches, subgraphs, human
intervention — is built on top of those four ideas. For any engineer who's
ever drawn a flowchart or a state machine, the model is familiar. That
familiarity is precisely the point: you're not learning a new paradigm,
you're putting a name and a runtime to something you already knew how to
draw.

Two levels within LangChain itself are worth separating, because they get
confused. The high-level shortcut `create_agent` (which replaces the older
`create_react_agent`) builds a ReAct-style agent in a few lines: a loop
where the model decides whether to call a tool or finish. That's exactly
the single-agent reason-and-act pattern. For that, a hand-rolled loop or
`create_agent` serve equally well. The low-level graph API, `StateGraph`,
is a different thing: you give explicit shape to topologies the ReAct loop
doesn't express well — several steps with dependencies, conditional
routing, parallelism, bounded cycles. That's the ground where the graph
earns its place.

In terms of project architecture, none of this changes the contract. The
graph lives inside the AI service, just like the loop it replaces. The
business backend keeps sending a transcript and receiving a structured
estimate with its `status` field, indifferent to whether underneath there's
a loop, a graph, or anything else. LangGraph is also model-agnostic: nodes
still wrap the same model call the service already uses. The framework
orchestrates; it doesn't replace your LLM-calling layer.

> *(Figure in the original: `S13-fig-01-decision-framework.jpg` — image not
> included in this repo. A decision tree: "what shape does your flow have?"
> branches three ways — "one model call, at most a queue behind it" →
> "NO FRAMEWORK: nothing to orchestrate, don't pay the tax"; "an agent that
> reasons and calls tools in a loop (ReAct)" → "HAND-ROLLED LOOP or
> create_agent: a well-instrumented loop is enough, the framework adds
> little"; "steps with dependencies, branches, and parallelism" → "GRAPH
> WITH LANGGRAPH: the flow has graph shape, this is where it earns its
> place" [highlighted purple]. Callout: measure the baseline before
> deciding — if reexpressing it gains nothing measurable, the loop was
> already the answer.)*

## 4. The complexity tax

A framework doesn't come free, and saying so is part of choosing it well.
LangChain 1.0 is stable, but it's heavy: it drags in a dependency surface,
layers of indirection, and scaffolding a homegrown loop doesn't have.
That's its complexity tax, and it has to be paid with eyes open.

The practical rule is direct. If your "flow" is a single model call with an
output format and, at most, a queue behind it, don't bring in an
orchestration framework — there's nothing to orchestrate, and you'd only be
paying the tax for nothing in return. If your flow is a single agent that
reasons and calls tools in a loop, a well-instrumented hand-rolled loop is
already a perfectly professional answer; the framework adds little. The
framework starts earning its place when the flow has genuine graph shape:
steps with dependencies, conditional branches, parallelism, a need to
persist and resume state, points where a human has to approve before
continuing. At that point, what the framework saves you — the
checkpointer, conditional routing, pause/resume, per-node instrumentation —
is exactly the work you'd otherwise reimplement, worse.

And there's a test you can't skip: **measure the baseline before
deciding.** The hand-rolled loop that already exists is the honest
reference. If reexpressing it as a graph doesn't gain the system anything
measurable — not clarity, not recovery capacity, not observability — then
the loop was already the right answer and the framework is unnecessary.
The decision gets made with data, not faith in the abstraction.

## 5. What's next

The conclusion isn't "use a framework" or "don't." It's that the
estimation flow, today, has exactly the shape that justifies a graph: a
sequence of steps with their own responsibility, at least one conditional
branch in validation, and work — the per-component budget search — that's
crying out to run in parallel. Reexpressing that flow as an explicit graph,
with typed, persistent state and per-node observability, is the natural
next step: it turns a loop that had to be read carefully into a structure
that can be seen, measured, and reasoned about. What follows is drawing
that graph and running it.

## Summary

- The system has a hand-rolled agentic loop that works but scales poorly.
  As soon as the flow adds steps, branches, and parallelism, the
  imperative loop fills up with implicit state and becomes hard to reason
  about.
- In 2026, three orchestration models coexist: graph-based (LangGraph),
  role-based (CrewAI), and conversational (the AutoGen lineage, now
  Microsoft Agent Framework). Google ADK and Pydantic AI round out a
  Python team's radar.
- The right question isn't "framework, yes or no" but what shape the flow
  has. The dominant pattern is hybrid; the dominant advice is start
  simple and instrument heavily.
- State management is where most agents fall over in production. That
  defines what a framework has to get right to be worth it.
- LangGraph is four ideas: typed state, node functions, (conditional)
  edges, and a checkpointer. `create_agent` covers the single ReAct loop;
  `StateGraph` covers real topologies with branches and parallelism.
- Every framework has a complexity tax. With no flow to orchestrate, don't
  pay it. With a real graph, what it saves you is the work you'd
  otherwise reimplement worse. Measure the baseline before deciding.

## Sources

- LangChain — LangChain 1.0 and LangGraph 1.0 announcement:
  https://www.langchain.com/blog/langchain-langgraph-1dot0
- LangChain — agent framework comparison:
  https://www.langchain.com/resources/ai-agent-frameworks
- Microsoft — Agent Framework overview, and the succession from AutoGen and
  Semantic Kernel: https://learn.microsoft.com/en-us/agent-framework/overview/
- Google — Agent Development Kit overview:
  https://docs.cloud.google.com/agent-builder/agent-development-kit/overview
- Production agent orchestration patterns:
  https://arahi.ai/blog/ai-agent-orchestration

---

> *(Editor's note — opens session 13 on the premise session 12 argued for,
> not yet the codebase's premise.)* This article's opening paragraph writes
> in the present tense as though the hand-rolled loop `s12-01` through
> `s12-06` argued for and built (in prose) already exists and runs in the
> estimation system. It doesn't, in `lidr/ai-engineering`: the branch tip is
> still `session 11 completed`, with no `run_agent`, `TOOL_REGISTRY`, or
> `search_budgets` tool anywhere in `app/`. This sharpens rather than
> breaks the pattern every `s12` article already established — the
> narrative voice describes a system one session ahead of the code it's
> nominally about, and by `s13` that gap is now two sessions wide, not one.
> Nothing here depends on the loop actually existing yet; the architectural
> argument (measure the hand-rolled baseline before adopting a graph) holds
> whether that baseline is real code or the fifty lines `s12-04` wrote out
> in full.

> *(Editor's note — directly strengthens `PLAYBOOK.md`'s existing Axis 5
> recommendation, not a new topic for this playbook.)* `PLAYBOOK.md` §2's
> Axis 5 already recommended, before this article existed, "a graph-based
> agent framework (e.g. LangGraph), only once the hand-rolled loop has been
> tried and the branching shape has actually been observed to need it." This
> article is that recommendation's first real citation, and it sharpens it
> in one respect worth carrying back: the decision isn't binary
> (loop-or-graph) but three-way — a single call needs no framework at all,
> a single reasoning-and-tool loop needs a loop but not a *graph*
> (`create_agent` or hand-rolled, equally fine), and only genuine graph
> shape — dependencies, conditional routing, parallelism, persist/resume,
> a human checkpoint — earns `StateGraph`. `PLAYBOOK.md` updated to cite
> this directly and carry the three-way shape rather than the two-way one
> it had.

> *(Editor's note — a number worth keeping, not just the headline.)* The
> "over 60% of incidents trace to state management" figure is doing real
> work in this article's argument for *why* a checkpointer specifically is
> the thing worth paying a framework's complexity tax for, rather than
> conditional routing or parallelism (both of which a hand-rolled loop can
> express without too much pain — `s12-04` already did the conditional
> branch, and `asyncio.gather` already did the parallelism). If a project's
> orchestration need is genuinely just "several tool calls with
> dependencies," reaching for a full graph framework on that basis alone is
> weaker justification than reaching for one because state needs to survive
> a restart or a pause for human approval — worth stating explicitly in a
> plan that selects Axis 5 and a graph framework, not just "the shape is
> graph-like."
