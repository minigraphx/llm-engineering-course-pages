---
title: "Profiling: connect the model to a resource budget"
sidebar:
  label: "E04 — Profiling & budgets"
---

<span id="profiling-connect-the-model-to-a-resource-budget" />


Prerequisites: the [decoder](/llm-engineering-course-pages/mini-gpt), [training](/llm-engineering-course-pages/pretraining), and
[evaluation](/llm-engineering-course-pages/evaluation) labs. Allow about 45 minutes. You need only a CPU.
The goal is to make a defensible resource decision, with assumptions visible.

## 1. Name the quantities before estimating them [#1-name-the-quantities-before-estimating-them]

A **parameter** is a trainable scalar. A **FLOP** is one floating-point operation;
we count a multiply followed by an add as two. **FLOPs** measure work, while
**FLOP/s** is a rate. **Throughput** here means processed training tokens per second.
A **profiler** records operations and their costs during execution. **Latency** is
the elapsed time to finish a step. None of these alone measures model quality.

Let `B` be batch size, `T` sequence length, `d` embedding width, `L` decoder layers,
`H` attention heads, `V` vocabulary size, and `C` configured context capacity.
The default lab uses `B=2,T=C=8,d=16,L=1,H=2,V=7`, so each step processes 16 tokens.
The model returns logits of shape `[2,8,7]`.

## 2. Count the architecture you actually built [#2-count-the-architecture-you-actually-built]

Token embedding and untied output head contribute `2*V*d` parameters; position
embeddings contribute `C*d`. Each decoder block has `4*d*d` attention projection
weights, `8*d*d` feed-forward weights, and `5*d` feed-forward biases. Its two
normalizations add `4*d` scalars; final normalization adds `2*d`.

With normalization enabled:

```text
P = 2*V*d + C*d + L*(12*d*d + 9*d) + 2*d
  = 224 + 128 + 3216 + 32
  = 3600 parameters
```

The count is exact for this MiniGPT, including its untied head and bias-free
attention projections. The test compares the formula to `sum(p.numel() ...)`,
including the no-normalization variant. A different architecture needs a new count.

**FP32** stores one scalar in four bytes. **AdamW** retains two moving statistics
per parameter. Parameters, gradients, and those two statistics occupy roughly
`4 * 4 * P = 16P = 57,600` bytes. This does not include Python, the tensor runtime,
allocator overhead, or temporary tensors. A **saved activation** is an intermediate
value retained for the backward pass. Our deliberately approximate activation
formula is `4*B*(L*(12*T*d + H*T*T) + T*V)`, or 13,760 bytes. The combined estimate
is 71,360 bytes; it is not the memory needed to launch a Python process.

For matrix multiplications, the dense forward estimate is:

```text
2*B*L*(12*T*d*d + 2*T*T*d) + 2*B*T*d*V = 110080 FLOPs
training ≈ 3 * forward = 330240 FLOPs per optimizer step
```

The first term includes projections and feed-forward matrices; the `T*T` term
comes from attention score/value multiplication. The vocabulary head adds the
last term. We omit nonlinearities, normalization, lookup, and optimizer arithmetic.
Backward often costs around twice forward for dense products; `3x` is a planning
approximation, not a measured identity. Masking future positions does not remove
the dense matrix multiplications in this teaching implementation.

## 3. Predict first, then measure the same work [#3-predict-first-then-measure-the-same-work]

Before timing anything, choose ten steps and assume 10,000 tokens/s. This is an
explicit hypothetical throughput, not a measurement from the upcoming run.
At batch2/context8 that means `10*2*8=160` timed tokens and a predicted
`160/10000=0.016` seconds. Predicted model-state plus activation memory is 71,360
bytes. For planning, reserve additional headroom; for example 2x the predicted
step time (0.032 seconds) for variability, plus separately budgeted startup,
evaluation and checkpoint time. Tensor-byte headroom cannot substitute for
measuring actual process peak memory.

```bash
python examples/profile_training.py --assumed-tokens-per-second 10000 --output artifacts/profiling/review.json
```

The CLI writes `artifacts/profiling/review.plan.json` **before** it calls the
profiler. That file records steps, tokens, assumed throughput, predicted seconds,
model-tensor bytes, architecture and scope. You can supply a separately calibrated
throughput with `--assumed-tokens-per-second`; zero, negative and nonfinite values
are rejected. Give each experiment a distinct output name to retain its prior.

The subsequent `review.json` embeds that unchanged plan. Its
`timed_step_total_seconds` sums the actual ten timed durations; it does not use
`10*median`. Both prediction and measurement exclude warm-up, model construction,
profiling, data loading, evaluation and checkpointing. Thus their scopes match.
A recorded repeat on the reference CPU measured **0.007748458 seconds**, versus
**0.016 predicted**. The signed deviation is `actual-predicted=-0.008251542` seconds;
the fractional deviation is `-0.008251542/0.016=-0.515721`, about **51.57% less elapsed time**
than predicted. The hypothetical throughput was conservative on this
machine. This does not show the analytic FLOP estimate was wrong, and the next
machine can differ. Recalibrate with a separate pilot instead of changing a saved
prior after seeing its result.

For memory the stored estimate remains 71,360 bytes, while the profiler records
115,904 bytes of positive operator allocations. Those scopes differ, so the report
explicitly declines to calculate a peak-memory percentage error. A meaningful
peak-memory comparison needs an appropriate additional process/device measurement.

The configuration comparisons used in the table below run with the same default
planning assumption and each also writes its own `.plan.json`:

```bash
python examples/profile_training.py --output artifacts/profiling/report.json
python examples/profile_training.py --context 16 --output artifacts/profiling/context16.json
python examples/profile_training.py --context 16 --width 32 --output artifacts/profiling/width32.json
```

The runner copies the model onto CPU, creates AdamW state with one warm-up step,
and times ten complete forward/backward/optimizer steps. **Warm-up** lets lazy
allocation and first-use initialization happen before steady-state timing.
The **median** is the middle duration, less sensitive to a single interruption
than a mean. Timing is outside the profiler to avoid including its instrumentation
cost. Separate instrumented passes record training operators and forward FLOPs.

Reference run on macOS arm64, Python 3.12.13, PyTorch 2.14.0, one CPU thread:

| Context / width | Parameters | Forward estimate / recorded FLOPs | Estimated memory | Median step | Tokens/s |
| --- | ---: | ---: | ---: | ---: | ---: |
| 8 / 16 | 3,600 | 110,080 / 111,728 | 71,360 B | 0.753 ms | 21,261 |
| 16 / 16 | 3,728 | 236,544 / 239,840 | 89,216 B | 0.799 ms | 40,055 |
| 16 / 32 | 13,600 | 866,304 / 872,672 | 271,744 B | 0.958 ms | 33,392 |

These timings are observations, not acceptance thresholds. Your machine, competing
processes, and library version affect them. Increasing context doubled useful
work without doubling time here: small operations have substantial fixed overhead.
Do not extrapolate that favorable ratio to long contexts.

`with_flops=True` supplies estimates for supported operators rather than a hardware
instruction counter. The baseline difference is `(111728-110080)/110080 ≈ 1.50%`.
Extra recorded arithmetic and differences in counted operators explain why matching
analytic and profiler totals exactly is unnecessary. See the
[PyTorch profiler API](https://docs.pytorch.org/docs/stable/profiler.html) and
[official profiling recipe](https://docs.pytorch.org/tutorials/recipes/recipes/profiler_recipe.html).

The report also records `positive_operator_allocation_bytes`: 115,904 bytes in the
reference. This sums positive per-operator net allocation events during a step.
It is **not peak live memory**, **RSS** (resident process memory), or GPU memory;
reused temporaries and persistent state make those different quantities. PyTorch
may warn that a pre-profiling allocation has no matching deallocation event.
That is another reason not to label this number as peak memory.

## 4. Locate a bottleneck and make a decision [#4-locate-a-bottleneck-and-make-a-decision]

A **bottleneck** is the part that limits progress at the current configuration.
The `operators` list ranks self CPU time: time attributed to that entry after
excluding recorded child operations. `course_forward` and `course_backward` are
our annotated regions, not individual tensor kernels. In the reference, their
self times were about 527 and 240 microseconds, with AdamW's step at about 226
microseconds. This indicates appreciable dispatch/Python/optimizer overhead at
tiny scale; it does not show that a large Transformer is Python-bound.

The default budget is 60 seconds and 64 MiB (`64*1024*1024` bytes). The estimate
fits that memory allowance and the measured median gives
`floor(60 / 0.00075256) = 79,727` steps. The JSON therefore says `decision: "run"`.
This is a local planning estimate for repeated synthetic steps; loading data,
evaluation, checkpointing, startup, and profiler work consume additional time.
It is not a promise of 79,727 end-to-end training steps in one minute.

Under a *model-tensor estimate* allowance of 100,000 bytes, context 16 / width 16
fits at 89,216 bytes, while width 32 does not at 271,744 bytes. Choose the smaller
width for that hypothetical tensor allowance. For a real process budget, measure
peak memory and leave headroom first; 100,000 bytes cannot hold the Python runtime.
Try `--memory-mib 0.001` to see the baseline fail the estimate and return zero feasible
steps. Zero, negative, and nonfinite budgets are rejected instead of yielding a
misleading resource recommendation.

## 5. Scaling and optional parallel execution [#5-scaling-and-optional-parallel-execution]

Increasing batch size scales these operation estimates linearly. Doubling width
approximately quadruples matrix weights and projection work. Doubling context
quadruples the attention `T*T` component, while most other components double.
Increasing layers scales repeated block work almost linearly. Learned position
embeddings also grow with configured context capacity, even if a batch is shorter.
Always remeasure: hardware utilization and memory traffic can change these ratios.

A **scaling law** is an empirical relationship between model loss and quantities
such as parameter count, training data, and compute. It is different from the
operation-count formulas above: those describe implementation work, not how much
quality that work buys. Larger models need enough suitable data and training to
use their capacity; more passes over the same tiny corpus can overfit instead of
improving held-out quality. Diminishing loss improvements can make the next unit
of compute less valuable. Compare measured loss across controlled model/data/step
budgets before spending more. This lab's three tiny configurations cannot fit or
validate a language-model scaling law. For the original empirical framing, see
[Kaplan et al., Scaling Laws for Neural Language Models](https://arxiv.org/abs/2001.08361).

**Data parallelism** puts a full model replica on each device and splits examples.
Replicas exchange gradients before each optimizer update. It can increase total
throughput, but does not make one replica's parameters smaller; preserve global
valid-token weighting when replica token counts differ.

**Tensor parallelism** splits an operation's matrices across devices. It reduces
per-device storage for those matrices but requires communication within layers.
**Pipeline parallelism** assigns successive layers to devices and streams
microbatches through them. Idle **bubbles** appear while the pipeline fills or
waits; more microbatches can reduce that idle fraction. Both require careful
communication and scheduling and are explanations here, not implemented runtimes.

Optional GPUs can run the training lab, with its CPU fallback. This profiler
intentionally always measures CPU. To make a separate CUDA profile, include CUDA
activities and synchronize asynchronous work around wall-clock timing; report
CUDA allocation/peak metrics separately. Never compare unsynchronized launch time
to the CPU median. For this 3,600-parameter exercise, device-launch and communication
overhead can cost more than the arithmetic they accelerate.

## 6. Practice and reference answers [#6-practice-and-reference-answers]

1. For the default architecture, calculate only the four FP32 parameter-state
   buffers. Then explain why a 57,600-byte process budget is insufficient.
2. Deliberately describe `positive_operator_allocation_bytes` as peak RAM in an
   experiment card. Identify the invalid inference and repair the sentence.
3. Predict the effect of doubling context on attention-score storage versus
   projection work, run the context comparison, and explain the observed timing.
4. Choose among the measured configurations for a 100,000-byte model-tensor estimate
   allowance, and say which missing measurement you need before deploying it.

Hints: four buffers each use four bytes per parameter; allocation over a time
interval differs from concurrently live storage; distinguish `T` from `T*T`.

Answers: (1) `3600*16=57600` bytes, excluding activations, temporaries, allocator,
Python, and libraries. (2) Write “115,904 bytes of positive per-operator net
allocations observed during the profiled step; peak live memory not measured.”
(3) Attention-score storage grows fourfold and projections roughly double; tiny
CPU operations can amortize fixed overhead, so wall time need not double.
(4) Both width-16 configurations fit that estimate; context 16 has higher measured
token throughput here. Measure actual process peak memory and verify quality at
the selected context before treating it as a deployable configuration.

Move on when you can derive 3,600 parameters, explain the FLOP deviation, distinguish
allocation events from peak memory, and state a budget choice with its limitations.
