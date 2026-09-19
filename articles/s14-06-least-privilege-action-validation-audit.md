---
title: "Least privilege, action validation, and audit for agents"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 14
series_part: 6
scope: evergreen
source: user-supplied article
reading_time: 19 min
added: 2026-09-19
summary: >
  Every agent so far only reads — the cost of a hallucination is a wrong
  number, caught by the validator or the human gate. The day one writes,
  the cost becomes a deleted row, a misdirected email, corrupted
  production state. Containment can't live in the prompt: a system prompt
  is an instruction the model weighs against everything else, not a
  restriction, and the same "never trust the client" discipline already
  applied to frontends and users applies unmodified — the model is an
  untrusted client proposing actions. Three layers, all outside the
  prompt. Least privilege in the tool grant itself (a table mapping agent
  to exactly the tools it needs, verified at startup so a miswired grant
  fails deployment rather than reaching production) — concentrating writes
  into one small, auditable persistence agent rather than scattering them
  is deliberate, the same reasoning that isolates payment-handling code.
  Action validation as a deterministic guard between intent and execution
  — plain code, not an LLM validating another LLM, checking both privilege
  and argument sanity (an estimation_id that doesn't match the current run
  is the check that matters most and is the one most often skipped), with
  irreversible actions routed to the human gate already built rather than
  auto-approved. Audit of every attempted action, allowed or denied —
  denials are the most valuable log lines in the system, an early warning
  the human gate itself is not; content gets redacted, not raw customer
  data. And a deliberately drawn boundary: everything here is
  application-level, answering what an agent can do within the logic; it
  is not process isolation, network policy, resource limits, or secrets
  management, which answer what damage a misbehaving process can do and
  belong to the next session's deployment concerns — the two layers
  complement, neither substitutes for the other.
keywords: [least privilege, tool grants, action validation, audit logging,
           ToolRisk, guard_action, execute_guarded, redact_sensitive,
           application-level security, process isolation, defense in
           depth]
---

# Least privilege, action validation, and audit for agents

*Antonio Perez* · 🔴 19 min

Up to now your agents only read. The searcher queries budgets, the
generator calculates, the validator checks coherence. If one of them
hallucinates, it produces a bad estimate, and a bad estimate gets caught
by the validator or by the human at the gate. The cost of an error is a
wrong number.

That changes completely the day an agent writes.

Someone's going to add the tool that was missing. `save_estimate`, to
persist the result. Or `update_budget_status`, to mark a budget as used.
Or `send_estimate_email`, because automating the send to the client would
be nice. And at that point you have a component governed by a language
model — a machine that sometimes hallucinates with complete confidence —
with permission to modify your company's data or send emails on its
behalf.

The cost of an error stops being a wrong number. It becomes a deleted row,
an email sent to the wrong person, corrupted state in production.

This article is about containing that. And the thesis is uncomfortable
from the start: containment can't live in the prompt.

## Why the prompt isn't a security mechanism

The instinctive reaction is to write something in the system prompt like
"never delete data" or "you should only read, never write." It's
understandable and it's useless.

A system prompt is an instruction, not a restriction. The model takes it
as one more input, weighs it alongside everything else in the context, and
most of the time respects it. Most of the time. An unusual prompt, a
transcript with content pushing in another direction, a chain of reasoning
that convinces itself the rule doesn't apply this time — any of those can
lead the model to do exactly what you told it not to.

Compare it with how you protect anything else. You don't trust the
frontend to "not send" a forbidden field: you reject it in the backend.
You don't trust a user to "not access" someone else's resource: you check
it with authorization. The rule you've been applying your whole career —
don't trust the client — applies here without a single modification. The
model is the client. It's an untrusted input proposing actions, and
untrusted inputs get validated in a layer the client doesn't control.

Everything that follows is that idea, applied three times.

## Layer 1: least privilege, in the tool grant

You already built the first containment, even though you didn't call it
security. When you gave `budget_searcher` only `search_budgets` and
`estimate_generator` only `calculate_estimate`, you applied least
privilege: each agent accesses only what its function needs.

The principle is old and it's the same as always: if an agent doesn't have
a tool in hand, it can't misuse it no matter how much it hallucinates. An
agent without `save_estimate` isn't going to corrupt the database, whether
by mistake or by manipulation, because the capability doesn't exist in its
world.

The design consequence is that the tool grant stops being a convenience
and becomes a security decision. And that forces you to name each tool's
nature, an exercise worth making explicit:

```python
from enum import Enum

class ToolRisk(str, Enum):
    PURE = "pure"          # no side effects: calculate_estimate
    READ = "read"          # reads state: search_budgets
    WRITE = "write"        # mutates state: save_estimate
    EXTERNAL = "external"   # acts on the world: send_estimate_email

AGENT_TOOL_GRANTS: dict[str, set[str]] = {
    "requirements_extractor": set(),
    "budget_searcher": {"search_budgets"},
    "estimate_generator": {"calculate_estimate"},
    "coherence_validator": {"validate_estimate"},
    "persistence_agent": {"save_estimate"},
}
```

Notice a decision that goes beyond the grant itself: we've pulled writing
out into its own `persistence_agent`. That's deliberate. Concentrating
write tools into one small agent, instead of scattering them across the
ones that already do other things, means the entire system's dangerous
surface fits in one file you can read in full in a minute. Most of your
agents stay in `read` and `pure`, and don't need to be watched with the
same intensity. It's the same logic that isolates the code handling
payments: not because the rest doesn't matter, but because concentrating
the risk makes it reviewable.

The grant is data, not an instruction, and that's what lets you check it
at startup:

```python
def verify_tool_grants(graph_agents: dict[str, Agent]) -> None:
    """Fail at startup if an agent was wired with a tool it was not granted."""
    for name, agent in graph_agents.items():
        granted = AGENT_TOOL_GRANTS.get(name, set())
        actual = {tool.name for tool in agent.tools}
        if not actual.issubset(granted):
            raise ConfigurationError(
                f"Agent '{name}' has ungranted tools: {actual - granted}"
            )
```

An agent mistakenly wired with a tool that isn't its own doesn't reach
production — it breaks the deployment. You've turned a security policy
into an invariant the system checks on its own.

> *(Figure in the original: `art_6_fig-01-privilegio-validacion-auditoria.png`
> — image not included in this repo. Three stacked sections. "1. Least
> privilege: each agent receives only its tools" — four teal boxes:
> `budget_searcher` [search_budgets, read], `estimate_generator`
> [calculate_estimate, pure], `persistence_agent` [save_estimate, write],
> `requirements_extractor` [tools: none]. "2. Validation: every action
> passes a guard before touching anything" — `agent intent` [purple] →
> `action guard: is this allowed?` [orange] branching to `execute` [green,
> "allow"] or `reject + log` [red, "deny"], both feeding into a shared
> dashed line down to section 3. "3. Audit: agent, tool, args, decision
> (allow/deny), result → structlog" [blue box], captioned "every intent
> gets logged, including denials. The whole execution is reconstructible."
> Caption: privilege in the code, validation before executing, and a
> record of everything attempted.)*

## Layer 2: validate the action, not just having it allowed

Least privilege decides whether an agent can use a tool. It says nothing
about with what arguments.

`persistence_agent` has permission for `save_estimate`. Fine. What if the
model decides to save a -400-hour estimate? Or overwrite an
`estimation_id` that isn't this run's? Or save an object missing half its
fields? It has permission to write; nobody said it had permission to write
anything at all.

You need a guard between the agent's intent and the actual execution: one
point every action with effects passes through, which approves or rejects
it before it touches anything.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ActionRequest:
    agent: str
    tool: str
    args: dict

@dataclass(frozen=True)
class GuardDecision:
    allowed: bool
    reason: str

def guard_action(request: ActionRequest) -> GuardDecision:
    # 1. Privilege: is this tool granted to this agent at all?
    if request.tool not in AGENT_TOOL_GRANTS.get(request.agent, set()):
        return GuardDecision(False, f"{request.agent} is not granted {request.tool}")

    # 2. Argument validation: rules the tool itself enforces, regardless of caller.
    if request.tool == "save_estimate":
        estimate = request.args.get("estimate", {})
        if estimate.get("hours", 0) <= 0:
            return GuardDecision(False, "estimate hours must be positive")
        if estimate.get("estimation_id") != current_estimation_id.get():
            return GuardDecision(False, "estimation_id does not match this run")

    return GuardDecision(True, "ok")
```

Three things worth calling out.

**The guard is plain, deterministic code.** No LLM validating another LLM
— that only adds a second fallible machine to the problem. The rules for
what counts as a valid action are business rules, and business rules get
written, tested, and reasoned about. A unit test can cover the guard
completely; it can't cover a prompt.

**The `estimation_id` check is the one that actually matters and the one
most often forgotten.** An agent that can write to any `estimation_id` is
an agent that, given the right input, modifies another client's estimate.
Tying every action to the run in progress — the same `estimation_id` you
already use as the checkpointer's `thread_id` — turns a generic permission
into one scoped to this context. It's the equivalent of not letting a user
edit resources that aren't theirs.

**Irreversible actions need something beyond validation.** Saving a row
can be undone. Emailing the client can't. For that class of action,
automatic validation isn't enough — the right move is routing them to the
human gate you already built. And here the session's two halves click
together: the human-in-the-loop wasn't only for low confidence; it's also
the approval mechanism for actions with no way back. Nothing new to
invent. The same pause, triggered by a different signal.

## Layer 3: audit, because what isn't logged didn't happen

The first two layers prevent. The third prevents nothing — it makes
everything reconstructible afterward. And it's what separates a system you
can operate from one you can only pray works.

When a client asks why their estimate changed, or when you want to
understand why the system did something strange on Tuesday at three, the
answer can't be "I don't know, the model decided that." It has to be a
record you can read.

The rule is blunt: every action with effects gets logged, including the
ones the guard denied. The denied ones are, in fact, the most valuable —
they're the system telling you exactly where an agent tried to step
outside its lane. A rising denial rate is an early warning, well before
anything breaks.

```python
async def execute_guarded(request: ActionRequest, tool: Callable) -> ToolResult:
    decision = guard_action(request)

    log = logger.bind(
        agent=request.agent,
        tool=request.tool,
        args=redact_sensitive(request.args),
        estimation_id=current_estimation_id.get(),
        allowed=decision.allowed,
    )

    if not decision.allowed:
        log.warning("action_denied", reason=decision.reason)
        raise ActionDeniedError(decision.reason)

    result = await tool(**request.args)
    log.info("action_executed", result_summary=summarize(result))
    return result
```

The log carries the `estimation_id`, which is — again — the same
identifier crossing all three layers and your traces. With that,
reconstructing everything the system did for a specific case is a query,
not archaeology. `redact_sensitive` is there for a reason best not learned
the hard way: an audit log that copies the client's personal data in plain
text is itself a privacy problem. You audit the action — who, what tool,
what shape of arguments, what result — not the full content of the data.

## What "sandboxing" is and isn't, here

The word sandboxing carries a specific image: containers, virtual
machines, isolated processes with no network access. It's worth being
honest about what this session covers and what it doesn't, because
confusing the two creates a false sense of security, which is worse than
having none.

Everything in this article lives at the application level, in your Python
code. It answers one question: what can this agent do within the
application's logic. And it's the right layer for privilege, argument
validation, and audit, because those are domain decisions — what counts as
a valid write, which actions are irreversible, what needs to be logged.
No container can answer any of those questions.

There's a second boundary, and it answers a different question: what
damage can the process do if an agent behaves unexpectedly. Process
isolation, network policies that stop an agent from calling somewhere it
shouldn't, CPU/memory/time limits, secrets management. That doesn't live
in your application code — it lives in the runtime and the deployment.
It's production-rollout material, and it arrives next session.

> *(Figure in the original: `art_6_fig-02-dos-fronteras-seguridad.png` —
> image not included in this repo. Two panels. "At agent level (this
> session)" [teal] — four green bullets: "Each agent accesses only its
> tools", "Validation of every action before executing", "Confirmation of
> irreversible actions", "Audit of every intent, allowed or not" —
> captioned "answers: what can this agent do within the application's
> logic. Lives in your Python code." "At infrastructure level (Session
> 15)" [blue] — four blue bullets: "Process and container isolation",
> "Network and egress policies", "Resource limits (CPU, memory, time)",
> "Secrets, credentials, rotation" — captioned "answers: what damage can
> the process do if an agent behaves unexpectedly. Lives in the runtime
> and the deployment." Header caption: two security boundaries, not one —
> what you do in the AI service's code doesn't substitute for what the
> infrastructure does, and vice versa.)*

The relationship between the two boundaries is what matters: they don't
substitute for each other, they complement each other, and neither alone
is enough. Action validation doesn't protect you from an agent executing
arbitrary code through a badly designed tool — process isolation is needed
for that. And process isolation doesn't protect you from an agent saving a
negative estimate with arguments perfectly valid from the operating
system's point of view — domain validation is needed for that. A serious
system needs both. This session leaves the first one built, and built
well; the second has its own place.

## Closing the module: what you've built

With this, the agent orchestration module is complete, and it's worth
looking back for a moment, because the journey has a shape.

You started with a linear graph that did its job. You questioned it: why
complicate it? And only once concrete limits showed up — overloaded
prompts, routes that depend on the case, decisions the code can't foresee
— did you reorganize it into a supervisor that routes and agents that
specialize. You chose how they communicate, knowing the shared blackboard
you already had was the right starting point. You gave the system the
ability to stop and ask for help when it knows it doesn't know, leaning on
the same checkpointer as always. You learned to make agents compete when
disagreement is information. And you've put limits on what each agent can
touch, validation on what it does, and a record of all of it.

None of that was a new paradigm. Every piece turned out to be an
engineering principle you already knew, applied to a component that
sometimes hallucinates: separation of responsibilities, contracts between
layers, don't trust the client, persistence to survive failures, defense
in depth. The genuinely new layer was small. That was the deal from the
start.

Your estimation system is functionally complete. It receives a transcript
and returns a defensible estimate, with a range, with explicit
assumptions, with a person in the loop when needed, and with no agent able
to do more than its part.

Functionally complete, yes. But it runs on your machine. Nobody's deployed
it, nobody's monitoring it, nobody's measured its latency under real load
or put a cap on what it costs per month. The other half of "in
production" — the half that starts where the code ends and operations
begins — is what's still ahead.

---

> *(Editor's note — `persistence_agent` is a sixth node this session never
> added to `s14-02`'s shared `AgentName` type, the same class of gap as
> `retry_count` and `human_review_gate`'s placement before it.)* `s14-02`'s
> `AgentName` `Literal` — the supervisor's complete routable destination
> set — was `["requirements_extractor", "budget_searcher",
> "estimate_generator", "coherence_validator", "finalize"]`. This article
> introduces `persistence_agent` as a graph node with its own tool grant
> but never updates that `Literal`. If routing to it goes through the
> supervisor (rather than a fixed edge after generation, the way `s14-04`'s
> own logic argued the human-review gate should), `AgentName` needs the
> same update `human_review_gate` did — worth deciding both at once rather
> than accreting Literal members one article at a time.

> *(Editor's note — `verify_tool_grants` presumes a node shape nothing in
> this session actually built.)* `agent.tools` requires each graph node to
> be wrapped in an object exposing its own tool list — the shape LangGraph's
> real "swarm" pattern uses (each agent its own tool-calling subgraph,
> discussed and flagged as a different topology in `s14-03`'s notes), not
> the shape every node function shown across `s13-02` through `s14-05`
> actually has. `budget_searcher`, `conservative_estimator`, and every
> other node in this whole arc are plain functions that call one fixed
> capability directly in code — `await retrieve_reference_budget(...)`, not
> a model choosing among a `.tools` list via function-calling. There is no
> `.tools` attribute to introspect on any node actually built so far. The
> `AGENT_TOOL_GRANTS` table itself is still valuable as documented intent
> and as the data `guard_action` checks against; `verify_tool_grants`'s
> specific implementation needs either a real `Agent` wrapper class built
> first, or a different startup check (e.g., static inspection of which
> capability functions each node's source references).

> *(Editor's note — checked against the code, `lidr/ai-engineering`,
> branch state at session 11 completed — the natural extension point
> already exists, and a related but distinct precedent for `redact_sensitive`
> too.)* No `ToolRisk`, `AGENT_TOOL_GRANTS`, `guard_action`, or
> `persistence_agent` exists yet. But `app/foundation/guardrails/` already
> holds `input.py` and `output.py` — two guardrail policies (moderation/
> injection/PII on the way in, scope/format filtering on the way out) this
> handbook has referenced since Part 5. This article's `guard_action` is a
> natural third policy in that same module — action validation, not input
> or output — rather than a new pattern. Separately, `redact_sensitive`'s
> underlying principle (don't persist raw sensitive data) already has a
> real precedent in `app/ingestion/pii/` (Session 6's Presidio-based
> pseudonymization), though at a different pipeline stage — ingestion-time
> document processing, not runtime audit-log redaction of tool-call
> arguments. Same discipline, different checkpoint.

> *(Editor's note — this article's own hand-off matches a reference
> already in this handbook's README, written before any article in
> Part 12-14 existed.)* This article explicitly closes session 14
> ("the agent orchestration module is complete") and hands off to
> deployment and operations — "the other half of 'in production'... is
> what's still ahead." The README's own Provenance section already named
> this, independently, months earlier in this repo's history: *"S15 is
> referenced but not in this repo — several articles defer production
> concerns (key rotation, distributed rate limiting, observability
> tooling) to it."* This article's own closing line and that pre-existing
> reference agree on where the story goes next without either having been
> written with the other in view.
