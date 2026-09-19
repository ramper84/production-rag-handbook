---
title: "Human-in-the-loop: interrupt, pause, and resume over the checkpointer"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-09-19
summary: >
  When the system knows it isn't in a position to answer alone, the
  correct response is uncomfortable: stop, show a person what it has, and
  wait — for minutes or days, in another process, maybe after a
  deployment, with state intact. That isn't a pause, it's persistence, and
  the checkpointer already built for crash recovery is the complete
  mechanism, unmodified: interrupt() raises a control exception LangGraph
  catches, writes state to the checkpoint, and returns control to the
  caller; Command(resume=...) continues from exactly that point later,
  through the same ainvoke, no special resume API. Three legitimate
  triggers, all boolean-evaluable conditions on state, never a vibe: low
  confidence the validator itself scored, an estimate outside the
  historical band (pure arithmetic, no model needed), no precedent above
  the similarity threshold. Three anti-patterns dressed as caution: routing
  agent failures to a human (that's an error, not a review), reviewing
  "just in case" until the reviewer rubber-stamps everything (a 60%
  trigger rate destroys the signal a 5% one preserves), and enforcing a
  hard business rule through the AI gate (that belongs in the business
  backend, which owns users and authorization). The pause crosses three
  layers, not one — a new status value already-existing contract fields
  can carry, and a /resume endpoint, with authorize! staying in the
  business backend because the AI service has no business knowing who's
  an authorized reviewer. The gotcha worth designing around from the
  start: a resumed node re-executes from its beginning, so the interrupting
  node does nothing but interrupt — no side effects before the call, ever.
  The human decision paired with what the system proposed is the single
  most valuable data point the system produces; persist it from day one,
  before you know what you'll do with it.
keywords: [human-in-the-loop, interrupt, Command resume, checkpointer,
           thread_id, confidence threshold, historical band, trigger
           conditions, authorize, business backend contract, idempotent
           resume, orphaned reviews]
---

# Human-in-the-loop: interrupt, pause, and resume over the checkpointer

*Antonio Perez* · 🔴 18 min

Your estimation system just produced 840 hours for a three-entity CRUD. In
the history, an equivalent CRUD has cost between 120 and 200. Nobody's
going to send that estimate to a client.

The question isn't how to stop the system from being wrong — it's going to
be wrong, and the interesting part here is that it *knew* something was
off: the validator had the historical band right in front of it. The
question is what the system should do when it detects it isn't in a
position to answer alone.

The easy answer is an `if` that returns an error. The correct answer is
more uncomfortable: the system has to stop, show what it has to a person,
and wait.

And waiting is the hard part. Because the person might take ten minutes or
three days. Your Python process can't sit there blocked on an `input()`.
The execution has to die and be able to resurrect exactly where it was,
with state intact, in another process, maybe on another machine, after a
deployment.

That isn't a pause. It's persistence. And it turns out you already have it
built.

## 1. The pause lives in the checkpointer

When you set up the checkpointer over the project's Postgres, you did it
for a practical reason: not losing work if something failed halfway
through the graph. That infrastructure is, without touching a line, the
complete human-intervention mechanism.

The reasoning is direct. The checkpointer persists the graph's state after
every node. If the state is persisted, the in-memory execution is
dispensable: it can be dropped and reconstructed from the checkpoint. And
if it can be reconstructed, it can be stopped indefinitely and continued
later.

A human-in-the-loop is exactly that: a stop that lasts as long as a human
takes.

```python
from langgraph.types import interrupt, Command

CONFIDENCE_THRESHOLD = 0.7

def human_review_gate(state: EstimationState) -> Command[Literal["finalize"]]:
    if not requires_human_review(state):
        return Command(goto="finalize")

    decision = interrupt(
        {
            "reason": build_review_reason(state),
            "estimate": state["estimate"],
            "confidence": state["confidence"],
            "budget_matches": state["budget_matches"],
        }
    )

    return Command(
        goto="finalize",
        update={
            "human_decision": decision,
            "estimate": apply_human_decision(state["estimate"], decision),
        },
    )
```

`interrupt()` isn't a `sleep`. It raises a control exception LangGraph
catches: the state is written to the checkpoint, the execution ends, and
the `invoke` you launched returns control to you along with the
interruption's information. The process is freed. And when the human
decision arrives — within a minute or a week — it resumes from that exact
point.

The payload you pass to `interrupt()` is what the person will see. Build it
carefully: it isn't a log, it's the interface. A reviewer shown a
`confidence: 0.42` and nothing else can't decide anything. A reviewer shown
the estimate, the historical band it collides with, and the analogous
budgets the system found, can.

## 2. Resuming

```python
config = {"configurable": {"thread_id": estimation_id}}

# First run: the graph may stop at the gate.
result = await graph.ainvoke({"transcript": transcript}, config)

# Later, once a human has decided:
result = await graph.ainvoke(Command(resume=human_decision), config)
```

`thread_id` is the piece that stitches it all together. It's the
identifier the checkpointer used to save the state, so it's what lets you
come back. Use the domain's own `estimation_id`, not a fresh UUID: the
business backend already has that identifier, already shows it in its own
UI, and it's going to be the same one the reviewer uses. When that
`thread_id` is also the key in your traces, you have a single identifier
crossing all three layers and one entire operation.

Notice the symmetry: `ainvoke` with an input starts the graph;
`ainvoke` with a `Command(resume=...)` continues it. Same function. There's
no special resume API, because for LangGraph, resuming isn't a special
case — it's what it always does, starting from a checkpoint that happens
not to be empty.

## 3. What should stop the graph

This is where it's decided whether your human-in-the-loop is useful or
theater.

The three legitimate signals in estimation:

**Low confidence.** The validator scores its own certainty. This isn't a
number that appears by magic: you ask for it explicitly, with a schema,
and give it criteria to produce it with.

```python
class ValidationResult(BaseModel):
    is_coherent: bool
    confidence: float = Field(ge=0.0, le=1.0)
    concerns: list[str]
    reasoning: str
```

**Outside the historical band.** This doesn't even need a model. You have
historical budgets indexed; you know what a CRUD has cost. If the estimate
falls outside the band by a large factor, it's an arithmetic comparison.

**No precedent.** If no analogous budget clears the similarity threshold,
the system is estimating blind. It isn't that it got something wrong — it
has no basis to get right.

```python
def requires_human_review(state: EstimationState) -> bool:
    validation = state["validation"]
    estimate = state["estimate"]

    low_confidence = validation["confidence"] < CONFIDENCE_THRESHOLD
    out_of_range = is_outside_historical_band(estimate, state["budget_matches"])
    no_precedent = len(state["budget_matches"]) == 0

    return low_confidence or out_of_range or no_precedent
```

The rule underneath: a trigger signal is a condition evaluable over the
state. If you can't write it as a boolean, it isn't a signal — it's an
intuition. And intuitions can't be tested, can't be tuned, and can't be
explained to a client.

> *(Figure in the original: `fig-02-senales-de-disparo.png` — image not
> included in this repo. Two panels. "Stop the graph. The system knows it
> doesn't know. The person brings judgment" — three orange bullets:
> "Confidence below threshold: the validator scores its own certainty,
> confidence < 0.7"; "Estimate outside the historical band: 840h for a
> CRUD that historically costs 120-200h"; "No precedent in the history: no
> analogous budget clears the similarity threshold" — below them, a boxed
> rule: "the signal is an evaluable condition over the state; if you can't
> write it as a boolean, it isn't a trigger signal, it's an intuition, and
> intuitions can't be tested." "Don't stop the graph. This isn't
> human-in-the-loop" — three red bullets: "An agent failed: that's an
> error — retry, fallback, or degradation, not a review"; "Just in case,
> on everything: if everything gets reviewed, nothing gets reviewed, the
> reviewer approves automatically"; "A hard business rule: if it always
> requires approval above 50k, that's an approval flow, not an AI gate".)*

## 4. And what shouldn't stop it

Three anti-patterns you'll see in production, all three disguised as
prudence.

**An agent failed.** That isn't a review, it's an error. A tool failure, a
timeout, or malformed JSON get resolved with a retry, a fallback, or
degradation. If you route errors to a human, you've turned your reviewer
into a manual retry service, and the day the provider's API goes down
you'll fill their queue with two hundred identical cases.

The distinction is clean: an error is the system failing to do its job; a
review is the system having done its job and the result needing judgment.

**Reviewing just in case.** The temptation to set the threshold high "until
we trust it." If 80% of estimates go through review, the reviewer doesn't
review — they approve automatically, in bulk, without looking. You've
built an elaborate interface for someone to click "accept" forty times in a
row, and on top of that you've destroyed the signal, because when the case
that actually mattered arrives, it'll arrive indistinguishable from the
rest. If everything gets reviewed, nothing gets reviewed. A
human-in-the-loop that triggers on 5% of cases is worth more than one that
triggers on 60%.

**A hard business rule.** If your company requires a partner to approve any
estimate over 50,000 euros, that's an approval flow, and it belongs in the
business backend. Don't put it in the graph. The AI service's gate exists
for a specific, different case: when the AI system knows it doesn't know.

## 5. The contract: the pause crosses three layers

And here's the part almost every tutorial skips, because in a notebook,
resuming is the next cell.

Not in your system. The pause happens in the AI service. The person
decides in the frontend. And in between is the business backend, which is
the one with users, permissions, and persistence. That pause is the first
thing in this entire program that modifies the contract between the
layers.

> *(Figure in the original: `fig-01-contrato-hitl-tres-capas.png` — image
> not included in this repo. Three horizontal lanes. "Frontend": `new
> estimation` → down into the business backend's `POST /estimations`; on
> the right, `review tray: approve/adjust/reject` receiving a `decision`
> arrow from `POST .../resume`, itself fed by `status ==
> "awaiting_human_review"`. "Business backend": `POST /estimations` down
> into `Servicio IA`'s `supervisor + agents` → `human_review_gate
> interrupt(...)` [orange] — dashed arrow to `finalize` — with a solid
> arrow labeled `status: awaiting_human_review + review payload` going back
> up to the business backend's `status ==` box, and `POST .../resume`
> feeding back down into the checkpointer. "Servicio IA": the same
> `supervisor + agents` → `human_review_gate` → `finalize` chain, all
> sitting above `AsyncPostgresSaver (checkpointer) — thread_id =
> estimation_id`, captioned "the paused state lives here. It can wait
> hours or days." Caption at bottom: `Command(resume=decision)` resumes
> from the checkpoint.)*

The minimal surface is two things: a new status in the response, and an
endpoint to come back through.

```python
@router.post("/estimations")
async def create_estimation(payload: EstimationRequest) -> EstimationResponse:
    config = {"configurable": {"thread_id": payload.estimation_id}}
    result = await graph.ainvoke({"transcript": payload.transcript}, config)

    if interrupts := result.get("__interrupt__"):
        return EstimationResponse(
            estimation_id=payload.estimation_id,
            status="awaiting_human_review",
            review_payload=interrupts[0].value,
            estimate=None,
        )

    return EstimationResponse(
        estimation_id=payload.estimation_id,
        status="completed",
        estimate=result["estimate"],
    )

@router.post("/estimations/{estimation_id}/resume")
async def resume_estimation(
    estimation_id: str, payload: HumanDecision
) -> EstimationResponse:
    config = {"configurable": {"thread_id": estimation_id}}
    result = await graph.ainvoke(Command(resume=payload.model_dump()), config)

    return EstimationResponse(
        estimation_id=estimation_id,
        status="completed",
        estimate=result["estimate"],
    )
```

The `status` field isn't new — it was already in your contract. All you're
doing is adding one more possible value to it. That detail matters more
than it looks: it means the business backend doesn't need a new
integration, only a new branch on a field it was already reading. The
architecture doesn't change; it extends.

On the business backend side, in the Rails reference (the pattern is
identical with any HTTP client):

```ruby
class EstimationsController < ApplicationController
  def create
    response = AiService.create_estimation(
      estimation_id: @estimation.id,
      transcript: @estimation.transcript
    )

    case response["status"]
    when "awaiting_human_review"
      @estimation.update!(
        status: :awaiting_review,
        review_payload: response["review_payload"]
      )
      ReviewMailer.with(estimation: @estimation).pending_review.deliver_later
    when "completed"
      @estimation.update!(status: :completed, estimate: response["estimate"])
    end
  end

  def resume
    authorize! :approve, @estimation

    response = AiService.resume_estimation(
      estimation_id: @estimation.id,
      decision: decision_params
    )

    @estimation.update!(status: :completed, estimate: response["estimate"])
  end
end
```

Notice where each responsibility lives. `authorize!` is in the business
backend, because that's what has users and roles: the AI service doesn't
know who's an authorized reviewer and has no business knowing. The
notification, the tray, the history of who approved what: all of that is
business. The AI service only knows how to pause and resume.

That separation isn't purism. It's what means that the day you change the
approval policy — now two reviewers are needed, or a junior can't approve
above a certain amount — you don't have to touch the graph.

## 6. What's going to bite you

**The node re-executes from the beginning on resume.** This is the most
expensive one and the least documented. When you resume, LangGraph doesn't
continue right after `interrupt()` — it re-runs the entire node from its
first line, and this time `interrupt()` returns the value instead of
stopping. Consequence: everything you did before `interrupt()` inside that
node runs twice. If you'd called the model in there, you pay for it twice.
If you'd inserted a database row, you insert it twice.

The rule, then: the node that interrupts does nothing but interrupt.
No side effects before `interrupt()`. Any real work goes in an earlier
node; the gate only evaluates a condition and stops.

**Orphaned resumptions.** What happens if nobody ever decides? The
checkpoint sits there, taking up space, and the estimate hangs forever in
`awaiting_human_review`. You need a policy: a deadline, an escalation to
another reviewer, or an expiration that closes the case. It isn't the AI
service's decision — it's business's — but somebody has to make it, and if
you don't, a client will discover it for you.

**Double resumption.** Two reviewers open the same estimate and both click
approve. The second call to `resume` arrives at a graph that already
finished. Your endpoint has to be idempotent, or the business backend has
to prevent it with a lock. It's the same problem as a form's double
submit, and it's solved the same way — it isn't an AI problem.

**The human decision is the most valuable data your system produces.**
Every time a person corrects an estimate, they're telling you exactly
where the model is wrong, on a real case, with the correct answer right
next to it. That pair — what the system proposed, what the human decided —
is gold: for tuning thresholds, for understanding which transcripts give
you trouble, and as evaluation material. Save it from day one, even before
you know what you're going to do with it. It's free now and unrecoverable
later.

## What's left

With this the system knows how to stop. It knows it doesn't know, tells
someone, and waits with its state safe.

But there's an odd asymmetry in what we've built. We've been very careful
deciding when the system shouldn't decide alone, and haven't spent a
paragraph on what each agent can do when it decides alone.

Your agents have tools in hand. They query databases. And the moment
someone adds the tool that was missing — the one that writes, the one that
sends, the one that deletes — the architecture question becomes a
different one: what can each agent touch, and what happens if it tries to
touch what it shouldn't.

---

> *(Editor's note — this is `s13-05`'s gotcha, restated with a stricter
> rule, not a repeat.)* `s13-05` already documented that a paused node
> re-executes on resume and said to keep prior work "cheap and idempotent."
> This article tightens that into something stricter and easier to hold to
> in practice: the interrupting node does **nothing** before `interrupt()`
> — no side effects at all, not even idempotent ones, moved to an earlier
> node instead. The looser s13-05 phrasing technically permits idempotent
> side effects before the call; this article's rule is simpler to audit
> ("does this node do anything but check a condition and call
> `interrupt()`?") and is the one worth following.

> *(Editor's note — where `human_review_gate` sits in `s14-02`'s graph is
> left implicit, and the article's own logic answers it.)* `s14-02`'s
> `AgentName` `Literal` — the supervisor's complete set of routable
> destinations — was `["requirements_extractor", "budget_searcher",
> "estimate_generator", "coherence_validator", "finalize"]`. It does not
> include `"human_review_gate"`. Wiring this article's gate into that
> graph requires deciding between two shapes this article doesn't name:
> add `"human_review_gate"` to the `Literal` so the supervisor can route
> to it as one more specialist, or wire it as a **fixed edge** after
> `coherence_validator`, bypassing the supervisor for this specific
> transition. `requires_human_review()` is itself a deterministic
> boolean check — no model call — which is exactly the case `s14-02` §6
> already argues belongs in code, not behind a model-driven router. By
> that article's own logic, the fixed-edge shape is the more consistent
> choice; routing review-eligibility through the supervisor would be
> paying a model call to make a decision this article just showed doesn't
> need one.

> *(Editor's note — checked against the code — Python side absent, Rails
> side present and a different shape entirely, not an extension point.)*
> On the Python side, no `interrupt`, `human_review_gate`, or
> `requires_human_review` exists — consistent with every article since
> `s12-01`. On the Rails side, unlike anything checked in this series so
> far, `estimator-web/app/controllers/estimations_controller.rb` **already
> exists** — but it's the Session 4 CAG-era controller: a synchronous
> `create` that builds an `Estimation::Request` from form params, calls
> `EstimatorAi::EstimationsClient#estimate`, and only creates the
> `Estimation` record *after* the response comes back
> (`Estimation.create!(response_payload: payload, ...)`), rescuing
> `GuardrailViolation`/`InvalidRequest`/`ServerError` into flash messages
> and a re-rendered form. There is no `resume` action, no `status`
> dispatch, no `authorize!`. More fundamentally, the request shape is
> backwards from what this article assumes: the real controller doesn't
> know an `estimation_id` until *after* the AI call returns, while this
> article's version needs the business backend to already have
> `@estimation.id` *before* calling `create_estimation`, to use as
> `thread_id`. Adopting this article's flow isn't adding a `resume` action
> to the existing controller — it's restructuring `create` itself to
> persist an `Estimation` record first, then call the AI service with that
> id already in hand.

> *(Editor's note — answers the third of `s14-01`'s four open questions,
> and reconnects to Part 12's still-open thread.)* Routing (`s14-02`) and
> communication (`s14-03`) are done; this is human intervention. Tool
> privilege remains, and this article's own closing section names it
> directly. Separately: the human-decision pairs this article says to
> persist from day one are exactly the kind of path-aware, human-graded
> material `s12-01`'s still-unresolved question (what does a golden set
> look like when the path itself isn't fixed?) would need to grade
> against. Nothing here answers that question, but this is the first
> article in either session naming a concrete source for the data it would
> require.
