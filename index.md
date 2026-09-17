---
title: "LLM Engineering Course"
sidebar:
  label: "Home"
---

<span id="llm-engineering-course" />


Learn how language models work by building the important pieces yourself, then
use those pieces to develop a model for a realistic company use case.

Start with [setup](/llm-engineering-course-pages/setup) and the [learning guide](/llm-engineering-course-pages/learning-guide).
The guide names prerequisites and checkpoints; the [glossary](/llm-engineering-course-pages/glossary)
explains unfamiliar terms. No prior machine learning is required.

## Two entry paths [#two-entry-paths]

- **Standalone path:** includes Python, mathematics, neural-network, and
  PyTorch foundations.
- **Godot RL bridge:** uses a diagnostic to skip concepts already demonstrated
  in the Godot RL course.

Both paths start with [Unit 0](/llm-engineering-course-pages/unit-00), continue through the
[diagnostic and learning-path check](/llm-engineering-course-pages/diagnostic), and converge before text
tokenization. Every skipped foundation keeps a re-entry link.

The standalone foundations track [F01–F06](/llm-engineering-course-pages/foundations-01-python-numpy)
covers Python/NumPy, tensor shapes, probability and cross-entropy, a trainable
neuron, MLP/backpropagation, a tiny autograd engine and a PyTorch checkpoint.
Godot-RL graduates take the [B01 bridge](/llm-engineering-course-pages/foundations-godot-rl-bridge) and
prove only the missing LLM-specific concepts. Both paths continue through the
core: the [T01 bigram baseline](/llm-engineering-course-pages/bigram-baseline), [T02 byte-level BPE](/llm-engineering-course-pages/bpe-tokenizer),
the [T03 data pipeline and embeddings](/llm-engineering-course-pages/data-pipeline), [T04 self-attention](/llm-engineering-course-pages/attention),
the [T05 Mini-GPT decoder](/llm-engineering-course-pages/mini-gpt) and its [T06 core gate](/llm-engineering-course-pages/mini-gpt-gate),
[I01 decoding, sampling and the KV cache](/llm-engineering-course-pages/inference-decoding), then
[E01 reproducible training](/llm-engineering-course-pages/pretraining), [E02 data quality](/llm-engineering-course-pages/data-quality),
[E03 evaluation](/llm-engineering-course-pages/evaluation) and [E04 profiling](/llm-engineering-course-pages/profiling). The company
track [C01–C04](/llm-engineering-course-pages/company-strategy) turns a requirement into a model decision,
auditable data, protected evaluation and continued pretraining; the adaptation
track [A01–A04](/llm-engineering-course-pages/sft-lora) covers SFT/LoRA, the adaptation comparison, DPO and
the RLHF bridge; the [capstone](/llm-engineering-course-pages/company-capstone) ties them into a reversible
company-model decision. The [Applied: RAG track](/llm-engineering-course-pages/rag-build) (R01–R02) is a
standalone entry that needs only Python and one LLM call.

Every required outcome has a CPU route. Run the
[hardware and cost preflight](/llm-engineering-course-pages/hardware) before choosing a larger profile.

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
