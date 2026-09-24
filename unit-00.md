---
title: "Unit 0 — First Tiny-LM Run"
sidebar:
  label: "Unit 0 — First Tiny-LM Run"
---

<span id="unit-0-first-tiny-lm-run" />


[← Learning guide](/learning-guide) · [Unit 0 troubleshooting →](/unit-00-troubleshooting) · [Glossary](/glossary)

## Learning outcome [#learning-outcome]

In 45–60 minutes, you will run a prepared Tiny-LM, generate from your own
prompt, inspect token IDs, probabilities, shapes, and loss, change one logit,
and explain the measured effect. No prior ML knowledge, learner data, cloud, or
accelerator is required.

You need to be able to type commands in a terminal. Everything else — including
every technical word on this page — is defined where it first appears. Each new
term is also a forward reference: [F03](/foundations-03-probability) and
[T01](/bigram-baseline) build the same ideas properly later. Here you only
need them well enough to predict, measure, and explain one change.

## 0–10 min · Predict and check setup [#010-min-predict-and-check-setup]

Before predicting anything, you need something to predict from. This is the
whole model in three sentences:

1. It has read a small amount of text and remembers, for every character, which
   characters tended to follow it.
2. To write, it looks at the character it just produced and picks the character
   that followed it most often.
3. It feeds that character back in as the new input, and repeats.

That is all. There is nothing else in this model.

Now write down two predictions. You will check both against measurements before
the hour is out.

1. Starting from the prompt `the `, what will the model write?
2. If you force `m` to be the most likely character after a space, will the
   written text change?

Then check your setup:

```
python examples/diagnose_unit0.py
```

You should see five lines like these:

```
python      pass  3.12.13
backend     pass  mps
artifacts   pass  artifacts/unit0-diagnostic
generation  pass  "the te t"
ready       yes
```

Each line names one check, its result, and what it found: your Python version
(3.11 or newer is needed); the **backend**, the processor that will do the
arithmetic — `cpu`, `mps` (an Apple GPU) or `cuda` (an NVIDIA GPU); the folder
where the prepared files were written; and a first smoke test — the prompt
`the ` plus the four characters the model wrote after it. The last line says whether Unit 0 is ready.
If any line says `fail`, use the [troubleshooting guide](/unit-00-troubleshooting).

## 10–20 min · First generation [#1020-min-first-generation]

```
python examples/run_unit0.py
```

The command prints a ten-line summary. This is the real output on the course
author's machine. Your `backend` line may say `cpu`; with `--device cpu` every
other line is identical:

```
backend      mps
prompt       "the " → token IDs [26, 14, 11, 1]
top 5 next   "t" 0.263 · "p" 0.176 · "a" 0.089 · "f" 0.089 · "m" 0.089
baseline     "the te te te te te te te te "
change       +4.0 on the score for "m" after " "
changed      "the mode mode mode mode mode"
probability  "m" after " ": 0.089 → 0.843
corpus loss  1.695 → 1.890 (worse; lower is better)
verdict      the probability of "m" rose and the text changed
full report  artifacts/unit0/report.json
```

<aside className="course-note" aria-label="What you should see">
<strong>What you should see</strong>

The `baseline` text looks repetitive — `"the te te te te te te te te "`.
That is the correct result, not a broken run. A model that only remembers
*one* character of context cannot do better, and seeing exactly how it
fails is the point of this unit. The whole rest of the course is about
giving a model more to work with.


</aside>

Read it line by line. Quotation marks show exactly where a piece of text starts
and ends, so `" "` is a single space.

- `backend` — which processor ran it, as in the diagnostic.
- `prompt` — your prompt, then the same prompt with each character replaced by
  its number in the model's alphabet. Those numbers are **token IDs**: `t` is 26
  and the space is 1.
- `top 5 next` — the five characters the model considers most likely right
  after the prompt's last character, each with its **probability**: a number
  from 0 to 1, where 0.263 means about 26 %. Equal probabilities are listed by
  token ID, so a tie can straddle the cut: `s` also has 0.089 and just misses
  the list.
- `baseline` — what the model wrote: your prompt plus 24 new characters.
- `change` — every run also makes one deliberate edit to a copy of the model.
  Here it makes `m` more likely after a space; the next section explains what
  the "score" is.
- `changed` — what the edited copy wrote from the same prompt.
- `probability` — how likely `m` is after a space, before → after the edit.
- `corpus loss` — one number for how surprised the model is by the text it
  learned from, before → after the edit; lower is better.
- `verdict` — one sentence the script computes from the numbers above; no
  model writes it.
- `full report` — the file with everything the run measured, in more detail.
  You will open it once, later in this unit.

Compare the `baseline` line with your first prediction now.

The first run materializes two ignored local artifacts under `artifacts/unit0/`:

- a deterministic 596-byte synthetic corpus dedicated to CC0-1.0;
- a prepared character-bigram checkpoint derived from adjacent-character counts.

Their hashes and provenance are written to `manifest.json`. They are generated
locally because datasets and checkpoints must not be committed to Git. The
report sits beside them as `report.json`, and every run replaces it.

## 20–30 min · Trace the model [#2030-min-trace-the-model]

The model knows 33 different characters. That set is called the **vocabulary**,
and its size is written `V`, so `V=33`.

For the character it just saw, the model holds one score for every character
that could come next — 33 scores. Each score is called a **logit**: an
unnormalised score, where bigger means more likely, but which is not yet a
percentage. All of them together form a **transition matrix** with shape
`[V,V]`: one row per current character, one column per possible next character,
so 33 × 33 = 1089 numbers, and the current character selects exactly one row.

Turning one row of 33 logits into 33 probabilities that add up to 1 is called
**softmax**. Choosing the single highest one and feeding that character back in
is called **greedy decoding** — the loop you predicted from in the first ten
minutes.

`corpus loss` is the **cross-entropy**: for every character in the corpus it
measures how much probability the model gave to the character that actually
came next, then averages. A lower value means the model was less surprised.

If you already program, verify the shapes yourself for a four-character prompt:

```
from llm_course import run_vertical_slice

report = run_vertical_slice("the ", preferred_device="cpu")
assert len(report["prompt_token_ids"]) == 4
assert len(report["baseline"]["top_next_tokens"]) == 5
assert 0.0 <= report["baseline"]["target_probability"] <= 1.0
```

If you do not program yet, do not skip this step — read the three `assert`
lines as three claims and check them by hand in the summary you already have:
the `prompt` line shows four token IDs for the four-character prompt, the
`top 5 next` line lists five candidates, and the first number on the
`probability` line — the target's probability before the edit — is between 0
and 1. You will write this code yourself in [F01](/foundations-01-python-numpy).

## 30–42 min · Build one controlled change [#3042-min-build-one-controlled-change]

The reference run adds `4.0` to one transition logit — the "score" on the
`change` line: after the prompt's final space, make `m` more likely. Because the
change is made to a logit and not to a probability, `+4.0` does not mean "four
more percent" — softmax decides how much of the probability actually moves, and
it takes it from the other candidates.

In the summary of your first run, read:

- the `probability` line — the target probability before and after;
- the `baseline` and `changed` lines — whether the generated continuation
  changed;
- the `corpus loss` line — whether the corpus loss improved or worsened.

This is where you check your second prediction.

### The full report [#the-full-report]

The summary is computed from a larger report. Look at it once:

```
python examples/run_unit0.py --json
```

`--json` prints the whole report instead of the summary; it is the same content
as `artifacts/unit0/report.json`. Here it is with the two five-candidate lists
shortened to their first entry and the two fingerprints cut after eight
characters:

```
{
  "backend": "mps",
  "model": "character-bigram",
  "vocabulary_size": 33,
  "assets": {
    "artifact_dir": "artifacts/unit0",
    "corpus_path": "artifacts/unit0/tiny_corpus.txt",
    "checkpoint_path": "artifacts/unit0/tiny_bigram.pt",
    "manifest_path": "artifacts/unit0/manifest.json",
    "dataset_sha256": "6d10e715[…]",
    "checkpoint_sha256": "855912a6[…]"
  },
  "prompt": "the ",
  "prompt_token_ids": [
    26,
    14,
    11,
    1
  ],
  "baseline": {
    "generation": "the te te te te te te te te ",
    "corpus_loss": 1.6954470872879028,
    "top_next_tokens": [
      {
        "token": "t",
        "token_id": 26,
        "probability": 0.2628726065158844
      },
      […]
    ],
    "target_probability": 0.08943088352680206
  },
  "controlled_change": {
    "source_character": " ",
    "target_character": "m",
    "logit_boost": 4.0,
    "generation": "the mode mode mode mode mode",
    "corpus_loss": 1.8896712064743042,
    "top_next_tokens": [
      {
        "token": "m",
        "token_id": 19,
        "probability": 0.8428245186805725
      },
      […]
    ],
    "target_probability": 0.8428245186805725
  },
  "measured_effect": {
    "target_probability_delta": 0.7533936351537704,
    "generation_changed": true
  }
}
```

Read a dotted name as a path into it: `baseline.corpus_loss` means the
`corpus_loss` field inside `baseline`. The blocks, top to bottom:

- `backend`, `prompt` and `prompt_token_ids` — the first two summary lines.
- `model` — the kind of model: a character bigram, one character of context.
  `vocabulary_size` — `V`, the 33 characters from the previous section.
- `assets` — provenance: where the corpus, the checkpoint and the manifest are,
  and a SHA-256 fingerprint of the corpus and of the checkpoint (a 64-character
  code that changes if a single byte of the file changes). It proves which data
  and which model produced these numbers. You can skip it on day one.
- `baseline` — the unedited model: its `generation`, its `corpus_loss`, its
  `top_next_tokens` (each with its token ID), and `target_probability`, the
  probability of the target character after the prompt.
- `controlled_change` — the same fields for the edited copy, plus which cell
  was edited: `source_character` (the row), `target_character` (the column) and
  `logit_boost` (how much was added).
- `measured_effect` — the answer to this section's question, already computed:
  `target_probability_delta` is after minus before (0.843 − 0.089 ≈ 0.753), and
  `generation_changed` says whether the two texts differ. The `verdict` line is
  written from these two fields.

Keep the summary for everyday runs; open the report when you need a number the
summary rounds or leaves out.

### Your own change [#your-own-change]

Now use your own prompt and target character:

```
python examples/run_unit0.py --prompt "a " --target-character t --boost 3
```

Predict the direction before running. The change affects exactly one cell in
the `[V,V]` matrix, so it can help one transition while harming the overall
corpus loss.

<details>
<summary>Check your prediction</summary>

```
backend      mps
prompt       "a " → token IDs [7, 1]
top 5 next   "t" 0.263 · "p" 0.176 · "a" 0.089 · "f" 0.089 · "m" 0.089
baseline     "a te te te te te te te te "
change       +3.0 on the score for "t" after " "
changed      "a te te te te te te te te "
probability  "t" after " ": 0.263 → 0.877
corpus loss  1.695 → 1.828 (worse; lower is better)
verdict      the probability of "t" rose, but the text did not change
full report  artifacts/unit0/report.json
```

The probability of `t` more than tripled, yet `baseline` and `changed` are the
same text, and the verdict says so. Look at `top 5 next`: `t` was already the
most likely character after a space. Greedy decoding only asks which character
is highest, and `t` was highest before and after, so the edit made the model
more certain without changing what it writes. A probability and the pick made
from it are different things: the pick — the position of the highest value —
is called the **argmax**. Later units build on this, starting with
[I01](/inference-decoding), where sampling draws from the probabilities
instead of always taking the argmax.


</details>



### Interactive · Boost one logit

**Interactive: boost one logit.** The browser rebuilds Unit 0's counted bigram table from the same corpus. Default as in `python examples/run_unit0.py`: prompt "the ", +4.0 on the score for "m" after " ". Probability of "m" after " ": 0.089 → 0.843; `baseline` `"the te te te te te te te te "`, `changed` `"the mode mode mode mode mode"`; corpus loss 1.695 → 1.890 (worse; lower is better); `verdict`: the probability of "m" rose and the text changed. In the browser you choose the prompt ("the " or "a "), the target character and a boost from −6 to +6 in steps of 0.5, or one of the page's three runs (+4 on m, +3 on t, −4 on t), and see the five most likely next characters before and after, both texts, the loss and the verdict.

## 42–50 min · Break an assumption [#4250-min-break-an-assumption]

Reverse the intervention:

```
python examples/run_unit0.py --prompt "a " --target-character t --boost -4
```

The target probability should fall. If your explanation still says the token
became more likely, your interpretation—not the model—is broken. Predict
whether the text changes as well, then run it.

<details>
<summary>Check your prediction</summary>

```
backend      mps
prompt       "a " → token IDs [7, 1]
top 5 next   "t" 0.263 · "p" 0.176 · "a" 0.089 · "f" 0.089 · "m" 0.089
baseline     "a te te te te te te te te "
change       -4.0 on the score for "t" after " "
changed      "a pa pa pa pa pa pa pa pa "
probability  "t" after " ": 0.263 → 0.006
corpus loss  1.695 → 1.815 (worse; lower is better)
verdict      the probability of "t" fell and the text changed
full report  artifacts/unit0/report.json
```

This time the text changes too: with `t` pushed down, `p`, second in
`top 5 next`, becomes the highest.


</details>

Then try an empty prompt:

```
python examples/run_unit0.py --prompt ""

run_unit0.py: error: --prompt must contain at least one character (characters outside the vocabulary are read as "?")
```

This is the expected error, not a crash: the script checks its input before it
runs the model, prints one line and stops without writing a report. A model
that predicts the next character needs at least one character to start from;
any character works, and one the model does not know is read as `?`. Read the
boundary error instead of removing the validation.

In Windows PowerShell the empty quotes can be dropped; you then see
`argument --prompt: expected one argument` — the same kind of input check, also
with exit code 2.

## 50–55 min · Measure reproducibility [#5055-min-measure-reproducibility]

Greedy decoding always takes the highest score, so it gives the same answer on
any backend. Sampling picks randomly among the candidates instead, so it needs a
seed — a fixed starting point for the randomness — to repeat itself. Sampling
here is performed on CPU so the same seed follows the same path (the `backend`
line still names your device, which runs the model; only the probabilities
are copied to the CPU for the random draws):

```
python examples/run_unit0.py --prompt "the " --sample --seed 11
```

Before running, predict whether the sampled text will look like the greedy
`te te te`. Run it twice and compare the generations. Accelerator users may
also run `--device cpu`; the mandatory evidence is always available on CPU.

<details>
<summary>Check your prediction</summary>

```
backend      mps
prompt       "the " → token IDs [26, 14, 11, 1]
top 5 next   "t" 0.263 · "p" 0.176 · "a" 0.089 · "f" 0.089 · "m" 0.089
baseline     "the aroke fchu.\nty thea tts?"
change       +4.0 on the score for "m" after " "
changed      "the m mke fchu.\nty thea mob?"
probability  "m" after " ": 0.089 → 0.843
corpus loss  1.695 → 1.890 (worse; lower is better)
verdict      the probability of "m" rose and the text changed
full report  artifacts/unit0/report.json
```

Sampled text looks nothing like the greedy `te te te`: it no longer always
takes the top candidate. `\n` inside the quotes is a line break the model
wrote.


</details>

## 55–60 min · Explain [#5560-min-explain]

Complete these sentences in your lab notes:

1. The prompt becomes token IDs by ...
2. The current character selects one row of scores, and that row has ... numbers
   in it, because ...
3. Softmax is needed because ...
4. The controlled change moved probability from ... to ...
5. The generation changed/did not change because ...
6. Corpus loss changed because the edited transition ...

## Completion gate [#completion-gate]

You pass Unit 0 when the diagnostic is ready, your own prompt generates text,
you can point to the token IDs, the probabilities and the loss in your own
report, and your prediction and explanation match the measured intervention.
Explaining it in your own words counts; reusing the words on this page does not.
Keep the full report of your own run, `artifacts/unit0/report.json`, as
evidence. Every run replaces it, so copy it once you have the run you want to
keep.

Course release acceptance additionally requires the documented
[five-learner pilot](/sources/unit-00-pilot).

## What's next [#whats-next]

Continue with the [diagnostic and learning-path check](/diagnostic). It
determines which foundations you can safely skip and preserves a re-entry link
for every shortcut.

[← Learning guide](/learning-guide) · [Unit 0 troubleshooting →](/unit-00-troubleshooting) · [Glossary](/glossary)
