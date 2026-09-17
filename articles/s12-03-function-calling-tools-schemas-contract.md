---
title: "Function calling in practice: tools, schemas, and the contract with the model"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 12
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 21 min
added: 2026-09-16
summary: >
  The model never executes your code — it emits a structured request, your
  code runs it, you return the result. Four beats: declare tools, the model
  emits a request, your code executes, you return the output keyed by
  call_id. A tool is four parts (name, description, parameters, strict
  schema), and the description is the highest-leverage one — a wrong tool
  choice or invented argument is almost always a vague description, not a
  model failure. Parallel tool calls in one turn are common and should be
  gathered together, not answered one by one. OpenAI and Anthropic use
  different vocabulary (function_call/call_id vs. tool_use/tool_use_id) for
  an identical contract — isolate the transport difference in a thin adapter
  or an aggregator like LiteLLM, never in tool logic. Design tools the way
  you'd design an API for a colleague who only reads the signature: one
  operation per tool, high-value results, informative errors, validation
  before anything irreversible executes.
keywords: [function calling, tool use, JSON schema, Responses API,
           Messages API, call_id, tool_use_id, parallel tool calls,
           strict schema, tool description, tool granularity, LiteLLM,
           provider portability]
---

# Function calling in practice: tools, schemas, and the contract with the model

*Antonio Perez* · 🔴 21 min

An agent that only reasons isn't much use. To estimate a project it needs to
do things: retrieve comparable historical budgets, calculate costs from
those references, validate that the result makes sense. Function calling is
the mechanism by which a model goes from talking to acting, and it's by far
the most useful piece of an agent's kit. It's also the most misunderstood.

So let's kill the misunderstanding at the root, because it contaminates
everything else: **the model does not execute your code. Ever.** It doesn't
query your database, doesn't run your calculation function, doesn't touch
anything. The only thing it does is emit a structured request — "I want to
call `search_budgets` with these arguments" — and your code decides what to
do with it. Function calling isn't the model executing functions; it's the
model asking you to execute them. Internalizing this changes how you design
the whole system.

For what follows, let's set the ground. The AI service has operations the
agent needs to execute during an estimation: `search_budgets` retrieves
comparable historical budgets, `calculate_estimate` computes costs from
references, `validate_estimate` runs checks on the result. For the model to
be able to invoke them, we declare them as tools. Let's see exactly what
that means.

## 1. The contract: who does what

Function calling is a contract between your code and the model, and it has
four beats.

First, you declare the available tools: what operations exist and what
shape their inputs take. Second, the model, if it decides it needs one,
emits a structured request with the tool's name and arguments. Third, your
code executes the operation — this is where the vector database actually
gets queried, or the calculation actually runs. Fourth, you return the
result to the model, which continues reasoning with that new data in hand.

Seen this way, none of this is exotic. It's a textbook typed interface: you
declare a schema, someone requests an operation conforming to that schema,
you execute it, you return the result. The only difference from any API
you've integrated is who's on the other side making the request: a language
model choosing the function based on the conversation, instead of a client
with a fixed flow. An engineer with API experience integrates function
calling exactly the way they'd integrate any interface: define the schema,
handle the callback, return a result. That's the right mental model, and
it's reassuringly familiar.

An example grounds it. The model reads a transcript that mentions an ERP
integration. Instead of inventing a cost, it emits a request: call
`search_budgets` with a query describing the integration and
`component_type` set to `"integration"`. Your code receives that request,
runs the real retrieval against the vector database, and returns the
historical budgets found. The model, now with real data in front of it,
reasons over it and decides the next step. At no point did the model touch
the database: it asked, you executed, you returned. That division of labor
is the entire mechanism.

> *(Figure in the original: `S12-fig-03a-contrato-function-calling.jpg` —
> image not included in this repo. It diagrams the four beats across two
> columns, "your code — AI service" and "the model (LLM)": 1) you declare
> the tools [name, description, parameters, strict] → sent with the
> request → 2) the model emits `function_call` {name, arguments, call_id} →
> 3) you execute the operation [queries the DB; the model never touches it]
> → 4) you return `function_call_output` {call_id, output} → 5) the model
> continues reasoning. Callout: the model asks, your code executes, the
> model never touches your database — `call_id` pairs each `function_call`
> with its `function_call_output`.)*

## 2. Anatomy of a tool: name, description, parameters, schema

A tool is defined with four pieces, and they deserve to be treated with the
respect they're due, because the model only ever sees this — never your
implementation.

- **Name.** A unique identifier. As the tool library grows, a
  disambiguating namespace (`budgets_search`, `estimate_calculate`) keeps
  the choice unambiguous.
- **Description.** What the model reads to decide when and how to use the
  tool. By far the highest-leverage piece, and we'll come back to it.
- **Parameters.** Defined as JSON Schema: types, required fields, enums,
  and a description per parameter.
- **Strict schema.** With `strict: true`, the arguments the model generates
  conform exactly to the JSON Schema you declared.

In OpenAI's Responses API, a tool looks like this:

```python
TOOLS = [
    {
        "type": "function",
        "name": "search_budgets",
        "description": (
            "Search historical project budgets for items comparable to a single "
            "software component. Call this once per component; do not combine "
            "unrelated components (for example, an ERP integration and a data "
            "migration) into one query."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "A focused description of one component to price.",
                },
                "component_type": {
                    "type": "string",
                    "enum": ["integration", "migration", "frontend", "backend", "mobile"],
                    "description": "Category of the component, used to filter results.",
                },
            },
            "required": ["query", "component_type"],
            "additionalProperties": False,
        },
        "strict": True,
    },
]
```

Notice how much intent lives in the description and the enums. They're not
there to document — they're there to steer the model's behavior. The
`component_type` enum restricts what the model can pass; the "one call per
component" instruction inside the description is what stops the model from
lumping the integration and the migration into a single search and getting
back a useless jumble. The schema doesn't just validate — it teaches.

## 3. The round trip in the Responses API

With tools declared, the exchange is direct. You call the model passing the
input and the tools; the model responds, and if it decides to act, its
output contains a `function_call`-type item.

```python
import json
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5",
    reasoning={"effort": "medium"},
    input=[{"role": "user", "content": transcript}],
    tools=TOOLS,
)

for item in response.output:
    if item.type == "function_call":
        args = json.loads(item.arguments)
        result = execute_tool(item.name, args)   # your code runs the operation

        response = client.responses.create(
            model="gpt-5",
            previous_response_id=response.id,
            input=[{
                "type": "function_call_output",
                "call_id": item.call_id,
                "output": json.dumps(result),
            }],
            tools=TOOLS,
        )
```

Three things worth not overlooking. First: the Responses API's output isn't
plain text, it's a list of typed items, and `function_call` is one of those
types — you have to iterate `response.output` and inspect each item's
`type`, not assume the first one is the answer. Second: every
`function_call` carries a `call_id`, and when you return the result with
`function_call_output` you have to reference that same `call_id`. That's
what pairs your result with the right request, and forgetting it is the
most common beginner mistake. Third: state chains via `previous_response_id`,
so the model keeps the previous turn's context without you having to resend
all of it by hand.

This exchange repeats: as long as the model keeps requesting tools, you
execute and return; when it stops requesting them and produces the final
answer, you're done. That's the loop, and the mechanics of every turn are
exactly the ones above.

A concrete heads-up that saves an hour of debugging: in the Responses API
the tool's schema is flat — `type`, `name`, `description` and `parameters`
all at the same level. If you're coming from the Chat Completions API,
there the schema is nested under a `function` key. Copying the format from
one to the other returns a parameter error that isn't always obvious.

## 4. Calls in parallel

A single model response can contain not one but several `function_call`
items. This is common and desirable: faced with a transcript with four
independent components, the model can request four `search_budgets` calls
at once — one per component — instead of chaining them one at a time.
Executing them concurrently and returning all four results together cuts
latency noticeably compared to four sequential turns, and it's one of the
cheapest optimizations you'll find.

The implementation nuance matters. When a turn contains several calls, you
don't answer one and call back for each: you collect all the
`function_call` items from the response, execute each operation — ideally
with `asyncio.gather`, since the AI service's tools are async — and return
all the `function_call_output`s, each with its own `call_id`, in a single
continuation request. The simplest loop handles the single-call case fine;
the moment you assume there can be several, the correct shape is to gather
the outputs before resuming. Assuming there's always exactly one is another
classic source of subtle bugs — it works in your tests with simple
transcripts and breaks on the first complex meeting.

## 5. The same contract, a different provider

Function calling isn't OpenAI's; it's a pattern each provider implements in
its own shape. In Anthropic's API, the same tool is declared with
`input_schema` instead of `parameters`:

```python
tools = [
    {
        "name": "search_budgets",
        "description": "Search historical project budgets for one software component.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "One component to price."},
                "component_type": {"type": "string"},
            },
            "required": ["query", "component_type"],
        },
    },
]
```

And the rest of the vocabulary shifts in parallel: the model returns
`tool_use` blocks (not `function_call` items), the response arrives with
`stop_reason: "tool_use"`, and you answer with a `tool_result` block (not a
`function_call_output`). The names differ; the contract is identical.
Declare a schema, the model requests an operation conforming to it, you
execute it, you return the result, the model continues.

That the contract is the same and only the shape changes has an
architectural consequence worth exploiting: you can isolate those transport
differences in a thin layer, or delegate them to an aggregator, and keep
your tools' logic — what `search_budgets` does, what it returns —
completely independent of the provider. Your functions don't change; only
the adapter translating between your schema and each API's does. It's the
same portability you'd want from any external dependency, applied to the
model provider.

> *(Figure in the original: `S12-fig-03b-mismo-contrato-openai-anthropic.jpg`
> — image not included in this repo. Three columns — OpenAI (Responses API),
> "the contract (identical)", Anthropic (Messages API) — pairing
> `parameters`/"declare the tool's schema"/`input_schema`,
> `function_call`/"the model requests an operation"/`tool_use`, "you inspect
> output items"/"signal that a request exists"/`stop_reason: tool_use`,
> `function_call_output`/"you return the result"/`tool_result`, and
> `call_id`/"pair request and result"/`tool_use_id`. Callout: isolate the
> shape in a thin layer, or an aggregator like LiteLLM — your tools' logic
> doesn't change when you change provider.)*

## 6. Where all this lives

All of this machinery lives inside the AI service, in Python. Tools aren't
new operations — they're the exposure, as model-invokable functions, of
capabilities the AI service already has. `search_budgets` wraps retrieval
over the vector database; `calculate_estimate`, the deterministic cost
logic; `validate_estimate`, the quality checks on the result. Declaring a
tool is, in practice, writing its schema and wiring its execution to a
function that already existed.

The business backend takes no part in any of this. It doesn't declare
tools, doesn't see `function_call` or `tool_use`, doesn't manage `call_id`.
It sends a transcript to the AI service's endpoint and gets back a
structured estimate. The entire function-calling exchange — the turns, the
arguments, the results — is an internal detail of the AI service, invisible
from outside. That boundary is what lets you completely change how the
agent uses its tools without touching a single line of the business
backend.

## 7. Designing tools the model uses well

The mechanics are easy; getting the model to use your tools well is where
the real work is. And it almost all comes down to two surfaces: the
description, which governs the input, and the result, which governs the
next decision.

**The description is the interface.** The model chooses the tool and fills
in its arguments reading only the description and the schema. If the model
picks the wrong tool or invents odd arguments, the cause is almost never the
model — it's a vague description. Write it for a reader who can't see your
code and has only those sentences to decide with. A well-placed enum or an
explicit constraint — "one call per component" — does more for reliability
than any temperature adjustment.

**Results should be high-value.** Return only what the model needs to
decide the next step, with stable, semantic identifiers, not a dump of two
hundred raw budget rows. A bloated result wastes context — which you also
resend every turn — and confuses the model. Less and cleaner is better.

**Errors are results too, and important ones.** When `search_budgets` finds
nothing useful, return it as an observation with information — "1 weak
match, low confidence for the legacy migration" — and the model can reason
and reformulate. Return a generic "error" and the model is left blind. An
informative error message is what lets an agent recover from its own
stumbles.

**Validate arguments before executing anything that hurts.** `strict: true`
guarantees the arguments have the schema's shape, not that they make sense.
For a read-only tool like `search_budgets`, executing directly is
acceptable. For an action with effects — persisting, sending, moving
something — add your own validation layer over the arguments the model
proposes before executing them. The schema is a type guard, not a
substitute for judgment.

**Mind the granularity.** It's an overlooked design decision that directly
affects the reliability of tool choice. Too many tools with overlapping
boundaries confuse the model, which hesitates over which to use; too few and
too generic force it to juggle arguments to express what it wants. The
sweet spot is usually one tool per operation with sharp boundaries:
`search_budgets` searches, `calculate_estimate` calculates,
`validate_estimate` validates, and none encroaches on the others' territory.
If you find yourself explaining in the description when *not* to use a
tool, that tool may be doing too much.

## 8. Closing: a typed interface with an unusual client

With the pieces in place, function calling stops being an AI mechanism and
reveals itself for what it is: a typed interface whose client happens to be
a model. You declare the schema, handle the callback, return the result.
The discipline is the usual one — clear contracts, input validation,
informative errors, high-value responses — and there's nothing in it you
haven't already done integrating any API.

The only genuinely new thing is that the one choosing the function and
filling in its arguments is the model, from a natural-language description.
And that's where the lever is: the reliability of your tools doesn't live
in a better model, it lives in better descriptions and better results.
That's interface engineering, not machine learning. Design the tool the way
you'd design a good API for a colleague who's only going to read the
signature and the documentation, because that, in essence, is exactly what
the model is going to do.

## Sources

- OpenAI, *Function calling* (Responses API) — flat schema,
  `function_call`/`function_call_output` items, `call_id` and `strict`:
  https://developers.openai.com/api/docs/guides/function-calling
- OpenAI, *Migrate to the Responses API* — shape differences from Chat
  Completions: https://platform.openai.com/docs/guides/migrate-to-responses
- Anthropic, *How tool use works* — the tool contract as a typed interface
  and the `tool_use`/`tool_result` blocks:
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
- Anthropic, *Define tools* — effective descriptions, namespacing, and
  high-value results:
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use

---

> *(Editor's note — this is s12-02 §4's promised detail, not a new topic.)*
> s12-02 named action and observation as organs and said, in passing,
> "mechanically, this is function calling." This article is that sentence
> expanded to the mechanics: §1's four beats are s12-02's action/observation
> pair made concrete, §4's parallel-calls handling is what a `model.decide`
> turn that requests more than one tool actually requires from
> `execute_tool`, and §7's "errors are results too" restates s12-02 §5's
> observation-quality argument from the tool-schema side rather than the
> agent-loop side. Read together, not as three separate topics.

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — and a distinction worth being precise
> about.)* No `tools=`, `function_call`, `responses.create`, `tool_use`, or
> `input_schema` appears anywhere in the codebase. `app/foundation/llm/wrapper.py`
> builds `self._instructor = instructor.from_litellm(litellm.completion)` and
> every generation call goes through `complete_structured`/
> `complete_structured_chat`, never the raw Responses API this article
> demonstrates. This is not a contradiction to flag the way s09-04's
> `reorder_u_pattern` was — it's two different applications of the same
> underlying provider primitive, and conflating them is an easy mistake for
> a reader to make. Instructor uses tool-calling internally to coerce **one**
> response into a Pydantic shape, in a single round trip, with no `model.decide`
> loop and no possibility of the model choosing among several tools — closer
> to this article's §3 mechanics used for structured output than to an agent
> selecting actions. This article's four-beat contract, with a real
> multi-turn `function_call`/`function_call_output` exchange the model can
> repeat, has no counterpart in the codebase yet — the same "session 12 is
> ahead of the code" finding as s12-01 and s12-02.

> *(Editor's note — reinforces an existing PLAYBOOK.md default with a
> reason it didn't have before.)* §5's "isolate the transport difference in
> a thin layer, or an aggregator like LiteLLM" names, directly, the same
> choice `PLAYBOOK.md` §4 already defaults to ("LiteLLM + Instructor, one
> wrapper, cross-provider fallback") — but that default previously had no
> stated reason connected to tool-schema portability specifically, only
> spend/retry tracking. Updated in `PLAYBOOK.md` §4 to cite this article for
> that half of the reason.
