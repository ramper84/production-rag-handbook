---
title: "The agentic loop, step by step"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-09-16
summary: >
  A working estimation agent fits in about fifty lines, built by hand with
  no orchestration framework — not because frameworks are bad, but because
  assembling the loop raw is the only way to know what they're automating
  for you. Four pieces: tools (wrapping capabilities the AI service already
  has), a tool registry decoupling "what tools exist" from "how the loop
  works," the loop itself (call the model, run whatever tools it requests
  in parallel via asyncio.gather, return the results keyed by call_id,
  repeat under a MAX_STEPS guard implemented as a for/else), and a
  Pydantic-typed final output — the path through the loop varies run to
  run, the shape of what it returns never does. A traced run shows the
  agent recovering from a weak observation by reformulating before
  computing on bad data, which a fixed pipeline structurally cannot do.
  Building it by hand is what turns adopting a framework later into an
  informed decision instead of an act of faith.
keywords: [agentic loop, tool registry, function calling, Responses API,
           structured outputs, MAX_STEPS, for-else, asyncio.gather, trace,
           observability, FastAPI, handover, from-scratch agent]
---

# The agentic loop, step by step

*Antonio Perez* · 🔴 18 min

Let's build the agent by hand. No LangChain, no orchestration framework of
any kind — not because frameworks are bad, they have their place, but
because assembling the loop raw is the only way to understand what they do
for you when you use them, and to be able to take control when you need to.
A working estimation agent fits in about fifty lines. Let's assemble them,
piece by piece, on the case of estimating software projects from meeting
transcripts.

By the end you'll have an agent that reads a complex transcript, breaks the
project down into components, searches historical budgets for each one,
computes partial estimates, validates the result, and consolidates it. And
you'll know exactly why it works, because you wrote it.

## 1. The pieces

Four elements, not one more.

**The tools** are the agent's executable capabilities. We have three, and
each wraps something the AI service already knows how to do:
`search_budgets` retrieves comparable historical budgets from the vector
database, `calculate_estimate` computes costs deterministically from
references, and `validate_estimate` runs quality checks on a candidate
estimate. The agent reimplements nothing; it only orchestrates.

**The model** is the orchestrator. We use `gpt-5` with medium reasoning
effort: it's the one reading the situation and deciding which tool to use
at each point.

**The loop** is the skeleton: call the model, execute whichever tools it
requests, return the results, repeat, until it produces a final answer or a
step limit runs out.

**The state** is the context accumulating turn by turn: every decision and
every observation get appended, and that accumulation is also the trace
you'll be able to inspect afterward.

## 2. The tools and their dispatch

Each tool is declared with a schema the model reads to decide when and how
to use it. In OpenAI's Responses API, `search_budgets` looks like this (the
other two follow the same shape):

```python
TOOLS = [
    {
        "type": "function",
        "name": "search_budgets",
        "description": (
            "Search historical budgets for one software component. Call once per "
            "component; keep unrelated components in separate calls."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "One component to price.",
                },
                "component_type": {
                    "type": "string",
                    "enum": ["integration", "migration", "frontend", "backend", "mobile"],
                },
            },
            "required": ["query", "component_type"],
            "additionalProperties": False,
        },
        "strict": True,
    },
    # calculate_estimate and validate_estimate follow the same structure
]
```

Declaring the schema is only half of it. The other half is wiring each tool
name to the function that executes it, and a simple pattern earns its keep
here: a registry mapping name to function.

```python
from typing import Any, Awaitable, Callable

ToolFn = Callable[[dict[str, Any]], Awaitable[dict[str, Any]]]

TOOL_REGISTRY: dict[str, ToolFn] = {
    "search_budgets": search_budgets,
    "calculate_estimate": calculate_estimate,
    "validate_estimate": validate_estimate,
}

async def execute_tool(name: str, args: dict[str, Any]) -> dict[str, Any]:
    fn = TOOL_REGISTRY.get(name)
    if fn is None:
        return {"error": f"unknown tool: {name}"}
    try:
        return await fn(args)
    except Exception as exc:
        # a failing tool must not crash the loop
        return {"error": str(exc)}
```

This registry decouples two things that shouldn't know about each other:
which tools exist, and how the loop works. Adding a new tool means adding a
schema to `TOOLS` and an entry to the registry; the loop doesn't change a
single line. And notice the `try`/`except`: if a tool fails, it doesn't
crash the agent. It returns an error as an observation, and that's an
important design decision — the model can read that error, reason about it,
and reformulate — not a defensive afterthought.

## 3. The loop

Here's the agent's heart. It's shorter than its reputation suggests.

```python
import asyncio
import json
from openai import AsyncOpenAI

client = AsyncOpenAI()
MAX_STEPS = 8

SYSTEM_PROMPT = (
    "You are a software estimation agent. Given a meeting transcript, identify "
    "the components to estimate, search historical budgets for each one "
    "separately, compute partial estimates, and consolidate them. Always run "
    "validate_estimate before producing the final answer."
)

async def run_agent(transcript: str) -> AgentResult:
    trace: list[Step] = []
    response = await client.responses.parse(
        model="gpt-5",
        reasoning={"effort": "medium"},
        instructions=SYSTEM_PROMPT,
        input=[{"role": "user", "content": transcript}],
        tools=TOOLS,
        text_format=Estimate,
    )

    for _ in range(MAX_STEPS):
        calls = [item for item in response.output if item.type == "function_call"]
        if not calls:
            break  # no tool calls: the model has its final answer

        results = await asyncio.gather(
            *(execute_tool(call.name, json.loads(call.arguments)) for call in calls)
        )

        tool_outputs = []
        for call, result in zip(calls, results):
            trace.append(Step(action=call.name, args=call.arguments, observation=result))
            tool_outputs.append({
                "type": "function_call_output",
                "call_id": call.call_id,
                "output": json.dumps(result),
            })

        response = await client.responses.parse(
            model="gpt-5",
            previous_response_id=response.id,
            instructions=SYSTEM_PROMPT,
            input=tool_outputs,
            tools=TOOLS,
            text_format=Estimate,
        )
    else:
        return AgentResult(status="max_steps_exceeded", trace=trace)

    return AgentResult(status="done", estimate=response.output_parsed, trace=trace)
```

Let's walk through the decisions, because each one matters.

The first call starts outside the loop, with the transcript as input. From
there, every iteration begins by looking at the model's output: we collect
every `function_call`-type item. If there are none, the model has stopped
requesting tools and produced its final answer; we exit with `break`.

If there are calls, we execute all of them in parallel with
`asyncio.gather`. A single turn can bring several — four components, four
`search_budgets` calls — and running them concurrently instead of serially
cuts latency noticeably. Assuming there's always exactly one is a classic
bug that shows up on the first complex transcript.

We return all the observations together, each paired with its `call_id`, in
a single continuation call. `call_id` is what pairs your result with the
right request; forgetting it is the most common beginner mistake. State
chains via `previous_response_id`, so the model keeps context without you
resending everything by hand. We do resend `instructions`, though, because
`previous_response_id` doesn't carry the system prompt along.

Meanwhile, every action and its observation get saved to the trace. That
list is your only window into what the agent decided and why; without it,
debugging an agent that "sometimes estimates wrong" is impossible.

And the whole loop lives inside a guard: `range(MAX_STEPS)`. The `for`
loop's `else` — which runs only if the loop finishes without hitting
`break` — captures the case where the agent keeps going without converging,
and cuts it off with an explicit error status. An agent loop with no step
limit is a bill waiting to happen.

## 4. The agent in motion

The theory clicks into place watching a run. Faced with a transcript
mentioning an ERP integration and a legacy data migration, the trace the
agent accumulates reads like this:

```
step 1  search_budgets(query="ERP integration via REST API", component_type="integration")
        -> 4 matches, median 120h
step 2  search_budgets(query="legacy data migration, undocumented schema", component_type="migration")
        -> 1 weak match, low confidence
step 3  search_budgets(query="data migration effort, mid-size dataset", component_type="migration")
        -> 3 matches, median 90h
step 4  calculate_estimate(components=[integration, migration])
        -> total 410h across 2 components
step 5  validate_estimate(estimate=...)
        -> ok, no issues found
```

The interesting part is step 3. In step 2 the migration search came back
with a single weak match, and the agent didn't compute on that thin data:
it read the observation, reformulated the query with different terms, and
searched again before moving on. That's exactly what a fixed pipeline
cannot do, and it's what justifies the loop. You didn't program that — it
emerged from the model being able to see the result of its own action and
decide the next step accordingly. That adaptive behavior, visible step by
step in the trace, is the agent earning its place.

> *(Figure in the original: `S12-fig-04a-ejecucion-bucle.jpg` — image not
> included in this repo. It diagrams the five-step trace above as a
> numbered vertical sequence inside "servicio IA — run_agent()", each step
> paired with its observation box [steps 1, 4 and 5 green/"ok"; step 2
> orange/"low confidence"; step 3 green, captioned "the agent recovers: it
> reads a poor observation and reformulates before calculating" — a fixed
> pipeline cannot do this, the path is built at run time], ending in a
> purple "return Estimate (fixed shape)" box.)*

## 5. The structured final output

When the model stops requesting tools, it produces the final estimate. And
here we don't want free text — we want a deterministic structure the rest
of the system can consume without surprises. That's why every call passes a
`text_format`: a Pydantic model the final response has to conform to.

```python
from pydantic import BaseModel

class ComponentEstimate(BaseModel):
    name: str
    hours: float
    reference_budget_ids: list[str]

class Estimate(BaseModel):
    components: list[ComponentEstimate]
    total_hours: float
    notes: str
```

When the loop exits via `break`, `response.output_parsed` is already a
validated `Estimate` object. This is the key point for the system's
stability: the agent can take a different path on every run — three
searches or five, one retry or none — but the shape of what it returns is
always the same. Non-determinism lives inside the loop; the output contract
is deterministic.

## 6. The agent behind an endpoint

The loop doesn't run in a vacuum — it lives in the AI service, behind an
endpoint the rest of the system calls. In FastAPI that's a few lines:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class EstimateRequest(BaseModel):
    transcript: str

@app.post("/estimate")
async def estimate(req: EstimateRequest):
    result = await run_agent(req.transcript)
    return {
        "status": result.status,
        "estimate": result.estimate,
        "trace": [step.__dict__ for step in result.trace],
    }
```

From the business backend, calling it is an ordinary HTTP call. In the
reference Ruby on Rails implementation it looks like this, though the
pattern is stack-independent and any HTTP client works the same way:

```ruby
# business backend: calling the servicio IA (any HTTP client works)
response = HTTP.post("#{AI_SERVICE_URL}/estimate", json: { transcript: transcript })
result = JSON.parse(response.body)

case result["status"]
when "done"
  save_estimate(result["estimate"])
when "max_steps_exceeded"
  flag_for_manual_estimation(result["trace"])
end
```

Notice what the business backend doesn't see. It doesn't see the loop, the
`function_call`s, the `call_id`s, or how many turns the agent took. It sends
a transcript and gets back a status and a structured estimate. The entire
agentic mechanism is an internal detail of the AI service, and that boundary
is what lets you completely rewrite how the agent uses its tools without
touching a single line of the business backend.

## 7. What building it by hand teaches you

Assembling this loop raw leaves lessons a framework would have hidden from
you.

The first is that error handling is yours, and it's where robustness is
won or lost. A tool can fail, the model can return something unexpected, a
call can time out. Dispatch's `try`/`except` turns a tool failure into a
recoverable observation; the `MAX_STEPS` guard prevents an infinite loop;
the output status distinguishes success from exhaustion. None of these
defenses is optional in production, and all of them are visible and
adjustable because you wrote them.

The second is that observability doesn't come free. The trace you're
accumulating is your debugging instrument, and its quality determines
whether you'll be able to understand why the agent did what it did. In a
real system you'll want to enrich it: per-step timing, tokens consumed, the
model's reasoning summary. But the basic structure — action, arguments,
observation, per turn — is already here.

The third is that you now understand what a framework would do for you.
When you consider adopting one, you'll see it largely offers you exactly
this loop, plus state management, retries, instrumentation, and, in the more
elaborate ones, orchestration as an execution graph. You might want that;
you might not. The difference is that now it's an informed decision, not an
act of faith. You've seen the machine from the inside and know which parts
it's automating for you.

## 8. Closing: fifty lines, and not one of them magic

Recap what you've built: a loop, a tool registry, a system prompt, and an
output schema. That's the entire agent. Every decision point is yours — how
many steps you allow, what you do when a tool fails, what you return when it
doesn't converge — every piece can be tested in isolation, and every turn
gets recorded in the trace.

There's no hidden layer where something happens you can't explain. The
model supplies the reasoning; you supply the control flow. And that's
exactly the feeling we were after: looking at something that sounds like
intelligent autonomy and recognizing, underneath, a carefully written
`while` loop with a stopping condition. Frameworks are conveniences built on
top of this. You now understand the thing they wrap.

> *(Figure in the original: `S12-fig-04b-anatomia-codigo.jpg` — image not
> included in this repo. It diagrams the code's anatomy in four boxes
> inside "servicio IA (Python + FastAPI)": `POST /estimate` [wraps
> `run_agent()`] at the top, feeding into a `run_agent()` group of four
> boxes — `SYSTEM_PROMPT` ["the agent's role and method: identify, search,
> calculate, validate and consolidate"], `TOOLS + TOOL_REGISTRY`
> [`search_budgets`, `calculate_estimate`, `validate_estimate`], "the loop:
> reason / act / observe" ["calls gpt-5 (reasoning: medium), repeats until
> is_final or MAX_STEPS, traces every step"], and `Estimate (Pydantic)`
> ["structured output, fixed shape — the path varies, the shape doesn't;
> stable output contract"] — with the business backend shown outside,
> connected only by `POST /estimate` in and `status + estimate` out.)*

## Sources

- OpenAI, *Function calling* (Responses API) — `function_call`/
  `function_call_output` items, `call_id`, and chaining with
  `previous_response_id`:
  https://developers.openai.com/api/docs/guides/function-calling
- OpenAI, *Structured outputs* — output conforming to a strict schema via
  `text.format`/`parse`: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic, *How tool use works* — the canonical shape of the loop governed
  by a stopping condition:
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works

---

> *(Editor's note — this is s12-01/02/03 assembled, not a fourth new idea.)*
> Nothing here is conceptually new against the first three parts of this
> session: `TOOLS` and dispatch are s12-03 §§2-3 with the registry pattern
> made explicit; the loop is s12-02 §7's schema with s12-01's fixed-vs-agent
> question already answered "agent"; `MAX_STEPS`/the `for`/`else` guard is
> s12-02 §1's stopping condition, now runnable; step 3's reformulation-on-a-
> weak-observation is s12-03 §7's "errors are results too" witnessed in a
> trace instead of argued in prose. What this article adds that the first
> three didn't is the one thing prose can't give you: a version specific
> enough to run, with the seams (the registry, the `for`/`else`, the
> `text_format`) named as decisions rather than left implicit.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — a real naming collision, not a
> contradiction.)* No `run_agent`, `AgentResult`, `TOOL_REGISTRY`, or
> `client.responses.parse` call exists in the codebase — consistent with
> s12-01/02/03. But this article's `Estimate`/`ComponentEstimate` Pydantic
> pair is **not** a green field: `app/generation/rag/schemas.py:416`
> already defines a real `Estimate` (Session 9), with a genuinely different
> shape — `total_engineer_days`, `modules: list[WorkModule]` (each
> `WorkModule` holding `tasks: list[TaskItem]`), `duration_weeks`,
> `sources: list[SourceCitation]`, `assumptions`, `confidence`, `reasoning`
> — not this article's flat `components: list[ComponentEstimate]` /
> `total_hours` / `notes`. The real class's own docstring already
> disambiguates it from a *different* earlier collision ("distinct from the
> Session 4 `EstimationResult`"), so this would be a third `Estimate`-shaped
> class in the same codebase if built as written. Anyone promoting this
> article's sketch to real code should rename one of the two, not merge
> them silently — `WorkModule`/`TaskItem`'s citation and confidence-gating
> machinery (`SourceCitation`, `insufficient_context_explanation`) has no
> equivalent in this article's schema and would be quietly lost in a
> same-name overwrite.

> *(Editor's note — a gap shared with s12-02, not unique to this article.)*
> `AgentResult` and `Step` are used throughout (`run_agent`'s return type,
> `trace: list[Step]`) but never defined here, exactly as in s12-02. Both
> read as small dataclasses (`Step` needs at least `action`, `args`,
> `observation`; `AgentResult` needs `status`, and optionally `estimate` and
> `trace`) but a reader typing this in verbatim needs to supply them —
> worth a one-line dataclass each if this becomes the actual `s12`
> implementation phase rather than staying a worked example.
