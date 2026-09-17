---
title: "E01 — Reproducible pretraining"
sidebar:
  label: "E01 — Reproducible training"
---

<span id="e01-reproducible-pretraining" />


[← I01 — Decoding, sampling & KV cache](/inference-decoding) · [E02 — Data quality →](/data-quality) · [Glossary](/glossary)

## Goal and prerequisites [#goal-and-prerequisites]

Allow 90–120 minutes. You need the [document pipeline](/data-pipeline),
[Mini-GPT](/mini-gpt), gradients and the meaning of cross entropy. Complete
[setup](/setup) first; every required command runs on CPU without downloads.
By the end you can explain one optimizer update, resume a run exactly on the
same CPU runtime, and distinguish quality measurements from speed measurements.

**Pretraining** minimizes next-token negative log likelihood (NLL) on training
text. One **microbatch** is one forward/backward pass. An **optimizer step** updates
parameters after one or more microbatches. Here a batch list is materialized once
from a seeded loader, copied into the trainer, then visited cyclically. This
small, deliberately transparent loader has no workers, random augmentation or
fresh shuffle each epoch. Its fingerprint covers order, shapes, dtypes and values.

## Predict, then reproduce [#predict-then-reproduce]

From the repository root, with the virtual environment activated:

```bash
python examples/run_pretraining.py --profile cpu-small --steps 20 --unstable
pytest tests/test_training.py -q
```

The command runs 20 uninterrupted steps, separately runs 10 steps, saves
`artifacts/pretraining-v1/resume.pt`, constructs a fresh trainer, resumes to step
20 and compares the results. It also saves `final.pt` and `report.json`; none of
these generated artifacts belongs in Git. Both runs initialize the same 28,544
parameter model: context 32, width 32, four heads, two layers, dropout 0.1 and
seed 7. Only training documents enter its batches. The report includes the
synthetic corpus provenance manifest and ordered batch fingerprint.

Predict whether equal initialization alone is enough to match resumed dropout.
The reference produced loss **3.71458 → 2.61039**, with **2,553 supervised tokens**
processed in 20 updates. `parameter_max_abs_difference` must be **0**;
`optimizer_exact` and `quality_metrics_exact` must both be **true** on the same
CPU runtime. The training losses use different batches and dropout each step,
so every individual step need not improve. This is an engineering baseline,
not held-out evidence of generalization.

## Trace one update with numbers [#trace-one-update-with-numbers]

`language_model_loss` receives logits `[B,T,V]` and already shifted labels `[B,T]`.
It averages NLL only where the label is not `-100`. Padding is excluded from both
loss and token counts; the attention mask also prevents attending to padding.

Suppose microbatch A has 4 supervised tokens with mean loss 2.0, while B has 1
with loss 5.0. Averaging the two means gives 3.5 and overweights B. The true loss
is `(4×2 + 1×5) / 5 = 2.6`. Backpropagate `0.8 * loss_A` and `0.2 * loss_B`, then
update once. The implementation counts the entire accumulation window **before**
backward. Different numbers of rows, padding positions and sequence lengths
therefore get the correct weights. An entirely empty window is an error; an
empty microbatch within a nonempty window is skipped.

```python
# The essential accumulation loop; the full Trainer also checks finite values.
optimizer.zero_grad(set_to_none=True)
counts = [(batch.labels != -100).sum().item() for batch in window]
for batch, count in zip(window, counts, strict=True):
    if count:
        loss = language_model_loss(model(batch.input_ids, batch.attention_mask), batch.labels)
        (loss * count / sum(counts)).backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

With dropout disabled, the tests compare this update against one concatenated
batch with unequal valid counts (4 and 3). They allow `atol=1e-7, rtol=1e-6`
for floating-point reduction order. Active dropout consumes random draws in
shape-dependent ways; microbatching and one large batch need not be bitwise
equal even though their objectives have the same token weighting.

## AdamW, schedule and clipping [#adamw-schedule-and-clipping]

**AdamW** keeps two memories for each parameter: an exponentially smoothed
gradient `m` and squared gradient `v`. With the PyTorch defaults, their decay
factors are 0.9 and 0.999. Bias correction accounts for their initial zero values.
The adaptive update divides the corrected `m` by `sqrt(v) + 1e-8`. Decoupled
weight decay additionally multiplies a parameter by `(1 - learning_rate × decay)`.
For a weight 2, learning rate 0.01 and decay 0.1, the decay alone gives 1.998.
`adamw_parameter_groups` decays matrices (including embeddings) and excludes
biases and normalization vectors. This is an explicit teaching convention.

**Warmup** starts with smaller updates; **cosine decay** smoothly reduces the
learning rate afterwards. Steps are zero-based in `learning_rate_at`. For five
steps, two warmup steps, peak LR 0.01 and minimum ratio 0.1, the schedule is
`[0.005, 0.010, 0.010, 0.0055, 0.001]`. The default 20-step run starts at 0.0015,
reaches 0.003, and ends at 0.0003. The full intended schedule is fixed before a
run: changing `total_steps` on resume would change the experiment and is rejected.
If there is only one update after warmup, that update uses the minimum LR. A
one-step run with zero warmup therefore uses `learning_rate * min_lr_ratio`.
For three steps with two warmup steps, the example becomes `[0.005, 0.010, 0.001]`;
there is no extra peak step when there is no room for it.

**Gradient clipping** rescales the entire gradient vector if its Euclidean norm
exceeds the threshold. For gradient `[3,4]`, the norm is 5; threshold 1 gives
approximately `[0.6,0.8]`. The logged `gradient_norm` is the **pre-clipping** norm.
Clipping happens once after accumulated backward, before AdamW. It limits this
gradient norm; it cannot repair nonfinite parameters or make any learning rate safe.

## Build a resume yourself [#build-a-resume-yourself]

```python
from llm_course.data_pipeline import PaddedDocumentDataset, build_split, make_batch_loader
from llm_course.mini_gpt import GPTConfig, MiniGPT
from llm_course.training import TrainConfig, Trainer

batches = list(make_batch_loader(
    PaddedDocumentDataset(build_split().train, max_length=33), batch_size=2, seed=7))
config = TrainConfig(total_steps=20, accumulation_steps=2)
trainer = Trainer(MiniGPT(GPTConfig(dropout=0.1)), batches, config)
trainer.train(10)  # additional optimizer steps
trainer.save_checkpoint('artifacts/my-pretraining/resume.pt')
restarted = Trainer(MiniGPT(GPTConfig(dropout=0.1)), batches, config)
restarted.load_checkpoint('artifacts/my-pretraining/resume.pt')
new_rows = restarted.train()  # all remaining steps; history includes the first ten
```

A checkpoint includes model and architecture, AdamW moments and step counters,
precision scaler, training configuration, next absolute batch position, processed
tokens, metric history and Torch CPU/device RNG states. Config plus completed step
fully determines the schedule; there is no separate mutable scheduler. Only Torch
is random inside this trainer; it saves a private stream and restores the caller's
stream after training. The Python/NumPy RNGs are not used by this loop. Data is
already materialized, so loading does not depend on a DataLoader iterator's RNG.

Only completed optimizer boundaries are saved; accumulated partial gradients are
not checkpointed. Model, training config, batch fingerprint, resolved device,
precision, Torch version, thread count and deterministic-algorithm setting are
checked before restoring state. Changing any of these identities is a new run.
The checkpoint is intended for course-produced files; loading arbitrary damaged
checkpoint contents is not an integrity-repair workflow.

PyTorch does not guarantee exact reproducibility across releases, platforms or
CPU/GPU devices. Our exact replay claim is deliberately confined to the same CPU
runtime. [PyTorch reproducibility notes](https://docs.pytorch.org/docs/2.14/notes/randomness.html)
explain the wider limits.

## Measure and deliberately break [#measure-and-deliberately-break]

The JSON logs loss, pre-clipping gradient norm, learning rate, supervised token
count, cumulative tokens, seconds, tokens/second and parameter bytes per step.
The CPU reference has **114,176 parameter bytes** (28,544 FP32 parameters). This
is not total training memory: gradients, AdamW moments, activations and framework
allocations add more. `peak_device_memory_bytes` is CUDA allocated peak memory
per update; it is `null` on CPU/MPS, where this measurement is not collected.
Timing includes transfer and update with accelerator synchronization, but excludes
constructing the loader/model and checkpoint I/O. Warmup and other system load
make throughput hardware dependent; it is not a pass/fail quality threshold.

`--unstable` runs a separate model with learning rate `1e20`, no warmup and FP32.
The reference stops at zero-based step 1 with `nonfinite loss`, after one completed
update. This is a controlled overflow demonstration, not a realistic tuning value.
A large finite loss is not automatically failure: inspect loss **and** gradient
norm, check token labels, lower LR and restore a good checkpoint. `TrainingDiverged`
stops before the current optimizer update when loss or norm is nonfinite. Restart
from a good checkpoint after this exception; do not continue a failed AMP update.

Optional profiles, each with CPU fallback if the device is absent:

```bash
python examples/run_pretraining.py --profile mps-small --precision float16
python examples/run_pretraining.py --profile cuda --precision bfloat16
```

The trainer requires **FP32 master parameters**: these are the stored weights that
the optimizer updates. Caller-supplied `.double()` or `.half()` models are rejected
before training, rather than mislabeled or silently converted. Use a normal FP32
`MiniGPT` and request compute precision through `TrainConfig.precision`; do not
change parameter dtype after constructing a trainer. Autocast may use lower
precision for operations while master parameters remain FP32.

The CUDA profile uses context 64, width 128, four layers and batch size 8 when
CUDA is available. CPU/MPS stay FP32 in this lesson, even when lower precision is
requested; the report names the fallback. CUDA FP16 uses autocast and gradient
scaling; BF16 uses autocast without scaling. Accumulation keeps a single scale
across the window, then unscales once before clipping. This ordering follows the
[PyTorch AMP examples](https://docs.pytorch.org/docs/2.14/notes/amp_examples.html).
Accelerator replay differences are reported honestly; CPU is the required gate.

## Exercise, hints and reference [#exercise-hints-and-reference]

1. Implement token-weighted accumulation in a scratch copy using A=2 valid tokens
   with loss 1 and B=6 valid tokens with loss 3. Predict the objective before running.
2. Save after step 4, replace one label, and attempt resume. Then restore the label
   and change only `total_steps`. Explain why both attempts should fail.
3. Explain why equal final loss does not prove exact resume. Name two additional
   checkpoint components whose omission can change the next update.

**Hint 1:** Count supervised tokens, not rows or sequence capacity. **Hint 2:** A
schedule depends on its original horizon. **Hint 3:** Think about the optimizer's
memory and dropout's next random draws.

**Reference:** The objective is `(2×1+6×3)/8 = 2.5`; backward weights are 1/4 and
3/4. Label mutation changes the data fingerprint; horizon mutation changes config.
Compare every parameter, AdamW's moments and step counters, RNG stream, batch
position, and deterministic quality metrics. A coincidentally equal scalar loss
cannot establish equality of these states.

Move on when you can derive the weights without code, explain clipping's position,
produce exact CPU replay, identify the deliberate failure, and state why falling
training loss says nothing yet about held-out quality. That question belongs in
the evaluation lab.

[← I01 — Decoding, sampling & KV cache](/inference-decoding) · [E02 — Data quality →](/data-quality) · [Glossary](/glossary)
