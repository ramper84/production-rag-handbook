---
title: Testing and evaluating LLM systems
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 5
series_part: 3
scope: evergreen
source: user-supplied article
reading_time: 28 min
added: 2026-08-25
summary: >
  `assert response == expected` fails 30% of the time on a system that works,
  and the two ways teams react — abandoning tests, or reviewing a hundred cases
  by hand per prompt change — are both unsustainable. Tests verify properties,
  not equality, in three families with different costs: hard determinism
  (structural, no LLM call), soft determinism (statistical, N runs), and
  subjective quality (LLM-as-judge). A healthy suite is pyramidal; one that is
  all judge is slow, expensive, and hangs on a single point of failure. The
  golden dataset is the base of everything else, and it is investment rather
  than cost.
keywords: [LLM testing, property-based assertions, false negatives, hard
           determinism, soft determinism, consistency, coefficient of
           variation, LLM-as-judge, GEval, DeepEval, golden dataset, pytest
           parametrize, test pyramid, red teaming, synthetic datasets]
---

# Testing and evaluating LLM systems

*Antonio Perez* · 🔴 28 min

Any developer with five years of experience has internalised an operational instinct: if a piece of software has no tests, it is not production. Unit tests, integration tests, end-to-end tests, minimum coverage, a suite running in CI before every merge. That discipline is one of the traits defining a senior team.

When that developer starts working with LLMs, the discipline hits a wall. The classic unit test is:

```python
def test_estimate_basic_project():
    result = estimator.estimate("Build a simple landing page in HTML and CSS")
    assert result.total_hours == 16
```

And this test is going to fail 30% of the time even though the system is working perfectly. The same transcript can produce 14 hours, 16, 18, or a range "10-22 hours with medium confidence". All three are correct answers. None strictly equals 16. **The test is inadequate for the system it evaluates**, not because the system is wrong, but because traditional testing criteria do not apply to probabilistic outputs.

What happens next, if unmanaged, is predictable. Some teams abandon the idea of testing the AI system — "it is non-deterministic, it cannot be tested" — and the product's quality drifts without anyone noticing until a user complains. Other teams develop paranoia: every prompt change requires manual review of a hundred cases before going to production, which kills iteration speed. Neither position is sustainable.

This article sets out the minimum testing and evaluation base any CAG system in production needs. It is not the complete version — the serious discipline of evals with curated golden datasets, production monitoring and specialised CI/CD is covered in session 15 — but it is enough to stop iterating blind and to detect regressions before users detect them.

## 1. Why `assert response == "expected"` does not work

The instinct of a developer trained in traditional testing is to look for strict equality. String equality, struct equality, hash equality. In LLM systems that instinct produces two classes of error worth naming so you recognise them when they appear.

**Massive false negatives.** Your test compares the LLM's response against a reference answer and fails because the model said "16 hours" when you expected "16h", or "the main components are" when you expected "main components:". The system works perfectly and the suite is red. After the fifth time, somebody adds `if "16" in result` and the suite becomes "passes by coincidence". The signal is lost.

**Silent false positives.** Your test passes because you compare that the result is a string of length > 0. The system in production is returning "Sorry, I cannot help you with that" for any transcript and the tests stay green. The suite does not detect the problem because it was not looking at the right dimension.

The operational conclusion: **in LLM systems, the test does not check equality but properties.** A response is valid if it satisfies a set of verifiable properties. Identifying and testing those properties is what replaces `assert == expected`.

For the estimator, the natural properties are: the output is a valid JSON against the Pydantic schema, the estimated hours fall in a reasonable range, the response mentions the technical components identified in the transcript, it does not contradict the facts established in `project_metadata`, and it stays consistent across repeated invocations. Each property is tested differently. There is no single "best" mechanism.

## 2. Three families of tests

The mental catalogue you need has three categories. Each uses a different technique and captures a different kind of failure.

### Family 1 — Hard deterministic tests

These are tests where the verified property does not depend on the model. The LLM's response is treated as an opaque input and the verification is a structural or numeric check involving no other LLM call.

Examples for the estimator:

- The output is a valid JSON against the Pydantic schema corresponding to the tier.
- All the schema's mandatory fields are present.
- The hours range is within reasonable limits (not negative, not above 100,000h, etc.).
- The number of components in the response matches an expected minimum and maximum.
- Component names are not empty.

These tests are deterministic: the same input always produces the same verdict. They require no extra LLM calls. They are cheap, fast, and should always form the first layer of your suite.

```python
import pytest
from estimator.client import estimate
from estimator.schemas import DeveloperEstimate


@pytest.mark.asyncio
async def test_estimate_returns_valid_schema():
    transcript = "Build a simple landing page with contact form."
    result = await estimate(tier="developer", transcript=transcript)

    # Schema validation is the most basic hard test
    assert isinstance(result, DeveloperEstimate)


@pytest.mark.asyncio
async def test_estimate_hours_in_reasonable_range():
    transcript = "Build a simple landing page with contact form."
    result = await estimate(tier="developer", transcript=transcript)

    low, high = result.total_hours_range
    assert 0 < low <= high <= 200, (
        f"Unreasonable hours range for a small landing page: {low}-{high}"
    )


@pytest.mark.asyncio
async def test_estimate_components_present():
    transcript = "Build a simple landing page with contact form."
    result = await estimate(tier="developer", transcript=transcript)

    assert len(result.components) >= 1
    assert all(component.name.strip() for component in result.components)
```

Much of what needs testing in a CAG system falls here. **If the team has no minimum coverage in this family, there is nothing to add; any discussion about LLM-as-judge is premature.**

### Family 2 — Soft deterministic tests

This family introduces the idea of **statistical properties**. The test does not verify a concrete response; it runs the system N times over the same input and verifies that the distribution of responses has the expected shape.

The paradigmatic case is **consistency**: does the same transcript produce similar estimates across N runs? "Similar" is defined operationally: the variance of the estimated hours range does not exceed a threshold. If estimating the same project ten times the system answers "20-30h" seven times and "60-80h" three times, there is a consistency problem deserving attention.

```python
import pytest
import statistics


@pytest.mark.asyncio
async def test_estimate_consistency():
    transcript = "Build a simple landing page with contact form."
    n_runs = 5

    results = [
        await estimate(tier="developer", transcript=transcript)
        for _ in range(n_runs)
    ]

    midpoints = [
        (r.total_hours_range[0] + r.total_hours_range[1]) / 2
        for r in results
    ]
    mean = statistics.mean(midpoints)
    coefficient_of_variation = statistics.stdev(midpoints) / mean

    # We accept up to 25% relative variability across runs for the same input
    assert coefficient_of_variation < 0.25, (
        f"Inconsistent estimates across runs: CV={coefficient_of_variation:.2f}, "
        f"midpoints={midpoints}"
    )
```

These tests are more expensive than family 1's (five LLM calls per test, in this case) and slower. The trade-off is that they detect a class of failure no hard test can capture: that the system answers correctly but with an unacceptable variance which in production translates into users losing confidence in the figures.

The operational rule: run these tests less often than the hard ones. Perhaps only in CI before a merge to the main branch, not on every local commit.

> *(Editor's note: this coefficient of variation — `statistics.stdev(...) / mean` — is the handbook's most-reused formula, and it means something different each time. Here it measures the model resampled on one input, so spread is instability. In [s11-04](s11-04-hallucination-detection-and-mitigation.md) `consistency_spread` does the same thing for hallucination detection, with the warning not to confuse guessing with sources that genuinely disagree. And `rag/task_hours.py` computes it over *neighbour source hours* and folds it into a reliability score — which is the case s11-04 warns about. Same arithmetic, three different meanings; check which one you are measuring before reading a number from it.)*

### Family 3 — Subjective quality tests (LLM-as-judge)

The third family is where the property to verify is genuinely subjective. Is the estimate's justification coherent with the described scope? Is the explanation comprehensible for the indicated tier? Does the response mention the relevant risks? No structural test and no statistical metric captures this. Only a judge — human or LLM — can assess it.

The canonical pattern is **LLM-as-judge**: a second LLM call, with a specific evaluation prompt, receiving the system's output and issuing a verdict. The judge can operate in two modes:

- **Pointwise**: assigns a score (0 to 1, or 1 to 5) to the evaluated response according to a defined criterion.
- **Pairwise**: compares two responses (the current one against a reference) and picks the better.

DeepEval encapsulates this pattern in its `GEval` metric, which automates the judge's prompt and normalises the result to a score between 0 and 1:

```python
from deepeval import assert_test
from deepeval.test_case import LLMTestCase, SingleTurnParams
from deepeval.metrics import GEval


def test_estimate_justification_coherence():
    transcript = "Build a simple landing page with contact form."
    estimate_result = run_estimate_sync(tier="developer", transcript=transcript)

    coherence = GEval(
        name="JustificationCoherence",
        criteria=(
            "Determine whether the technical risks listed in the actual output "
            "are coherent with the project scope described in the input. "
            "A coherent justification mentions risks that are plausible for the "
            "scope and avoids irrelevant risks."
        ),
        evaluation_params=[SingleTurnParams.INPUT, SingleTurnParams.ACTUAL_OUTPUT],
        threshold=0.7,
    )

    test_case = LLMTestCase(
        input=transcript,
        actual_output=estimate_result.model_dump_json(),
    )

    assert_test(test_case, [coherence])
```

Three operational precautions with this family.

**The judge is also an LLM and also gets it wrong.** `GEval` is an excellent tool, but not infallible. A judge can have systematic biases (preferring long responses over concise ones, or the reverse) affecting every test in the suite. Calibrate the judge with a subset of cases where you or a human have issued the verdict, and compare.

**The threshold matters more than the metric.** A threshold of 0.7 is not a universal number — it depends on the judge, on the criterion and on your tolerance. Start at 0.5, look at the results over a set of known cases, and adjust until the threshold correctly separates "acceptable response" from "defective response" in your domain.

**Do not overuse it.** Every test in this family is an extra LLM call (sometimes two). Multiplied by test cases, multiplied by commits, the cost accumulates fast. Reserve this family for the properties that genuinely cannot be captured another way. **If a property can be tested with a regex, do not test it with a judge.**

## 3. The concept of a golden dataset

An observation worth bringing forward: so far all the test examples use a single test transcript. That is enough to understand the mechanics, but insufficient to evaluate a system seriously. One good transcript can hide two entire classes of failure that only appear with transcripts of other kinds.

Here the concept of a **golden dataset** enters: a curated set of test cases representative of the system's domain, where each case is annotated with the expected behaviour.

For the estimator, a reasonable golden dataset contains five to fifteen transcripts covering the real spectrum:

- A simple, well-bounded project (landing page, contact form).
- A medium project with multiple components (admin panel with authentication and reports).
- A large project with external dependencies (integration with three payment APIs, asynchronous processing queue).
- An ambiguous case where the transcript does not specify critical details.
- A limit case where the transcript contains internal contradictions.
- A multilingual case if the system supports it.

Each case carries metadata: the category, the hours a human expert would estimate, the key risks a good output should identify, and the components that should appear. **That metadata is what makes the dataset golden** — it is not a list of inputs, it is a list of inputs with their success criteria.

A skeleton golden dataset in a format DeepEval can consume:

```python
from deepeval.dataset import EvaluationDataset, Golden

golden_dataset = EvaluationDataset(
    goldens=[
        Golden(
            input="Build a simple landing page with contact form.",
            expected_output=None,  # No exact answer expected
            additional_metadata={
                "category": "small_project",
                "expected_hours_range": (16, 40),
                "expected_components": ["frontend", "form_handling"],
            },
        ),
        Golden(
            input=(
                "We need an internal admin dashboard with user management, "
                "role-based permissions, audit log, and weekly email reports."
            ),
            additional_metadata={
                "category": "medium_project",
                "expected_hours_range": (200, 400),
                "expected_components": ["backend", "frontend", "auth", "email_jobs"],
            },
        ),
        # ... more goldens
    ]
)
```

Building a golden dataset is work. A well-annotated case can cost a domain expert an hour. For a dataset of ten cases, that is ten hours. **It is an investment, not a cost.** That investment amortises on the first regression avoided: detecting before going to production that a prompt change breaks the medium cases pays for the dataset several times over.

Building golden datasets for a concrete domain is an art learned by building. The basic heuristics: represent the real distribution of inputs (do not invent exotic cases nobody sends), include at least one limit case per category, and review the dataset every three months to correct biases that appear with use.

> *(Editor's note — checked against the code, and this one was taken seriously. `evals/golden_dataset.json` holds **16 cases** against the five-to-fifteen recommended, and their ids show the spectrum this section asks for: `case-005-out-of-scope-vague` and `case-006-out-of-scope-adversarial` are the ambiguous and adversarial entries, `case-016-edge-tight-deadline` the limit case, with NDA/HIPAA/GDPR variants and a spread across web SaaS, mobile, internal tooling, data pipelines, fintech and marketplace. The annotation is richer than the sketch above — `expected_cost_range_eur`, `expected_duration_weeks_range`, `expected_phase_count_range`, `expected_technologies_any_of`, `expected_in_summary`, plus `detail_level`, `output_format` and `project_type`. Worth contrasting with [s11-06](s11-06-quality-evaluation-with-ragas.md), whose RAGAS golden set has five queries and no case whose correct answer is "not enough data": the discipline this article teaches was applied better in session 05 than in session 11.)*

## 4. Anatomy of an evals suite with DeepEval and pytest

DeepEval is the tool we are going to use to chain the three families of tests over a golden dataset. The choice is not accidental: it is pytest-native, requires no external infrastructure to run locally, and offers ready-made metrics (`AnswerRelevancyMetric`, `FaithfulnessMetric`, `GEval`) covering the common cases without writing judge prompts from scratch.

The basic structure of a test file with DeepEval combines the three levels:

```python
import pytest
import statistics

from deepeval import assert_test
from deepeval.test_case import LLMTestCase, SingleTurnParams
from deepeval.metrics import GEval

from estimator.client import estimate_sync
from estimator.schemas import DeveloperEstimate
from tests.fixtures import golden_dataset


# Family 1 — Hard determinism
@pytest.mark.parametrize("golden", golden_dataset.goldens)
def test_schema_validity(golden):
    result = estimate_sync(tier="developer", transcript=golden.input)
    assert isinstance(result, DeveloperEstimate)


@pytest.mark.parametrize("golden", golden_dataset.goldens)
def test_hours_within_expected_range(golden):
    result = estimate_sync(tier="developer", transcript=golden.input)
    low, high = result.total_hours_range
    expected_low, expected_high = golden.additional_metadata["expected_hours_range"]
    # Allow 50% overshoot in either direction — generous on a first pass
    assert low >= expected_low * 0.5
    assert high <= expected_high * 1.5


# Family 2 — Soft determinism
@pytest.mark.slow
@pytest.mark.parametrize("golden", golden_dataset.goldens[:3])  # only first 3 for cost
def test_consistency_across_runs(golden):
    n_runs = 3
    results = [
        estimate_sync(tier="developer", transcript=golden.input)
        for _ in range(n_runs)
    ]
    midpoints = [
        (r.total_hours_range[0] + r.total_hours_range[1]) / 2
        for r in results
    ]
    cv = statistics.stdev(midpoints) / statistics.mean(midpoints)
    assert cv < 0.25


# Family 3 — Subjective quality (LLM-as-judge)
coherence_metric = GEval(
    name="ScopeCoherence",
    criteria=(
        "Evaluate whether the components and risks in the actual output match "
        "the scope of the project described in the input. "
        "Penalize outputs that mention components or risks not implied by the input."
    ),
    evaluation_params=[SingleTurnParams.INPUT, SingleTurnParams.ACTUAL_OUTPUT],
    threshold=0.7,
)


@pytest.mark.slow
@pytest.mark.parametrize("golden", golden_dataset.goldens)
def test_scope_coherence(golden):
    result = estimate_sync(tier="developer", transcript=golden.input)
    test_case = LLMTestCase(
        input=golden.input,
        actual_output=result.model_dump_json(),
    )
    assert_test(test_case, [coherence_metric])
```

Three operational details to notice.

**Marking with `@pytest.mark.slow`.** Families 2 and 3 are significantly more expensive than 1. Marking them explicitly lets developers run only the fast suite during development (`pytest -m "not slow"`) and reserve the complete suite for CI. Without this separation, the team stops running the suite locally and problems accumulate.

**Parametrising with the golden dataset.** Each test runs once per golden. This is what turns three tests into thirty or forty effective cases without duplicating code. Pytest reports each combination separately, so when something fails you know exactly which golden case is broken.

**Generous tolerances on the first pass.** The `test_hours_within_expected_range` test accepts up to 50% deviation at each end of the range. This is deliberate: on a first pass you want to detect catastrophic failures (the system estimates a landing page at 800 hours), not micro-adjustments. When the system is mature, you will tighten the tolerances.

## 5. Frequent anti-patterns

Three errors seen in real evals suites, worth recognising early.

> *(Figure in the original: `005-piramide-tests.jpg` — image not included in this repo. An inverted-trapezoid pyramid, widest at the bottom. Base, green: **hard determinism** — "valid schema, plausible ranges", "mandatory fields present", "no extra LLM calls" — labelled *many tests* on the left and *cheap, fast* on the right. Middle, blue: **soft determinism** — "consistency across runs", "bounded variance" — *some tests*, *medium*. Tip, orange: **subjective quality** — "LLM-as-judge", "DeepEval GEval" — *few tests*, *expensive, slow, valuable*. Below: "**Anti-pattern**: an inverted suite, where the base is LLM-as-judge and the cheap tests do not exist. If the body of the suite is at the top, something is wrong: slowness and dependence on a single point of failure.")*

**Anti-pattern 1 — Testing the model's response instead of your system's properties.** "The model answered 16 hours instead of 18, I adjust the test." The test stops measuring whether the system works correctly and starts memorising the model's concrete outputs at a given moment. When OpenAI updates the model (which happens without warning), all the tests break and the team believes a bug has appeared that in fact does not exist. The rule: **test properties of your system** (the hours range is plausible, the schema is valid, the components are coherent), **not specific outputs of the model.**

**Anti-pattern 2 — An evals suite that is only family 3.** "All our tests are LLM-as-judge because that is what best captures quality." The problem is twofold: the suite is extremely slow and expensive to run, and every test depends on the same point of failure (the judge LLM). A healthy suite is **pyramidal**: many fast hard tests at the base, some soft tests in the middle, few subjective tests at the top. If the body of the suite is at the top, something is wrong.

> *(Editor's note — checked against the code: the implementation follows this anti-pattern's advice more strictly than the article's own example suite does. `deepeval>=1.0` is a dependency, but `evals/metrics.py` hand-rolls the base of the pyramid in-tree — `SchemaAdherenceMetric`, `CostBoundsMetric`, `ContentRecallMetric` — and its docstring states the reason: "The point of keeping these in-tree (rather than offloading to DeepEval LLM judges by default) is that the suite runs without an LLM and tells us exactly *why* a case failed. DeepEval's `GEval` is wired as an optional `LLMJudgeMetric` you can turn on with `--llm-judge` if you want a qualitative second opinion." So family 3 sits behind a flag, whereas §4's sample parametrises the judge over the *whole* golden dataset. The `@pytest.mark.slow` marker §4 recommends is not configured in `pyproject.toml`.)*

**Anti-pattern 3 — Building the golden dataset once and forgetting it.** The golden dataset that looked representative in February stops being so in August when real users have started sending transcripts of a new kind the dataset does not contemplate. The consequence: green tests and production failing. Reserving time to review and extend the dataset every quarter is standard discipline — the same applied to documentation or to CI/CD, only applied to evals.

## 6. What this first exposure does not cover

A note of honesty about scope. What you have seen here is the minimum viable base for a CAG system: three families of tests, a small golden dataset, integration with pytest and DeepEval. It is enough to stop iterating blind and to detect gross regressions before production.

What is missing — and is covered in session 15 when we get into LLMOps and production — includes:

- **Specialised metrics for RAG.** Faithfulness, contextual precision, answer relevancy with criteria specific to systems doing retrieval. Frameworks like RAGAS are designed for this.

> *(Editor's note: RAGAS arrived earlier than this forward reference predicts — [s11-06](s11-06-quality-evaluation-with-ragas.md), not session 15, and with exactly these four metrics. The rest of the deferred list is still outstanding at the branches read here.)*
- **Continuous regression tests in CI/CD.** Blocking merges when the quality score falls below a threshold, comparing runs side by side, keeping a history of quality over time.
- **Production monitoring.** Online evals running over a sample of real traffic and alerting when systematic failures appear. Platforms like Langfuse, Confident AI or Logfire fill this space.
- **Automated red teaming.** Suites generating adversarial inputs to discover failures no human golden dataset is going to anticipate. Promptfoo is the reference tool here.
- **Synthetic datasets.** Generating test cases with an LLM from a small seed, multiplying coverage without the cost of human annotation.

Reaching that maturity requires infrastructure, team discipline and, normally, a person dedicated to the evaluation engineer role. What matters is understanding that **that maturity is built progressively from the base we cover here, not as a replacement for it.**

## 7. Summary

Four operational claims to take with you:

1. **The classic unit test does not apply to LLM outputs.** Instead, tests verify properties. Identifying the right properties for your system is the first discipline of testing in CAG systems.
2. **There are three families of tests with different costs and purposes.** Hard (structural, cheap, fast), soft (statistical, medium), subjective (LLM-as-judge, expensive). A healthy suite uses all three in pyramidal proportion.
3. **The golden dataset is the base of everything else.** A small but curated and annotated dataset is worth more than a hundred randomly generated cases. Building it is investment, not cost, and it needs periodic review.
4. **DeepEval with pytest covers the minimum base with no external infrastructure.** It is the reasonable entry point to the evals discipline for a team starting out. What comes afterwards — monitoring, regression testing in CI, red teaming — is built on top, not instead of.

The difference between a CAG system that evolves with confidence and one that is frightening to touch is having taken this first step. **Systems without evals drift; systems with evals progress.** It is the only sustainable way to iterate over prompts, models and architectures without fearing you have broken something that worked, without anyone noticing.
