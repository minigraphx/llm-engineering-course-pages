---
title: "Attention reference v1"
sidebar:
  hidden: true
---

[Course home](/) · English source reference

This is the reproducibility contract for the self-attention lab. It runs on the
embedded batches of the `data-pipeline-v1` pipeline; no dataset, checkpoint, or
rendered artifact is stored in Git.

## Reference command

```bash
python examples/run_attention_lab.py --embedding-dim 16 --heads 4 --head-dim 8 \
  --batch-size 4 --max-length 24 --head 0 --seed 7
```

Artifacts are written below `artifacts/attention/` (ignored by Git):
`report.json` and `heatmap-head-0.svg`.

## Hand-check contract

Queries, keys, and values are the same three two-dimensional vectors
`[[1,0],[0,1],[1,1]]`, with a causal mask and $d_k=2$.

| Row | Allowed keys | Weights | Output |
| ---: | --- | --- | --- |
| 0 | 0 | `[1, 0, 0]` | `[1, 0]` |
| 1 | 0, 1 | `[0.330, 0.670, 0]` | `[0.330, 0.670]` |
| 2 | 0, 1, 2 | `[0.248, 0.248, 0.503]` | `[0.752, 0.752]` |

Row 1 follows in closed form from $1/(1+e^{1/\sqrt{2}})$. Any deviation beyond
float tolerance means the implementation, not the arithmetic, changed.

## Reference evidence

One CPU run in the course environment produced:

| Measurement | Value |
| --- | ---: |
| single-head output shape | `[4,23,8]` |
| single-head weights shape | `[4,23,23]` |
| multi-head output shape | `[4,23,16]` |
| multi-head weights shape | `[4,4,23,23]` |
| head dimension (4 heads, width 16) | 4 |
| attention parameters | 1024 |
| future attention mass | 0.0 |
| maximum row-sum error | 1.8e-07 |
| difference to the head-by-head reference | 0.0 |
| difference to `F.scaled_dot_product_attention` | 2.4e-07 |
| gradient check, maximum relative error | 3.3e-11 |

Acceptance requires exactly zero future attention mass, agreement with both
references below `1e-5`, and a gradient error below `1e-5`. Exact float values
may vary slightly across supported PyTorch backends or versions.

## Defect contract

Each deliberately broken variant must stay detectable by a named measurement:

| Variant | Measurement | Correct | Broken |
| --- | --- | ---: | ---: |
| `no_mask` | future attention mass | 0.00 | 179.05 |
| `no_scaling` | mean weight entropy (nats) | 2.11 | 1.82 |
| `wrong_transpose` | score-matrix shape | `[B,H,T,T]` | raises |
| `softmax_over_queries` | maximum row-sum error | 0.00 | 3.82 |

`wrong_transpose` raises only while `T != d_k`. With `sequence_length=4` and
`head_dim=4` the shapes match, nothing raises, and only the values differ:
maximum output difference `1.007` and future attention mass `1.21`. Shape tests
alone therefore do not protect against it.

## Change protocol

Change one factor, keep both reports, and compare the hand-check rows, the two
reference differences, and the defect table. A change that alters the hand-check
numbers is a change of the model, not of the configuration, and needs a new
version string.


## Review regression contract

The reference configuration and numeric values above remain unchanged.
Additional masks intersect the causal mask; neither head implementation may
attend to future keys when an explicit mask is supplied. Each query must retain
one allowed key, otherwise a clear exception replaces a NaN softmax row.
Heatmap labels must come from the selected sequence, including nonzero indices.
The report must roundtrip through strict JSON (`allow_nan=False`): masked
`scaled_scores` use the string `"-inf"`; all finite scores remain numbers.
