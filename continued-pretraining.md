---
title: "C04 — Continued pretraining without losing the baseline"
sidebar:
  label: "C04 — Continued pretraining"
---

<span id="c04-continued-pretraining-without-losing-the-baseline" />


[← C03 — Protected evaluation](/company-evaluation) · [A01 — SFT & LoRA →](/sft-lora) · [Glossary](/glossary)

Prerequisites: [C02](/company-data), [E01](/pretraining). Allow one session.
The code is `src/llm_course/continued_pretraining.py`; read `document_batches`,
`adaptation_decision` and `run_continued_pretraining` beside this page.

## What DAPT changes and what it cannot [#what-dapt-changes-and-what-it-cannot]

Continued pretraining (also called domain-adaptive pretraining, DAPT) continues
the language-model objective on approved domain text. It teaches vocabulary and
style distributions; it does not directly teach a JSON response contract.
`document_batches` encodes each document with a zero-merge byte tokenizer
(vocabulary 260, so nothing is fitted to held-out text) and cuts it into windows
of 48 next-byte targets that never cross a document boundary. The model is T05's
`MiniGPT` at one layer, width 32 and four heads: 30,816 parameters, the 123,264
bytes of float32 storage the report lists as `parameter_bytes`.

The example splits whole documents before any window is cut: every fourth
`tiny_lm` sentence is held out as a general control (12 train, 4 held out) and
every fourth `company_documents()` record, at another offset, is a domain
held-out (9 train, 3 held out: `access-secret-v1`, `escalation-v1` and
`support-format-v1`). Instruction
and gold cases are never read. `run_continued_pretraining` normalises both
training corpora against both held-outs and raises `ValueError` on any overlap.

## Trace the decision gate [#trace-the-decision-gate]

`adaptation_decision` reads four NLLs and two preregistered thresholds:
gain `= domain_before − domain_after` must reach `minimum_gain = 0.1` nats, and
regression `= general_after − general_before` must stay within
`maximum_regression = 0.15`. Both gates must pass; a lower domain NLL cannot
hide forgetting. For seed 7 the domain held-out moves from 3.7068 to 3.2961, a
gain of `0.4107`; the general control moves from 2.5574 to 2.6662, a regression
of `0.1087`. Both gates pass and the decision is `keep_for_further_review`,
with the scope string `raw-text validation gates only; no instruction or production-quality claim`. C01's `choose_strategy`
applies the same two thresholds when it is given this evidence.

| Seed | Domain NLL before → after | Gain | General NLL before → after | Regression |
| --- | --- | ---: | --- | ---: |
| 7 | 3.7068 → 3.2961 | 0.4107 | 2.5574 → 2.6662 | 0.1087 |
| 19 | 3.6247 → 3.2551 | 0.3696 | 2.4640 → 2.5500 | 0.0860 |
| 43 | 3.5880 → 3.2296 | 0.3584 | 2.5191 → 2.5515 | 0.0323 |

## Run, resume, roll back [#run-resume-roll-back]

`run_continued_pretraining.py` uses the approved company snapshot, byte-level
fallback tokenization, disjoint domain held-outs and two general controls. It
trains a tiny CPU model, saves an immutable base, resumes from an interrupted
checkpoint, and can roll back by reloading the base.

```bash
.venv/bin/python examples/run_continued_pretraining.py --seeds 7 19 43
```

The measured seeds improved domain NLL by 0.359–0.411 nats while general-control
NLL regressed by 0.032–0.109 nats. The report therefore keeps the candidate for
further review rather than promoting it. Exact resume and rollback identity are
strong engineering checks; they are not a quality claim.

Each seed writes `artifacts/continued-pretraining/<seed>/report.json` after 80
base steps and 40 adaptation steps at learning rate 0.002, 2,940 adaptation
tokens. The resume check trains a second copy for 20 steps, saves
`interrupted.pt`, loads it into a fresh trainer and finishes the remaining 20:
`parameter_max_difference` is `0.0` and `optimizer_exact` is `true` for every
seed. The rollback check reloads `general-reference.pt` and compares logits on
one held-out batch: `logits_exact` is `true`. Because the decision is keep,
`selected_checkpoint` is `adapted-model.pt`; a `rollback` decision would point
it at the reference instead. A 60-second budget per seed stops training at an
optimizer boundary and saves `budget-stop.pt`.

## Predict → Trace → Build → Break → Measure → Explain [#predict-trace-build-break-measure-explain]

1. **Predict:** Which of gain 0.05 with regression 0.02, or gain 0.5 with regression 0.2, does `adaptation_decision` roll back? Both? Write it down.
2. **Trace:** Run the command above and read `decision`, `resume` and `rollback` for seed 7 in `artifacts/continued-pretraining/7/report.json`.
3. **Build:** Call `adaptation_decision(3.7068, 3.2961, 2.5574, 2.6662)` and reproduce the seed-7 row; then pass `maximum_regression=0.1` and read `reasons`.
4. **Break:** Append one `domain_eval` document to `domain_train` in a copy of the example and observe the `ValueError` before any step runs.
5. **Measure:** Run `--seeds 7 --adapt-steps 80` and record gain and regression next to the 40-step row; state which gate moves.
6. **Explain:** Why does a passing gate still not promote the checkpoint? Name the evaluation that has to run next.

### Independent exercise [#independent-exercise]

Why are a domain gain and a general regression reported separately? Which change
would make this experiment invalid: adding a general held-out sentence to the
training text, or changing only the learning-rate seed?

<details>
<summary>Hint</summary>

Look at what `run_continued_pretraining` checks before training and at what
the report records for each seed. One change is caught by code; the other is
caught only by the person keeping the record.


</details>

<details>
<summary>Reference answer</summary>

Separate scores expose forgetting. Adding held-out text contaminates the test;
a preregistered seed change is a new run that must be recorded, not hidden.


</details>

Checkpoint: state a promotion threshold, name the rollback artifact and explain
why DAPT must be followed by instruction evaluation.

[← C03 — Protected evaluation](/company-evaluation) · [A01 — SFT & LoRA →](/sft-lora) · [Glossary](/glossary)
