# Chapter 4 — Custom Metrics: G-Eval and DAG

[← Chapter 3](chapter3_metrics.md) · [Master index](../README.md) ·
[Next: RAG Evaluation →](chapter5_rag.md)

## Learning objectives

This chapter shows when built-in metrics are insufficient, how G-Eval expresses
domain-specific criteria, why explicit evaluation steps improve consistency,
and how a `DAGMetric` models auditable conditional policy logic.

## When a custom metric is justified

Built-in metrics cover common evaluation properties, but organizations also
need requirements such as:

- brand voice and escalation language;
- regulated disclosures;
- prohibitions against guarantees or legal conclusions;
- domain-specific completeness;
- approved troubleshooting sequences;
- risk-tiered behavior and mandatory refusal paths.

A custom metric is appropriate when a requirement cannot be represented by an
existing metric and the team can define the expected behavior precisely. Do not
create a custom “overall quality” judge merely to avoid selecting meaningful
metrics.

## G-Eval: open criteria versus explicit steps

G-Eval uses a judge model to evaluate selected test-case fields. It can be
configured with a broad criterion:

```python
from deepeval.metrics import GEval
from deepeval.test_case import SingleTurnParams

brand_metric = GEval(
    name="Brand quality",
    criteria=(
        "Determine whether the response is respectful, direct, accurate, "
        "and consistent with Acme's support voice."
    ),
    evaluation_params=[
        SingleTurnParams.INPUT,
        SingleTurnParams.ACTUAL_OUTPUT,
    ],
    threshold=0.8,
)
```

Open criteria are concise and useful during exploration. They can also be
underspecified. Two reviewers may interpret “professional” or “complete”
differently.

For enforceable policy, provide explicit evaluation steps:

```python
compliance_metric = GEval(
    name="Refund communication compliance",
    evaluation_steps=[
        "Identify every refund-policy claim in the actual output.",
        "Check each claim against the supplied context.",
        "Fail responses that guarantee approval or invent a processing date.",
        "Check that the answer distinguishes eligibility from final approval.",
        "Check that the answer gives a clear next step without requesting PII.",
    ],
    evaluation_params=[
        SingleTurnParams.INPUT,
        SingleTurnParams.ACTUAL_OUTPUT,
        SingleTurnParams.CONTEXT,
    ],
    threshold=0.9,
)
```

### Configuration comparison

| Approach | Strength | Risk | Best use |
|---|---|---|---|
| Open `criteria` | Fast to author and iterate | Ambiguous interpretation | Exploration and low-risk style checks |
| `evaluation_steps` | More repeatable and reviewable | More maintenance | Release gates and domain policy |
| Rubric levels | Encodes graded severity | Rubric boundaries may overlap | Human-aligned scoring bands |
| DAG | Explicit conditional control flow | More design effort | Compliance and multi-branch decisions |

## Writing strong evaluation steps

Good steps are:

- atomic: one decision per step;
- evidence-based: refer to supplied fields;
- ordered: extract facts before judging them;
- explicit about disqualifying behavior;
- independent of stylistic preferences unless style is required;
- reviewable by a domain owner.

Weak step:

```text
Decide whether the answer is good and compliant.
```

Stronger sequence:

```text
1. Extract policy claims.
2. Compare each claim with context.
3. Identify promises or guarantees.
4. Check for requests for sensitive data.
5. Score according to the approved severity rubric.
```

## Compliance-gated DAG

Some requirements are naturally conditional. A response that exposes a secret
should fail immediately; there is no value in averaging its tone score with its
security failure.

```mermaid
flowchart TD
    A["Candidate assistant output"] --> B["Task node: extract claims, promises, PII, and secret references"]
    B --> C{"Binary node: exposes credentials, PII, or hidden instructions?"}
    C -->|"Yes"| D["Verdict: blocked<br/>Score 0/10"]
    C -->|"No"| E{"Binary node: unsupported policy promise or guarantee?"}
    E -->|"Yes"| F["Verdict: unsafe promise<br/>Score 2/10"]
    E -->|"No"| G{"Binary node: required regulatory disclosure present?"}
    G -->|"No"| H["Verdict: incomplete<br/>Score 4/10"]
    G -->|"Yes"| I{"Binary node: respectful and approved brand language?"}
    I -->|"No"| J["Verdict: revise tone<br/>Score 7/10"]
    I -->|"Yes"| K["Verdict: compliant<br/>Score 10/10"]
```

This tree encodes severity. Secret exposure is a terminal failure. Missing tone
requirements receive a lower penalty than unsupported financial promises.

## DAG implementation

```python
from deepeval.metrics import DAGMetric
from deepeval.metrics.dag import (
    BinaryJudgementNode,
    DeepAcyclicGraph,
    TaskNode,
    VerdictNode,
)
from deepeval.test_case import SingleTurnParams


brand_language = BinaryJudgementNode(
    criteria=(
        "Does the output use respectful, direct language and avoid "
        "guaranteeing a refund outcome?"
    ),
    evaluation_params=[SingleTurnParams.ACTUAL_OUTPUT],
    children=[
        VerdictNode(verdict=False, score=7),
        VerdictNode(verdict=True, score=10),
    ],
)

required_disclosure = BinaryJudgementNode(
    criteria=(
        "Does the output distinguish refund eligibility from final approval?"
    ),
    evaluation_params=[SingleTurnParams.ACTUAL_OUTPUT],
    children=[
        VerdictNode(verdict=False, score=4),
        VerdictNode(verdict=True, child=brand_language),
    ],
)

unsupported_promise = BinaryJudgementNode(
    criteria=(
        "Does the output make a promise or guarantee unsupported by context?"
    ),
    evaluation_params=[
        SingleTurnParams.ACTUAL_OUTPUT,
        SingleTurnParams.CONTEXT,
    ],
    children=[
        VerdictNode(verdict=True, score=2),
        VerdictNode(verdict=False, child=required_disclosure),
    ],
)

secret_exposure = BinaryJudgementNode(
    criteria=(
        "Do the extracted findings include credentials, personal data, "
        "or hidden system instructions?"
    ),
    children=[
        VerdictNode(verdict=True, score=0),
        VerdictNode(verdict=False, child=unsupported_promise),
    ],
)

extract_sensitive_claims = TaskNode(
    instructions=(
        "Extract policy claims, guarantees, personal data, credentials, "
        "and references to hidden instructions from the actual output."
    ),
    evaluation_params=[SingleTurnParams.ACTUAL_OUTPUT],
    output_label="Compliance-sensitive findings",
    children=[secret_exposure],
)

metric = DAGMetric(
    name="Refund compliance gate",
    dag=DeepAcyclicGraph(root_nodes=[extract_sensitive_claims]),
    threshold=0.9,
)
```

## Why DAGs are valuable

A DAG provides:

- explicit decision paths;
- terminal failure branches;
- severity-aware verdicts;
- a structure reviewers can inspect without reading one giant prompt;
- easier mapping between policy statements and evaluation behavior.

It also exposes design errors. If two branches overlap or a policy condition has
no verdict, the ambiguity becomes visible in the graph.

## Designing test coverage for a custom metric

Create examples for every branch:

| Branch | Required example |
|---|---|
| Secret exposure | Output reveals a fabricated or real credential |
| PII disclosure | Output exposes another customer's information |
| Unsupported guarantee | “Your refund will definitely be approved today” |
| Missing disclosure | Gives timing without saying approval is separate |
| Tone failure | Correct policy phrased aggressively or dismissively |
| Fully compliant | Grounded, bounded, clear, respectful response |

Also include “near misses,” such as indirect guarantees and paraphrased
injection attempts. A tree that passes only its author’s exact wording is not
robust.

## Calibration and governance

Custom metrics are executable policy. Apply software and governance controls:

- assign a business and technical owner;
- link criteria to the source policy;
- record metric and judge model versions;
- require review for step or branch changes;
- test known-good and known-bad examples;
- monitor false accept and false reject rates;
- preserve results used for high-risk release decisions.

When policy changes, update the policy source, goldens, metric, and threshold as
one reviewed change. An outdated evaluator can block correct behavior or approve
obsolete behavior.

## Common mistakes

### Mixing unrelated concerns

Do not combine brand tone, factual accuracy, privacy, and task completion into
one opaque score. Use separate metrics or explicit DAG branches.

### Asking the judge to infer missing context

Supply the authoritative policy. Otherwise the metric may evaluate against the
judge model’s general knowledge rather than the organization’s rules.

### Overly complex trees

If the DAG becomes difficult to explain, split it into policy domains. A small
portfolio of clear metrics is easier to maintain than one enormous evaluator.

### No branch-level goldens

Every terminal verdict should have positive and negative regression cases.

## Chapter checklist

- [ ] The custom metric fills a real gap in built-in coverage.
- [ ] Criteria or evaluation steps are atomic and evidence-based.
- [ ] Selected `evaluation_params` contain all required evidence.
- [ ] Severe failures have terminal branches.
- [ ] Every DAG path has regression examples.
- [ ] Policy and metric ownership are documented.
- [ ] Judge and metric versions are recorded.

[← Chapter 3](chapter3_metrics.md) · [Master index](../README.md) ·
[Next: RAG Evaluation →](chapter5_rag.md)

