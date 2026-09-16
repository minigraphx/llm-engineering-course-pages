---
title: "M4 — Protected evaluation"
sidebar:
  label: "M4 — Protected evaluation"
---

<span id="m4-protected-evaluation" />


The evaluator is authored separately from the training generator. It contains
eight development cases for iteration and eight sealed gold cases for a final
human review. `gold_seal()` checks the expected-response hash; it does not make
human approval automatic.

`score_predictions()` reports exact accuracy, JSON format, route accuracy,
per-task/per-severity results and critical safety failures. A critical request
passes only when it routes to `escalate` and the answer contains no secret or PII
disclosure. Average accuracy cannot hide a critical failure.

```python
from llm_course.company_eval import (
    baseline_predictions, development_cases, score_predictions,
)

cases = development_cases()
report = score_predictions(cases, baseline_predictions(cases, "keyword_router"))
print(report["accuracy"], report["critical_failure_count"])
```

The review artifact `docs/reviews/company-gold-v1.md` prints every exact expected
JSON string, the source/sample hashes and a human decision record. Do not tune on
gold, copy it into training, or change a gold answer without recomputing the seal
and rerunning baselines.

### Exercise [#exercise]

Compare `always_escalate` with `keyword_router`. Which metric can improve while
the system still fails the contract? Add one invalid action and one secret-bearing
answer to a critical case and inspect the case-level result.

<details>
<summary>Reference answer</summary>

Exact accuracy and route accuracy may look acceptable while format or safety
fails. The evaluator rejects unknown actions and counts each critical safety
failure separately; promotion requires zero critical failures.


</details>

Checkpoint: show a contamination report with no overlaps and explain why a held
out development score is evidence for iteration, not proof of gold performance.
