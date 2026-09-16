---
title: "Your route through the course"
sidebar:
  label: "Learning guide"
---

<span id="your-route-through-the-course" />


[← Setup](/llm-engineering-course-pages/setup) · [Course home](/llm-engineering-course-pages/) · [Glossary](/llm-engineering-course-pages/glossary)

You do not need prior machine learning, calculus, or PyTorch. You do need a
computer on which you can install Python, and time to try small programs.
If programming is new, take the Python primer in [F01](/llm-engineering-course-pages/foundations-01-python-numpy)
at your own pace. Unit 0 is a guided first success; it does not require you to
understand the whole training system yet.

## Choose the right starting point [#choose-the-right-starting-point]

| Starting point | Route |
| --- | --- |
| New to programming and ML | Setup → Unit 0 → diagnostic → F01–F06 → T01–T03 → T04 → M2 → M3 |
| Python experience, new to ML | Same route; demonstrate F01 skills in the diagnostic before skipping it. |
| Godot-RL course experience | Unit 0 → diagnostic → [B01 bridge](/llm-engineering-course-pages/foundations-godot-rl-bridge) → missing foundations → common F06 gate → same core. |

The [diagnostic](/llm-engineering-course-pages/diagnostic) is not a barrier. An unanswered question means
"learn this next". You can always choose the full lesson instead of a shortcut.

## What each stage adds [#what-each-stage-adds]

| Stage | Question you will be able to answer | Check before continuing |
| --- | --- | --- |
| [Unit 0](/llm-engineering-course-pages/unit-00) | What changes when I increase one next-character score? | Predict, run and explain your own intervention. |
| [F01–F06](/llm-engineering-course-pages/foundations-01-python-numpy) | How does a program calculate an error and update numbers to reduce it? | Shape check, numerical gradient check and one PyTorch update. |
| [T01 bigram](/llm-engineering-course-pages/bigram-baseline) | Can the previous token predict the next? | Explain train/validation separation and a baseline comparison. |
| [T02 tokenizer](/llm-engineering-course-pages/bpe-tokenizer) | How does text become integers without losing Unicode? | Encode/decode roundtrip and a merge by hand. |
| [T03 pipeline](/llm-engineering-course-pages/data-pipeline) | How do documents become batches of next-token targets? | Explain each tensor, boundary and padding mask. |
| [T04 attention](/llm-engineering-course-pages/attention) | Which earlier positions contribute to this prediction? | Hand calculation, heatmap and causality test. |
| [M2 Mini-GPT](/llm-engineering-course-pages/mini-gpt) | How do these pieces become a trainable decoder? | Tiny-batch overfit, exact reload and [core gate](/llm-engineering-course-pages/mini-gpt-gate). |
| [M3 training](/llm-engineering-course-pages/pretraining) | Can I interrupt a run and resume the same experiment? | Match uninterrupted CPU parameters and explain accumulation. |
| [M3 data quality](/llm-engineering-course-pages/data-quality) | Is my apparent improvement caused by contaminated data? | Audit, repair and measure a deliberately leaky evaluation. |
| [M3 evaluation](/llm-engineering-course-pages/evaluation) | Does a change improve held-out predictions across seeds? | Paired baseline/change report with uncertainty and limitations. |
| [M3 profiling](/llm-engineering-course-pages/profiling) | What fits my time and memory budget? | Compare an estimate to a measured CPU run. |

M2/M3 form the implemented core. Company adaptation, interpretability extensions,
inference systems and the full Engineering certificate follow in later
milestones. Completing these labs is evidence for the core rubric; it is not an
automatically issued certificate or a public-release approval.

## How to study one lesson [#how-to-study-one-lesson]

1. **Predict:** write one expected number, shape or direction before executing.
2. **Trace:** follow a single example through the code. Name each axis.
3. **Build:** implement the small exercise yourself in `artifacts/my-work/`.
4. **Break:** change the suggested assumption and keep the error as evidence.
5. **Measure:** compare to the unchanged baseline on the same data and seed.
6. **Explain:** write why the result happened; "the test passed" is not a mechanism.

Start with the CPU command. Read a JSON report as a tree of named measurements;
`validation.loss` means the `loss` field inside `validation`. Timing can change
between runs even when the learned parameters match. Keep notes of commands,
configuration, versions and results. Generated reports/weights stay in `artifacts/`.

## When you get stuck [#when-you-get-stuck]

| Difficulty | Return to |
| --- | --- |
| `def`, brackets, loops or an exception are unfamiliar | [F01](/llm-engineering-course-pages/foundations-01-python-numpy) |
| You cannot say what `[B,T,D]` means | [F02](/llm-engineering-course-pages/foundations-02-shapes) |
| Scores, probabilities and loss seem interchangeable | [F03](/llm-engineering-course-pages/foundations-03-probability) |
| You cannot predict an update's direction | [F04](/llm-engineering-course-pages/foundations-04-neuron) |
| Backward pass or a gradient check is mysterious | [F05](/llm-engineering-course-pages/foundations-05-mlp-autograd) |
| An optimizer or seed feels like magic | [F06](/llm-engineering-course-pages/foundations-06-pytorch-checkpoint) |
| A new abbreviation blocks reading | [Glossary](/llm-engineering-course-pages/glossary) |

Try the exercise before opening its hint or reference. If the reference is still
unclear, reduce it to one row and one number. Repeat only the missing checkpoint;
you do not need to restart the course. The real beginner pilot is still required
to validate pacing and comprehension with learners.
