---
title: Hallucination detection and mitigation
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 11
series_part: 4
scope: evergreen
source: user-supplied article
reading_time: 21 min
added: 2026-08-24
summary: >
  Referential integrity proves the cited source exists, not that it says what
  the claim says — a hallucination with an alibi. Three kinds, each caught by a
  different technique: fabrication and unsupported extrapolation fall to a
  deterministic numeric anchor costing no tokens, false attribution needs a
  semantic judge. Layer them cheap-first, because what an `if` can reject should
  not spend a model call. The judge is itself a model and has a reliability
  floor no amount of further LLM can raise. Abstention is the honest exit and a
  feature, not a failure — but over-abstaining makes the system useless, which
  is not prudence but declining to do the work.
keywords: [hallucination detection, fabrication, false attribution,
           extrapolation, numeric grounding, semantic verification, LLM judge,
           self-consistency, dispersion, verification funnel, abstention,
           insufficient status, confidence degradation, prevention in prompt]
---

# Hallucination detection and mitigation

*Antonio Perez* · 🔴 21 min

We arrive at the point where the estimate looks impeccable. Every component carries its figure, every figure its citation, and the citations resolve to real budgets that were in the retrieved context. Referential integrity is guaranteed: no source points into the void. And yet, **the system may be lying.**

Because the fact that `fin-2024-07#c3` exists and resolves to a real budget says nothing about whether that budget contains "40h for payments". The model could have cited, with a perfectly valid identifier, a fragment that actually talks about authentication. It could have attributed to that source a figure that does not appear in it. Or it could have invented the number outright and hung on it the most plausible citation it had to hand. The citation is impeccable. The claim is false. **This is a hallucination with an alibi**, and it is the most dangerous precisely because it has passed every structural filter.

Detecting it is no longer checking identifiers. It is checking that the source's content genuinely sustains what the estimate says. That is the system's last layer of trust, and the hardest to build, because it demands verifying meaning, not form.

## Three ways to hallucinate in an estimate

Not all hallucinations are alike, and distinguishing them matters because they are detected with different techniques.

**Fabrication** is the pure case: a figure that appears in no source. The model writes "40h for payments" and none of the retrieved fragments contains that number for that component. It invented it, sometimes because it "sounds reasonable", sometimes because it fills a gap it did not know how to leave empty. It is the easiest to detect if you have the sources' figures to hand.

**False attribution** is subtler: the figure exists, but not in the source it is assigned to. Perhaps the 40h really is in some budget, but the cited `chunk_id` corresponds to another component or another project. Or two sources say 40h and 90h, and the model presents 40h citing the fragment that actually said 90. **The figure is real; the traceability is a lie.** And because the citation resolves, the structural filters do not see it.

**Unsupported extrapolation** is the hardest to bound: the model reasons beyond what the sources support. The sources give the payments module at 40h and 90h, and the model concludes that "a complex payments module with anti-fraud will be around 160h" — a figure that is nowhere and that derives not from the data but from a generalisation that sounds expert. Here there is no wrong source to point at; there is a logical leap nobody asked for.

> *(Figure 10 in the original: `art4-fig10-tres-alucinaciones.jpg` — image not included in this repo. Three panels. **Fabrication**: "40h" with a dashed arrow to an empty box "(no source)", noted "figure appearing in no budget" — hunted by *numeric anchoring*. **False attribution**: "40h" curving to a red card `fin-2024-07#c3` "talks about authentication", noted "real figure, wrong source" — hunted by the *semantic judge*. **Extrapolation**: "160h" marked "outside [40–90]", leaping from two boxes reading 40h and 90h, noted "unfounded jump" — hunted by *anchoring (out of range)*. Caption: "Each type of hallucination is detected by a different technique; none sounds absurd, and that is the problem.")*

The three share an uncomfortable property: **they produce plausible outputs.** A hallucination that sounded absurd would not be a problem. The problem is exactly that they sound right.

## Detect: cheap first, expensive afterwards

Verification costs — in latency, in calls, in money — so it is not all applied to everything. The strategy that works is layered: first deterministic checks costing almost nothing that catch the flagrant cases, and only over what survives, expensive semantic verification. **What an `if` can discard should not spend a model call.**

> *(Figure 11 in the original: `art4-fig11-embudo-verificacion.jpg` — image not included in this repo. A funnel from "all generated lines" down through three stages, with a cost-per-line axis running cheap to expensive alongside. **1 · Numeric anchoring** (deterministic, no call, cheap) drops out "fabrication + extrapolation". **2 · Semantic judge** (cheap model call) drops out "false attribution". **3 · Consistency** (N generations, critical lines only) sits at the expensive end. Output: "verified lines". Caption: "Anchor everything; judge what the anchor does not settle; consistency only on what is critical.")*

### Numeric anchoring: the check that costs no call

The first layer is purely arithmetic. Every figure in the estimate should be traceable to the figures of the cited sources. If the estimate says 40h citing two fragments, and those fragments say 40h and 55h, the 40 is anchored. If it says 160h and the cited fragments say 40 and 90, the 160 is outside the range of everything it cites: it is an extrapolation, and it is flagged.

```python
def numeric_grounding(line: SynthesizedComponent, evidence_by_id: dict[str, BudgetEvidence]) -> bool:
    """Cheap, deterministic check: is the claimed figure traceable to cited sources?

    Interpolation within the cited range is allowed; a figure outside the
    range of every cited source is an unsupported extrapolation.
    """
    cited_hours = [
        evidence_by_id[cid].hours
        for cid in line.source_chunk_ids
        if cid in evidence_by_id and evidence_by_id[cid].hours is not None
    ]
    if not cited_hours:
        return False  # no numeric support at all -> fabrication
    return min(cited_hours) <= line.low_hours and line.high_hours <= max(cited_hours)
```

The design decision behind that `min <= ... <= max` deserves explaining. **Allowing interpolation within the cited range is deliberate**: if the sources give 40 and 90, a 65 is a defendable mixture, not an invention. What gets flagged is extrapolation outside the range, which is where the model stops combining data and starts inventing it. If your domain is stricter and you only accept figures appearing literally in a source, harden the condition; but for estimation, where combining is legitimate, the range is the right anchor. This layer alone already catches pure fabrication (no numeric support) and extrapolation (out of range) without spending a token.

### Semantic verification: the judge, with its limits

Numeric anchoring does not see false attribution: a 40h citing a fragment that talks about something else passes the arithmetic check if the 40 happens to be in range. For that you need to look at meaning, and there a model-based verifier comes in, instructed to be strict and to doubt in favour of "not supported".

```python
class ClaimVerdict(BaseModel):
    component: str
    supported: bool
    reason: str
    confidence: float


VERIFY_INSTRUCTIONS = """\
You are a strict verifier of software estimate lines against their cited sources.
A claim is SUPPORTED only if the cited sources actually mention this component
and a figure consistent with the claim. A number present in no cited source is
NOT supported. Attributing a figure to a source that discusses a different
component is NOT supported. Do not be charitable: when in doubt, return supported=false.
"""


def verify_claim(line: SynthesizedComponent, cited_evidence: list[BudgetEvidence]) -> ClaimVerdict:
    response = client.responses.parse(
        model=settings.verifier_model,  # a cheaper, separate model from the generator
        input=[
            {"role": "system", "content": VERIFY_INSTRUCTIONS},
            {"role": "user", "content": render_verification_input(line, cited_evidence)},
        ],
        text_format=ClaimVerdict,
    )
    return response.output_parsed
```

Here one has to be honest about what this is and what it is not. **Using a model to detect another model's hallucinations is circular**: the verifier can also hallucinate, and can declare supported what is not. It does not eliminate the risk; it reduces it, and it reduces it in three concrete ways. First, **a narrow verdict schema** — `supported` is a boolean, not free prose in which to hide ambiguity. Second, **using a different and cheaper model than the generator**, so the two do not share exactly the same blind spots. Third, and most important, **instructing the verifier to doubt against**: when in doubt, not supported. **A permissive verifier is worse than no verifier**, because it gives a false sense of security.

> *(Editor's note — the codebase's nearest equivalent, `agentic/critic.py`, is stricter in shape and more permissive in failure. `CriticFeedback` carries `verdict`, `issues` and `confidence_in_review`, with a validator requiring at least one critical or major issue before `needs_iteration` — a narrower contract than `ClaimVerdict`'s bare boolean. But when the critic call itself errors it returns a synthetic *accept with zero confidence* rather than failing closed. The zero confidence is what keeps that honest; if any consumer reads `verdict` without reading `confidence_in_review`, the fail-open becomes exactly the permissive verifier this paragraph warns about.)*

### Consistency: not cheap, but sometimes worth it

There is a third signal, independent of the sources: **stability**. If you ask for the same figure several times and the model returns 40, 42 and 38, it is leaning on something; if it returns 40, 110 and 70, it is guessing. Dispersion across samples is an indicator of how much the model is making up.

```python
def consistency_spread(transcript: str, component: str, n: int = 3) -> float:
    """Regenerate a single figure n times; high variance suggests guessing.

    Expensive (n generations). Reserve it for high-stakes or low-confidence
    lines, not for every figure in every estimate.
    """
    samples = [generate_single_figure(transcript, component) for _ in range(n)]
    mean = statistics.mean(samples)
    return statistics.pstdev(samples) / mean if mean else 0.0
```

The comment in the code is not optional: consistency costs N generations per figure, so applying it to everything is unviable. It is reserved for high-impact or low-confidence lines, where the cost of being wrong justifies that of checking. And it has a conceptual trap to keep firmly in mind: **consistency confuses "the model is guessing" with "the data genuinely disagree".** A component that really runs from 40 to 90 hours depending on scope will produce dispersed samples, and that is not a hallucination: it is honest uncertainty, and the range is the correct answer. **Do not punish legitimate doubt as though it were invention.** Dispersion is suspicious when the sources agree and the model does not; not when the sources already disagreed.

> *(Editor's note — checked against the code, where this warning has already been half-ignored. `numeric_grounding`, `verify_claim`, `gate_line` and `consistency_spread` are **not implemented**, but `rag/task_hours.py` computes the identical coefficient of variation — `statistics.pstdev(hours_values) / mean_hours` — over the *neighbour source hours* rather than over resampled generations, and folds it straight into a confidence score: `reliability = weighted_similarity * (1 - min(dispersion, 1))`. That is dispersion among sources lowering the system's stated reliability, which is the penalty this paragraph says not to apply and which [s11-02](s11-02-synthesising-contradictory-budgets.md) argues is the most valuable signal the data carries. Defensible as a confidence signal rather than a hallucination verdict — a wide range genuinely is less actionable — but worth deciding deliberately, because as written the system is least confident exactly where s11-02 says it is most informative.)*

## Mitigate: prevent, validate, abstain

Detecting is half. The other half is what to do, and it starts before generating.

**Preventing** is the cheapest and the most effective. The generator's instructions must explicitly forbid what you do not want: use only figures present in the evidence, mark a component as unfounded rather than inventing it, do not extrapolate beyond the sources. The structured schema helps, because a `grounded` field the model has to fill forces it to take a position on every figure. Prevention does not eliminate hallucinations — no prompt does — but it reduces the volume reaching detection, and that makes everything else cheaper.

**Validating** is the post-generation net that combines the three previous signals into a per-line decision. And the decision is not binary: it is graduated, from more to less strict according to what fails.

```python
class VerifiedLine(BaseModel):
    component: str
    low_hours: float | None
    high_hours: float | None
    status: Literal["grounded", "insufficient", "rejected"]
    confidence: float


def gate_line(
    line: SynthesizedComponent,
    evidence_by_id: dict[str, BudgetEvidence],
    verdict: ClaimVerdict,
) -> VerifiedLine:
    anchored = numeric_grounding(line, evidence_by_id)

    if anchored and verdict.supported:
        return VerifiedLine(
            component=line.component,
            low_hours=line.low_hours,
            high_hours=line.high_hours,
            status="grounded",
            confidence=verdict.confidence,
        )

    if not anchored and not verdict.supported:
        # No numeric anchor and the verifier rejects it: do not emit a figure.
        log.warning("ungrounded_line_dropped", component=line.component)
        return VerifiedLine(
            component=line.component,
            low_hours=None,
            high_hours=None,
            status="insufficient",
            confidence=0.0,
        )

    # Mixed signals: keep the figure but degrade confidence and flag for review.
    log.info("line_degraded", component=line.component, anchored=anchored, supported=verdict.supported)
    return VerifiedLine(
        component=line.component,
        low_hours=line.low_hours,
        high_hours=line.high_hours,
        status="grounded",
        confidence=min(verdict.confidence, 0.4),
    )
```

> *(Figure 12 in the original: `art4-fig12-gate-line-matriz.jpg` — image not included in this repo. A 2×2 of *anchored: yes/no* against *judge: supports / does not support*. Both yes → green **grounded**, "figure + high confidence". Anchored but unsupported, and supported but unanchored → amber **grounded**, "degraded confidence + review flag". Neither → red **insufficient**, "no figure · abstain". Caption: "The decision is not binary: it abstains only when neither signal sustains the figure.")*

> *(Editor's note: `"rejected"` is declared in the `status` Literal and never returned — the three branches yield `grounded`, `insufficient` and `grounded`. Figure 12 agrees with the code, showing three outcomes over four quadrants, so the dead value is in the type rather than in the design. Drop it from the Literal, or decide what distinguishes a rejected line from an insufficient one and give it a branch.)*

**Abstaining** is the honest exit when a line does not hold up. `status="insufficient"` is not a system failure: **it is the system doing the right thing.** For a module with no comparable data, saying "I do not have enough budgets to estimate this reliably" is infinitely more valuable than inventing a number somebody will use to commit to a deadline. Abstention turns a confident lie into a useful question: the project manager now knows which part of the estimate needs a human expert, instead of discovering it when the deadline is missed.

> *(Editor's note: abstention is implemented, and as an invariant rather than a convention. `validation.check_coherence` enforces that an estimate whose `confidence` is `"insufficient"` must have `total_engineer_days` and `duration_weeks` both `None`, no `modules`, and a non-empty `insufficient_context_explanation` — so a system cannot claim it abstained while still shipping numbers. That is the structural counterpart to this section's argument.)*

## Honest trade-offs

**Detection adds latency and cost, and is never complete.** The three layers together can double the response time and multiply the calls. That is why they are ordered cheap to expensive and applied selectively: numeric anchoring to everything, the judge to what the anchor does not settle, consistency only to what is critical. Even so, **assuming you will detect every hallucination is itself a hallucination.** The objective is reducing the rate to a level acceptable for the domain's risk, not reaching zero.

**The judge is a model, and models hallucinate.** Detecting hallucinations with an LLM has a reliability floor you cannot beat by adding more LLM. The narrow schema, the different model and the bias toward "not supported" lower the risk, but a verifier is not a guarantee: it is another probabilistic layer. When the cost of an error is very high, the last verification is still a human.

**Consistency punishes honest uncertainty if you do not calibrate it.** It is the easiest trap to step in: flagging as a hallucination a component that legitimately has a wide range. Dispersion is only a signal of invention when it contradicts sources that agreed. If the sources already disagreed, the dispersion is the truth, not the error.

**Over-abstaining makes the system useless.** An estimator answering "insufficient data" to half the components is used by nobody. Abstention has to be calibrated to the right threshold: abstain when something is genuinely unfounded, not every time there is a shred of doubt. For estimation, a false "grounded" (a confident lie) is usually more expensive than a false "insufficient" (lost utility), but both have a cost, and pushing the threshold toward total abstention is not prudence: **it is declining to do the work.**

**Preventing in the prompt has diminishing returns.** Each new anti-hallucination instruction helps less than the last, and a prompt overloaded with prohibitions starts to degrade the generation's general quality. Prevention reduces the volume, but post-generation detection is what really sustains the guarantee. **Do not try to solve in the prompt what belongs to verification.**

## What this leaves unresolved

With this, the system verifies every estimate it produces: it anchors the figures to the sources, checks the meaning with a strict judge, measures consistency where it matters, and abstains instead of inventing when there is no foundation. For a concrete estimate, it knows line by line whether it can be trusted.

But all these checks look at **one answer at a time.** They tell you whether *this* estimate is well founded; they do not tell you whether your *system* is getting better or worse. If tomorrow you change the generation prompt, or the verifier model, or the way the context is assembled — does the hallucination rate go up or down? Does the average faithfulness of the answers improve, or have you just introduced a silent regression you will only see when a client complains? **Per-request verification is a guardrail: it stops a bad answer getting out. It is not a measurement:** it does not tell you how good the system is as a whole, nor whether a design decision made it better or worse.

Knowing that — measuring generation quality systematically, over a representative set, with numbers you can compare between versions — is a different discipline. Without it, every change to the system is a blind bet, however good the per-response guardrails are.
