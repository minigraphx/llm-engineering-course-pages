---
title: "B01 — Godot-RL neural-foundations bridge"
sidebar:
  label: "B01 — Godot-RL bridge"
---

<span id="b01-godot-rl-neural-foundations-bridge" />


[← Diagnostic](/llm-engineering-course-pages/diagnostic) · [Course home](/llm-engineering-course-pages/) · [→ Common checkpoint](/llm-engineering-course-pages/foundations-06-pytorch-checkpoint)

## Purpose [#purpose]

This compact page recognizes demonstrated Godot-RL skills without treating a
course certificate as proof. Complete the challenges below; revisit only the
foundation blocks where evidence is missing.

## Transfer map [#transfer-map]

| Godot-RL foundation | LLM foundation | Verify or learn now |
| --- | --- | --- |
| weighted sum, bias, activation | neuron and MLP | verify F04 shapes and one gradient |
| layers and forward pass | token representation pipeline | add the `[B,T,C]` time axis in F02 |
| loss and backpropagation | next-token cross-entropy | learn vocabulary softmax and F03 loss |
| policy logits | next-token logits | both normalize scores, but pretraining has labelled next tokens |
| PyTorch training | PyTorch LM training | verify gradient clearing and causal target shapes |
| failure diagnosis | experiment diagnosis | retain trace, baseline, seed, and repair evidence |
| ONNX/native inference | optimized LM inference | later add autoregressive decoding and KV cache |

## Twenty-minute evidence check [#twenty-minute-evidence-check]

1. Infer `[4,8,64]` from activations `[4,8,32]` and weight `[32,64]`.
2. Calculate softmax and both target losses for logits `[2,0]`.
3. Explain why reward is not used in ordinary next-token pretraining.
4. Obtain gradient 28 for `(2x+1)²` at `x=3` by chain rule and central difference.
5. Run one PyTorch `[4,3] → [4,2]` update with seed 7 and finite gradients.

| Result | Route |
| --- | --- |
| all five independent | go directly to the F06 common checkpoint |
| one supported answer | review the corresponding F02–F05 section, then F06 |
| any critical shape, loss, or gradient error | complete that full block, then F06 |

Do not skip F03 merely because policy networks also emit logits: vocabulary
targets, sequence axes, causal masking, and likelihood training are new.

## What's next [#whats-next]

Take the shared [F06 checkpoint](/llm-engineering-course-pages/foundations-06-pytorch-checkpoint). It is the
same gate for both entry paths.

[← Diagnostic](/llm-engineering-course-pages/diagnostic) · [→ Common checkpoint](/llm-engineering-course-pages/foundations-06-pytorch-checkpoint)
