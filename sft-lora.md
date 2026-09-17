---
title: "A01 — SFT and LoRA: teach a response, preserve a base"
sidebar:
  label: "A01 — SFT & LoRA"
---

<span id="a01-sft-and-lora-teach-a-response-preserve-a-base" />


[← C04 — Continued pretraining](/llm-engineering-course-pages/continued-pretraining) · [A02 — Adaptation comparison →](/llm-engineering-course-pages/adaptation-comparison) · [Glossary](/llm-engineering-course-pages/glossary)

Prerequisites: [the decoder](/llm-engineering-course-pages/mini-gpt), [training](/llm-engineering-course-pages/pretraining), and the fictional
company data/evaluation contracts. Run the explicit CPU mechanism before the
optional pretrained lab. Both entry paths need no private data or teacher API.

## Trace the response mask before training [#trace-the-response-mask-before-training]

An instruction contains a prompt and a desired response. The prompt conditions the
response but is not itself the answer we want to score. For token IDs
`[11, 12 | 21, 22]`, construct unshifted labels `[-100, -100, 21, 22]`.
`logits[1]` predicts 21 and `logits[2]` predicts 22. The loss function shifts once;
passing already shifted labels would silently teach the wrong transition.
Padding is also `-100`, while its attention mask is zero.

If the two correct-token probabilities are 0.5 and 0.25, response mean NLL is
`(-log(0.5) - log(0.25)) / 2 = 1.0397`. Adding ten prompt tokens must not dilute
that mean. Changing masked target content leaves this loss unchanged for fixed
logits; changing a prompt input can still change the model's response logits.

The code tokenizes prompt and response separately to make the boundary explicit.
It includes the assistant chat header in the prompt and the end-of-turn token in
the response. A record that exceeds the context is rejected instead of truncating
away a safety instruction or its answer.

## Build the low-rank update before PEFT [#build-the-low-rank-update-before-peft]

Full SFT updates all weights. LoRA computes
`y = Wx + (alpha / r) BAx`, where `A` is `[r, input]` and `B` is `[output, r]`.
The original `W` and bias are frozen. For a 512×512 layer with rank 4, full weight
training uses 262,144 parameters; A and B use `4×512 + 512×4 = 4,096`, or 1.5625%.
This ratio excludes other layers, biases, optimizer state and activations.

Initialize A randomly and B to zero: the initial adapted output equals the base.
On the first backward pass B can receive a gradient while A's gradient is zero
because B is zero. After B changes, A can learn too. A test that demands a nonzero
A gradient on the very first step misunderstands the mechanism.

`LoRALinear.merged()` copies `W + (alpha/r)BA` into a fresh linear layer. Adapter
files contain a version and a hash of the exact base weights; another base with
the same dimensions is rejected. Core checkpoints use `torch.load(weights_only=True)`;
the productive library path uses safetensors. Merge checks reject nonfinite weights.
Preserve the immutable base to roll back.

## Run and measure [#run-and-measure]

```bash
.venv/bin/pytest tests/test_post_training.py -q
.venv/bin/python examples/run_sft_lora.py --steps 120
.venv/bin/pip install -e '.[post-training]'
.venv/bin/python examples/run_open_model_lora.py --download --device cpu
.venv/bin/python examples/run_open_model_lora.py --device mps --output artifacts/open-model-lora-mps
```

The first experiment pairs random MiniGPT full SFT and manual LoRA with the same
training/development examples, seed, optimizer, learning rate, steps and batch
size. The pretrained experiment uses the real Apache-2.0
[SmolLM2-135M-Instruct model](https://huggingface.co/HuggingFaceTB/SmolLM2-135M-Instruct),
fixed at revision `12fd25f77366fa6b3b4b768ec3050bf629380bac`. It downloads only
allowlisted configuration/tokenizer files and `model.safetensors`, loads with
`trust_remote_code=False`, and trains rank-4 q/v adapters in float32.

All methods consume training instructions and separately authored development
cases; none opens gold. The same evaluator checks actual generated predictions
before/after. Response NLL and exact/format/safety metrics answer different
questions. A lower teacher-forced NLL does not prove usable generated answers.
Reload and merged outputs are checked against the trained adapter. See the
[adaptation comparison](/llm-engineering-course-pages/adaptation-comparison) and `docs/baselines/sft-lora-v1.md`
for measured results, runtime, memory scope and limitations.

## Predict → Trace → Build → Break → Measure → Explain [#predict-trace-build-break-measure-explain]

1. **Predict:** Which token receives the first response loss? What is A's first gradient?
2. **Trace:** Print IDs, labels and attention masks for unequal-length examples.
3. **Build:** Implement `response_batch`, `sft_loss` and `LoRALinear` from the formulas.
4. **Break:** Unfreeze W, remove the label shift, or load onto a different base. Observe a test fail.
5. **Measure:** Record train/dev NLL, format failures, general-control NLL, time and trainable count.
6. **Explain:** Decide whether to keep the adapter. A regression is evidence for rollback.

### Independent exercise [#independent-exercise]

Use a 12-input, 8-output layer, rank 2 and alpha 6. Compute adapter count and scale.
Then replace the target of a prompt token and a response token separately while
holding logits fixed. Add a test proving which edit affects loss. Save, reload and
merge after two optimizer steps; verify base gradients remain absent.

<details>
<summary>Hint</summary>

Count both rectangular factors. The response label at position t is predicted
by logits at t−1. Do not move the mask independently of its target.


</details>

<details>
<summary>Reference answer</summary>

A has 24 and B has 16 parameters: 40 total versus 96 base weights; scale is 3.
Masked prompt target edits leave loss unchanged. Response target edits usually
change loss; choose unequal log-probabilities to make the assertion meaningful.
A learns after the zero-initialized B has begun changing.


</details>

Checkpoint: explain loss alignment without a Trainer; demonstrate zero initial
LoRA delta, frozen base, safe reload and merge; report a failed quality gate honestly.
The mechanism follows [LoRA](https://arxiv.org/abs/2106.09685); production wrappers
are documented by [PEFT](https://huggingface.co/docs/peft/v0.18.0/index).

[← C04 — Continued pretraining](/llm-engineering-course-pages/continued-pretraining) · [A02 — Adaptation comparison →](/llm-engineering-course-pages/adaptation-comparison) · [Glossary](/llm-engineering-course-pages/glossary)
