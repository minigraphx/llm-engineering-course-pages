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

## 2026-09-24 — Unit 0: boost one logit yourself [#2026-09-24-unit-0-boost-one-logit-yourself]

Before "Break an assumption", Unit 0 now has an interactive element: pick the prompt
(`"the "` or `"a "`), a target character and a boost from −6 to +6, and watch the
target's probability, the top 5 next characters before and after, both greedy texts,
the corpus loss and the verdict change as you drag. It rebuilds the same counted
bigram table the script uses, so its numbers match the page's summaries; three
buttons replay the page's runs (+4 on m, +3 on t, −4 on t).
→ [Unit 0 — First Tiny-LM Run](/unit-00)

## 2026-09-23 — R01 and R02 are now available in German [#2026-09-23-r01-and-r02-are-now-available-in-german]

Both RAG lessons now have a complete German version, from preparing the corpus and
the embedding trace through the guardrails, the reference run and the independent
exercises. Commands, code and program output stay in English, as on the other German
pages; the questions you send to the index stay English because the index is built
from the English pages.
→ [R01 — From document to answer](/rag-build) · [R02 — So it does not lie](/rag-quality)

## 2026-09-23 — Unit 0: a ten-line summary instead of a JSON dump [#2026-09-23-unit-0-a-ten-line-summary-instead-of-a-json-dump]

`run_unit0.py` now prints ten lines — prompt and token IDs, the top candidates,
both generations, the target's probability and the corpus loss before → after, and
one computed verdict — and saves the full report to `artifacts/unit0/report.json`;
`--json` prints it. The page explains every line, walks through the full report
once, and shows why `--boost 3` moves a probability without changing the text.
`diagnose_unit0.py` prints one line per check, and an empty prompt now gives a
one-line error instead of a traceback. AG01 now runs `run_unit0.py` without
redirecting its output. Characters with equal probability are now listed by token
ID, so the top-candidate lists in Unit 0's and T01's reports read the same on
every platform.
→ [Unit 0 — First Tiny-LM Run](/unit-00)

## 2026-09-23 — R01 and R02: every number measured on today's course [#2026-09-23-r01-and-r02-every-number-measured-on-todays-course]

The RAG lessons quoted a reference run from before the agent pages joined the corpus.
Every number R01, R02 and the RAG README quote now comes from a fresh run on today's
41 pages: `improved` still beats `naive` on retrieval (Recall@1 0.878 against 0.561),
but its lexical groundedness is lower (0.557 against 0.631), and R02 now says so
instead of calling it unchanged. The offline example in R01 shows a different top
passage. A test now flags when a new page moves these numbers.
→ [R02 — So it does not lie](/rag-quality) · [R01 — From document to answer](/rag-build)

## 2026-09-23 — AG03: measure what a defence against prompt injection buys [#2026-09-23-ag03-measure-what-a-defence-against-prompt-injection-buys]

One attack set, the same poisoned document, run against R02's retrieval pipeline and
AG02's agent through a cumulative defence ladder — delimiters, escaping,
spotlighting, a pattern filter, ingest cleaning, a confirmation step, then scoping
the send tool so no call can choose a recipient. Every measurement runs on local
Qwen models only, never a hosted API. On the small model, retrieval success falls
from 7 of 8 to 3 of 8 and never reaches zero; the 4B agent never takes the bait at
any step, yet the same defences still cost it the task, from 6 of 8 completions
down to 0 of 8.
→ [AG03 — Prompt injection](/agent-injection)

## 2026-09-23 — R02: defeat the injection filter [#2026-09-23-r02-defeat-the-injection-filter]

The independent exercise no longer asks for a seventh regex. You now write two
phrasings of the 999-euro attack that slip past all six patterns, watch the filter
drop R02's own Guardrails section, and explain which guardrail still catches you.
The passage tags are escaped, so a document can no longer close its own tag and
forge a passage.
→ [R02 — So it does not lie](/rag-quality)

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
