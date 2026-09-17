---
title: "LangGraph from scratch: StateGraph, nodes, edges, and state"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 13
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 13 min
added: 2026-09-16
summary: >
  Four primitives: a shared typed state every node reads and updates, a
  node (a pure function reading state, returning only the fields it
  changes), an edge (direct edges fix sequence, conditional edges inspect
  state and route), and a checkpointer (covered separately). StateGraph is
  the builder combining the first three, compiled into an executable
  graph. The state schema is the highest-leverage decision: reducers
  (Annotated[list[X], operator.add]) decide whether an update overwrites
  (the default — right for status/estimate) or accumulates (right for
  budget_matches, and required for parallel branches not to clobber each
  other). Keep state minimal — everything in it serializes on every
  transition. A routing function inspecting state and returning the next
  node's name (or END) is the entire mechanism for conditional control,
  and the same mechanism later supports retries, cycles, and richer
  branching. The upfront design cost is real for a trivial flow and is
  exactly what buys the control back for a real one.
keywords: [LangGraph, StateGraph, nodes, edges, reducers, operator.add,
           TypedDict, conditional edges, START, END, routing function,
           state schema, graph compilation]
---

# LangGraph from scratch: StateGraph, nodes, edges, and state

*Antonio Perez* · 🔴 13 min

The estimation flow, told as a list of steps, is easy to state: from a
meeting transcript, extract requirements; group the requirements into
components; for each component, search reference budgets; generate an
estimate from those budgets; and finally validate and consolidate the
estimate. Five steps, each with a clear responsibility.

Stating it is easy; expressing it in code that stays readable once there
are branches and parallelism, less so. An imperative loop works, but mixes
each step's work with the logic of what comes after what, in the same
place. LangGraph separates those two things: the work lives in nodes, the
control lives in edges, and the data lives in a shared state. This article
builds that model from scratch over the estimation flow.

## 1. Four primitives and nothing else

LangGraph's model is deliberately small. Four pieces:

- **State**: a shared, typed object every node reads and updates.
- **Node**: a function that receives the state and returns a partial
  update of it.
- **Edge**: the connection between nodes. Direct edges fix a sequence;
  conditional ones decide where to go by looking at the state.
- **Checkpointer**: persists state after every step (covered separately;
  here it's enough to know it exists).

`StateGraph` is the builder that joins the first three. You instantiate it
with a state schema, add nodes to it, draw the edges, fix the entry point,
and compile it. The result is an executable graph.

## 2. State: the most consequential decision

In a LangGraph project, designing the state schema is the highest-weight
decision. Everything else — what each node does, how edges route — is read
and written against that object. A badly thought-out schema is paid for in
every node.

State is a `TypedDict` (Pydantic or dataclasses also work — pick one and be
consistent). For the estimation flow:

```python
from typing import Annotated, Optional, TypedDict
import operator

class Component(TypedDict):
    name: str
    category: str

class BudgetMatch(TypedDict):
    component: str
    reference_budget_id: str
    amount: float

class EstimationState(TypedDict):
    transcript: str
    requirements: list[str]
    components: list[Component]
    # Accumulator: each search appends its matches.
    budget_matches: Annotated[list[BudgetMatch], operator.add]
    estimate: Optional[dict]
    status: Optional[str]  # "validated" | "needs_review"
    errors: Annotated[list[str], operator.add]
```

The key is in the annotated fields. By default, when a node returns a value
for a field, that value overwrites whatever was there. That's what you want
for `status` or `estimate`: the last value wins. But for `budget_matches`
you don't want to overwrite — you want to accumulate, because every search
contributes its results and all of them need to add up. That's a reducer.
`Annotated[list[BudgetMatch], operator.add]` tells LangGraph that when a
node returns `budget_matches`, instead of replacing the list it should
concatenate it with the existing one. The `operator.add` reducer is what
makes parallel execution make sense: several branches can write at once
without stepping on each other.

One discipline that saves grief: keep the state light. Everything in the
state gets serialized on every transition between nodes. If you put raw
model responses with their metadata in there, the state object bloats and
persistence becomes the bottleneck. Store identifiers and already-distilled
data; leave the transient stuff in function scope.

> *(Figure in the original: `S13-fig-02a-flujo-grafo.jpg` — image not
> included in this repo. The estimation flow as a graph: `START` →
> `extract_requirements` → `classify_components` → `search_budgets`
> [teal, "retrieval"] → `generate_estimate` → `validate_and_consolidate`
> [orange-bordered, "conditional edge"], branching to `END`/`validated`
> [green] or `flag_for_review`/`needs_review` [orange]. Caption: five nodes
> with their own responsibility; control lives in the edges. Callout:
> transcript comes in from the business backend, estimate with status goes
> out — the AI service's contract doesn't change.)*

## 3. Nodes: pure functions that return a slice of state

A node is a normal Python function. It receives the state and returns a
dict with only the fields it changes — not the whole state. Treat it as a
pure function: don't mutate what you receive, return the update. That makes
nodes trivial to test and keeps routing predictable.

```python
def classify_components(state: EstimationState) -> dict:
    components = group_requirements_into_components(state["requirements"])
    return {"components": components}

def search_budgets(state: EstimationState) -> dict:
    matches: list[BudgetMatch] = []
    for component in state["components"]:
        matches.append(retrieve_reference_budget(component))
    # Only the changed field is returned; the reducer merges it in.
    return {"budget_matches": matches}
```

Every node reuses domain logic that already exists in the AI service —
retrieval over the vector store, deterministic estimate calculation —
wrapped in this shape. The node doesn't orchestrate: it does its work and
returns its slice of state. Who goes next is the edges' decision.

> *(Figure in the original: `S13-fig-02b-anatomia-nodo.jpg` — image not
> included in this repo. Three columns: "shared state (TypedDict)" listing
> `transcript`, `requirements`, `components`, `budget_matches
> [accumulator]` (highlighted), `estimate`, `status`, `errors
> [accumulator]` (highlighted) — reads into "node = pure function"
> [`def search_budgets(state): matches = [...]  # only the changed field;
> return {"budget_matches": matches}`] — writes a partial update into "the
> reducer combines the update" [`operator.add`: accumulates, concatenates
> the lists; default: overwrites, the last value wins] — looping back to
> write the combined state. Caption: afterward, a conditional edge
> inspects the combined state and picks the next node.)*

## 4. Edges: fixed sequence and decisions

Direct edges fix the order when the order is fixed. Conditional ones route
when a decision has to be made. `START` and `END` are the graph's entry and
exit sentinels.

```python
from langgraph.graph import StateGraph, START, END

def route_after_validation(state: EstimationState) -> str:
    # A routing function inspects the state and returns the next node's name.
    return END if state["status"] == "validated" else "flag_for_review"

builder = StateGraph(EstimationState)
builder.add_node("extract_requirements", extract_requirements)
builder.add_node("classify_components", classify_components)
builder.add_node("search_budgets", search_budgets)
builder.add_node("generate_estimate", generate_estimate)
builder.add_node("validate_and_consolidate", validate_and_consolidate)
builder.add_node("flag_for_review", flag_for_review)

builder.add_edge(START, "extract_requirements")
builder.add_edge("extract_requirements", "classify_components")
builder.add_edge("classify_components", "search_budgets")
builder.add_edge("search_budgets", "generate_estimate")
builder.add_edge("generate_estimate", "validate_and_consolidate")
# Conditional edge: the routing function decides where validation leads.
builder.add_conditional_edges("validate_and_consolidate", route_after_validation)
builder.add_edge("flag_for_review", END)

graph = builder.compile()
```

The routing function is the mechanism behind all dynamic control: it
inspects the state and returns the next node's name. Here it decides
whether a validated estimate finishes or gets diverted to a node that flags
it for review. That same mechanism — a function that looks at the state and
chooses — is what later supports retries, cycles, and richer branching.

Compiling closes the design and produces an executable graph. From there it
gets invoked (synchronously or asynchronously), can be streamed, and can be
persisted. The typed state you defined is the contract running through the
whole execution.

## 5. The cost of upfront design

There's an honest trade-off in all of this. You write a hand-rolled loop in
one pass; a graph forces you to decide, ahead of time, the state schema,
what counts as a node, and where a conditional edge goes. That upfront
design is real work, and for a trivial flow it's work that doesn't pay off.

The counterpart is that this upfront design is exactly what gives you
control afterward. A couple of rules keep the graph healthy. Reducers, only
where you genuinely need to accumulate; simple overwrite for everything
else. Conditional edges, only at real decision points, not on every
transition. And minimal, typed, validated state, because every byte gets
serialized at every step. A graph that respects this reads at a glance and
debugs per node; one that stuffs extra logic into the state or adds
branches that weren't needed loses exactly the advantage it was adopted
for.

## 6. What's next

With this, the estimation flow stops being a loop that has to be read
carefully and becomes an explicit structure: typed state, five nodes with
their own responsibility, and edges that say, unambiguously, what comes
after what. That structure is visible and reasons about itself. What's
missing for it to hold up in production is that this state doesn't live
only in memory — that it persists after every step, survives a restart,
and distinguishes what lasts for a session from what lasts forever. That's
the checkpointer's and memory's territory.

## Summary

- LangGraph separates work, control, and data: nodes do the work, edges
  decide the control, shared state carries the data.
- The state schema is the highest-weight decision. It's a `TypedDict`;
  reducers decide how updates combine. `operator.add` accumulates
  (essential for parallelism); the default overwrites.
- A node is a pure function that receives the state and returns only the
  fields it changes. Easy to test, predictable routing.
- Direct edges fix the sequence; conditional ones route with a function
  that inspects the state and returns the next node's name. `START` and
  `END` are the sentinels.
- Compiling produces an executable graph with the typed state as the
  contract for the whole execution.
- Upfront design is both the cost and the advantage. Reducers only where
  you accumulate, conditional edges only at real decisions, minimal
  state: that's what keeps the graph readable at a glance and debuggable
  per node.

## Sources

- LangGraph — StateGraph, nodes, edges, and state (official docs):
  https://docs.langchain.com/oss/python/langgraph
- LangGraph — reducers and state schema:
  https://docs.langchain.com/oss/python/langgraph/graph-api
- State and edge design best practices in LangGraph:
  https://www.swarnendu.de/blog/langgraph-best-practices/

---

> *(Editor's note — checked against the code, `lidr/ai-engineering`, branch
> state at session 11 completed — present in the lockfile, absent from the
> code, and worth being precise about the difference.)* No
> `StateGraph`/`EstimationState`/`extract_requirements`/`classify_components`
> exists in `app/`, consistent with every article since s12-01. But unlike
> those, `langgraph` is not simply missing from the project — it's already
> **installed**: `uv.lock` resolves it as a transitive dependency of
> `langchain` (pulled in by `langchain-openai`, one of three `langchain-*`
> packages `pyproject.toml` declares directly, for chunking and embeddings
> from earlier sessions, not for orchestration). `import langgraph` would
> succeed today in this environment. Nobody evaluated or chose it for this
> purpose — it rode in as a side effect of unrelated dependencies, which is
> exactly the kind of thing worth checking explicitly (`uv tree` or
> equivalent) rather than assuming: "the package is importable" and "the
> package was deliberately adopted" are different facts, and conflating
> them is an easy way to think a framework decision already got made when
> it didn't.

> *(Editor's note — this is s13-01's recommendation made concrete, not a
> new decision.)* s13-01 argued that a graph framework earns its place
> specifically when the flow has "steps with dependencies, conditional
> routing, parallelism" — this article is that argument turned into code
> for the exact flow s13-01 named. The five-node backbone plus one
> conditional edge shown here is deliberately modest: it doesn't yet use
> parallel branches (the per-component `search_budgets` call inside the
> node is a Python loop, not parallel graph edges) or the checkpointer
> s13-01 identified as the actual justification (over 60% of incidents
> trace to state management, not routing). Read together with `PLAYBOOK.md`
> §2's updated Axis 5 guidance: this article's graph is the "conditional
> routing" third of that justification demonstrated; the parallelism and
> persistence thirds are, by this article's own closing section, still to
> come.
