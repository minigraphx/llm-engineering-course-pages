---
title: "T05 — Mini-GPT: assemble a tiny decoder"
sidebar:
  label: "T05 — Mini-GPT"
---

<span id="t05-mini-gpt-assemble-a-tiny-decoder" />


[← T04 — Self-attention](/attention) · [T06 — Core gate →](/mini-gpt-gate) · [Glossary](/glossary)

**Outcome:** build a trainable next-token model and explain every operation from
integer IDs to a probability distribution. Allow 90–120 minutes, then take the
[core gate](/mini-gpt-gate). You need [tensor shapes](/foundations-02-shapes),
[autograd](/foundations-05-mlp-autograd), the [document pipeline](/data-pipeline),
and [causal attention](/attention). Either the standalone route or the
[Godot-RL bridge](/foundations-godot-rl-bridge) leads here.

## Start with one prediction [#start-with-one-prediction]

For the document `cat`, the pipeline gives inputs `[<bos>, c, a, t]` and targets
`[c, a, t, <eos>]`. Position 1 reads `<bos>, c` and predicts `a`. The dataset has
already shifted the targets: shifting them again would ask `c` to predict `t`.
A **decoder** uses only earlier positions and the current position. A **logit** is
an unconstrained score for one candidate token; it becomes a probability only
after softmax. The **vocabulary** is the set of permitted token IDs.

The implementation is `src/llm_course/mini_gpt.py`; read `MiniGPT.forward` beside
this page. It calls the explicit `MultiHeadSelfAttention` from the previous lab.
There is no ready-made Transformer layer to hide the masks or projections.

## Follow the shapes and numbers [#follow-the-shapes-and-numbers]

For two three-token sequences, width 4, two heads, and a toy vocabulary of 5:

| Stage | Shape | Meaning |
| --- | --- | --- |
| IDs | `[2,3]` | 6 integers, not continuous features |
| Token + position embeddings | `[2,3,4]` | 4 learned features per position |
| Q, K, V per head | `[2,2,3,2]` | 2 heads with 2 features each |
| Attention scores | `[2,2,3,3]` | each query compares with 3 keys |
| Merged head output | `[2,3,4]` | head features concatenated and projected |
| Feed-forward expansion | `[2,3,16]` | 4× wider features at each position |
| Block output | `[2,3,4]` | residual additions preserve width |
| Final logits | `[2,3,5]` | 5 next-token scores per position |

1. **Lookup and position.** Token ID 2 selects one embedding row, for example
   `[1,0,-1,2]`. At position 1, add `[0.1,0.2,0.3,0.4]` to obtain
   `[1.1,0.2,-0.7,2.4]`. The position vector tells identical tokens where they occur.
   Padded positions are multiplied by zero.
2. **Layer normalization.** At each position separately, subtract the feature
   mean and divide by `sqrt(variance + 1e-5)`, then multiply/add learned scale/bias.
   For `[1,3]`, mean is 2 and variance is 1, so the initial result is approximately
   `[-1,1]`. It does not average across time or across documents.
3. **Attention.** Linear maps produce Q, K and V. A query `[1,0]` and keys
   `[1,0]`, `[0,1]` give scaled scores `[0.7071,0]` after dividing by `sqrt(2)`.
   Softmax gives roughly `[0.670,0.330]`. Values `[2,0]`, `[0,4]` mix to
   `[1.340,1.320]`. Future and padded keys are forbidden before softmax.
4. **Residual addition.** Add the attention update to the original vector:
   `[1,2] + [0.3,-0.2] = [1.3,1.8]`. This keeps a direct path for information and
   gradients through the block. Normalize *before* attention; this is **pre-norm**.
   The placement can affect training stability; the motivation is discussed in
   [Xiong et al.](https://arxiv.org/abs/2002.04745). Our small experiment below is
   a separate local measurement, not a replication of that paper.
5. **Feed-forward network (FFN).** At every position, apply `W1`, GELU, then `W2`:
   width 4 → 16 → 4. A scalar linear example is `2 × 0.5 + 0.1 = 1.1`.
   GELU is `x × Φ(x)`, so `GELU(1) ≈ 0.8413`, `GELU(-1) ≈ -0.1587`;
   it is a smooth nonlinearity, not a probability distribution over tokens.
   See [PyTorch's GELU definition](https://docs.pytorch.org/docs/2.14/generated/torch.nn.GELU.html).
   Add this update through another residual connection. The FFN mixes features
   at one position; attention mixes information between permitted positions.
6. **Dropout.** During training, probability `p=0.25` drops some features and scales
   survivors by `1/0.75`: a surviving value 3 becomes 4. Evaluation disables this
   randomness. The mandatory baseline uses `p=0` so this variable stays fixed.
7. **Final norm and LM head.** After all blocks, normalize features and linearly
   project width 4 to vocabulary size 5. An output weight row `[1,0,0,-1]` and
   hidden state `[2,1,0,0.5]` yield logit 1.5. No softmax goes inside `forward`.
8. **Loss.** For toy logits `[0,log(2),0]`, probabilities are `[0.25,0.5,0.25]`.
   If the correct target is index 1, negative log-likelihood (NLL) is
   `-log(0.5)=0.6931`. Two valid targets with losses 0.6931 and 1.3863 average
   to 1.0397; adding ten padded targets changes neither that mean nor gradients.
   Perplexity is `exp(mean NLL)`, here approximately 2.828.

The block equations are `h = x + Attention(LN(x))`, then
`y = h + FFN(LN(h))`, with dropout on updates and padded states zeroed after
both additions. All-masked rows need care: softmax over only `-inf` is undefined.
We allow padded queries a dummy diagonal key and then zero their outputs; valid
queries still cannot read padding. Try an entirely padded batch and inspect
`torch.isfinite(logits).all()`.

## Run the reference experiment [#run-the-reference-experiment]

From the repository root after [setup](/setup):

```bash
python -m pytest tests/test_mini_gpt.py -q
python examples/run_mini_gpt.py --steps 120 --seeds 7 19 42
```

The command writes `artifacts/mini-gpt/report.json`, with configuration, document
hashes, batch fingerprints, shapes, losses, fixed generated samples and paired
ablation deltas. It generates the small course corpus in memory, uses CPU and one
thread, and downloads nothing. The actual corpus has 12 training and 2 validation
documents; the runner caps these at 32/16 for future larger corpus versions.
Each document is truncated to 25 IDs, giving at most 24 prediction targets.
Training uses batch size 8, width 32, 4 heads, 2 blocks, AdamW at 0.003, weight decay
0.01, gradient clipping at norm 1, and 120 updates. The second arm removes all layer
normalization; data split, batch order, optimizer, steps and seed remain paired.

Reference measurements on Python 3.12.13 / PyTorch 2.14.0, CPU:

| Seed | Norm train NLL | Norm validation NLL | No-norm validation NLL | Paired difference |
| --- | --- | --- | --- | --- |
| 7 | 0.130419 | 2.082053 | 4.471570 | +2.389516 |
| 19 | 0.127252 | 2.228553 | 3.334080 | +1.105527 |
| 42 | 0.131081 | 2.154143 | 3.479235 | +1.325092 |

The mean no-norm minus norm difference is **+1.606712 NLL**, sample standard
 deviation **0.686760**. This is descriptive variation over only three seeds,
not a confidence interval. All three paired runs favor normalization on these two
validation documents. Training NLL is much lower than validation NLL: fitting this
tiny corpus does not demonstrate general language ability. Removing normalization
also removes its 320 learned scale/bias parameters, an inherent part of this
ablation. The complete contract is `docs/baselines/mini-gpt-v1.md`.

## Generate, save, reload [#generate-save-reload]

```python
import torch
from llm_course.data_pipeline import BOS_ID
from llm_course.mini_gpt import GPTConfig, MiniGPT, load_model, save_model

model = MiniGPT(GPTConfig())  # untrained: output has no learned language quality yet
prompt = torch.tensor([[BOS_ID]], dtype=torch.long)
sample = model.generate(prompt, max_new_tokens=8, temperature=1.0, seed=7)
save_model(model, "artifacts/mini-gpt/model.pt")
restored = load_model("artifacts/mini-gpt/model.pt")  # CPU, evaluation mode
assert torch.equal(model.eval()(prompt), restored(prompt))
```

Run the baseline command first to create the artifact directory. Use the character
encoder to describe special token IDs; ordinary character decoding alone does not
handle BOS/EOS/PAD. Sampling divides logits by temperature: `[0,2]` at temperature
2 becomes `[0,1]`, reducing the sharper token's probability from 0.881 to 0.731.
A local seed makes repeated sampling reproducible and preserves the model's prior
train/eval state. When context exceeds the block size, only the latest window is
read and its learned positions restart at zero; the returned prefix is retained.
Prompts are unpadded, and generation always produces the requested number of
new tokens, without an EOS stop rule or KV cache ([I01](/inference-decoding)). Checkpoints store architecture
and weights; optimizer/RNG resume belongs to E01.

## Controlled break and next step [#controlled-break-and-next-step]

In a local experiment, change the dataset targets to equal its inputs. The loss
may fall even faster because the model sees the answer at the current position.
Explain why this is no longer next-token learning, then restore shifted targets.
Do not commit generated corpora or checkpoints. Move to the [core gate](/mini-gpt-gate)
when you can trace every shape, distinguish the attention mask from loss masking,
and explain why tiny-batch overfitting is a debugging check rather than held-out
quality evidence.

[← T04 — Self-attention](/attention) · [T06 — Core gate →](/mini-gpt-gate) · [Glossary](/glossary)
