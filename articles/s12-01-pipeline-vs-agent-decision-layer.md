---
title: "From pipeline to agent: when your RAG system needs a decision layer"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 23 min
added: 2026-09-16
summary: >
  A fixed three-step pipeline — reformulate, retrieve, generate — is
  predictable, cheap and testable, and stays the default for any problem
  whose decision tree can be pre-mapped. An agent buys exactly one thing:
  adaptive orchestration over a problem whose shape isn't known until the
  input is read — at the cost of latency, token spend, non-determinism and
  compounding errors. A five-question checklist (pre-mappable? variable
  shape? does the value justify the spend? is the error cheap and
  verifiable? is the model actually good at the domain?) decides which.
  Applied to the estimator: the pipeline stays the default for simple
  transcripts; an agent becomes a decision layer above it for multi-component
  ones, promoting the pipeline's own steps to tools rather than replacing
  them, behind a cheap router and an unchanged business contract.
keywords: [agents, workflows, tasks, tool use, orchestration, decision layer,
           Building Effective Agents, non-determinism, cost, latency,
           observability, code agents, router]
---

# From pipeline to agent: when your RAG system needs a decision layer

*Antonio Perez* · 🔴 23 min

Our estimation system works. You hand it a meeting transcript, it reformulates
the query, retrieves comparable historical budgets, and generates a structured
estimate. It's a fixed pipeline: three steps, always the same, always in the
same order. It's predictable, easy to test, cheap to run, and it fails in ways
you can anticipate.

From that starting point, the interesting question isn't "how do I build an
agent." It's "why would I want to." Because an agent isn't a free upgrade to
the pipeline. It's an architectural decision with a concrete cost, and most of
the time the right answer is not to add it. This article is about when the
answer is yes.

## 1. The pipeline that already works

It's worth being honest about how good a fixed pipeline is, because agent
marketing tends to make us forget it.

A pipeline is a sequence of steps you write. In our case:

```python
def estimate_from_transcript(transcript: str) -> Estimate:
    query = reformulate(transcript)
    budgets = search_budgets(query)
    return generate_estimate(transcript, budgets)
```

Control flow is yours. You decided that reformulation happens first, then
retrieval, then generation. The model fills in each blank, but it doesn't
decide the structure. And that has very desirable consequences:

- **It's predictable.** The same input always takes the same path. If
  something fails, you know which step.
- **It's cheap.** You know exactly how many LLM calls you make per request.
  Cost doesn't explore, it doesn't spike.
- **It's testable.** You can test each step in isolation with known inputs
  and deterministic assertions.
- **It's fast.** There's no back-and-forth negotiation with the model about
  what to do next.

For a huge share of real problems, this is all you need. A classification
task, a field extraction, a single retrieval followed by a generation — none
of that requires the model to take the wheel. Adding agency there means
buying non-determinism, latency and cost in exchange for nothing.

So the pipeline is the default state. The question is what has to break to
justify leaving it.

## 2. Where the pipeline breaks

Take two transcripts.

The first says, in essence: "We need a landing page with a contact form,
deployed on basic hosting." One component, one search for comparable budgets,
one estimate. The pipeline nails it. There's no decision to make: the shape
of the problem is fixed and you already encoded it.

The second is a kickoff meeting for a real project: a customer portal with
its own business layer, an integration with the client's ERP over an API, a
mobile app that consumes that portal, and a data migration from a legacy
system that "nobody's quite sure how it's put together." Here the fixed
pipeline starts to creak, and it's worth seeing exactly why.

The problem doesn't have a known shape ahead of time. You don't know how many
components there are until you've read the transcript. You don't know how
many budget searches you need, or over what. The ERP integration and the
legacy migration are very different beasts, and searching for both with a
single reformulated query returns a useless mix. You also don't know in what
order to tackle them, or whether the result of estimating the migration
changes how you estimate the integration.

Within the pipeline paradigm you have two bad options. One: fire a single
giant search and ask the generator to make do with a jumble of incomparable
budgets. Quality collapses. Two: hand-code a decision tree — "if there's an
integration, take this branch; if there's a migration, this other one" — that
you then have to maintain for every project shape that shows up. That doesn't
scale: every client brings a new combination.

What's missing isn't more retrieval or better generation. It's run-time
decision capacity: something that reads the transcript, decides there are
four components, searches for budgets for each one separately, computes
partial estimates, and consolidates them. A path built case by case, not one
you wrote in advance.

> *(Figure in the original: `S12-fig-01-pipeline-vs-agente.jpg` — image not
> included in this repo. It contrasts a fixed pipeline, where you write the
> step order once, against an agent loop, where `model.decide` chooses the
> next action each turn from a fixed tool set — same operations, different
> owner of the control flow.)*

## 3. Three levels: task, workflow, agent

A precise vocabulary helps, because "agent" gets used for almost everything.
Anthropic, in its guide on building effective agents, and Barry Zhang from
its agents team, propose a three-level scale that's genuinely useful for
deciding.

**Task.** A single model call. Summarize this, classify that, extract these
fields. Two years ago this looked like magic; today it's the foundation
everything else is built on. Cost is predictable and failure modes are
bounded.

**Workflow.** Several model calls chained in a control flow *you* define.
Reformulate, retrieve and generate is a workflow. You write the steps; the
model fills them in. This is where most of a well-built RAG system lives,
and that's exactly right.

**Agent.** The model directs its own process. It decides the next action
from what it observes in the environment — the result of a search, the
output of a calculation — and keeps going until it judges it's done. You own
the goal and the guardrails; you don't own every branch of the path.

The line that sums it up best is Zhang's: with a workflow, *you* control the
plumbing; with an agent, the *model* controls the plumbing. Everything else —
cost, latency, testability, observability — follows from that one structural
difference.

Seeing it in code makes it concrete. A workflow is the function from before:
the sequence is written. An agent, in its most honest form, is a loop:

```python
def run_agent(transcript: str) -> Estimate:
    messages = build_initial_context(transcript)
    for _ in range(MAX_STEPS):
        decision = model.decide(messages, tools=TOOLS)
        if decision.is_final():
            return decision.estimate
        observation = execute_tool(decision.tool_call)
        messages.append(observation)
    raise AgentDidNotConverge()
```

(This is a schema, not a real API.) The difference from the workflow isn't
the loop itself — loops are nothing new. It's the `model.decide` line: who
chooses the next step. In the workflow, you chose it when you wrote the
sequence. In the agent, the model chooses it every turn, based on what it
just observed. That's the entire novelty. Everything else is control flow you
recognize from any program you've written.

## 4. What an agent actually buys you

What you buy with an agent is exactly one thing: the ability to solve
problems whose decision tree you cannot pre-map.

In our estimation system, that translates into something concrete. The agent
reads the complex project's transcript, decides there are four components
with different profiles, and treats each as a sub-task: it searches historical
budgets for the ERP integration on one side, for the legacy migration on the
other, computes partial estimates with the right references, and
consolidates. When a search comes back thin or empty, it can reformulate and
retry before computing on bad data. That path — four searches, two retries,
one calculation per component, one consolidation — wasn't written by you. The
model built it on the fly, from the content of that specific transcript.

Notice what you *don't* buy. You don't buy better retrieval: the agent
searches with the same tools as always. You don't buy better generation: it
consolidates with the same model. You don't buy new intelligence. You buy
**exclusively adaptive orchestration**. If your problem has a fixed shape,
there's nothing here for you, because you already had fixed orchestration
solved, and cheaper.

## 5. The price of autonomy

This is where the marketing usually goes quiet. An agent pays for its
flexibility, and the price isn't small.

**Latency.** Every turn of the loop is a round trip to the model. The
pipeline made one or two calls; the agent can make eight before converging.
The user who expected two seconds now waits twenty.

**Cost.** Exploration costs tokens. As a rule of thumb, Zhang suggests that
about ten cents of a dollar per task is equivalent to thirty to fifty
thousand tokens. An agent that reasons, searches four times and consolidates
eats through that budget easily, multiplying what the pipeline cost several
times over. At scale, the difference stops being anecdotal: an operation
processing a million requests a month while spending five times the
necessary tokens burns on the order of an extra million and a half dollars a
year. Whether that's reasonable depends entirely on the value of each task.

**Non-determinism.** The same transcript can take different paths on two
runs. That complicates testing — checking output against an expected value
is no longer enough — and turns reproducing a bug into an exercise in
patience.

**Errors compound.** In a pipeline, a bad retrieval produces a bad answer —
one bounded failure. In an agent, a bad retrieval at step two can turn into
three more steps built on that rotten foundation. Each iteration multiplies
the failure rate of the weakest link. Autonomy amplifies both successes and
mistakes.

**Observability debt.** With a pipeline, logging inputs and outputs was
enough. With an agent you need to trace decisions: what it reasoned, which
tool it chose, what it observed, why it continued. Without that trace,
debugging an agent that "sometimes gets the estimate wrong" is nearly
impossible.

None of these costs is an argument for never using agents. They're the price
you pay, and the sensible decision is to check that what you buy is worth
more than what you pay.

## 6. The decision criteria

Translated into questions you can ask in front of a concrete problem:

- **Can you pre-map the decision tree?** If you can enumerate the steps and
  their branches, build it as a workflow. You'll get more precision, more
  control and less cost than any agent. Being able to map it is the
  strongest signal that you don't need agency.
- **Does the problem have a variable shape?** If the number of steps, their
  order, or their nature depend on the input in ways you can't enumerate
  ahead of time, you're in agent territory. The complex transcript is; the
  landing page isn't.
- **Does the value justify the spend?** Exploration costs money. A
  high-volume, low-per-unit-value task — classifying millions of tickets —
  is workflow territory, almost always. A low-volume, high-value task —
  estimating a six-figure project — can easily justify an agent's overhead.
- **What's the cost of an error, and can you verify it?** If an error is
  expensive and hard to detect, autonomy becomes a liability. This is where
  mitigations matter: read-only tools, automatic validation of the output
  before accepting it, and a human in the loop at critical points. An agent
  whose actions are all reversible and verifiable is far less risky than one
  that writes to production.
- **Is the model actually good enough at your domain?** If the model doesn't
  reason reliably about software budgets, and doesn't recover from its own
  mistakes, don't hand it the wheel. Agency on top of a model that doesn't
  master the domain just produces more elaborate failures.

There's an agent use case that works well today and illustrates the criteria
perfectly: coding agents. The problem is ambiguous (there's no fixed path to
fixing a bug), the output's value is obvious, current models are good at it,
and — crucially — the result can be verified with tests. When your problem
meets those four conditions, the agent earns its place. When it doesn't, be
suspicious.

## 7. How this applies to our estimation system

Time to commit, because the conclusion isn't "replace the pipeline with an
agent."

The pipeline stays the default path. The simple transcripts, which are the
majority, go through it as-is: faster, cheaper, deterministic. There's no
reason to pay the agency tax on a problem that already has a fixed shape.

The agent enters as a **decision layer above the pipeline, not as its
replacement.** And the detail that makes this clean instead of a redesign is
that the pipeline's pieces become the agent's tools. The budget retrieval you
already had becomes a tool, `search_budgets`. The cost calculation becomes
another, `calculate_estimate`. Output validation, `validate_estimate`. You
reimplement nothing: you promote the workflow's steps to invokable actions
and let the model sequence them when the problem's shape is unknown. For
complex transcripts, the agent orchestrates exactly the same primitives the
pipeline executed in a fixed order.

This suggests a two-path architecture with a cheap router up front: a
lightweight classifier decides whether the transcript is simple — pipeline —
or complex — agent. That way you pay for autonomy only when the problem
demands it, and keep the average cost under control.

One point worth not blurring: all of this lives inside the AI service. It's
an internal detail of it. The business backend still sends a transcript and
receives a structured estimate over the same contract as always; it doesn't
care whether a three-step pipeline or an agent that made eight calls produced
it behind the scenes. That separation is what lets you introduce the agent
without touching the business layer or the frontend, and what will let you
remove it or change it tomorrow if the cost doesn't pay off. Agency is an
implementation decision of the AI service, not a product architecture
change.

## 8. The agent, from the code's point of view

If you've made it this far, the conclusion should sound almost
disappointing, and that's exactly the point.

An agent isn't a new paradigm that retires your software engineering. It's a
control-flow decision. Instead of you writing the `if`/`else` that picks the
next step, the model emits the next action and you execute it in a loop.
That's the novelty, and it's bounded. Everything around that one line — the
tools and their contracts, output validation, decision observability, cost
and latency control, the stopping condition so the loop doesn't run away —
is engineering you already know how to do. Interface design, error handling,
limits and timeouts, traceability. None of it is AI-specific.

That's why the hard part of working with agents isn't building them. The
loop is twenty lines. The hard part is deciding whether you actually needed
one, and resisting the temptation to reach for it where a workflow would
have done the job better, cheaper and with fewer surprises. Always start
with the simplest solution that passes your tests. Move up a rung from task
to workflow, from workflow to agent, only when the shape of the problem
forces you to. By default, the pipeline. The agent, when you have no other
choice.

## Sources

- Anthropic, *Building Effective Agents* — the workflow/agent distinction
  and the recommendation to start with the simplest solution:
  https://www.anthropic.com/research/building-effective-agents
- Barry Zhang (Anthropic), *How We Build Effective Agents* — the
  task/workflow/agent taxonomy and the cost math, synthesized in:
  https://shellypalmer.com/2026/04/how-anthropic-thinks-about-agents-workflows-and-tasks/
- OpenAI, *Function calling* (Responses API) — the mechanics of tools and the
  execution loop: https://developers.openai.com/api/docs/guides/function-calling

---

> *(Editor's note — relationship to s05-05.)* Part 5's Actor-Critic-Boss
> (s05-05) is not an instance of the general agent this article defines, and
> the distinction is worth being precise about because both live in
> `PLAYBOOK.md`'s architecture decision as separate axes. ACB is a **bounded**
> loop over a **fixed** set of roles (Actor generates, Critic evaluates
> against a rulebook, Boss decides iterate/accept/stop), capped at a stated
> iteration count, with no tool selection happening anywhere in it — closer to
> this article's "workflow" level than its "agent" level, even though it
> iterates. The agent this article describes chooses **which** tool to call
> and **how many times**, from an open-ended set, based on what it observes —
> that's the `model.decide` line, and ACB has no equivalent to it. A system
> can legitimately have both: a bounded Critic checking a rulebook (Axis 4)
> and, separately, a decision layer choosing how many budget searches a
> complex transcript needs (this article's territory) — they answer different
> questions and neither substitutes for the other.

> *(Editor's note — domain transfer.)* The five-question checklist in §6
> transfers cleanly; the two worked examples in §6 and the coding-agent
> aside do not, on their own terms. Two independent reference systems in this
> handbook's own orbit already ran this checklist and landed on "no,"
> without having read this article: `fantasy`'s own `ARCHITECTURE.md` states
> as an explicit ADR (ADR-007) that nothing in it "chooses its own next
> action, calls tools in a loop, routes between specialists, or decides when
> it is finished" — its weekly path pre-maps its decision tree exactly
> because, per §6's first question, it could. A second system built against
> this same handbook's `PLAYBOOK.md` (a personal trading advisor, unpublished
> here) rejected orchestration on the same grounds: a reference multi-agent
> implementation it studied runs a fixed set of five analysts in a fixed
> order every time, which is Phase composition dressed as agency, not this
> article's variable-shape trigger. Both are corroborating evidence for §6's
> framework, arrived at independently — not domain transfer failures, the
> opposite: the checklist held up outside the estimator's own domain.

> *(Editor's note — checked against the code, `lidr/ai-engineering`,
> branch state at session 11 completed.)* The article's own pseudocode is
> explicitly marked "a schema, not a real API," so this is a mapping, not a
> correction. `estimate_from_transcript` and `generate_estimate` exist under
> those exact names in `app/generation/rag/estimator.py`. `reformulate`
> is `reformulate_query()` in `app/generation/rag/query_reformulator.py`.
> `search_budgets` has no single real counterpart — the real pipeline splits
> it into `compose_search_text()` + an embedding call + `retrieve()` (which
> wraps `search_chunks()` in `app/generation/rag/retriever.py`), and adds a
> soft-fail branch, hybrid/rerank runtime config (Session 10), a
> token-budget truncation, and a citation-repair retry the article's
> three-line sketch doesn't show. None of that contradicts the article — the
> three-step shape is the right conceptual compression for the point being
> made — but a reader promoting these steps to literal tools, as §7
> recommends, is promoting nine real stages, not three, and `search_budgets`
> as a tool name would need to wrap that whole retrieval sub-pipeline, not
> just `search_chunks`. As of this commit, no `TOOLS`, `model.decide` loop,
> or router of the kind §7 describes exists in the codebase — session 12 is
> ahead of the code, not behind it, which is the first time in this handbook
> that's been true.
