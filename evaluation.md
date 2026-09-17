---
title: "E03 — Evaluation: make a claim the evidence can support"
sidebar:
  label: "E03 — Evaluation"
---

<span id="e03-evaluation-make-a-claim-the-evidence-can-support" />


[← E02 — Data quality](/data-quality) · [E04 — Profiling & budgets →](/profiling) · [Glossary](/glossary)

Prerequisites: complete the [decoder](/mini-gpt) and [training](/pretraining)
labs. You should be able to identify logits, shifted labels, and an optimizer step.
Allow about 50 minutes. Everything below runs offline on a CPU.

By the end, you can aggregate held-out loss correctly, compare paired runs,
reject an unsupported improvement, and explain what a tiny behavioral probe misses.

## 1. Define the experiment before looking at results [#1-define-the-experiment-before-looking-at-results]

**Held-out data** is data excluded from parameter updates. **Validation data** is
held out but used to select settings. A **test set** is reserved for the final
comparison after those settings are fixed. If you repeatedly adjust settings after
reading test results, you have turned the test set into validation data.

This lab fixes two learning rates, 0.001 and 0.003, before running either arm.
Everything else is held constant: 30 optimizer steps, architecture, training order,
training and test sequences, decoding settings, and paired initialization seeds
7, 19, and 43. A **seed** initializes the pseudorandom generator. Pairing means
comparing the two settings with seed 7 to each other, then seed 19, then seed 43.

The primary experiment uses the existing Unit 0 synthetic course corpus and the
character encoder. The document split with seed 7 contains 12 training, two
validation, and two test documents. Training uses batches of four; each document
is truncated to 25 encoded tokens, producing at most 24 next-token targets. The
final metric evaluates 48 targets from the two test documents. Validation is
unused because both comparison arms were fixed in advance. The model has width
32, two decoder layers, four attention heads, and context capacity 24. Its
synthetic source manifest is `datasets/unit0/manifest.json`, with the accompanying
card at `docs/data-cards/unit0-synthetic-v1.md`.

A separate, smaller symbol-model experiment is stored under `symbol_task` in the
same report. It makes task answers and smoke-test behavior easy to inspect; its
metrics must never be compared directly with the character model's NLL.
The tiny dataset consists of seven rotations of the rule `0,1,2,3,4,5,6,0,...`.
Each document has nine symbols. Rotations beginning at 0–3 train the model;
rotations beginning at 4–6 are the frozen test set. Each nine-symbol document
produces eight input/target pairs. Training has 32 targets and the test has 24.
The test sequences are distinct, but their transitions occur in training. This is
an intentionally small test of rule completion, not a benchmark for language.

The generated JSON includes a **data card**: provenance, license scope, generator,
split definitions, counts, fingerprints, and limitations. A **fingerprint** is a
SHA-256 digest of the serialized tensors and protocol. It lets us detect that two
reports actually used different evaluation inputs. The data exists only in memory;
no downloaded corpus, personal data, or weights are required.

## 2. Count tokens, not batches [#2-count-tokens-not-batches]

**Negative log likelihood (NLL)** is `-log(p(correct next token))`, using natural
logarithms, so the unit is a **nat**. Lower is better. An unlikely correct token
costs more. If its probability is 0.5, its NLL is about 0.693.

**Perplexity** is `exp(mean NLL)`. It translates log loss to a multiplicative scale;
it is not an accuracy percentage. Always compare it with the same tokenizer,
vocabulary, labels, and evaluation protocol.

Suppose a short batch has one valid token with mean NLL 1, and a long batch has
three valid tokens with mean NLL 3. The correct combined NLL is
`(1*1 + 3*3)/(1+3) = 2.5`, so perplexity is about `12.1825`. Averaging batch means
gives 2 and unfairly lets the single token count as much as three tokens.
Padding labels equal `-100` and must contribute neither loss nor token count.

The implementation calls cross entropy with `reduction="sum"` for each batch,
adds those sums, then divides once by the number of valid labels. Its model input
has shape `[B,T]`, logits have shape `[B,T,V]`, and labels already contain the next
tokens. Evaluation must not shift them a second time. It temporarily disables
training behavior and gradients, then restores each module's previous mode.

## 3. Run all three seeds and read the report [#3-run-all-three-seeds-and-read-the-report]

From the repository root after the normal editable installation. For PowerShell,
use the [portable Python-file convention](/setup#shell-conventions) for the
multiline readout below:

```bash
python examples/run_evaluation.py --output artifacts/evaluation/report.json
python - <<'PY'
import json
from pathlib import Path
report = json.loads(Path("artifacts/evaluation/report.json").read_text())
for arm in ("baseline", "candidate"):
    for run in report[arm]:
        print(arm, run["seed"], round(run["nll"], 4),
              run["task_predictions"], run["generation_ids"])
print(report["comparison"])
print("unchanged claim:", report["no_change_control"]["claim_supported"])
PY
```

Primary course-corpus reference, PyTorch 2.14.0, Python 3.12.13:

| Seed | Baseline test NLL | Candidate test NLL | Reduction | Next-character task correct, baseline → candidate |
| --- | ---: | ---: | ---: | --- |
| 7 | 3.244165 | 2.974400 | 0.269765 | 0/3 → 0/3 |
| 19 | 3.456174 | 2.964097 | 0.492077 | 0/3 → 0/3 |
| 43 | 3.173621 | 2.866494 | 0.307127 | 1/3 → 1/3 |

This primary comparison has mean NLL reduction 0.356323 and a 95% paired interval
`[0.060585, 0.652061]`. Its next-character task uses prefixes of lengths 4, 8, and
12 from the first held-out test sequence. Correct token IDs are `[21,1,18]`.
Generation uses its four-token prefix and 16 sampled continuation tokens at seed
101. Candidate seed 7 produces `<bos>a tdtammf measutse `: lower NLL still leaves
unconvincing text and no improvement on the three argmax task questions.

For the symbol task, inspect `report["symbol_task"]` with the same keys.
Its reference is:

| Seed | Baseline NLL | Candidate NLL | NLL reduction | Task correct, baseline → candidate |
| --- | ---: | ---: | ---: | --- |
| 7 | 1.914761 | 1.340504 | 0.574257 | 0/3 → 3/3 |
| 19 | 1.791130 | 1.273541 | 0.517589 | 1/3 → 1/3 |
| 43 | 1.732459 | 1.316821 | 0.415639 | 0/3 → 2/3 |

Small numerical differences across software builds are possible. The report records
both architecture and complete training configuration; changing `--steps` creates a
different experiment and does not reproduce this table.

The symbol **task metric** uses three fixed prompts `[4]`, `[5]`, `[6]`, with expected
next tokens `[5,6,0]`. It takes the most likely next token, called **argmax**.
The **generation check** samples eight tokens from prompt `[4,5]` using temperature
1 and sampling seed 101. Temperature controls the sharpness of the probability
distribution. The report retains the prompt plus continuation. In the candidate
reference runs this was `[4,5,2,3,3,4,2,3,4,5]` for every seed: clearly imperfect
rule following, despite lower NLL. A sampled sequence and task accuracy answer
different questions from probability-sensitive NLL. None replaces the others.

## 4. Uncertainty and the unchanged control [#4-uncertainty-and-the-unchanged-control]

For each seed, compute `d = baseline NLL - candidate NLL`. Positive means improvement.
The three symbol-task differences above average 0.502495. The **sample standard deviation**
describes their spread; the **standard error**, `s/sqrt(n)`, describes uncertainty
in the estimated mean under independent seed runs.

We use a two-sided 95% **Student-t confidence interval**:
`mean(d) ± t * s(d)/sqrt(n)`. With three runs the degrees of freedom are `n-1=2`,
so the multiplier is 4.303, much wider than 1.96. The observed interval is
`[0.302806, 0.702184]` nats/token. The code accepts 3–31 paired seeds and checks
for duplicate or unmatched seeds and differing evaluation identities.

A claim passes only when the entire interval is above zero. Comparing the baseline
to itself gives `[0,0]` and `claim_supported: false`. A positive sample mean alone
is insufficient. For differences `[0.1,0.2,0.3]`, the interval crosses zero and the
claim must also be rejected. The formulas and paired-observation treatment follow
[NIST's paired-observation guide](https://itl.nist.gov/div898/handbook/prc/section3/prc311.htm)
and [confidence-limit guide](https://itl.nist.gov/div898/handbook/eda/section3/eda352.htm).

The interval assumes approximately normal independent paired differences. Three
seeds cannot diagnose that assumption well. Its uncertainty concerns training
seeds on this fixed dataset; it does not include new documents, changed domains,
human ratings, or repeated test-driven tuning. Under the assumptions, the method
covers the population mean in 95% of repeated experiments; it does not assign a
95% probability to the mean being inside this one already observed interval.

## 5. Smoke checks that actually read the model [#5-smoke-checks-that-actually-read-the-model]

A **smoke test** is a small check capable of catching a conspicuous defect. It
cannot establish general quality. The harness performs four scoped checks:

| Check | Actual evidence | Interpretation and missing coverage |
| --- | --- | --- |
| Leakage | Exact intersection of train/test document strings | Detects copied documents; misses paraphrases and shared subspans |
| Memorization warning | Greedy model continuation of training prefix `[0,1]` compared to `[2,3,4]` | A reproduced suffix may reflect either learning the rule or remembering it |
| Safety plumbing | Greedy continuation of `[4]`, flagging synthetic sentinel token 6 | Tests whether a configured forbidden-output detector works; token 6 is not real harmful content |
| Bias plumbing | Difference in probability of token 6 after `[0]` and `[3]`, threshold 0.1 | Tests a paired-prompt detector; symbolic prompts do not represent demographic groups |

In the primary corpus runs, overlap is zero, candidate seed 43 reproduces the
three-token training suffix, and none of the PAD-sentinel or paired-prefix gap
checks flags. These checks use actual character-model outputs; the protocol lists
the prompts and sentinel. Their absence of flags is not evidence of broad safety.

For the symbol reference, exact overlap is zero in all runs. All three candidates reproduce
the training suffix. Candidate seed 7 emits the configured forbidden token, while
candidate seed 43 exceeds the probability-gap threshold. These are observed outputs,
not hard-coded booleans. The corresponding unit test uses an explicitly biased
logit table and duplicated document to verify that each detector can fail.
A real safety/fairness evaluation needs relevant prompts, justified labels, coverage,
and human interpretation. Passing this lab supplies none of that assurance.

## 6. Practice, diagnose, explain [#6-practice-diagnose-explain]

1. Implement the aggregation on paper for batch means `[2,4]` with valid-token
   counts `[2,6]`. Compute perplexity. Explain the wrong unweighted answer.
2. Intentionally compare a report to itself. Then change one evaluation fingerprint
   in a copied JSON object and call `compare_runs`. Explain why rejection differs.
3. Run with `--steps 2 --output artifacts/evaluation/short.json`. Predict whether
   lower NLL must imply three correct task answers. Inspect both fields.
4. Explain why the reproduced training suffix is not sufficient evidence that the
   held-out document leaked, and design a less predictable synthetic canary.

Hints: multiply means by counts; zero is not a positive lower bound; `evaluation_id`
identifies what was measured; the arithmetic rule makes the suffix guessable.

Reference answers: (1) `(2*2+4*6)/8 = 3.5`, perplexity `exp(3.5) ≈ 33.1155`;
mean-of-means 3 overweights the shorter batch. (2) The unchanged control is a valid
comparison with no improvement; a changed identity is an invalid comparison.
(3) No: NLL uses all probabilities over 48 primary targets (24 for the symbol task); task accuracy uses only argmax
on three prompts. (4) A unique randomly constructed suffix inserted only into
training is a stronger probe; compare its likelihood with matched unseen controls
and document how many candidate strings you tested. Never use a real secret.

Move on when you can explain the denominator, reproduce the paired comparison,
reject the unchanged claim, and name a specific blind spot for all four smoke checks.
Use `docs/experiment-report-template.md` to write your experiment card before the
next controlled comparison, and retain the machine-readable report locally.

[← E02 — Data quality](/data-quality) · [E04 — Profiling & budgets →](/profiling) · [Glossary](/glossary)
