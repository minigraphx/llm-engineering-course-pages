---
title: "T03 — Data pipeline & embeddings"
sidebar:
  label: "T03 — Data pipeline & embeddings"
---

<span id="t03-data-pipeline-embeddings" />


[← T02 — Byte-level BPE](/llm-engineering-course-pages/bpe-tokenizer) · [T04 — Self-attention →](/llm-engineering-course-pages/attention) · [Glossary](/llm-engineering-course-pages/glossary)

## Learning outcome [#learning-outcome]

In 75–90 minutes, you will build a versioned mini data pipeline that turns raw
text into embedded batches, and you will be able to name the shape and meaning
of every tensor on the way. You will split documents before creating windows,
pack a token stream, pad variable-length sequences, mask padding out of the
loss, seed the DataLoader, and measure leakage instead of hoping it is absent.

The pipeline runs on two encoders: the Unit-0 character vocabulary plus three
special tokens, or the byte-level BPE tokenizer from T02. Only the
`DocumentEncoder` object changes; every later stage keeps working, which is what
makes the boundary between tokenization and pipeline visible.

## Predict · Where does a batch come from? [#predict-where-does-a-batch-come-from]

Before running anything, predict:

1. how many `[4,12]` batches 470 training tokens produce;
2. which positions of a padded batch must not reach the loss;
3. whether splitting fixed-length windows randomly is as safe as splitting
   documents randomly.

Then run the CPU reference:

```
python examples/run_data_pipeline.py
```

Keep the JSON output. It records the version, configuration, split hashes,
window counts, batch shapes, fill rates, batch hashes, embedding trace, and
leakage numbers.

## Trace · Raw text to token IDs [#trace-raw-text-to-token-ids]

`normalize_documents` strips whitespace, drops empty lines, and keeps the first
copy of each exact duplicate. Deduplication happens **before** splitting: a
document present twice would otherwise land in two partitions and turn
memorization into apparent generalization.

`encode_document` then frames each document with the encoder's boundary tokens:

| Token | Character ID | BPE ID | Meaning |
| --- | ---: | ---: | --- |
| content | 0–32 | 0–295 | bytes, merged pieces, or characters |
| pad | 33 | 296 | filler; never contributes to the loss |
| bos | 34 | 297 | document start |
| eos | 35 | 298 | document end |

`decode_tokens` prints special tokens by name, so `<bos>a token<eos>` is
readable evidence rather than a mystery integer.

`build_encoder` trains BPE on the **training documents only**. A tokenizer
trained on the whole corpus would carry validation and test statistics into
every later measurement — leakage that no split can repair afterwards.

## Build · Two ways to reach a fixed shape [#build-two-ways-to-reach-a-fixed-shape]

A model needs rectangular batches, but documents have different lengths. The
pipeline implements both standard answers and measures them.

**Packing.** `build_token_stream` concatenates the encoded training documents
into one stream of 470 tokens. `SequenceWindowDataset` cuts it into windows of
`block_size + 1` tokens and shifts them by one:

| Value | Shape | Meaning |
| --- | --- | --- |
| `input_ids` | `[4,12]` | current token IDs |
| `labels` | `[4,12]` | the same window shifted by one |
| `attention_mask` | `[4,12]` | all ones; every position is real |

With `stride=12` the windows do not overlap and 1 tail token is dropped. With
`--stride 6` you get roughly twice as many windows over the same tokens, at the
price of correlated examples.

**Padding.** `PaddedDocumentDataset` keeps one document per sample, and
`collate_batch` right-pads a batch to its longest sequence:

| Value | Shape | Meaning |
| --- | --- | --- |
| `input_ids` | `[4,39]` | tokens, `<pad>` after the document ends |
| `labels` | `[4,39]` | targets, `-100` after the document ends |
| `attention_mask` | `[4,39]` | 1 for real tokens, 0 for padding |

`-100` is PyTorch's default `ignore_index`: cross-entropy skips those positions
entirely. Padding that reaches the loss teaches the model to predict `<pad>`.

`make_batch_loader` wraps either dataset in a single-worker `DataLoader` with an
explicit `torch.Generator` seed, so the batch order is part of the contract.

## Build · Token and position embeddings [#build-token-and-position-embeddings]

`TokenPositionEmbedding` holds two tables:

- `token_embedding`: `[36,16]` — one vector per token identity;
- `position_embedding`: `[64,16]` — one vector per position index.

The forward pass adds them and zeroes padded positions:

$$
h_{b,t} = E_{\text{token}}[x_{b,t}] + E_{\text{pos}}[t]
$$

The token table alone is position-blind: the same ID gets the same vector at
position 0 and position 30. The sum is what lets a later attention layer tell
`the model` from `model the`. Reference sizes: `[4,39]` IDs become `[4,39,16]`
embeddings from 1600 parameters, and `embedding.trace(batch)` prints the whole
chain with meanings.

## Break · Split after windowing [#break-split-after-windowing]

Run the deliberately wrong order, recorded in the same report under
`leakage.window_split_demo`: build all 601 sliding windows first, then split
them randomly. The result is 63 of 151 held-out windows appearing verbatim in
training, and 149 of 151 overlapping training text by one or two positions.

Evaluation on that split measures copying, not learning. Compare it with the
document split under `leakage.document_split`: 0 shared documents, and a
measured window overlap of 0.300 (validation) and 0.216 (test), which stays
inside the 0.5 budget.

Then force a failure to see the gate work:

```
python examples/run_data_pipeline.py --max-window-overlap 0.0
```

`assert_no_leakage` raises `LeakageError`. Overlap on template-generated text is
expected and reported; unlimited overlap is not tolerated.

## Measure · Which shape wins [#measure-which-shape-wins]

| Loader | Batch shape | Real positions | Fill rate |
| --- | --- | ---: | ---: |
| packed | `[4,12]` | 468 / 468 | 1.000 |
| padded | `[4,39]` | 458 / 480 | 0.954 |

Packing wastes nothing but crosses document boundaries, so a window may predict
the start of one document from the end of another; `<eos>` is the signal that
makes this learnable. Padding respects boundaries but pays for the longest
sequence in every batch — the cost grows with length variance, not with the
number of documents.

Now change only the encoder:

```
python examples/run_data_pipeline.py --encoder bpe
```

| Measurement | character | byte-level BPE |
| --- | ---: | ---: |
| training tokens | 470 | 223 |
| packed windows | 39 | 18 |
| padded batch shape | `[4,39]` | `[4,19]` |
| padded fill rate | 0.954 | 0.894 |
| embedding parameters | 1600 | 5824 |
| window overlap (validation) | 0.300 | 0.000 |

Subwords halve the sequence length for the same text, so a fixed `block_size`
covers roughly twice as much content — the reason context length is discussed in
tokens, not characters. The vocabulary grows from 36 to 300, so the token table
grows with it. The document split hashes stay identical, because the split
happens on text and does not depend on the encoder.

Determinism is measurable too: `batch_stream_fingerprint` hashes every batch of
a full pass. Two runs of the same version, config, and seed must return the same
hashes. See the versioned
[data pipeline reference](https://github.com/minigraphx/llm-engineering-course/blob/main/docs/baselines/data-pipeline-v1.md)
for the reference values and the change protocol.

## Explain · Your pipeline paragraph [#explain-your-pipeline-paragraph]

Write one paragraph that names:

- each stage from raw text to embeddings, with tensor shapes;
- why deduplication and splitting happen before windowing;
- what `attention_mask` and `-100` each protect;
- the trade-off between packing and padding, in fill rate;
- what changes and what stays identical when you switch the encoder;
- which numbers prove your batches are deterministic;
- what your leakage budget is and why it is not zero here.

## Completion gate [#completion-gate]

You pass when you can redraw the chain
text → documents → split → tokens → windows/padding → batch → embeddings
without looking, and your saved report proves it. Required evidence:

- split document counts and SHA-256 hashes;
- `[B,T]` inputs, labels, and mask plus `[B,T,D]` embeddings;
- padded positions holding `<pad>` in inputs and `-100` in labels;
- identical batch hashes across two runs with the same seed;
- packed fill rate of 1.0 against the measured padded fill rate;
- one character run and one BPE run of the same corpus;
- 0 shared documents and a reported window overlap inside the budget.

Run the automated contract checks:

```
pytest tests/test_data_pipeline.py
```

## What's next [#whats-next]

The batches now carry meaning per position but no way to compare positions with
each other. Next, you will build self-attention as a visible number experiment
and give these embeddings context.

[← T02 — Byte-level BPE](/llm-engineering-course-pages/bpe-tokenizer) · [T04 — Self-attention →](/llm-engineering-course-pages/attention) · [Glossary](/llm-engineering-course-pages/glossary)
