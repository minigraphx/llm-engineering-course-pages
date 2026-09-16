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

The standalone foundations track is now executable: Python/NumPy, tensor
shapes, probability and cross-entropy, a trainable neuron, MLP/backpropagation,
a tiny autograd engine, and a PyTorch checkpoint. Godot-RL graduates can take
the [compact bridge](/llm-engineering-course-pages/foundations-godot-rl-bridge) and prove only the missing
LLM-specific concepts. Both paths now continue into the executable
[bigram language-model baseline](/llm-engineering-course-pages/bigram-baseline), the from-scratch
[byte-level BPE tokenizer](/llm-engineering-course-pages/bpe-tokenizer), the versioned
[data pipeline and embedding path](/llm-engineering-course-pages/data-pipeline), and
[self-attention from first principles](/llm-engineering-course-pages/attention). Continue with the
[Mini-GPT decoder](/llm-engineering-course-pages/mini-gpt) and its [core gate](/llm-engineering-course-pages/mini-gpt-gate), then
[reproducible training](/llm-engineering-course-pages/pretraining), [data quality](/llm-engineering-course-pages/data-quality),
[evaluation](/llm-engineering-course-pages/evaluation) and [profiling](/llm-engineering-course-pages/profiling).

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

Unit 0 now has an executable Tiny-LM vertical slice and is ready for its
required five-learner pilot. The diagnostic, hardware gates, and neural-
foundations track, reproducible bigram baseline, byte-level BPE lab,
versioned data pipeline with embeddings, and self-attention lab are
available. M2 adds the decoder, core gate and model card; M3 adds resumable
CPU training, provenance audits, paired evaluation and measured profiling.
Later company-model units and larger supplied datasets remain planned.
The real five-learner pilot is still open; automated checks do not establish
learner comprehension or issue a certificate.

</aside>
