---
title: "LLM Engineering Course"
sidebar:
  label: "Home"
---

<span id="llm-engineering-course" />


Learn how language models work by building the important pieces yourself, then
use those pieces to develop a model for a realistic company use case.

Start with [setup](/setup) and the [learning guide](/learning-guide).
The guide names prerequisites and checkpoints; the [glossary](/glossary)
explains unfamiliar terms. No prior machine learning is required.

## Two entry paths [#two-entry-paths]

- **Standalone path:** includes Python, mathematics, neural-network, and
  PyTorch foundations.
- **Godot RL bridge:** uses a diagnostic to skip concepts already demonstrated
  in the Godot RL course.

Both paths start with [Unit 0](/unit-00), continue through the
[diagnostic and learning-path check](/diagnostic), and converge before text
tokenization. Every skipped foundation keeps a re-entry link.

The standalone foundations track [F01–F06](/foundations-01-python-numpy)
covers Python/NumPy, tensor shapes, probability and cross-entropy, a trainable
neuron, MLP/backpropagation, a tiny autograd engine and a PyTorch checkpoint.
Godot-RL graduates take the [B01 bridge](/foundations-godot-rl-bridge) and
prove only the missing LLM-specific concepts. Both paths continue through the
core: the [T01 bigram baseline](/bigram-baseline), [T02 byte-level BPE](/bpe-tokenizer),
the [T03 data pipeline and embeddings](/data-pipeline), [T04 self-attention](/attention),
the [T05 Mini-GPT decoder](/mini-gpt) and its [T06 core gate](/mini-gpt-gate),
[I01 decoding, sampling and the KV cache](/inference-decoding), then
[E01 reproducible training](/pretraining), [E02 data quality](/data-quality),
[E03 evaluation](/evaluation) and [E04 profiling](/profiling). The company
track [C01–C04](/company-strategy) turns a requirement into a model decision,
auditable data, protected evaluation and continued pretraining; the adaptation
track [A01–A04](/sft-lora) covers SFT/LoRA, the adaptation comparison, DPO and
the RLHF bridge; the [capstone](/company-capstone) ties them into a reversible
company-model decision. The [Applied: RAG track](/rag-build) (R01–R02) is a
standalone entry that needs only Python and one LLM call.

Every required outcome has a CPU route. Run the
[hardware and cost preflight](/hardware) before choosing a larger profile.

## What you will build [#what-you-will-build]

- a bigram language-model baseline;
- a BPE tokenizer;
- self-attention and a decoder-only transformer;
- a small GPT-style model with training and evaluation;
- instruction and preference fine-tuning;
- a synthetic-data pipeline with quality gates;
- an optimized local inference service;
- a reproducible company-model capstone.

<aside className="course-note" aria-label="Development status">
<strong>Development status</strong>

Every course unit from Unit 0 through the capstone is shipped and executable
on CPU: the foundations F01–F06 with the B01 bridge, the core T01–T06 and
I01, the engineering units E01–E04, the company units C01–C04, the
adaptation units A01–A04 and the capstone, plus the Applied: RAG track
R01–R02. German localisation is complete except R01/R02 (MIN-129). Still
open: the supplied larger data packs (M1A), the interpretability extension
(M2A), the quantization/batching/serving units (M6), the human sign-off on
the sealed company gold set and synthetic sample (MIN-95/96/100), and the
real five-learner pilot. Automated checks do not establish learner
comprehension or issue a certificate.

</aside>
