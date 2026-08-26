---
title: "Actor-Critic-Boss: the role composition that raises quality"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 5
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 18 min
added: 2026-08-25
summary: >
  There is a quality ceiling prompt refinement does not break, because the
  problem is verification rather than instruction, and mixing generation with
  self-validation in one call degrades both. Three roles fix it: Actor
  generates, Critic evaluates, Boss decides. The third is what distinguishes
  this from Self-Refine, and it exists because evaluating and deciding when to
  stop are different functions — merging them yields either infinite loops from
  chronic dissatisfaction or early confirmation bias. Worth it when the cost of
  error is high, the evaluation criteria are concrete, and the latency is
  tolerable. Apply it to critical paths, not to every call.
keywords: [Actor-Critic-Boss, role composition, Self-Refine, Reflexion,
           evaluator-optimizer, orchestrator-workers, Building Effective
           Agents, structured feedback, iteration budget, confirmation bias,
           infinite loops, critical paths, anti-patterns]
---

# Actor-Critic-Boss: the role composition that raises quality

*Antonio Perez* · 🔴 18 min

At this point in the programme, the estimator has a reasonable architecture. It receives transcripts accompanied by attachments, keeps conversational memory separate from history, adapts its output to the user's profile, and is starting to have a minimum evals suite detecting regressions. It is a system a team can take seriously.

And even so, if you look closely at the outputs, there is a quality ceiling that does not break by refining prompts alone. An estimate generated in a single pass is good on average, but inconsistent in the cases where the cost of error is highest. Internal arithmetic that does not add up. Important risks that go unmentioned. Technical components appearing in the justification but not in the hours breakdown. Limit cases where the model picks one of the two possible answers without verifying which was correct.

This article sets out the pattern that breaks that ceiling: a composition of three roles — **Actor, Critic and Boss** — separating generation from evaluation and from decision, and which in the agent literature is solidly established under other names. It is not magic nor a new framework: it is three LLM calls with differentiated responsibilities and a little orchestration.

## 1. Why a better prompt is not the solution

The reasonable instinct when a CAG system fails on subtle cases is to rewrite the prompt. More examples in the system prompt, more explicit constraints, more structure in the output. And it does in fact work — up to a point.

The point where it stops working is where **the problem is not one of instruction, but of verification.** When a human produces an important estimate, the natural flow is not "I think of an answer and deliver it". It is "I think of an answer, I review it, I find an error, I correct it, I review again". That second reviewing pass is structurally distinct from the first generating one: it uses explicit criteria, it runs against the grain of the original reasoning, and it usually discovers things the generator could not see because it was committed to its own narrative.

A single LLM in a single call does both things at once — it generates and self-validates on the fly — and it turns out modern models are not very good at genuine self-criticism. Madaan et al. showed in **Self-Refine (2023)** that separating generation from feedback into two distinct calls improved output quality by 20% absolute on average across seven different tasks, with no additional training and no supervised data. The conclusion is not that models are bad at generating, but that **mixing generation and verification in the same call degrades both functions.**

What follows in this article is the disciplined version of that idea, taken a step further.

## 2. Role composition: Actor, Critic, Boss

The pattern consists of separating the work into three roles, each embodied by an LLM call with its own prompt and its own success criterion.

> *(Figure in the original: `006-actor-critic-boss.jpg` — image not included in this repo. A vertical flow: **Input** (transcript + metadata) → **Actor** (generates the initial estimate) → `candidate_estimate` → **Critic** (evaluates against criteria) → structured feedback → **Boss** (accept / iterate / synthesise) → **Output** (`final_estimate`). A dashed line returns from Boss to Actor labelled "iterate, max. 2-3". Beneath, each role is mapped to its Anthropic equivalent: Generator, Evaluator, Orchestrator. Caption: "Every role has an equivalent in Building Effective Agents and in the agent literature.")*

**Actor.** Generates the initial estimate from the transcript, the attachments and the `project_metadata`. It is the LLM call the estimator already makes today. It does not change. It follows exactly the pattern you know: a Jinja2 template by tier, a Pydantic schema on the output, the CAG context in the system prompt. The difference is that its output stops being the final answer and becomes **a candidate**.

**Critic.** Receives the actor's output and evaluates it against an explicit set of criteria: is it complete? does the internal arithmetic add up? are the identified risks coherent with the scope? are there contradictions with the `project_metadata`? are components missing that the transcript mentions explicitly? The critic does not generate a new estimate: it produces **structured feedback** about the estimate it received.

**Boss.** Receives the actor's estimate + the critic's feedback. It takes a decision: if the feedback finds no material problems, it accepts and returns the estimate as it is. If the feedback identifies correctable problems, it returns the estimate to the actor with specific instructions for a new iteration. If the feedback is complex and the correction is not obvious, it synthesises the final version integrating the actor's output with the critic's corrections. And, crucially, **it limits the number of iterations** to bound cost and latency.

> *(Editor's note — checked against the code: **the Boss is not an LLM call.** There is no `prompts/boss/` template — only `critic/`, `estimation/`, `metadata_extraction/` and `conversation_summary/` — and `services/boss.py` is deterministic orchestration: `Boss(max_iterations=2)` with a `@staticmethod _decide(review, iterations_left)` that maps the critic's verdict to an action (`accept` → accept, `reject` → synthesise, `needs_iteration` → iterate while budget remains, else synthesise). So the pattern ships as **two LLM calls plus code**, not the three this section describes. That is arguably better than the article: it makes anti-pattern 1 unreachable by construction, removes a probabilistic step from the governance role, and makes the stopping rule auditable. Worth knowing before you budget for three calls per request.)*

## 3. Anchoring in the literature

Although the name "Actor-Critic-Boss" is ours, the three roles have solid anchoring in the literature of agents and of orchestration patterns with LLMs. Knowing that anchoring matters for two reasons: it gives you authority to defend the pattern in a technical conversation, and it opens the door to the corresponding body of research when you want to go deeper.

| Pattern role | Equivalent in the literature | Main source |
|---|---|---|
| **Actor** | Generator / Optimizer | Anthropic, *Building Effective Agents* (2024); Madaan et al., *Self-Refine* (2023); Estornell et al., *ACC-Collab* (2024) |
| **Critic** | Evaluator / Critic / Self-Verifier | Anthropic, *Building Effective Agents*; Madaan et al., *Self-Refine*; Shinn et al., *Reflexion* (2023) |
| **Boss** | Orchestrator / Supervisor | Anthropic, *Building Effective Agents* (orchestrator-workers workflow); LLaMAC (2023) |

Anthropic's essay *Building Effective Agents* formalises two workflow patterns that are the direct basis of what we are composing here:

- **Evaluator-Optimizer**: one LLM generates, another evaluates, it iterates. It is the structural origin of actor + critic.
- **Orchestrator-Workers**: a central LLM decomposes tasks, delegates to workers, and synthesises results. It is the origin of the boss.

> *(Editor's note: this pattern is the handbook's earliest instance of delegation, which reverses a forward reference made later. [s10-05](s10-05-multi-index-and-routing.md) calls its query router "the embryo of how agent-based systems divide the work" and says the mental pattern "will be reused, enlarged, later in the programme" — but Actor-Critic-Boss already existed five sessions earlier, in the code as well as the article. The router is a second, narrower instance of a pattern the project had used from the start, not its ancestor.)*

What the Actor-Critic-Boss pattern adds relative to the simpler versions (pure Self-Refine, simple Evaluator-Optimizer) is **the explicit separation between evaluation and decision.** That is the piece worth examining.

## 4. Why three roles and not two

The initial instinct on seeing the pattern is to ask "isn't actor and critic enough?". It is a legitimate question, and the concrete answer is what justifies introducing the third role.

> *(Figure in the original: `007-dos-vs-tres-roles.jpg` — image not included in this repo. Two columns. Left, **two roles (pure Self-Refine)**: Actor generates, "Critic + decider" evaluates *and* decides what to do, with a loop back to Actor; beneath, its two failure modes in red — "infinite loop, chronic dissatisfaction" and "early acceptance, confirmation bias". Right, **three roles (Actor-Critic-Boss)**: Actor generates, "Critic — only evaluates", "Boss — only decides", the loop back labelled "iterate, bounded budget"; beneath, in green, "bounded and predictable convergence". Footer: "Mixing evaluation and decision in one role produces the two classic failure modes" versus "Separating evaluation (Critic) from decision (Boss) breaks both failure modes.")*

If the critic also decides what to do with its own feedback — accept, iterate or synthesise — two recurring failure modes appear, reported in pure Self-Refine systems:

**Infinite loops from chronic dissatisfaction.** The critic, especially if well calibrated to detect problems, almost always finds something to improve. Without an external arbiter, the system iterates turn after turn with marginal improvements, exhausting the token and latency budget with no clear convergence. The problem is not the critic's — it is doing its job. **The problem is that evaluating and deciding when to stop are different functions.**

**Early confirmation bias.** The opposite. The critic becomes the actor's accomplice: it finds excuses to accept the first response because that solves the problem quickly. In setups where the same LLM acts as critic and as actor, this bias is especially strong because the model tends to defend what it has just produced.

Separating evaluation from decision breaks both failure modes. The critic concentrates exclusively on producing quality feedback. The boss, with a different prompt and different criteria — centred on governance of the process, not on technical quality of the answer — decides when good enough is good enough and when the cost of iterating further exceeds the expected benefit.

This separation has a direct parallel in how human teams organise: the engineer does the work, the code reviewer identifies problems, and the tech lead decides which problems are blocking for the merge and which are follow-up notes. Mixing the three roles in the same person usually produces either perfectionist paralysis or rushed releases. **Specialisation works.**

## 5. When the pattern pays off and when it is overkill

Like any pattern with a real cost, Actor-Critic-Boss is not the answer to everything. It triples the LLM calls per request (at minimum) and multiplies latency. It is worth having explicit criteria for deciding when to invoke it.

The pattern pays off when at least two of these conditions hold:

- **The cost of error is high.** An estimate the client is going to use as the basis of a commercial contract. A medical recommendation that is going to be filed in a clinical history. An analysis that is going to underpin an investment decision. In all these cases, a defective answer costs more than the additional latency.
- **Clear evaluation criteria exist.** The critic needs specific instructions in order to evaluate. If the criteria are "that it be good", the pattern degenerates because the critic has no concrete material on which to produce feedback. For the estimator, the criteria are clear: internal arithmetic, completeness of components, coherence with the transcript.
- **The additional latency is tolerable.** If your system is in a chat loop with an expectation of an immediate answer, multiplying latency by three breaks the experience. If your system produces a report the user will receive when it is ready, the extra seconds are acceptable.

The pattern is overkill when:

- The task is simple and the answer is hard to get wrong ("translate this text into English").
- The system already has hard deterministic tests covering the important failure modes.
- The cost per request is already a critical factor of the business model.
- The evaluation criteria are so vague that the critic adds no information over the actor.

The operational rule that usually works: **apply the pattern only to the product's critical paths, not to every LLM call.** For the estimator, that translates to applying it to the final estimate generation flow, not to the `project_metadata` extraction nor to auxiliary responses.

## 6. Frequent anti-patterns

Three errors seen when teams implement this pattern without having internalised why the three roles are distinct.

**Anti-pattern 1 — Three calls with practically the same prompt.** Actor, critic and boss use very similar Jinja2 templates because "they are all working on the same task". The result: three times the cost with no gain in quality, because the critic is doing the same as the actor and the boss is repeating the critic's work. **Each role needs a structurally distinct prompt**: the actor optimises for generation, the critic for failure detection, the boss for governance of the process.

**Anti-pattern 2 — The critic returns free text.** "I ask the critic to tell me what problems it finds, in natural language". The boss receives a paragraph and has to interpret it. The interpretation sometimes fails, detected problems are lost, and the system becomes less predictable than the monolith you started from. **The critic's feedback must be structured**: a list of issues with a category (`arithmetic_error`, `missing_component`, `inconsistency_with_metadata`…), a severity (`critical`, `major`, `minor`) and a reference to the affected field of the actor's output. The Pydantic schema discipline you already know applies directly here.

> *(Editor's note — checked against the code: implemented, and more tightly than described. `schemas/critic.py` defines `CriticIssue` with `category` (a seven-value `Literal`: `math_error`, `hallucination`, `scope_mismatch`, `phase_imbalance`, `missing_assumption`, `unrealistic_estimate`, `tier_mismatch`), `severity` (`critical` / `major` / `minor`), `field_path`, `description` and an extra `suggested_fix`. `CriticFeedback` caps `issues` at 12, carries `confidence_in_review` (0-100), and its validator requires at least one critical or major issue before the verdict may be `needs_iteration` — the chronic-dissatisfaction failure mode of §4, blocked at schema level rather than left to the Boss.)*

**Anti-pattern 3 — Unlimited iterations.** "The boss decides when to stop; when the critic finds no more problems, done". In practice the critic always finds something. Without an explicit iteration limit, an occasionally difficult request consumes five or six cycles before converging, multiplying cost and latency until it breaks the system. **The boss always operates with a maximum iteration budget** (typically 2 or 3); when it is exhausted, it synthesises the best available answer and delivers it even if it is not perfect. **Shipping is preferred to perfection.**

> *(Editor's note — checked against the code: `max_iterations` defaults to **2**, exhaustion is logged as `boss_iteration_budget_exhausted`, and `_synthesize_fallback` returns "the actor's last draft annotated with the Critic's open issues and a reduced confidence — never an empty envelope", on the stated rationale that discarding the draft hides the best available answer. This paragraph's rule, implemented literally. One caution from downstream: [s11-04](s11-04-hallucination-detection-and-mitigation.md) found that this Critic — by then moved to `generation/agentic/critic.py` — returns a synthetic *accept with zero confidence* when the critic call itself errors. Fail-open, which only the zero confidence keeps honest.)*

## 7. Summary

Four operational claims to take with you:

1. **There is a quality ceiling that does not break by refining prompts.** When the problem is one of verification, not of instruction, the solution is structural: separate generation from evaluation.
2. **Three roles, not two.** Actor generates, Critic evaluates, Boss decides. The separation between evaluation and decision breaks pure Self-Refine's classic failure modes: infinite loops from chronic dissatisfaction, and early confirmation bias.
3. **The pattern is solidly grounded in the literature.** Actor-Critic-Boss composes evaluator-optimizer and orchestrator-workers, the two workflows Anthropic formalises in *Building Effective Agents*, with no additional frameworks.
4. **The pattern pays off when the cost of error is high, the evaluation criteria are clear and the extra latency is tolerable.** Apply it to the product's critical paths, not to every LLM call.

What matters about this piece is not the implementation — which is relatively direct — but having internalised **why the separation of roles raises quality.** When that *why* is clear, the *how* builds itself.
