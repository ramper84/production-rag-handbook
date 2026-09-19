---
title: "When a multi-agent system stops being 'a graph with more nodes'"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-09-19
summary: >
  From outside, a multi-agent system looks a lot like a graph with more
  nodes and more ambitious names — and if the only difference is calling
  what used to be search_budgets "budget_searcher," that's architecture
  theater, paid for in latency and 3am debugging. The linear graph's real
  ceiling shows up in four recognizable ways: a node's prompt accumulating
  rules from unrelated domains (coupling); a tool set grown too large for
  one decision space (choice accuracy degrades with option count); the
  step order genuinely not known ahead of time, so a hand-coded decision
  tree of conditional edges ages badly; and responsibilities evolving on
  different cadences, an org problem before a technical one. What separates
  a workflow from an agentic system isn't node count — it's who owns
  control flow, code or the model. Once justified, specialists either
  cooperate (each contributes a distinct piece, one pass, one weak link
  contaminates everything downstream) or compete (two+ attack the same
  task with different criteria, a synthesizer resolves the divergence,
  which is itself signal about uncertainty — at 2-3x the cost, and only
  worth it when the competing criteria actually diverge, not when two
  near-identical prompts create the illusion of a second opinion). The
  real costs — per-hop routing calls with no work product, context loss at
  every handoff, non-determinism in control flow, a larger failure
  surface, and a real cognitive cost for the next engineer — are the price,
  not a reason never to pay it.
keywords: [multi-agent systems, supervisor pattern, control flow ownership,
           tool set size, cooperation topology, competition topology,
           synthesizer, architecture theater, routing cost, context loss,
           non-determinism, failure surface]
---

# When a multi-agent system stops being "a graph with more nodes"

*Antonio Perez* · 🔴 18 min

You have an estimation graph that works. It receives a transcript, extracts
requirements, classifies components, searches historical budgets, generates
an estimate, and validates it. Five nodes, typed state, a checkpointer over
Postgres, clean traces. It does what it promises.

And now someone tells you it needs to become a multi-agent system.

The right question in the face of that sentence isn't "how?" It's "why?"
Because from outside — and this is the first thing worth defusing — a
multi-agent system looks a lot like a graph with more nodes and more
ambitious names. If the only difference were calling what used to be
`search_budgets` `budget_searcher`, we'd be doing architecture theater. And
architecture theater gets paid for in latency, in token cost, and in
3am debugging sessions over why the system made a decision nobody wrote.

This article is about the boundary. About what concrete limit has to have
hit you for multiplying agents to stop being a free complication and start
being the right answer.

## 1. The single graph's ceiling

Let's start with what does work, because last session's graph isn't a
draft to be outgrown. It's a legitimate architecture that solves an entire
class of problems, and many production systems don't need to go past it.

What a directed graph with function-nodes does is fix control flow in the
code. You decide, at write time, that requirements get classified into
components after extraction, and that budgets get searched after that.
Conditional edges give flexibility, but the flexibility is bounded: they're
branches someone foresaw and wrote. The model fills in blanks; it doesn't
choose the path.

That property is a huge advantage. A deterministic flow is predictable,
cheap to trace, and easy to test. When the business process you're
modeling has a stable sequence — and estimating software, in its canonical
form, does — the linear graph is the right answer. Starting with
multi-agent when the flow is fixed is like standing up a microservices
architecture for a three-table CRUD: it isn't wrong for being complex, it's
wrong because the complexity buys nothing.

The ceiling shows up when that mental model stops holding. And it shows up
in fairly recognizable ways.

**A node's prompt starts accumulating rules from unrelated domains.** Look
at the node that generates the estimate. If its system prompt says how to
calculate hours, and also how to interpret historical budgets that don't
quite match, and also how to adjust for the client team's seniority, and
also how to react if the transcript mentions a legacy ERP integration...
that node isn't a step anymore. It's an agent overloaded with four
responsibilities fighting each other inside the same context window. The
classic symptom: you touch one prompt rule to fix a case and something
unrelated breaks. That's coupling — the same coupling you'd recognize in an
800-line class.

**The tool set grows within a single decision space.** If one node has
access to six, eight, twelve tools, the model has to discriminate among all
of them on every call. The rate of picking the wrong tool rises with the
number of options, the same way a human's would rise if you handed them
twelve poorly labeled buttons. Splitting those tools across agents with
fewer options each isn't just safety hygiene — it improves accuracy.

**The order stops being known ahead of time.** This is the most decisive
signal, and it's worth pausing on. In your graph, you fixed the order.
But imagine a transcript where the client describes three independent
modules: one is a familiar CRUD, another is an integration with a system
that has no precedent, and the third is a data migration. The optimal path
for each module is different. The CRUD just needs a budget search and a
calculation. The unprecedented integration needs requirements extracted in
much more detail, a search by analogy, and probably shouldn't be trusted at
face value. Coding all those branches as conditional edges is possible, but
the graph becomes a hand-written decision tree that ages badly.

When whoever decides what runs next stops being the code and becomes the
model, you've crossed the boundary. That's what separates a workflow from
an agentic system: not the number of nodes, but who owns the control flow.

**Responsibilities evolve at different rates.** A purely software-
engineering argument, and probably the most familiar one. If the data team
iterates weekly on how budgets get searched and the business team touches
validation rules once a quarter, keeping them in the same node is an
organizational problem before it's a technical one. Different axes of
change ask for different components. You've been applying this your whole
career; nothing changes here.

> *(Figure in the original: `fig-01-grafo-lineal-vs-supervisor.png` — image
> not included in this repo. Two panels side by side. "S13 - Linear graph:
> fixed route, the code decides the order" — five purple boxes in a
> straight vertical chain: `extract_requirements` → `classify_components`
> → `search_budgets` → `generate_estimate` → `validate_estimate`, captioned
> "all the tools, a single prompt." "S14 - Supervisor and workers: dynamic
> route, the model chooses at runtime" — an orange `supervisor` box at top,
> dashed lines labeled `Command(goto=...)` fanning out to four teal worker
> boxes in a 2×2 grid: `requirements_extractor` [tools: -],
> `budget_searcher` [tools: search_budgets], `estimate_generator` [tools:
> calculate_estimate], `coherence_validator` [tools: validate_estimate] —
> all sitting above a shared box captioned "typed shared state (the same
> one from S13)".)*

Notice what the figure does **not** show. No new infrastructure. The shared
state is the same typed state. The nodes are the same functions. The only
thing that changed is that there's now a node that decides, and nodes that
only see their own tools. A multi-agent system, in the shape that's going
to serve you in production, is your graph reorganized.

## 2. Two ways to split the work

Once going multi-agent is justified, there's a second decision people skip
that determines the whole system's cost and behavior: do the agents
**cooperate** or **compete**?

### Cooperation: each agent contributes a distinct piece

This is the split-by-specialization. The extractor produces requirements,
the searcher produces analogous budgets, the generator produces hours, the
validator produces a verdict. None does another's work; the result is the
composition of all of them.

This is the topology that makes sense by default for estimation, and for a
concrete reason: the contributions are orthogonal. Extracting requirements
from a transcript and validating cost coherence are tasks that don't
compete with each other — they need each other. Asking two agents to
extract requirements in parallel just to keep the best one would cost
double to get, almost always, the same thing.

The cost is one pass through the flow. The risk is the classic one for
chains: a weak link contaminates everything downstream. If the extractor
drops a requirement, neither the searcher, nor the generator, nor the
validator has any way to know — none of them saw the original transcript
with that responsibility.

### Competition: several agents propose, someone synthesizes

Here two or more agents attack the same task with different criteria, and
a third decides. In estimation the example is almost too natural: one
agent estimates in conservative mode (assumes friction, integrations that
go sideways, requirements that grow) and another in aggressive mode
(assumes a competent team and stable scope). A synthesizer receives both
proposals and produces the final result.

The interesting part isn't that the synthesizer "picks the good one." The
interesting part is that the divergence between the two proposals is
information you didn't have. If the conservative one says 340 hours and
the aggressive one says 190, that gap is telling you the project has a lot
of structural uncertainty. If both converge on 250 and 270, the case is
predictable. In the single-estimator graph that signal doesn't exist: you
get a number and don't know how much to trust it.

> *(Figure in the original: `fig-02-cooperan-vs-compiten.png` — image not
> included in this repo. Two panels. "Cooperate: each agent contributes a
> distinct piece" — four teal boxes [`extractor`, `searcher`, `generator`,
> `validator`] converging into one green box "one estimate," captioned
> "cost: one pass. Risk: a weak link contaminates the final result."
> "Compete: two agents propose, a third synthesizes" — two teal boxes
> [`conservative_estimator` 340h, `aggressive_estimator` 190h] converging
> into one green box [`synthesizer`: "260h + justified range"], captioned
> "cost: double or triple. Gain: the divergence between proposals is a
> signal of uncertainty.")*

Competition is paid for literally at double or triple: two generations plus
a synthesis. And it has a subtle trap. If the two competing agents share
the same context, the same model, and prompts that differ only by one
adjective, their outputs will correlate far more than you'd expect, and
you'll be paying for three calls for the illusion of a second opinion.
Competition earns its keep when the criteria genuinely diverge: a
different prompt, a different evidence set, ideally a different model.

Rule of thumb: cooperation to decompose work; competition to attack
uncertainty. And don't mix them by default — applying competition to
everything multiplies your bill without multiplying quality.

## 3. What it's going to cost you

A multi-agent system is an architecture decision with real trade-offs.
They deserve to be on the table before the first line gets written.

**Cost and latency.** Every routing hop the supervisor makes is a model
call that produces no work of its own — it only decides. In a
central-supervisor topology, a task touching two specialists means four
calls (supervisor → agent A → supervisor → agent B) where the linear graph
made two. You're paying for routing. It can be worth it; what can't happen
is discovering that on the bill.

**Context loss at handoffs.** When one agent passes the baton, what does
the next one carry? Pass the entire message history and context grows
without bound, back to the exact problem you were trying to avoid. Pass
only a summary and that summary becomes a semantic bottleneck: whatever
isn't in it doesn't exist for the next agent. There's no universal answer.
There's an explicit decision you have to make, and no library makes it well
for you.

**Non-determinism in control flow.** This is the direct counterpart of the
main advantage. If the supervisor decides the route, two runs over the
same transcript can take different paths. That complicates tests,
complicates reproducing bugs, and complicates explaining to a client why
yesterday's estimate doesn't match today's. It's manageable — traces,
checkpoints, low temperature on routing, tests against the result rather
than the path — but it's a permanent tax.

**A larger failure surface.** Five agents are five places the model can
hallucinate, five tool sets that can be invoked wrong, and a router that
can get stuck in a loop. The linear graph, for all its rigidity, had a
valuable property: when it failed, you knew exactly where.

And the cost nobody counts: the cognitive one. The next developer who opens
the repository has to understand five prompts, a routing protocol, and a
shared-state whiteboard, instead of reading five functions in order. If the
system isn't solving a problem that justifies that, you've done them a
disservice.

## 4. When not to do it

Be honest about these three:

- **If the flow is fixed, you don't need a supervisor.** A supervisor whose
  only policy is "first A, then B, then C" is an expensive conditional
  edge, with one extra model call and a free source of randomness. The
  linear graph already expressed that, and better.
- **If the real problem is a bad prompt, fix the prompt.** Splitting a
  mediocre prompt across four agents leaves you with four mediocre prompts
  and a coordination problem on top.
- **If you don't have observability, don't add agents.** A multi-agent
  system without per-node traces isn't a system — it's a black box with
  opinions. Instrumentation isn't an extra added afterward; it's the
  precondition for this architecture to be debuggable at all.

## What's still to decide

If you've made it this far with the sense that multi-agent looks
suspiciously like what you already know how to do — separate
responsibilities, limit what each component can touch, define contracts
between pieces — you've understood it. There's no new paradigm. There's a
small new layer over principles you've been applying for years.

But justifying the architecture is the easy part. What's still open is
harder and more concrete: someone has to route, and that someone is a node
deciding at runtime which specialist acts — how do you build it so every
one of its decisions is visible, not an act of faith? Agents have to
communicate, and how they do — shared state, a direct handoff, messages —
changes the whole system's cost and traceability. There will be cases
where the system shouldn't decide alone, and the pause for a person to
step in can't be an improvised `if` — it has to be persisted state and an
outward-facing contract. And every agent is going to have tools in hand,
which turns an architecture question into a privilege question: who can do
what, and what happens if it tries to do more.

Those four questions — routing, communication, human intervention, and
privilege — are the rest of the road.

---

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed.)* No supervisor node, `Command(goto=...)`
> routing, or per-agent tool scoping exists — consistent with every article
> since `s12-01`. `app/generation/agentic/` holds only `boss.py` and
> `critic.py` (Session 5's Actor-Critic-Boss), not a multi-node supervisor.
> One near-miss worth flagging precisely: "conservative" and "aggressive"
> both appear in existing prompt files
> (`app/foundation/prompts/estimation/v3/system.j2`,
> `app/foundation/prompts/critic/v1/system.j2`), but only as single-word
> tuning adjectives inside one prompt ("be conservative: round hours up...",
> "conservative-industry rates") — not this article's §2 pattern of two
> *separate* agents run under two different strategies and reconciled by a
> synthesizer. Same word, unrelated pattern; a repo search for
> "conservative"/"aggressive" alone would over-suggest this competition
> topology is already half-built. It isn't.

> *(Editor's note — competition is not a fan-out/fan-in instance of
> `s13-04`'s pattern, despite the surface resemblance.)* `s13-04`'s
> `Send`-based fan-out parallelizes **identical** work over **independent**
> inputs (one budget search per component, same prompt, same tool, no
> disagreement possible — the reducer just concatenates). This article's
> "compete" topology parallelizes **different** work (different prompts,
> different strategies) over the **same** input, and the fan-in is a
> synthesis call, not a reducer concatenation — the aggregation step is
> itself a model call reasoning about the disagreement, not
> `operator.add`. Building "compete" as a `Send`-based fan-out with a
> merge-node synthesizer is the right graph mechanics, but don't reach for
> `operator.add` expecting it to do the synthesizing; that reducer has
> nothing to say about which of two divergent estimates, or what blend of
> them, is right.

> *(Editor's note — directly and substantially backs `PLAYBOOK.md`'s Axis
> 5, more than any article since `s12-01` itself.)* Axis 5's existing
> trigger 2 ("multiple independent specialist roles... data-dependent")
> was written as this playbook's own extrapolation, before any article
> gave it concrete, recognizable symptoms. This article supplies exactly
> that: prompt coupling across unrelated domains, tool-set size degrading
> choice accuracy, order genuinely unknowable ahead of time (the sharpest
> of the four, matching what the axis already called "genuinely depends on
> intermediate results"), and responsibilities changing at different
> cadences — the last one new to this playbook entirely, an organizational
> signal rather than a technical one. It also supplies design guidance this
> playbook never had: once Axis 5 is selected, cooperate-vs-compete is a
> second, distinct decision with its own cost/benefit shape, not a detail
> of "how you build the supervisor loop." `PLAYBOOK.md` updated to cite
> this directly.
