---
title: "F02 — Tensors and shapes"
sidebar:
  label: "F02 — Tensors & shapes"
---

<span id="f02-tensors-and-shapes" />


[← F01](/llm-engineering-course-pages/foundations-01-python-numpy) · [Course home](/llm-engineering-course-pages/) · [→ F03](/llm-engineering-course-pages/foundations-03-probability)

## Learning outcome [#learning-outcome]

You will predict matrix and batch shapes before execution, verify them with
assertions, and diagnose one deliberate broadcasting error.

## Start with rows and columns [#start-with-rows-and-columns]

A scalar is one number, a vector is a row of numbers, a matrix is a table, and a
tensor generalizes arrays to more axes. `shape` is the list of axis lengths;
`[2,3]` is two rows of three entries. In matrix multiplication, one output entry
is a **dot product**: multiply matching entries and sum. The top-left output
below is `1×1 + 2×0 + 3×1 = 4`; the top-right is `1×0 + 2×1 + 3×1 = 5`.

`reshape` repackages the same ordered values into new dimensions; `transpose`
exchanges axes. A 2×3 table becomes a 3×2 table under transpose. **Broadcasting**
reuses a smaller array across compatible axes: adding `[10,20,30]` to every row
of a 2×3 table does not copy the data in your source code. Axis sizes must agree
or one must be 1. `*` multiplies corresponding elements; `@` contracts matrix
axes. They answer different questions even when their output shapes happen to match.

## A concrete matrix multiplication [#a-concrete-matrix-multiplication]

Let

```text
X = [[1, 2, 3],
     [0, 1, -1]]
W = [[1, 0],
     [0, 1],
     [1, 1]]
```

`X` has shape `[2, 3]`, `W` has shape `[3, 2]`, and the shared dimension 3
contracts. Therefore `X @ W` has shape `[2, 2]` and the concrete result is
`[[4, 5], [-1, 0]]`.

```
import numpy as np

x = np.array([[1, 2, 3], [0, 1, -1]])
w = np.array([[1, 0], [0, 1], [1, 1]])
result = x @ w
assert x.shape == (2, 3)
assert w.shape == (3, 2)
assert result.shape == (2, 2)
assert np.array_equal(result, [[4, 5], [-1, 0]])
```

## Batch, time, channel [#batch-time-channel]

Language-model activations normally use `[B, T, C]`: examples, token
positions, channels. For `X:[4, 8, 32]` and `W:[32, 64]`, NumPy/PyTorch applies
the same projection at every batch and time position, producing `[4, 8, 64]`.
Only the last axis is contracted.

A bias with shape `[64]` broadcasts over both leading axes. A bias shaped
`[8]` does not describe channels; make that mismatch fail early:

```
activations = np.zeros((4, 8, 64))
bias = np.zeros(64)
assert bias.shape == (activations.shape[-1],)
assert (activations + bias).shape == (4, 8, 64)
```

## Build and break [#build-and-break]

Write `linear(x, weight, bias)` with explicit shape checks. Test a 2D input and
a `[B,T,C]` input. Then transpose the weight accidentally. Capture the failing
assertion and explain which two dimensions no longer agree.

## Completion evidence [#completion-evidence]

- every intermediate shape is written before code execution;
- assertions cover batch, time, feature, hidden, and class axes;
- you can distinguish reshape, transpose, and broadcast;
- the broken transpose fails at your boundary, not deep inside training.

## What's next [#whats-next]

Continue with [F03 — Probability and loss](/llm-engineering-course-pages/foundations-03-probability).

[← F01](/llm-engineering-course-pages/foundations-01-python-numpy) · [→ F03](/llm-engineering-course-pages/foundations-03-probability)
