---
title: "Competition and synthesis between agents"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-09-19
summary: >
  A single estimator returns a number with no measure of its own fragility
  — it looks identical whether it nailed a trivial case or improvised
  confidently on an impossible one, and asking the model to self-score
  confidence inherits the model's own bias toward trusting what it just
  said. Two agents with deliberately opposed criteria attacking the same
  problem is a more honest way to get that measure: a conservative
  estimator assuming friction, an aggressive one assuming the best
  reasonable case, run in parallel via plain multi-edge fan-out (fixed
  width, not Send's dynamic kind), fanned in by the same reducer pattern
  already established. Divergence between their proposals is computed
  arithmetic, never an LLM call — paying tokens for a subtraction is pure
  waste, and it's the signal itself: convergence means the outcome doesn't
  depend on which assumptions hold, so the system can close on its own;
  divergence means the result depends entirely on unresolved assumptions,
  which is a judgment call for a person, not a calculation. The
  synthesizer's job is explicitly not to average — the assumptions driving
  the disagreement and the open questions that would resolve it are the
  valuable output, not a false-precision midpoint. Competition is fraud
  when the two competitors aren't genuinely different (same model, same
  context, prompts differing by one adjective correlate far more than
  expected, producing false confidence from fake diversity) or when
  there's nothing to disagree about (extracting a stated requirement has a
  right answer; if you couldn't defend both positions yourself, your
  agents can't either). Real divergence needs substantively different
  criteria, different evidence, ideally different models. Stay at two
  competitors — a third usually lands in the middle and adds cost without
  signal. Repeated sampling of one prompt is a cheaper alternative that
  measures the model's own noise, not the domain's actual uncertainty —
  a different, less useful thing. The decision is triple cost for real
  triple value, or nothing: worth it only when what's at stake justifies
  the multiplier.
keywords: [competition topology, synthesizer, divergence, fan-out, fan-in,
           operator.add, correlation trap, repeated sampling, EstimateProposal,
           conservative estimator, aggressive estimator, open questions]
---

# Competition and synthesis between agents

*Antonio Perez* · 🔴 18 min

Your system returns 260 hours.

How much do you trust that number? You have no way to know. It's the
output of an agent that did its job as best it could, and it could come
from a trivial case the system nailed or an impossible one the model
confidently improvised on. The figure looks exactly the same either way.
That's a single estimator's underlying weakness: it produces no measure of
its own fragility.

You can ask the model to score its own confidence, and it's better than
nothing. But a model evaluating its own answer has the bias you'd expect:
it tends to trust what it just said, and produces numbers that sound
reasonable without correlating much with whether it was right.

There's another way to get that measure, and it's more honest: have two
agents with opposite criteria attack the same problem and look at how far
apart they land.

## 1. From cooperating to competing

Until now all your agents cooperate: each contributes a distinct piece and
the result is the composition. The extractor doesn't compete with the
searcher; they need each other.

Competition is something else. Two or more agents attack the same task,
with deliberately different criteria, and a third decides what to do with
the disagreement.

In software estimation the example is almost embarrassingly natural,
because it's exactly what happens in any real estimation meeting. There's
the one who says two weeks and the one who says two months, and both are
right under their own assumptions. The disagreement isn't a process
failure: the disagreement *is* the process.

Two estimators, then:

- **`conservative_estimator`**: assumes friction. Integrations that go
  sideways, requirements that grow, a staging environment nobody's set up.
- **`aggressive_estimator`**: assumes the best reasonable case. A
  competent team, stable scope, no surprises.

And a **`synthesizer`** that receives both proposals.

## 2. Implementing it

### Proposals have a schema

The first thing to fix is what an estimator produces. If it returns a
bare number, the synthesizer can't do anything intelligent with the
disagreement — it can only average, which is the worst option. An
estimator has to return its number and the assumptions holding it up.

```python
from pydantic import BaseModel, Field

class EstimateProposal(BaseModel):
    stance: Literal["conservative", "aggressive"]
    hours: float = Field(gt=0)
    assumptions: list[str] = Field(description="What this estimate takes for granted.")
    risks: list[str] = Field(description="What would make this estimate wrong.")
    reasoning: str
```

The `assumptions` are the payload. When the conservative one says 340 hours
because it assumes the legacy ERP integration is undocumented, and the
aggressive one says 190 because it assumes the client provides
documentation and a test environment, the difference between them isn't a
number — it's a concrete question someone can go resolve. That's the
valuable output.

### The two estimators run in parallel

They're independent, so there's no reason to pay two latencies in series.
LangGraph runs in parallel the nodes coming out of the same point, and the
state gathers them:

```python
class EstimationState(TypedDict):
    requirements: list[str]
    budget_matches: list[dict]
    proposals: Annotated[list[dict], operator.add]   # <- the reducer does the fan-in
    estimate: dict | None
    confidence: float | None

builder.add_edge("requirements_extractor", "conservative_estimator")
builder.add_edge("requirements_extractor", "aggressive_estimator")

builder.add_edge("conservative_estimator", "synthesizer")
builder.add_edge("aggressive_estimator", "synthesizer")
```

The fan-in is free, and it's free because of the reducer. `proposals` is
annotated with `operator.add`, so the two concurrent writes concatenate
instead of clobbering each other. If that field were unannotated, one of
the two estimators would vanish silently and you'd have a competition
system with a single competitor. It's the quietest bug in this entire
article.

```python
CONSERVATIVE_INSTRUCTIONS = """You estimate software projects assuming friction.
Undocumented integrations, scope creep, environments that are not ready,
requirements that turn out to hide complexity. Be explicit about every
assumption you make. You are not pessimistic for its own sake: you are the
estimate that survives contact with reality.
"""

async def conservative_estimator(state: EstimationState) -> dict:
    with logfire.span("agent.conservative_estimator"):
        response = await client.responses.parse(
            model="gpt-5",
            input=[
                {"role": "system", "content": CONSERVATIVE_INSTRUCTIONS},
                {"role": "user", "content": render_estimation_brief(state)},
            ],
            text_format=EstimateProposal,
        )
        return {"proposals": [response.output_parsed.model_dump()]}
```

### Divergence is computed, not opined on

And here's the design decision that makes all of this worth doing.

The temptation is to pass both proposals to the synthesizer and ask it to
"assess how much they differ." Don't. The divergence between two numbers
is an arithmetic operation. Asking an LLM for it is paying tokens for a
subtraction, and on top of that accepting that it'll sometimes get it
wrong.

```python
def compute_divergence(proposals: list[dict]) -> float:
    """Relative spread between proposals. Deterministic, cheap, testable."""
    hours = [p["hours"] for p in proposals]
    lo, hi = min(hours), max(hours)
    return (hi - lo) / hi
```

One line of arithmetic. Zero tokens. And with that, before ever calling
the synthesizer, you already have the signal you were missing: 190 against
340 gives a divergence of 0.44; 250 against 270 gives 0.07.

Notice what each case means, because it isn't symmetric.

**They converge.** Two deliberately opposed criteria landed almost in the
same place. That's strong information: the outcome doesn't depend on the
assumptions. It doesn't matter whether the team is senior or the
integration goes sideways — the project costs what it costs. The system
can close on its own with high confidence.

**They diverge.** The outcome depends entirely on which assumptions hold.
It isn't that the system calculated wrong — it's that the question has no
computable answer without first deciding whether the client is going to
deliver the ERP documentation. That's a judgment call, and a person makes
it.

In other words: competition doesn't just give you a better estimate. It
gives you the criterion for knowing when you shouldn't be estimating
alone. Divergence feeds confidence directly, and confidence feeds the
decision to stop.

### The synthesizer doesn't average

```python
SYNTHESIZER_INSTRUCTIONS = """You receive two estimates for the same project,
produced under opposite assumptions, plus a computed divergence score.

Do not average them. Your job is to:
- Identify which assumptions actually drive the difference.
- Produce a final estimate with an explicit range, not a single point.
- State what would have to be true to move the estimate toward either end.
- Report your confidence, taking the divergence score into account.
"""

class SynthesizedEstimate(BaseModel):
    hours: float
    range_low: float
    range_high: float
    confidence: float = Field(ge=0.0, le=1.0)
    driving_assumptions: list[str]
    open_questions: list[str]

async def synthesizer(state: EstimationState) -> dict:
    divergence = compute_divergence(state["proposals"])

    response = await client.responses.parse(
        model="gpt-5",
        input=[
            {"role": "system", "content": SYNTHESIZER_INSTRUCTIONS},
            {"role": "user", "content": render_proposals(state["proposals"], divergence)},
        ],
        text_format=SynthesizedEstimate,
    )
    result = response.output_parsed

    return {
        "estimate": result.model_dump(),
        "confidence": result.confidence,
    }
```

The explicit instruction not to average isn't paranoia. It's the default
behavior a model tends toward when you give it two numbers and ask for
one, and averaging is precisely what destroys the value of having paid for
two estimates. The average of 190 and 340 is 265, a number nobody can
defend and which also hides that the range exists. Your client doesn't
need a falsely precise point — they need a range and the two or three
questions whose answers would narrow it.

And `open_questions` is, to me, the single most useful field this whole
system produces. It's the list of what needs to go get asked of the
client. A single estimator never produces it, because it doesn't know what
the number was riding on.

## 3. When this is a fraud

Three ways to throw money away, ordered by frequency.

### The correlation trap

Take two agents, give them the same model, the same context, and prompts
differing by one adjective ("estimate conservatively" / "estimate
aggressively"), and watch what happens: their outputs look a lot alike.
You'll have paid for three calls to get the illusion of a second opinion.

The reason is obvious once said out loud: two samples from the same model
over the same context are enormously correlated. The adjective moves the
number 10% and that's it. And the worst part isn't the cost — it's that
the resulting low divergence is a false confidence signal. The system will
tell you the case is predictable when what actually happened is that you
never created any real diversity.

For competition to be worth anything, the competitors have to genuinely
differ:

- **Substantively different criteria in the prompts, not adjectives.** Give
  the conservative one instructions about undocumented integrations and
  technical debt; give the aggressive one instructions about reuse and
  teams with context.
- **Different evidence.** This is the one with the most impact and the
  least use: give the conservative one the historical budgets that ran
  over schedule, and the aggressive one the ones that came in clean. Now
  they're not arguing about style — they're looking at different worlds.
- **Ideally, different models.** Two providers have different biases, and
  their disagreement is more informative than a model's disagreement with
  itself.

If you can't get any of the three, competition won't give you signal.
Don't build it.

### Competing where there's nothing to judge

Competing on requirements extraction is throwing money away: there's a
reasonably correct answer and two agents will converge on it. Competition
only pays off when there's a judgment call involved — that is, when two
reasonable experts could legitimately disagree. Estimating hours qualifies.
Extracting that the client asked for Google login from a transcript
doesn't.

Rule: if you couldn't defend both positions yourself, your agents can't
either.

### Scaling to N competitors

It's tempting to think if two is good, five is better. It isn't: returns
decay fast and cost is linear. With two genuinely opposed criteria you
already have the signal you were after — the uncertainty band. A third
usually lands in the middle and tells you nothing you didn't know. Stay at
two, unless you can name a third criterion that's genuinely orthogonal to
the other two.

### And a cheaper alternative worth knowing

Before setting up three agents, know that a simpler technique exists:
sample the same prompt several times and look at the spread. It's cheaper
to build, doesn't require designing opposed criteria, and also gives you a
measure of stability.

What it doesn't give you is the good part: it produces no opposing
assumptions, no open questions, and no explanation of why the numbers
differ. Repeated sampling measures the model's own noise. Competition
measures the domain's uncertainty. They're different things, and only the
second is actionable in front of a client.

## 4. The cost, plainly

One estimator: one call. Competition: two calls in parallel plus one
synthesis — that is, three calls, and latency equal to the slower of the
two. In cost, ×3. In latency, roughly ×2.

Is it worth it? If your system's output is a budget sent to a client that
commits the company for months, tripling one inference's cost to get a
defensible range, a list of assumptions, and a real measure of uncertainty
is, by a wide margin, the best money you'll spend in the whole system.

If the output is a rough estimate for prioritizing an internal backlog, no.
Use one estimator and move on.

That's the entire decision, and it depends on what being wrong costs.

## A loose end

We're at five agents, six with the synthesizer, and we've been adding them
fairly freely. Each has its prompt, its schema, and its tools.

Its tools. There's something there we haven't looked at.

`budget_searcher` queries the budgets database. Only queries? The day
someone gives it a tool that writes — to save the estimate, to mark a
budget as used, whatever — there will be an agent governed by a model with
write permission over company data, and nobody will have made that
decision explicitly: it'll just have happened.

The question that remains is the most boring and the most important one:
exactly what can each agent touch, and what happens when it tries to touch
what it shouldn't.

---

> *(Editor's note — a confirmed, precise arithmetic mismatch between the
> code and both figures, not a rounding difference.)* `compute_divergence`
> as shown returns `(hi - lo) / hi`, and the prose's own worked numbers
> match that formula exactly: `(340-190)/340 = 0.441…` → "0.44", and
> `(270-250)/270 = 0.074…` → "0.07". **Neither figure agrees with this.**
> Both `fig-01` (the fan-out diagram) and `fig-02` (the convergence
> panels) show `divergence = 0.79` for the 190/340 pair and
> `divergence = 0.08` for the 250/270 pair — which are exactly
> `(hi - lo) / lo` instead: `(340-190)/190 = 0.789…` → "0.79",
> `(270-250)/250 = 0.08` → "0.08", both confirmed by direct computation.
> The code and the prose agree with each other; the figures were
> generated from a different formula (normalized against the *low* value,
> not the *high* one) than the function actually shown. Both are
> legitimate ways to define relative spread — this isn't a claim that one
> formula is wrong — but the two don't produce the same numbers, and a
> reader implementing `compute_divergence` verbatim will get numbers that
> don't match either figure. Pick one deliberately; don't assume the code
> and the diagrams already agree.

> *(Editor's note — a simpler, correctly-scoped alternative to `s13-04`'s
> mechanism, not a variant of it.)* The parallel fan-out here uses plain
> `add_edge` calls (two static edges from one source), not `s13-04`'s
> `Send` API. That's the right call, not an inconsistency: `Send` exists
> for *data-dependent* fan-out width (one branch per component, and the
> component count isn't known until the transcript is read); here the
> width is always exactly two, fixed at graph-definition time, so static
> edges are simpler and sufficient. Reaching for `Send` when the branch
> count is already known would be unnecessary machinery — worth naming
> explicitly so a reader doesn't assume every parallel fan-out needs the
> dynamic mechanism.

> *(Editor's note — this is `s14-01`'s "compete" topology, fully built,
> not a new proposal — and its own near-miss warning still applies
> verbatim.)* `s14-01` §2 introduced `conservative_estimator` /
> `aggressive_estimator` / `synthesizer` as the worked example for the
> competition topology, and flagged there that "conservative"/"aggressive"
> already appear in the codebase's prompts, but only as single-word tuning
> adjectives inside one prompt — not two separate competing agents. That
> finding is unchanged here: still no `EstimateProposal`,
> `compute_divergence`, or `SynthesizedEstimate` anywhere in
> `lidr/ai-engineering` (branch state at session 11 completed). This
> article's own §3 "correlation trap" warning is, pointedly, a more
> detailed version of the exact mistake a naive reading of those existing
> prompt adjectives would walk into if someone tried to stand up
> "competition" by just writing two prompts that differ by a word.

> *(Editor's note — directly backs a guidance point `PLAYBOOK.md` already
> carries from `s14-01`, now with the concrete mechanics `s14-01` didn't
> have yet.)* `PLAYBOOK.md`'s Axis 5 cites `s14-01` for the
> cooperate-vs-compete decision and already warns against near-identical
> prompts producing false second opinions. This article supplies what was
> missing: the divergence computation is deterministic code, not a model
> judgment call, and the fan-in mechanism is plain multi-edge routing, not
> `Send`. `PLAYBOOK.md` updated to cite this directly for both.
