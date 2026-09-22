---
title: "What's new"
sidebar:
  label: "What's new"
---

<span id="whats-new" />


[Course home](/)

Dated changes a learner can notice, newest first. Each entry links to the page
that changed. Internal work — tests, tooling, dependency updates — is not
listed here; the Git history has it.

## 2026-09-22 — AG02: the same agent on native tool use [#2026-09-22-ag02-the-same-agent-on-native-tool-use]

Run AG01's agent three ways on one question — the text protocol on Claude, Claude's
native tool use, and Qwen3's own `<tool_call>` format — and read what the API took
over (the schema, the parse, the call ids) against what is still yours. Includes the
first Break exercise where the API refuses a broken conversation.
→ [AG02 — Native tool use](/agent-native)

## 2026-09-21 — F01 takes the Python basics one idea at a time [#2026-09-21-f01-takes-the-python-basics-one-idea-at-a-time]

The section that introduced slices, loops, `zip`, functions, `assert` and
pytest in one go is now five short steps, each with one thing to predict. A
slice visualizer on the page lets you type a slice and see which positions of
`["a", "b", "c", "d"]` light up, and line up two slices to see the pairs `zip`
produces — the bigram idea in miniature.
→ [F01 — Python & NumPy](/foundations-01-python-numpy)

## 2026-09-21 — AG01: build an agent loop by hand [#2026-09-21-ag01-build-an-agent-loop-by-hand]

The first unit of the new **Applied: Agents** group is live. You build an agent
with nothing hidden — a plain-text tool protocol you parse yourself, a step
budget, and a trace that shows what the model actually saw — compare two local
models on the same question, and watch a small model invent a source. Two more
agent units are planned and marked *(coming soon)* on the home page.
→ [AG01 — The agent loop, by hand](/agent-loop)

## 2026-09-20 — Unit 0 no longer assumes what the course teaches later [#2026-09-20-unit-0-no-longer-assumes-what-the-course-teaches-later]

The whole model is stated in three sentences before the first prediction, every
technical term is defined where it first appears, and the repetitive output is
explained as the expected result rather than a broken run. The Python check
gains a hand-check route for learners who do not program yet.
→ [Unit 0 — First Tiny-LM Run](/unit-00)

## 2026-09-19 — The home page says what you will be able to do [#2026-09-19-the-home-page-says-what-you-will-be-able-to-do]

Nine skills in plain language, each linked to the unit that proves it, and a
list of planned units marked *(coming soon)* so you can see what is not written
yet.
→ [Course home](/)

## 2026-09-19 — Applied: RAG moved below the Course group [#2026-09-19-applied-rag-moved-below-the-course-group]

The retrieval track now sits after the core course in the sidebar. Its content
and its route in the learning guide are unchanged.
→ [R01 — From document to answer](/rag-build)

## 2026-09-17 — The course is served from llm.onlinekurs.training [#2026-09-17-the-course-is-served-from-llmonlinekurstraining]

The published site moved to its own domain. Old links redirect.

[Course home](/)
