---
title: "What an agent costs"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 6
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-09-16
summary: >
  "The agent is more expensive" isn't a budgetable number. Four additive
  sources are: more model calls (one per loop turn vs. one or two for a
  pipeline), growing context (the dominant one — every turn resends
  everything accumulated so far, so cost isn't linear in step count, it
  fattens with each one), reasoning tokens paid every turn instead of once,
  and exploration/retries. A worked illustrative example — an 8-turn agent
  against a 2-call pipeline — lands at roughly 48k vs. 8k tokens, about 6x,
  almost entirely in growing input tokens, not model output. Every model
  call already returns exact token usage, so a small ledger accumulating it
  per turn gives full visibility; measure per-step cost (reveals context
  growth), the p95 not the mean (the long tail of a confused agent is what
  wrecks the average budget), and the ratio against the pipeline on the
  same inputs, not the agent's absolute cost alone. Five levers, ranked by
  impact: route (biggest — don't send the pipeline's job to the agent),
  trim context, cap the tail (MAX_STEPS plus a per-run budget), match model
  and reasoning effort to the sub-task, cache what's deterministic.
keywords: [agent cost, token economics, context growth, reasoning tokens,
           usage tracking, cost ledger, p95, routing, MAX_STEPS,
           reasoning effort, caching, cost attribution]
---

# What an agent costs

*Antonio Perez* · 🔴 18 min

An agent does the same job as a pipeline — produce an estimate from a
transcript — and can cost several times more. That's not an implementation
flaw; it's the structural price of autonomy. The problem is that "the agent
is more expensive" isn't a number you can budget with. This article is
about turning that vague sentence into something measurable: where the
overcost comes from, how to instrument it precisely, and what levers you
have to control it.

The scenario, to set the stage: an estimation agent that, given a
transcript, uses three tools — `search_budgets`, `calculate_estimate`,
`validate_estimate` — iterating in a loop until it produces the result. Its
alternative is a fixed pipeline doing the same steps in a predetermined
order. We'll compare the cost of both over the same work.

## 1. Where the overcost comes from

An agent's overcost doesn't come from one place — it comes from four that
add up.

**More model calls.** A pipeline makes one or two calls per estimate. An
agent makes one per loop turn: decide, run tools, observe, decide again.
Eight turns are eight calls where the pipeline made two. Right there you
have a factor of four in call count.

**Context grows every turn, and it's the dominant factor.** This is the one
people underestimate. On every iteration, the agent resends the model
everything accumulated so far: the transcript, the prior decisions, and
every tool's observation. The eighth call doesn't cost what the first one
did — it costs what the first one did, plus seven rounds of dragged-along
observations. Input tokens, which are what's billed on volume the most,
grow turn by turn. A loop that accumulates has a cost that isn't linear in
the number of steps — it fattens with each one.

**Reasoning tokens.** Reasoning models spend internal tokens deliberating
before answering, and those tokens are billed. In an agent, the model
reasons every turn — which tool to use, how to interpret the observation —
so that cost is paid repeatedly, not once.

**Exploration and retries.** An agent can take suboptimal paths: reformulate
a search that came back thin, back out of a line that led nowhere. Every
one of those steps is correct — it's the agent adapting — but it costs
tokens a pipeline, by not exploring, doesn't spend.

None of these four is entirely avoidable; they're the counterpart of
flexibility. But all of them are measurable, and the second — context
growth — is where most of the money is, and therefore the biggest lever.

## 2. The 5x math

Let's put approximate numbers on a complex transcript, to see where the
multiplier comes from.

The pipeline makes two calls — reformulate and generate — with a bounded,
stable context. Call that one unit of baseline cost.

The agent, over the same four-component transcript, takes on the order of
six to eight turns. The first calls are cheap, but the last ones drag along
every prior observation: four budget searches, their results, maybe a
reformulation, the calculation, the validation. Add each turn's reasoning
tokens on top, and the total billed tokens ends up several times the
pipeline's. Five times is a realistic figure for a complex case; it can be
less on simple ones and more when the agent gets tangled.

With round, illustrative numbers you can see the mechanism. Suppose the
pipeline consumes about 8,000 tokens total across its two calls. The agent
takes eight turns, and its input tokens aren't constant: the first call
sends only the transcript, say 2,000 tokens; but every turn adds the prior
observation, so the input climbs to 4,000, then 6,000, and by the eighth
round it's around 9,000, because it's dragging along everything seen so
far. Averaging out, that's on the order of 40,000 input tokens, plus about
8,000 of output and reasoning combined: close to 48,000 tokens against the
pipeline's 8,000. There's your factor, and notice that almost all the
overcost is in those input tokens growing turn by turn, not in the model's
responses. The exact number will vary, but the shape — input that fattens
with every step — is always the same.

> *(Figure in the original: `S12-fig-06a-coste-por-vuelta.jpg` — image not
> included in this repo. A bar chart, "input tokens per agent call, turn by
> turn (illustrative figures)": 8 bars rising 2.0k, 3.2k, 4.3k, 5.2k, 6.0k,
> 7.0k, 8.0k, 9.0k across turns 1-8, a dashed purple trend line labeled
> "every turn resends what's accumulated: the input fattens," and a dashed
> green reference line at ~4k labeled "pipeline (~4k per call, two calls)."
> Callout: Agent ~48k tokens vs. Pipeline ~8k tokens = ~6x. Almost all the
> overcost is in the input that grows, not in the model's responses.)*

To translate that into money, a mental rule of thumb circulating in the
ecosystem puts the order of ten cents of a dollar per task around thirty to
fifty thousand tokens. An agent that reasons and searches several times eats
through that budget easily. And at scale, the difference stops being
anecdotal: an operation processing a million tasks a month while spending
five times the necessary tokens burns on the order of an extra million and
a half dollars a year. The multiplier that looks negligible on one loose
task becomes, multiplied by volume, a budget line.

The conclusion isn't "agents are expensive, full stop." It's that the
multiplier depends entirely on the case, and that without measuring it
you're budgeting blind. So let's measure it.

## 3. How to measure it

The good news is that an agent's cost is one of the most measurable problems
you'll face: every model call returns you exactly how many tokens it
consumed. All you have to do is capture and accumulate it.

Every API response carries a `usage` field with input tokens, output
tokens, and, in reasoning models, the breakdown of how many of those output
tokens were reasoning. A small ledger accumulating this across the loop
gives you full visibility:

```python
from dataclasses import dataclass

@dataclass
class Pricing:
    input_per_1k: float
    output_per_1k: float  # reasoning tokens are billed as output tokens

@dataclass
class CostLedger:
    steps: int = 0
    input_tokens: int = 0
    output_tokens: int = 0
    reasoning_tokens: int = 0  # tracked for visibility, already inside output_tokens

    def add(self, usage) -> None:
        self.steps += 1
        self.input_tokens += usage.input_tokens
        self.output_tokens += usage.output_tokens
        self.reasoning_tokens += usage.output_tokens_details.reasoning_tokens

    def cost(self, pricing: Pricing) -> float:
        return (
            self.input_tokens / 1000 * pricing.input_per_1k
            + self.output_tokens / 1000 * pricing.output_per_1k
        )
```

Wiring it into the loop is one line per turn: after every model call,
`ledger.add(response.usage)`. When it finishes, you have that run's cost,
how many steps it took, and how much of its spend was pure reasoning.

One detail worth not getting wrong: reasoning tokens are billed as output
tokens and are already counted inside `output_tokens`. Don't add them
separately — you'd double the cost. You track them separately only to see
what fraction of your spend is deliberation; sometimes you discover half
the bill is the model thinking, and that's an actionable signal.

Three things worth measuring beyond the per-run total.

**Cost per step**, because it reveals context growth: you'll see input
tokens climb turn by turn, and that tells you how much dragging the state
along is costing you. If the last call costs five times the first, you now
know where your money is.

**The distribution, not the average.** Agents have a long tail: most runs
are reasonable, but a confused agent that iterates to the step limit is
your worst case, and it's the one that wrecks the average budget. Measure
the 95th percentile, not just the mean, because that's where the surprises
live.

**The comparison against the pipeline** on the same inputs. The number that
matters isn't the agent's absolute cost, but its overcost against the
cheapest alternative that would solve the case. That ratio is what
justifies the autonomy, or doesn't.

And one attribution that more than pays for the effort of instrumenting:
which tool is inflating the context. If you log the size of the observation
each tool returns, you'll quickly discover whether the growing cost comes
from `search_budgets` returning huge payloads that then get dragged along
turn after turn. Attributing the spend to its source — this tool, this kind
of observation, this stretch of the loop — is what turns "the agent is
expensive" into "60% of the cost is unslimmed search results being dragged
along," which is already a problem with a solution.

## 4. How to control it

With the cost measured, controlling it is a matter of known levers. Ranked
by impact for this case:

**Route.** The biggest lever isn't inside the agent — it's before it. Don't
send the agent what a pipeline would solve. A cheap classification up front
that sends simple transcripts to the fixed pipeline and only complex ones to
the agent keeps the average cost low, because you pay for autonomy only
when the problem demands it. Most of your inputs probably don't.

**Trim the context.** Since context growth dominates per-run cost, cutting
it is the biggest lever inside the loop. Summarize old observations that no
longer inform the decision, drop the irrelevant ones, and keep identifiers
instead of full payloads. Having `search_budgets` return five clean
references instead of two hundred raw rows doesn't just improve the model's
decisions — it reduces what you resend every turn from then on.

**Cap the tail.** A step limit — the classic `MAX_STEPS` — puts a ceiling on
the worst case. Complement it with a per-run budget: if an estimate crosses
a token or cost threshold, cut it off and treat it as a case for review
instead of letting it run. The long tail is where the money is lost, and
this amputates it.

**Match model and reasoning effort to the work.** Not every decision needs
maximum reasoning effort or the most expensive model. Reserve the power for
the orchestration — where the agent decides — and consider a cheaper model
or lower reasoning effort for bounded sub-tasks. Reasoning level is a
direct cost dial.

**Cache what's deterministic.** If the agent repeats equivalent searches
across runs, a result cache avoids paying twice for the same thing. It
doesn't apply everywhere, but where it does, it's free money.

> *(Figure in the original: `S12-fig-06b-palancas-de-coste.jpg` — image not
> included in this repo. Five ranked cards, impact bars from high to low:
> 1) Route — "simple to the pipeline, complex to the agent: pay for
> autonomy only when it's needed" [high]; 2) Trim the context — "summarize
> old observations; store identifiers, not full payloads" [high]; 3) Cap
> the tail — "MAX_STEPS + a per-run budget: amputate the worst case"
> [medium-high]; 4) Match model and reasoning — "reserve power for the
> orchestration; less effort on bounded sub-tasks" [medium]; 5) Cache the
> deterministic — "avoid paying twice for the same search across runs"
> [situational]. Callout: none of these is exotic — routing, state
> management, limits, resource selection and caching, the usual repertoire
> for operating any expensive process.)*

None of these levers is exotic. They're routing, state management, limits,
resource selection, and caching — the usual repertoire for operating any
expensive process.

## 5. Closing: cost is a design decision, not a mystery

An agent's cost isn't an obscure AI problem. It's measurable down to the
token, attributable per step, and controllable with ordinary engineering:
you instrument, you budget, you route, you cache. The only unusual part is
that the billing unit is tokens over a non-deterministic loop; the
discipline for managing it is the same you'd apply to any expensive
operation.

And that's where the right decision framework comes from. An agent isn't
"better" than a pipeline — it's a different trade between cost and
capability. The question isn't whether the agent works — it almost always
does — but whether the value it delivers in your case justifies the
multiplier you just measured. If an estimate is worth enough, five times a
pipeline's cost is a bargain. If it isn't, the agent is the wrong tool and
the pipeline was serving you better. Measuring the cost isn't an accounting
exercise — it's what lets you make that decision with data instead of
faith.

## Sources

- Anthropic, *Building Effective Agents* — agents trade latency and cost for
  better task performance, and the recommendation not to pay that bill when
  a simpler flow suffices: https://www.anthropic.com/research/building-effective-agents
- Barry Zhang (Anthropic), *How We Build Effective Agents* — the
  cost-per-task heuristic and the arithmetic at scale, synthesized in:
  https://shellypalmer.com/2026/04/how-anthropic-thinks-about-agents-workflows-and-tasks/
- OpenAI, *Usage and costs* — the `usage` field with input, output and
  reasoning tokens per call: https://platform.openai.com/docs/guides/production-best-practices

---

> *(Editor's note — the section's own numbers, not the code: "5x" heading,
> 6x math.)* §2 is titled "The 5x math" and states "five times is a
> realistic figure," then works through an example that sums to ~48,000
> tokens against the pipeline's ~8,000 — a factor of **6**, not 5, exactly
> as the section's own paired figure captions it ("~6x"). This isn't a
> rounding quibble: the heading's number and the worked arithmetic
> immediately beneath it disagree, and the figure sides with the
> arithmetic. Left as the author wrote it, per this handbook's convention
> of flagging rather than silently correcting — "5x" reads as the rule-of-
> thumb figure quoted before the worked numbers commit to a specific
> example, and "several times, commonly in the 5-6x range for a complex
> case" is the more defensible way to carry this forward than either number
> alone.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — a real, close precedent, not an
> absence.)* Unlike s12-01 through s12-05, this article's core proposal
> already has a working ancestor in the codebase:
> `app/foundation/llm/wrapper.py` defines `MODEL_COSTS` (USD per 1M tokens,
> covering `gpt-5`, `gpt-5-mini`, `claude-haiku-4-5`, `claude-sonnet-4-5`
> among others) and `_estimate_cost(model, tokens_in, tokens_out)`, and
> every `LLMWrapper` call already returns a `cost_usd`. What's missing is
> exactly this article's contribution: accumulation *across a loop's turns*
> — the real code prices one call at a time, with nothing corresponding to
> `CostLedger` summing several, because no multi-turn loop exists yet to
> sum over — and separate visibility into reasoning-token share, which
> `MODEL_COSTS`'s flat input/output split doesn't distinguish. Two smaller
> mismatches worth knowing before wiring this in: the real code's pricing
> table is per-1M tokens, this article's `Pricing` is per-1k (a factor-of-
> 1000 bug if copied without converting), and the real usage object uses
> Chat-Completions-style field names (`prompt_tokens`, via LiteLLM) where
> this article's ledger assumes Responses-API-style `input_tokens` /
> `output_tokens_details.reasoning_tokens` — the same vocabulary split
> s12-03's editor's note already flagged between the two APIs, showing up
> again here on the billing side.

> *(Editor's note — relationship to s12-01 and this session so far.)* The
> $0.10/task ≈ 30-50k tokens rule of thumb and the "$1.5M/year at a million
> requests" example are not new here — s12-01 already cited them in passing
> while arguing *whether* to build an agent at all. This article is where
> those numbers get derived rather than quoted: §§1-2 show the mechanism
> producing them (growing input tokens, not model output), and §3-4 turn
> "measure it" and "control it" from s12-01's closing advice into concrete,
> buildable steps. `PLAYBOOK.md`'s Axis 5 already cites s12-01 for the
> headline figures; nothing here changes those figures, only where they
> come from.
