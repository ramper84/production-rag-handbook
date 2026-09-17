---
title: "Anatomy of an agent: what happens inside the loop"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 22 min
added: 2026-09-16
summary: >
  "A loop" doesn't tell you what happens inside one turn of it, and that's
  where control of the system is won or lost. Five organs, named so they can
  be debugged: reasoning (decide the next action), planning (decide the
  shape of the whole), action (the one point where the agent touches
  anything outside itself, mediated by least-privilege tool access),
  observation (the agent's only ground truth, and an informative error is
  what lets it recover), and handover (to a human via a needs_review status,
  or to another agent). Reasoning-model-era addendum: the ReAct Thought is
  now largely native and opaque by default — reasoning summaries are the
  observability lever, captured deliberately or not at all. State grows one
  decision-and-observation pair every turn, so cost grows with it — the
  eighth turn pays to resend the first seven observations.
keywords: [ReAct, reasoning, planning, action, observation, handover,
           human in the loop, tool use, function calling, agent loop,
           AgentResult, reasoning summaries, state growth, least privilege]
---

# Anatomy of an agent: what happens inside the loop

*Antonio Perez* · 🔴 22 min

An agent, reduced to its most honest form, is a loop: the model decides an
action, your code executes it, the result goes back to the model, and it
repeats until there's an answer. That's true, but "a loop" tells you nothing
about what happens inside each turn. And that's where control of the system
is won or lost.

This article opens one turn of the loop and names its parts: reasoning,
planning, action, observation, and the handover that lets the agent step
aside when it should. This isn't a taxonomy exercise. Naming the parts is
exactly what lets you debug, measure cost, and decide where to intervene.
You can't debug what you can't name, and an agent without anatomy is a black
box that "sometimes gets the estimate wrong."

Let's set the scene, because we need it for everything else. We have a
software project estimation system: it receives a meeting transcript and
produces a structured estimate. The agent has tools available —
`search_budgets`, which retrieves comparable historical budgets;
`calculate_estimate`, which computes costs from references;
`validate_estimate`, which runs checks on the result — and its job is to
orchestrate them when the transcript is complex enough not to have a fixed
shape.

## 1. The skeleton: reason, act, observe, repeat

The canonical formulation of this loop comes from Yao and co-authors' work
on ReAct, in which a model interleaves reasoning traces and actions:
reasoning is used to induce, track and update a plan and to handle
exceptions; actions are used to interact with external sources and bring in
new information. The idea is that reasoning and acting separately is worse
than interleaving them: reasoning without action runs out of fresh data and
hallucinates; action without reasoning doesn't know what to do with what it
brings back.

In its original form, ReAct was a prompting technique: you wrote example
trajectories with an explicit text format.

```
Thought: The transcript describes two independent components; I will price them separately.
Action: search_budgets(query="ERP integration REST")
Observation: 4 historical budgets found; median 120h for a similar integration.
Thought: The migration component has no clear match; I need to reformulate.
Action: search_budgets(query="legacy data migration undocumented schema")
Observation: 1 weak match; low confidence.
```

That pattern — Thought / Action / Observation, looped, until a final answer
— is the skeleton. The rest of the article is the organs hanging off it.
And there's an implementation detail worth flagging up front, because it
changes how this gets built today: with current reasoning models, most of
the Thought is no longer something you write into the prompt; it happens
natively inside the model. We'll come back to this.

A skeleton also needs a stopping condition. An agent loop without a guard is
a bug waiting to happen: either the model gives a final answer, or a step
maximum is reached, or an error budget runs out. Without that guard, a
confused agent iterates until it burns through your quota. The stopping
condition isn't a detail — it's part of the skeleton.

> *(Figure in the original: `S12-fig-02a-anatomia-bucle.jpg` — image not
> included in this repo. It diagrams one turn: `model.decide` [reasoning +
> planning] → `act` → an action node ["executes a tool"] → `observe` → an
> observation node ["result from the environment"], looping back to
> `model.decide` until `is_final` or `MAX_STEPS`, with a side branch from
> `model.decide` labeled `needs_review` going to a human-review node, and a
> callout: state grows every turn — each iteration adds decision +
> observation to the context, more resent context, more cost per turn.)*

## 2. Reasoning: deciding what to do

Reasoning is the faculty of interpreting the situation and choosing the next
action. It's where the agent looks at what's in front of it — the
transcript, what it has observed so far — and concludes something like:
"this describes an ERP integration and a legacy data migration; they're
different beasts, I'll estimate them separately."

Here's the nuance that separates the mental model from the real
implementation. The ReAct pattern was born making reasoning explicit in
text, and that had a virtue: you could read it. Current reasoning models do
that work natively, spending internal reasoning tokens you don't write or
see directly. The Thought doesn't disappear; it moves inside the model.
What you're left with for observability are the **reasoning summaries**
some providers expose: summaries of the reasoning chain, useful for
debugging and auditing without having to force the text format by hand.

The practical consequence is twofold. On one hand, you no longer have to
teach the model to reason with `Thought:` examples; it does it on its own,
and often better. On the other, you lose fine-grained control over that
trace: if you need strict auditability, you have to deliberately capture the
reasoning summaries, because by default the reasoning is opaque. It's
neither a regression nor a clean improvement — it's a change in where the
reasoning lives and what levers you have over it.

## 3. Planning: decomposing the problem

If reasoning decides the next step, planning decides the shape of the whole:
how the problem breaks down into steps. In our case, planning is reading the
transcript and concluding there are four components to estimate — customer
portal, ERP integration, mobile app, legacy migration — and that each
deserves its own budget search.

There are two moments this planning can happen, and it's worth
distinguishing them because they have different implications. One is
upfront planning: the agent sketches the steps at the start and then
executes them. The other is continuous planning: the agent decides the next
step every turn, in light of what it just observed. Upfront planning is more
auditable — you have the plan written down before spending — but more rigid
in the face of surprises; continuous planning adapts to what it finds but is
harder to anticipate and budget for.

With capable models, planning tends to be emergent rather than an explicit
step: the model decomposes on the fly without you asking for a formal plan.
That's usually enough. But naming planning as a component gives you a lever:
when you need auditability — justifying to a client why the estimate came
out the way it did — you can force an explicit plan as the loop's first step
and save it. The decision to force it or let it emerge is yours, and it's a
design decision, not a model detail.

## 4. Action: touching the world

Action is the one point where the agent affects something outside itself.
Everything else — reasoning, planning, observing — happens inside the
model's head or in your state management. Action is where it calls
`search_budgets` and something actually happens: the vector database gets
queried, budgets get retrieved.

Mechanically, this is function calling, and its contract is exactly that of
any typed interface you've always known: you declare what operations exist
and the shape of their inputs and outputs; the model emits a structured
request; your code executes it and returns the result. The model never
executes anything on its own. It emits an intent — "call `search_budgets`
with these arguments" — and you decide what to do with it. An engineer with
API experience integrates this the same way as any other interface: define
the schema, handle the call, return the result. The difference is that the
caller, on the other side, is a model choosing the function based on the
conversation.

That mediation of yours over the action is where the agent's safety lives,
and it's worth not giving it away. Not all actions are equal. `search_budgets`
is read-only: reversible, cheap to get wrong, safe to grant. An action that
writes to production, sends an email, or moves money is something else. The
principle is least privilege: give the agent the actions it needs and not
one more, and irreversible actions go through a check — or a human — before
executing. That the model requests an action doesn't obligate you to execute
it as-is; you can validate the arguments first, and you must, for the
actions that hurt.

## 5. Observation: reading the environment's response

Observation is what you return to the model after executing an action, and
it's how the agent gets ground truth from the environment at every step.
Without observation, the model reasons about its own imagination; with it,
it corrects course from facts.

What's underestimated is how much the quality of the observation governs
the quality of the next decision. A high-value observation — structured,
concrete, just enough — feeds good reasoning. A bloated observation — two
hundred raw budget items when the five relevant ones would do — wastes
context and confuses the model. Returning stable, semantic identifiers, and
only the fields the agent needs to decide the next step, isn't cosmetic:
it's what keeps the loop focused.

Errors are a special case of observation, and probably the most important
one. When `search_budgets` finds nothing useful, that's an observation, and
a good one. If you return it to the model with information — "1 weak match,
low confidence for legacy migration" — the agent can reason and reformulate
the query. If you return it as a generic "error," or worse, hide it, the
agent is left blind and stumbles. An informative error message returned as
an observation is what lets an agent recover from its own failures; a mute
error is what makes it fail in incomprehensible ways.

## 6. Handover: knowing when to step aside

An agent shouldn't always finish the work alone. Handover is the transfer of
control to another party, and it has two directions.

The first is toward a human. This is the human-in-the-loop pattern: the
agent stops at a checkpoint and asks for judgment, or escalates when its
confidence is low or when the next action is expensive and irreversible. In
our system, imagine the legacy migration component has no reliable
historical reference — undocumented schema, nothing similar in the budget
history. The agent can estimate the rest competently and, for that piece,
step aside: mark the estimate as needing human review instead of inventing a
number with false precision. That's not a failure of the agent; it's a
well-designed agent recognizing the limit of what it can verify.

The second direction is toward another agent: delegating a sub-task to a
specialist better equipped for it. In single-agent systems this doesn't come
up, but it's the foundation multi-agent architectures are built on.

In both cases, the handover needs an explicit contract: what state gets
transferred, who becomes the owner of the decision, and how control comes
back (if it comes back at all). A handover without a contract is a ball
thrown in the air with nobody told they have to catch it.

This is where the agent's anatomy crosses into the system's architecture.
The agent lives inside the AI service, and the handover toward a human
translates cleanly into a contract: the AI service returns a status — say,
`needs_review` — along with whatever it did manage to compute and the
reason. The business backend routes that status to a person. From the
client side, in the reference Rails implementation, that's as simple as
this (the pattern is stack-independent: any HTTP client works):

```ruby
# business backend: routing the servicio IA response
result = ai_service.estimate(transcript)

case result.status
when "done"
  save_estimate(result.estimate)
when "needs_review"
  enqueue_for_human_review(result.partial_estimate, result.reason)
end
```

> *(Figure in the original: `S12-fig-02b-handover-capas.jpg` — image not
> included in this repo. It diagrams the same case-status routing: frontend
> ↔ business backend ↔ AI service [the agent loop, emitting `AgentResult`],
> the business backend branching on `case status` into `done` →
> `save_estimate` → back to the frontend as `estimacion`, or `needs_review`
> → `enqueue_for_human_review` → human review.)*

Handover stops being an abstract concept and becomes a `status` in a
response and a `case` that routes it. Ordinary software.

## 7. The anatomy, from the code's point of view

With all the pieces in place, one turn of the loop reads like this. It's a
schema, not a concrete API, but every line is an organ:

```python
def run_agent(transcript: str) -> AgentResult:
    state = build_initial_state(transcript)          # the accumulating context
    for step in range(MAX_STEPS):
        decision = model.decide(state)               # reasoning + planning
        if decision.needs_human:                     # handover to a person
            return AgentResult(status="needs_review", state=state)
        if decision.is_final:                        # stopping condition
            return AgentResult(status="done", estimate=decision.estimate)
        observation = execute_tool(decision.action)  # action + observation
        state.append(decision, observation)          # the loop carries the trace
    return AgentResult(status="max_steps_exceeded", state=state)
```

Notice the state. The loop isn't pure model memory: it's a structure you
maintain, and it grows every turn with the decision and its observation.
That accumulated state is the agent's trace — the Thought / Action /
Observation loop — and it's what you can log, inspect, and use to debug.
Every organ has its line: reasoning and planning live in `model.decide`;
action and observation, in `execute_tool` and its result; handover, in the
`needs_human` branch; stopping, in `is_final` and the `range(MAX_STEPS)`
wrapping it all. All inside the AI service. The business backend only sees
the final `AgentResult` and its status.

That growing state has a cost worth keeping in view, because it's the
direct counterpart of the anatomy. Every turn adds the decision and the
observation to the context, and that context gets resent to the model on
the next turn. In other words: the loop doesn't just make more calls, each
call is more expensive than the last, because it drags along everything
observed so far. An agent that takes eight turns over a complex transcript
is paying, on the eighth, to resend the previous seven observations. Naming
state as an organ is also acknowledging that it fattens, and that in long
agents you end up needing strategies to trim it — summarizing old
observations, discarding ones that no longer inform the decision, keeping
the identifier instead of the full content. This isn't premature
optimization: it's the structural consequence of a loop that accumulates,
and knowing it up front avoids the surprise on the bill.

## 8. Closing: anatomy demystifies

Name the parts and the agent stops being a mystery. Reasoning is decision
logic, just living inside the model now instead of in your `if`/`else`.
Planning is decomposing a problem, something you do every time you design a
function. Action is a function call with effects. Observation is a return
value you feed back into the flow. Handover is escalation and delegation, a
pattern from any serious work system. And the loop is control flow with a
guard, like any carefully-written `while` you've ever written.

That's the real usefulness of the anatomy: it isn't theory, it's what lets
you instrument the system. You can measure how many steps it takes, test
each organ in isolation — feed a fixed observation and check the decision,
run an action with known arguments and verify the result — and put limits
where they hurt. An agent whose organs you can name is a system you can
operate. One whose organs you can't tell apart is a black box you can only
pray to.

## Sources

- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*
  (ICLR 2023), arXiv 2210.03629 — the interleaved reasoning-and-action
  cycle: https://arxiv.org/abs/2210.03629
- Anthropic, *Building Effective Agents* — the agent that gets ground truth
  from the environment at every step and stops to ask for human judgment at
  checkpoints: https://www.anthropic.com/research/building-effective-agents
- Anthropic, *How tool use works* — action as a typed contract and the loop
  governed by a stopping condition:
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works

---

> *(Editor's note — builds directly on s12-01, not independent.)* Same
> three tools (`search_budgets`, `calculate_estimate`, `validate_estimate`),
> same `run_agent` shape, now with `AgentResult`/`needs_human`/`MAX_STEPS`
> filled in. Read in order — s12-01 establishes *whether* to build the
> agent at all; this article is what's inside it once s12-01's checklist
> says yes.

> *(Editor's note — handover vs. Axis 4's Critic, and the trading-advisor
> precedent.)* §6's `needs_review` handover and `PLAYBOOK.md`'s Axis 4
> (Actor-Critic-Boss) both put a human between a model and an consequential
> action, but they are not the same mechanism, and the difference matters
> for where each is documented. Axis 4's Critic is **deterministic code**
> checking a rulebook with a computable answer — no judgment call about
> *whether* to check, only about the rule's outcome. This article's
> handover is the **agent's own judgment call**, made by the same model
> that's doing the reasoning, that a case is outside what it can verify.
> The two compose rather than substitute: a system can have an agent whose
> `needs_human` branch is itself gated by a deterministic Critic before a
> human ever sees it. The unpublished trading-advisor project referenced in
> s12-01's domain-transfer note independently built exactly this
> `needs_review`-shaped contract — a human-approval gate between a
> recommendation and any executed action — for reasons unrelated to this
> article (it hadn't been read yet): the same shape recurring under
> different names is a fair signal it's the load-bearing pattern, not a
> stylistic choice.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed.)* None of `AgentResult`, `needs_human`,
> `needs_review`, `MAX_STEPS`, or `build_initial_state` exist in the
> codebase yet — same finding as s12-01: this article is still ahead of the
> reference implementation, not describing it. The Ruby snippet's
> `ai_service.estimate(transcript)` / `case result.status` shape does not
> yet exist in `estimator-web` either; the current Rails client consumes a
> single structured `EstimationResponse`, not a `status`-discriminated
> result with a `needs_review` branch — building §6's handover contract
> means extending that response shape, not just the FastAPI side.
