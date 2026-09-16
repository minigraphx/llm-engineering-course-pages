---
title: "F06 — PyTorch translation and foundations checkpoint"
sidebar:
  label: "F06 — PyTorch & checkpoint"
---

<span id="f06-pytorch-translation-and-foundations-checkpoint" />


[← F05](/llm-engineering-course-pages/foundations-05-mlp-autograd) · [Course home](/llm-engineering-course-pages/) · [Godot-RL bridge](/llm-engineering-course-pages/foundations-godot-rl-bridge)

## Learning outcome [#learning-outcome]

You will translate the NumPy pipeline to PyTorch and pass the common gate by
implementing, checking, breaking, and explaining it independently.

## One complete framework update [#one-complete-framework-update]

PyTorch stores tensors and remembers operations for the backward pass.
`nn.Linear(3,2)` holds a weight matrix and bias: three input values become two
scores. `model.parameters()` returns the adjustable tensors. SGD (stochastic
gradient descent) performs the familiar subtraction of learning rate times
gradient. The library manages bookkeeping; the mathematics stays the same.
Save this in `artifacts/my-work/one_update.py` and run
`python artifacts/my-work/one_update.py`:

```python
import torch
from torch import nn

torch.manual_seed(7)
x = torch.tensor([[1., 0., 0.], [0., 1., 0.], [0., 0., 1.], [1., 1., 0.]])
y = torch.tensor([0, 1, 1, 0])
model = nn.Linear(3, 2)
optimizer = torch.optim.SGD(model.parameters(), lr=0.1)
logits = model(x)
assert logits.shape == (4, 2)
loss = nn.functional.cross_entropy(logits, y)
optimizer.zero_grad()
loss.backward()
assert all(torch.isfinite(p.grad).all() for p in model.parameters())
optimizer.step()
after = nn.functional.cross_entropy(model(x), y)
print(round(loss.item(), 6), round(after.item(), 6))
assert after.item() < loss.item()
```

Output shows loss before and after the update; the second number is smaller.
`.item()` reads a scalar tensor as a Python number. Pass **logits** to
`cross_entropy`, not probabilities already transformed by softmax: the function
includes stable log-softmax. Integer targets select the correct column.
`backward()` computes gradients; only `step()` changes parameters. `zero_grad()`
prevents unintended addition of the previous step's gradients. After tracing this
example, complete the independent checkpoint below.

## Why custom models use a class [#why-custom-models-use-a-class]

A **class** groups stored data and related functions (**methods**).
`class SmallMLP(nn.Module)` extends PyTorch's module class. `__init__` sets up
layers once, `self` means this particular model object, and `super().__init__()`
initializes PyTorch's parameter bookkeeping. Storing layers as `self.hidden`
registers their parameters with the optimizer. `forward` describes the calculation;
`mlp(x)` invokes it through PyTorch's bookkeeping. Append this below the earlier example:

```python
class SmallMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.hidden = nn.Linear(3, 4)
        self.output = nn.Linear(4, 2)

    def forward(self, inputs):
        hidden = torch.tanh(self.hidden(inputs))
        return self.output(hidden)

mlp = SmallMLP()
assert mlp(x).shape == (4, 2)
```

This is the familiar MLP: 3 → 4 → 2 with `tanh` between layers. The Transformer
uses the same class structure with a longer `forward` calculation.
`nn.Sequential` chains layers; `nn.ModuleList` registers a list of blocks you call
in a loop. A `dataclass` such as the later `GPTConfig` groups named settings;
it is not a learning algorithm.

## Translate concepts, not just syntax [#translate-concepts-not-just-syntax]

| From scratch | PyTorch |
| --- | --- |
| parameter arrays | `torch.nn.Parameter` inside a module |
| manual forward equations | `module(batch)` |
| manual reverse pass | `loss.backward()` |
| subtract learning-rate times gradient | `optimizer.step()` |
| reset stored gradients | `optimizer.zero_grad()` |

For a batch `[4,3]` and `Linear(3,2)`, logits must be `[4,2]`; four integer
targets must be `[4]`. Predict these shapes before running anything.

## Independent checkpoint [#independent-checkpoint]

Create your own `checkpoint.py` without importing `llm_course` implementations.
It must:

1. implement stable batched softmax and cross-entropy;
2. implement a one-hidden-layer NumPy MLP with manual backpropagation;
3. compare one analytic parameter gradient with central difference and require
   absolute error below `1e-6`;
4. train the scalar neuron on the supplied four points, diagnose a copy whose
   update sign is reversed, and repair it;
5. translate one MLP training step to PyTorch with seed 7, shape assertions,
   gradient clearing, finite-gradient assertions, backward, and update;
6. print initial/final loss, gradient-check error, shapes, fault evidence, and
   a two-sentence diagnosis.

The supplied points are generated in code, so no learner-owned data is needed:

```
x = np.array([-1.0, 0.0, 1.0, 2.0])
y = 2.0 * x + 1.0
```

Run twice from a clean process. Values must match within `1e-7` on CPU.

## Reference oracle [#reference-oracle]

After your independent version passes, run:

```
python examples/foundations_checkpoint.py
pytest tests/test_foundations.py tests/test_autograd.py
```

Compare contracts and evidence, not variable names. The reference deliberately
exposes healthy and wrong-sign traces; it does not replace your diagnosis.

## Pass rubric [#pass-rubric]

| Evidence | Pass condition |
| --- | --- |
| implementation | no course foundation functions imported in learner solution |
| shapes/numerics | all asserted; probabilities finite and normalized |
| gradient check | absolute error below `1e-6` |
| failure diagnosis | names wrong sign, cites rising loss/parameter direction, repairs it |
| PyTorch | correct forward → zero → backward → finite check → step order |
| reproducibility | two CPU runs agree within `1e-7` |

If one row fails, return only to its F01–F05 re-entry link in the
[diagnostic](/llm-engineering-course-pages/diagnostic), then retry that row.

## What's next [#whats-next]

The neural-foundations milestone is complete. Record the common gate in the
[diagnostic](/llm-engineering-course-pages/diagnostic), then continue with the
[bigram language-model baseline](/llm-engineering-course-pages/bigram-baseline).

[← F05](/llm-engineering-course-pages/foundations-05-mlp-autograd) · [Diagnostic](/llm-engineering-course-pages/diagnostic) · [→ Bigram baseline](/llm-engineering-course-pages/bigram-baseline)
