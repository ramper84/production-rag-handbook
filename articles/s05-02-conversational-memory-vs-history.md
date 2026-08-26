---
title: "Conversational memory vs history: strategies for CAG systems"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 5
series_part: 2
scope: evergreen
source: user-supplied article
reading_time: 26 min
added: 2026-08-25
summary: >
  History answers "what did the user say on turn 7?"; memory answers "what do
  we know about this project?". Systems that merge them answer both badly.
  Memory is distilled facts in a typed structure, independent of the turn that
  produced them, so it survives sliding-window truncation — which is what lets
  you keep the simplest history strategy without its classic cost. Injected
  into the system prompt as established facts, updated after each turn by
  heuristic or by LLM extractor. Forgetting needs three explicit policies in
  three different places: user revision, session TTL, explicit reset.
keywords: [conversational memory, history, ProjectMetadata, sliding window,
           cumulative summary, anchors, distilled facts, Jinja2 injection,
           LLM extractor, heuristic extraction, forgetting policy, TTL,
           session reset, in-memory sessions, anti-patterns]
---

# Conversational memory vs history: strategies for CAG systems

*Antonio Perez* · 🔴 26 min

In session 02 we worked on the architecture of conversations: the message array travelling to the API, the system/user/assistant roles, the problem of the history growing and the three classic strategies for managing it (sliding window, cumulative summary, hybrid). If you go back to that material with session 05 in mind, you will notice something: in that article **everything was history**. The word "memory" appeared as an approximate synonym, with no operational distinction.

That elision was deliberate — in the estimator's initial phase you did not need more. But on introducing a multi-turn system conversing about a project in progress, the simplification breaks. The user expects the system to remember the project's name, the agreed technologies, the assumed team, the settled scope, without having to repeat them every turn. And they expect it **even when the turn where they mentioned those things has already fallen outside the sliding window.**

The conclusion is that **history and memory are two distinct things that are best treated separately in architecture.** Confusing them is one of the most expensive errors seen in CAG systems in production: a single growing blob is managed and truncated by eye, the system loses coherence about the project in progress, and the team ends up inflating the system prompt with instructions like "do not forget the project's name" that do not solve the underlying problem.

This article formalises that distinction and turns it into concrete pieces of code for the estimator.

## 1. Operational definitions

Let us start by anchoring the vocabulary.

**Conversational history** is the array of messages (system, user, assistant, user, assistant…) that travels to the LLM's API on every call. It is a raw structure: each message contains exactly what the user wrote and what the model replied, in chronological order. Its management — how many turns survive, how they are truncated, how they are summarised — is what we covered in the session 02 article.

**Conversational memory** is the set of relevant facts the system has learned about the conversation's domain across the turns. Facts are not turns; they are **distilled claims**. "The project is called BookFlow", "the assumed team is 3 full-time people", "the agreed stack is Rails + React + PostgreSQL", "the client has explicitly rejected using microservices for the first phase". Each fact has an origin (a turn where it was mentioned) but the memory is independent of the turn: it persists even if the original turn has been discarded from the history.

The distinction is subtle but critical. **The history answers "what did the user say on turn 7?". The memory answers "what do we know about this project?".** They are two different questions and systems that mix them end up answering both badly.

### Why keep the two separate

There are three operational reasons for keeping them as independent structures:

**Cost and latency.** The history grows linearly with the conversation; the memory does not. A well-bounded project may have twenty relevant facts after a hundred turns. If the facts travel in a compact structure separate from the history, you resend them on every LLM call without paying the cost of the full history.

**Resistance to truncation.** When you apply a sliding window, the old turns disappear. If the memory depends on the history, it disappears too. If the memory is an independent structure, it survives truncation. This is what stops the system "forgetting" the project's name when the conversation gets long.

**Auditability.** In a CAG system in production, you are going to be asked "why did the LLM assume X?". If the assumed facts are explicitly in an inspectable structure, you can answer. If they are scattered across an array of raw turns, you cannot.

## 2. Anatomy of the conversational state in the estimator

Let us make it concrete. An estimator session has three state components, not one:

> *(Figure in the original: `003-anatomia-estado-conversacional.jpg` — image not included in this repo. A `Session` root object holding a `session_id` (UUID v4) and two side-by-side blocks. **`history`** — "the raw" — listing `system: role and CAG context`, `user: turn N-2`, `assistant: turn N-2`, `user: turn N-1`, annotated "grows linearly with the conversation; suffers the sliding window's truncation". **`project_metadata`** — "the distilled" — listing `project_name: BookFlow`, `assumed_team_size: 3`, `technologies: Rails, React`, `agreed_scope: MVP phase 1`, `rejected_options: microservices`, annotated "grows only when new facts appear; survives the history's truncation".)*

`ConversationHistory` is what you already know: the list of messages with the sliding-window logic. `ProjectMetadata` is this session's novelty: a typed structure capturing the facts of the project in progress.

In code:

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import uuid4


class ProjectMetadata(BaseModel):
    """Distilled facts about the project under estimation.

    Survives history truncation. Updated after each turn.
    """
    project_name: str | None = None
    assumed_team_size: int | None = None
    mentioned_technologies: list[str] = Field(default_factory=list)
    agreed_scope: str | None = None
    explicit_constraints: list[str] = Field(default_factory=list)
    rejected_options: list[str] = Field(default_factory=list)


class Message(BaseModel):
    role: str  # "system" | "user" | "assistant"
    content: str


class Session(BaseModel):
    session_id: str = Field(default_factory=lambda: str(uuid4()))
    history: list[Message] = Field(default_factory=list)
    project_metadata: ProjectMetadata = Field(default_factory=ProjectMetadata)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

> *(Editor's note — checked against the code: `app/sessions/models.py` has all of this, with three differences. `ProjectMetadata` carries **four** fields, not six — `explicit_constraints` and `rejected_options` do not exist, which matters because the template below renders "Rejected options (do not propose these again)" and Policy 1 in §6 depends on that field. Its fields do carry the validation §4 recommends as a belt-and-braces check, at schema level: `max_length=120` on the name, `ge=1, le=50` on team size, `max_length=2000` on scope. And it has a `merge_with()` method implementing the extractor's merge rules in code — non-null fields from the update win, technologies are a case-insensitive union — rather than leaving them to the prompt, which is sturdier than what this article describes. `Session` names the field `metadata`, not `project_metadata`, and has no `updated_at`.)*

The separation is structural, not decorative. When the system receives a new turn, it does three things in order:

1. Injects `project_metadata` into the system prompt via a Jinja2 template, along with the current window of `history`.
2. Calls the LLM and obtains the response.
3. Updates both `history` (adding the new user/assistant pair) and `project_metadata` (extracting new facts).

Steps 1 and 3 are where the interesting piece lives. Step 2 is the LLM call you already know.

## 3. Injecting memory into the system prompt

The system prompt's Jinja2 template is updated to receive a `<project_metadata>` block with the known facts. When the session starts, the block is empty; with the turns, it fills up.

```jinja2
You are a senior software estimation expert. Produce realistic, well-justified
estimates for software projects based on meeting transcripts and complementary
documentation.

{% if project_metadata %}
<project_metadata>
{% if project_metadata.project_name %}
Project name: {{ project_metadata.project_name }}
{% endif %}
{% if project_metadata.assumed_team_size %}
Assumed team size: {{ project_metadata.assumed_team_size }} full-time engineers
{% endif %}
{% if project_metadata.mentioned_technologies %}
Technologies mentioned: {{ project_metadata.mentioned_technologies | join(", ") }}
{% endif %}
{% if project_metadata.agreed_scope %}
Agreed scope: {{ project_metadata.agreed_scope }}
{% endif %}
{% if project_metadata.explicit_constraints %}
Explicit constraints:
{% for constraint in project_metadata.explicit_constraints %}
- {{ constraint }}
{% endfor %}
{% endif %}
{% if project_metadata.rejected_options %}
Rejected options (do not propose these again):
{% for option in project_metadata.rejected_options %}
- {{ option }}
{% endfor %}
{% endif %}
</project_metadata>
{% endif %}

<context_examples>
{# CAG static reference estimates as in previous sessions #}
{% include "reference_estimates.j2" %}
</context_examples>

When producing the estimate, treat the project_metadata as established facts.
Do not contradict them unless the user explicitly revises them in the current turn.
```

Two details worth noticing:

**Conditional rendering.** Each metadata field is included only if it has a value. This stops the LLM seeing fragments like "Project name: None", which pollute the context and sometimes lead the model to invent values for the empty fields.

**Explicit treatment as established facts.** The system prompt's last instruction — *"treat the project_metadata as established facts"* — is not decorative. Without it, the LLM tends to treat the memory as one more suggestion among others and to renegotiate facts that are already closed. With it, the model understands the memory has authority over new interpretations.

The result: even though the turn where the user said "we are going to use Rails" has disappeared from the sliding window, the fact `mentioned_technologies: ["Rails"]` is still in the system prompt and the model does not ask itself again which stack is going to be used.

## 4. Updating the memory after each turn

Injecting the memory is the easy part. Keeping it alive is where the interesting piece is. After each LLM response, the system needs to inspect the new turn (what the user said + what the model replied) and update `project_metadata` with the relevant facts.

There are two canonical approaches, with very different cost and robustness profiles.

### Approach 1 — Simple heuristic

You define explicit rules that extract facts from the turn. For example: if the user or the model mention a proper noun appearing as the subject of "the project is called" or "named X", you capture it as `project_name`. If tokens known as technology names appear (from a maintained list), you add them to `mentioned_technologies`.

```python
import re

KNOWN_TECHNOLOGIES = {"rails", "react", "postgresql", "redis", "node", "python", ...}


def update_metadata_heuristic(
    metadata: ProjectMetadata,
    user_turn: str,
    assistant_turn: str,
) -> ProjectMetadata:
    combined = f"{user_turn}\n{assistant_turn}".lower()

    # Project name: simple regex
    if metadata.project_name is None:
        match = re.search(r"(?:project (?:is )?(?:called|named) )['\"]?([\w\s]+?)['\"]?[\.\,]", combined)
        if match:
            metadata = metadata.model_copy(update={"project_name": match.group(1).strip()})

    # Technologies: vocabulary match
    found = {tech for tech in KNOWN_TECHNOLOGIES if tech in combined}
    if found:
        merged = sorted(set(metadata.mentioned_technologies) | found)
        metadata = metadata.model_copy(update={"mentioned_technologies": merged})

    return metadata
```

**Advantages.** Zero cost per turn (no extra LLM call), negligible latency, predictable and debuggable behaviour.

**Disadvantages.** Fragile. The `project_name` regex assumes a concrete English formulation and breaks with any variation. If the user says "let's call it Bookflow internally", the regex does not capture it. Heuristics grow in complexity fast and end up being a small home-made NLP that is hard to maintain.

### Approach 2 — LLM extractor

You use a second LLM call, with a specific prompt, to return a JSON with the updated `ProjectMetadata` fields. The input is the current metadata + the last turn; the output is the new metadata.

```python
EXTRACTION_PROMPT = """\
You receive the current ProjectMetadata of a software estimation session and the
latest conversation turn. Your task: produce an updated ProjectMetadata that
incorporates any new facts revealed in the turn.

Rules:
- Only update fields when the turn provides clear evidence.
- Preserve existing values unless the user explicitly revises them.
- If the user retracts a previous fact, remove it.
- For lists (technologies, constraints, rejected_options), append new items
  without duplicating existing ones.

Current metadata: {current_metadata_json}

Latest turn:
USER: {user_turn}
ASSISTANT: {assistant_turn}

Return ONLY a valid JSON matching the ProjectMetadata schema.
"""


async def update_metadata_llm(
    metadata: ProjectMetadata,
    user_turn: str,
    assistant_turn: str,
    client,
) -> ProjectMetadata:
    response = await client.responses.create(
        model="gpt-4o-mini",
        input=EXTRACTION_PROMPT.format(
            current_metadata_json=metadata.model_dump_json(),
            user_turn=user_turn,
            assistant_turn=assistant_turn,
        ),
        response_format={"type": "json_object"},
    )
    return ProjectMetadata.model_validate_json(response.output_text)
```

**Advantages.** Robust against language variations, multilingual with no extra work, captures subtle facts no reasonable regex would catch. And it reuses a pattern you already know from session 04 (structured data with a schema).

**Disadvantages.** A real cost: an extra LLM call per turn. With `gpt-4o-mini` we estimated a per-call cost in cents in session 04, so multiplied by turns it is still very cheap — but not zero. Additional latency of 500-1500 ms per turn. And a new risk: **the extraction can be wrong.** If the extractor invents a false fact, you put it into the memory and from there it affects every subsequent call.

### How to choose

The practical rule:

- If the domain is very bounded and the relevant facts fit predictable formulaic patterns: **heuristic**.
- If the domain is open, multilingual or with high language variability: **LLM extractor**.
- If you have budget for both: use the LLM extractor with a subsequent heuristic validation that discards obviously wrong extractions (fields changing type, absurdly long values, internal contradictions).

For the estimator, the domain has a certain structure but the conversation is free and possibly bilingual. The balance tips slightly toward the LLM extractor — but the heuristic is completely defendable if you want to minimise cost and latency in this phase.

> *(Editor's note — checked against the code: the LLM extractor won. `app/sessions/metadata_extractor.py` exposes `update_metadata(...)`, calling a small model named by a `METADATA_EXTRACTOR_MODEL` setting — its docstring says the call is kept "narrow on purpose" — and it uses structured output (`response_model=ProjectMetadata`) rather than the raw `json_object` mode plus manual validation shown here. No heuristic path was implemented alongside it.)*

## 5. The history-management strategy returns

A quick reminder from the session 02 article: there are three canonical strategies for managing the history when it grows.

| Strategy | How it works | When to choose it |
|---|---|---|
| **Sliding window** | Keep the last N turns, discard the oldest | Short conversations, or when the relevant facts are already in `project_metadata` |
| **Cumulative summary** | When the history grows, summarise the old turns into a single compact message | Long conversations where the nuances of the original language matter |
| **Hybrid with anchors** | Cumulative summary of the oldest + a window of the last N + critical turns that are never discarded | Serious production, with conversations stretching over days or weeks |

What changes now relative to session 02 is that **the decision about which strategy to use becomes less critical when you have `project_metadata` separate.** The simple sliding window stops being risky because the facts that matter are not lost when a turn falls out of the window — the facts live in the memory, which is independent.

This is the most practical consequence of separating memory and history: **it lets you use the simplest history-management strategy without suffering the classic consequences.** A sliding window with `MAX_TURNS = 6` and `project_metadata` updated per turn is a completely reasonable architecture for initial production.

> *(Editor's note — checked against the code: the estimator uses `MAX_TURNS = 6` exactly, but not the simple sliding window this paragraph settles for. `ConversationHistory` carries `max_turns=6` **plus `anchors: list[Message]` and `summary: str | None`** — row three of the table, hybrid with anchors, with a `compression/` package holding `policy.py`, `anchors.py` and `summarizer.py`. So the implementation reached for the strategy the table reserves for "serious production" while the article argues the separation makes the simplest one safe. Both can be right; the article's claim is that you *may* choose row one, not that you must.)*

## 6. When to forget

A piece that is systematically underestimated: **systems need explicit policies for forgetting.** Without forgetting policies, the memory grows uncontrolled, old facts contaminate new decisions, and the session ends up answering based on obsolete information.

Three policies the estimator must implement from the beginning:

**Policy 1 — Explicit revision by the user.** If the user says "we are not going to use Rails any more, we are going with Node", the memory must reflect it: `mentioned_technologies` updated and, if the structure allows it, `rejected_options` extended with "Rails". Both the heuristic and the LLM extractor must recognise this revision pattern.

**Policy 2 — Per-session TTL.** The complete session must have a reasonable lifetime. A session inactive for 24 hours is probably no longer the same business conversation: the user's context has changed, and resuming with the old memory can induce mistaken assumptions. A simple policy: any session with no activity for 24 hours is archived, and on resuming the user is offered a new session with inherited memory or a fresh start.

**Policy 3 — Explicit reset.** The user must be able to say "forget everything from before and let's start again" and get a clean session. This materialises as a `POST /sessions` endpoint creating a new session, leaving the previous one intact for auditing but out of the active flow.

Policy 1 lives inside the memory-update logic. Policy 2 lives in the session's life cycle (a scheduled job archiving expired sessions). Policy 3 is an ordinary REST endpoint.

> *(Editor's note — checked against the code: one of the three is implemented. Policy 3 exists as `POST /sessions` in `app/routers/sessions.py`. **Policy 2 does not**: there is no TTL, no archiving job, and `Session` has no `updated_at` to measure inactivity from — the field this article's own schema includes and the implementation drops. Policy 1 is partial at best, since `rejected_options` is absent from `ProjectMetadata`, so a revision can update the technology list but cannot record what was rejected. The `Session` docstring is candid about the wider gap: "Lives in memory of the FastAPI process. State is lost on restart, which is intentional for the exercise.")*

**Three policies, three different locations in the architecture.** The temptation is to treat the whole forgetting problem as a single piece; the reality is that they are three independent mechanisms best kept separate.

## 7. Persistence: what does not enter yet

A natural question at this point: where does all this state live?

In a mature architecture, sessions live in a persistent database — Redis for fast access, PostgreSQL for long-term auditing, or both — managed by the business backend. The AI service receives the `session_id` on each request and loads the corresponding state. This allows continuity across service restarts, horizontal distribution and historical traceability.

In the programme's current phase, however, the reasonable choice is keeping the sessions in an in-memory dictionary in the AI service's process. It is deliberately simple, does not scale beyond a single process, and is lost with every restart. It is accepted because persistence and session federation are topics belonging to the deployment and production module, not to the CAG architecture phase.

What matters architecturally is that **the separation between `history` and `project_metadata` we are defending survives intact when persistence is introduced.** The two components serialise and load independently. Migrating from an in-memory dict to Redis requires refactoring nothing of the model; only the storage backend changes.

## 8. Frequent anti-patterns

Three errors seen repeatedly in conversational CAG systems that this architecture prevents.

**Anti-pattern 1 — Memory in the system prompt as a free-form string.** "The project is called X, the team is 3 people, the technologies are…" as a plain text block inside the system prompt, updated by hand. It works on the first turn. By turn 20 the block is full of inconsistencies and nobody knows how it got there. A typed structure with explicit fields is longer to write but infinitely more maintainable.

**Anti-pattern 2 — Trusting that the LLM "will remember".** Reasoning of the type "the LLM already read that the project is called X on turn 3, no need to repeat it on every call". It is false. **The LLM has no state between calls.** If turn 3 has fallen outside the sliding window, that fact disappears unless it lives explicitly somewhere else. Explicit memory is not redundancy; it is the only way for the fact to survive.

**Anti-pattern 3 — Mixing memory and history in a single structure.** "I am going to store the whole raw history in the database and also store the extracted facts in the same blob so as not to have two tables". It happens all the time, especially in small projects where the separation looks like over-engineering. The bill arrives when you need to truncate the history without touching the memory, or when a change in a fact's schema forces you to migrate old conversations. Two structures from the start come cheap compared with that migration.

## 9. Summary

The core of this piece is four operational claims worth travelling to your mental model:

1. **History and memory are two distinct structures with distinct responsibilities.** The history is the message array; the memory is the set of distilled facts about the domain.
2. **The memory survives the history's truncation.** That is the property making the system keep coherence in long conversations without paying the cost of a growing context.
3. **The memory materialises as a typed structure (Pydantic)** injected into the system prompt via a template and updated after each turn. The two ways of updating (simple heuristic, LLM extractor) are both valid and are chosen according to context.
4. **Forgetting needs explicit policies.** Three distinct mechanisms: user revision, session TTL, explicit reset. Each in its place.

Applying these four claims turns a conversation that "keeps forgetting things" into a system that maintains coherence about the domain while controlling cost and latency. It is the difference between a functional chat and a product tool.
