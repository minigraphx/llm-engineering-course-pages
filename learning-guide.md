---
title: "Your route through the course"
sidebar:
  label: "Learning guide"
---

<span id="your-route-through-the-course" />


[← Setup](/setup) · [Course home](/) · [Glossary](/glossary)

You do not need prior machine learning, calculus, or PyTorch. You do need a
computer on which you can install Python, and time to try small programs.
If programming is new, take the Python primer in [F01](/foundations-01-python-numpy)
at your own pace. Unit 0 is a guided first success; it does not require you to
understand the whole training system yet.

## Choose the right starting point [#choose-the-right-starting-point]

| Starting point | Route |
| --- | --- |
| New to programming and ML | Setup → Unit 0 → diagnostic → F01–F06 → T01–T06 → I01 → E01–E04 → C01–C04 → A01–A04 → Capstone |
| Python experience, new to ML | Same route; demonstrate F01 skills in the diagnostic before skipping it. |
| Godot-RL course experience | Unit 0 → diagnostic → [B01 bridge](/foundations-godot-rl-bridge) → missing foundations → common F06 gate → T01–T06 → I01 → E01–E04 → C01–C04 → A01–A04 → Capstone. |
| Python and one LLM call, wants retrieval first | Setup → [R01](/rag-build) → [R02](/rag-quality), then the core route above. |

The [diagnostic](/diagnostic) is not a barrier. An unanswered question means
"learn this next". You can always choose the full lesson instead of a shortcut.

## What each stage adds [#what-each-stage-adds]

| Stage | Question you will be able to answer | Check before continuing |
| --- | --- | --- |
| [Unit 0](/unit-00) | What changes when I increase one next-character score? | Predict, run and explain your own intervention; keep the JSON report. [Troubleshooting](/unit-00-troubleshooting) if a check fails. |
| [Diagnostic](/diagnostic) | Which foundations can I skip safely, and where do I re-enter if one fails? | Common gate: one batched shape, one stable cross-entropy value, one backward/update cycle, gradient clearing explained, seeded reproduction. |
| [B01 bridge](/foundations-godot-rl-bridge) | Which Godot-RL skills transfer directly to language models? | Complete the transfer challenges, then take the shared F06 checkpoint. |
| [F01–F06](/foundations-01-python-numpy) | How does a program calculate an error and update numbers to reduce it? | F06 pass rubric: own `checkpoint.py`, gradient check below `1e-6`, wrong-sign diagnosis, two CPU runs within `1e-7`. |
| [T01 bigram](/bigram-baseline) | Can the previous token predict the next? | Recreate the data → loss → update → validation → sampling loop and explain a baseline comparison. |
| [T02 tokenizer](/bpe-tokenizer) | How does text become integers without losing Unicode? | Roundtrip and edge-case tests pass; discuss the vocabulary/sequence-length/OOV trade-off. |
| [T03 pipeline](/data-pipeline) | How do documents become batches of next-token targets? | Redraw the chain text → batch → embeddings and explain each tensor, boundary and mask. |
| [T04 attention](/attention) | Which earlier positions contribute to this prediction? | Hand calculation of one attention row, heatmap and causality test in the report. |
| [T05 Mini-GPT](/mini-gpt) | How do these pieces become a trainable decoder? | Tiny-batch overfit, exact reload; trace every shape and separate the attention mask from loss masking. |
| [T06 core gate](/mini-gpt-gate) | Can I implement, diagnose and explain a decoder independently? | At least 8/10 including all coding and defect points, tiny-batch NLL below 0.1 and exact reload. |
| [I01 decoding](/inference-decoding) | How does one row of logits become one token, when does generation stop, and what does the cache keep? | Temperature columns and masks by hand; name the stop rule and what the KV cache stores per block. |
| [E01 training](/pretraining) | Can I interrupt a run and resume the same experiment? | Match uninterrupted CPU parameters, derive the accumulation weights and explain clipping's position. |
| [E02 data quality](/data-quality) | Is my apparent improvement caused by contaminated data? | Audit, repair and measure a deliberately leaky evaluation; write a data card. |
| [E03 evaluation](/evaluation) | Does a change improve held-out predictions across seeds? | Paired baseline/change report with uncertainty; reject the unchanged control; name a smoke-check blind spot. |
| [E04 profiling](/profiling) | What fits my time and memory budget? | Derive the parameter count, compare an estimate to a measured CPU run and state the memory scope. |
| [C01 strategy](/company-strategy) | Is training the right next action for this company problem? | Trace every chosen component to a requirement, a measured baseline gap and an acceptance test. |
| [C02 company data](/company-data) | How does a source snapshot become auditable training data? | Explain snapshot, training record, development case and protected gold case. |
| [C03 protected evaluation](/company-evaluation) | Why is a development score not gold evidence? | Contamination report with no overlaps; explain iteration versus sealed proof. |
| [C04 continued pretraining](/continued-pretraining) | Does domain text help without losing the general baseline? | State a promotion threshold, name the rollback artifact, explain why DAPT needs instruction evaluation. |
| [A01 SFT & LoRA](/sft-lora) | How do I teach a response while preserving the base? | Loss alignment without a Trainer; zero initial LoRA delta, frozen base, safe reload and merge; report a failed gate. |
| [A02 adaptation comparison](/adaptation-comparison) | Which adaptation does the evidence justify? | Read a fair paired scorecard; separate storage savings from peak memory; reject an adapter that fails a gate. |
| [A03 DPO](/dpo) | How does a preference pair change a policy? | One response log-probability sum, the clipped PPO ratio, proxy reward versus task success. |
| [A04 RLHF bridge](/rlhf-bridge) | How does a reward model get exploited, and what would trigger a rollback? | Describe one rollback monitor and the immutable reference needed to reproduce it. |
| [Capstone](/company-capstone) | Can another person reproduce my company-model decision? | Decision record: CPU recipes from a clean checkout, baseline hashes, two rejected alternatives, the rollback step. |
| [R01 RAG build](/rag-build) | How does a document become a cited answer without training? | Add one question to the evaluation set and report its Recall@5 for both presets. |
| [R02 RAG quality](/rag-quality) | Does my retrieval system refuse and cite correctly? | Say, with numbers from your own run, which preset retrieves better, how often each refuses and what the judge cannot see. |

Completing these labs is evidence for the course rubrics; it is not an
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
| `def`, brackets, loops or an exception are unfamiliar | [F01](/foundations-01-python-numpy) |
| You cannot say what `[B,T,D]` means | [F02](/foundations-02-shapes) |
| Scores, probabilities and loss seem interchangeable | [F03](/foundations-03-probability) |
| You cannot predict an update's direction | [F04](/foundations-04-neuron) |
| Backward pass or a gradient check is mysterious | [F05](/foundations-05-mlp-autograd) |
| An optimizer or seed feels like magic | [F06](/foundations-06-pytorch-checkpoint) |
| You cannot say which positions an attention row reads | [T04](/attention) |
| Temperature, top-k/top-p or the KV cache are unclear | [I01](/inference-decoding) |
| A paired comparison, seed or held-out claim is unclear | [E03](/evaluation) |
| A new abbreviation blocks reading | [Glossary](/glossary) |

Try the exercise before opening its hint or reference. If the reference is still
unclear, reduce it to one row and one number. Repeat only the missing checkpoint;
you do not need to restart the course. The real beginner pilot is still required
to validate pacing and comprehension with learners.
