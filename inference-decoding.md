---
title: "I01 — Decoding, sampling and the KV cache"
sidebar:
  label: "I01 — Decoding, sampling & KV cache"
---

<span id="i01-decoding-sampling-and-the-kv-cache" />


[← T06 — Core gate](/llm-engineering-course-pages/mini-gpt-gate) · [E01 — Reproducible training →](/llm-engineering-course-pages/pretraining) · [Glossary](/llm-engineering-course-pages/glossary)

Prerequisites: [T05](/llm-engineering-course-pages/mini-gpt), [T06](/llm-engineering-course-pages/mini-gpt-gate). Allow one session.
T05's `MiniGPT.generate` draws a fixed number of tokens at one temperature and
nothing else. Every generation loop makes three further decisions: how one row of
logits becomes one token, when the loop stops, and what it keeps between steps.
The code is `src/llm_course/decoding.py`; read `sample_next`, `decode` and
`step_logits` beside this page. Nothing here trains. The model is read, never
changed, and every function returns a new value instead of editing its input.

## From logits to one token [#from-logits-to-one-token]

Take five logits `[2.0, 1.0, 0.5, 0.0, -1.0]`. Softmax divides each exponential by
their sum. At temperature $T = 1$ the exponentials are 7.3891, 2.7183, 1.6487,
1.0000 and 0.3679, the sum is 13.1239, and the probabilities follow. At $T = 0.5$
the logits are divided by 0.5 first, giving `[4, 2, 1, 0, -2]`, exponentials
54.5982, 7.3891, 2.7183, 1.0000, 0.1353 and sum 65.8408:

| Logit | $T = 1$ | $T = 0.5$ |
| --- | --- | --- |
| 2.0 | 0.5630 | 0.8292 |
| 1.0 | 0.2071 | 0.1122 |
| 0.5 | 0.1256 | 0.0413 |
| 0.0 | 0.0762 | 0.0152 |
| -1.0 | 0.0280 | 0.0021 |

Temperature never reorders the tokens; it changes their ratios. The first entry
beats the second by $e^{1} = 2.718$ at $T = 1$ and by $e^{2} = 7.389$ at $T = 0.5$.
As $T$ approaches zero the first entry approaches 1, so
`sample_next(logits, temperature=0)` skips the division and returns the argmax:
that is greedy decoding, and `distribution(logits, temperature=0)` returns the
matching one-hot vector. For $T > 0$, `distribution` subtracts the largest logit
before dividing, so a temperature of 0.01 still fits in float64, then
`sample_next` draws one index with `torch.multinomial`. The draw takes an optional
`torch.Generator`; the same seed and the same logits give the same token, which is
what "seed 1" means in the lab output below. `tests/test_decoding.py` checks both
columns above within 0.001.

## Sampling that stays on topic [#sampling-that-stays-on-topic]

Sampling at $T = 1$ gives the fifth token 2.8 % of the draws. Over a hundred tokens
that tail adds up, and one improbable token drags every later token with it. Two
masks cut the tail. Top-k keeps the $k$ largest entries and renormalises: with
$k = 2$ the first two entries hold 0.5630 + 0.2071 = 0.7701 of the mass, so they
become 0.5630 / 0.7701 = 0.7311 and 0.2689. Top-p (nucleus) sorts the
probabilities and keeps the shortest prefix whose cumulative mass reaches $p$: the
running sums are 0.5630, 0.7701, 0.8958, 0.9720, so $p = 0.9$ keeps four entries
and drops the fifth. Renormalised by 0.9720 they are 0.5792, 0.2131, 0.1292 and
0.0784. Both exist because they fail differently: top-k keeps two tokens whether
the model is certain or clueless, top-p keeps one token when the first entry
already holds 0.9 and many when the distribution is flat. `distribution` applies
top-k first and then top-p, and the cumulative masses top-p compares against are
those of the full softmax; the single renormalisation comes last.

Now the real model. `examples/run_mini_gpt.py` never writes a checkpoint, only
`report.json`, so the lab trains `artifacts/mini-gpt/model.pt` itself on first
run — T05's recipe, seed 7, 120 steps, about a second on CPU — and reuses that
checkpoint on every run after:

```bash
python examples/run_decoding_lab.py
```

The prompt `'the small model '` becomes 17 tokens with `<bos>`, which leaves 7 of
the 24 context positions for generation. The printed top-5 after the prompt:

```text
  T = 0.0
         'f'  1.0000
         ' '  0.0000
         '!'  0.0000
         ','  0.0000
         '-'  0.0000
  T = 0.7
         'f'  0.3076
         'p'  0.3031
         'r'  0.2944
         'l'  0.0911
         'o'  0.0008
  T = 1.2
         'f'  0.2677
         'p'  0.2654
         'r'  0.2609
         'l'  0.1317
         'o'  0.0082
```

Three characters are within 0.014 of each other and the temperature only moves the
gap to `'l'` and `'o'`. The three continuations at $T = 0.7$ with seeds 1, 2 and 3
are `follows`, `reveals` and `predict`, one per leader. At $T = 1$ the full softmax
gives `'f'` 0.2858, `'p'` 0.2829, `'r'` 0.2771, `'l'` 0.1219 and `'o'` 0.0044; the
running sum is 0.8458 after three entries and 0.9677 after four, so top-p = 0.9 keeps
four of 36 tokens and prints `'f'` as 0.2858 / 0.9677 = 0.2953. Top-k = 5 keeps
`'o'` as well and prints it at 0.0045.

## When to stop [#when-to-stop]

`decode(model, prompt_ids, max_new_tokens=..., eos_id=...)` appends one token per
step and stops after it has appended `eos_id`, so the returned tensor ends with the
EOS token when the stop rule fired. The lab passes `CHARACTER_ENCODER.eos_id`, and
all three samples still report `7 new tokens, length limit`: this checkpoint never
emits `<eos>`. Every training document is 30 to 40 characters long, longer than the
25-token window T05 trains on, so `<eos>` was truncated away from every training
target and the model has never seen it as an answer. The rule is wired and never
fires; the test suite proves it by choosing each token of the greedy continuation
as the EOS id in turn. The second limit is `max_new_tokens`. The third is the
context: `decode` refuses `prompt + max_new_tokens > block_size` with a
`ValueError`, while `MiniGPT.generate` crops to the last `block_size` tokens and
restarts the learned positions at zero. A rolling window keeps generating but
silently forgets the prompt.

[R01](/llm-engineering-course-pages/rag-build)'s default generator, a 4-billion-parameter model, makes the
same choices. `TransformersGenerator.generate` in `src/llm_course/rag/generate.py`
calls
`model.generate(**inputs, max_new_tokens=max_tokens, do_sample=False)` with
`max_tokens=300` and renders the chat template with `enable_thinking=False`.
`do_sample=False` is greedy decoding, temperature 0 in this lesson's terms, so the
same prompt gives the same answer on every run, which is what lets R02 measure
rates. Generation stops at the model's end-of-turn token or after 300 new tokens,
whichever comes first; rule 4 of the system prompt, "at most five sentences", is
what usually ends it long before 300. Streaming is nothing more than this loop with
the print statement inside it: each step yields one token, and a server can send it
before the next step begins.

## Why every step recomputes everything [#why-every-step-recomputes-everything]

`decode` gets its logits from `model(ids[None])[0, -1]`: the full forward over all
$t$ tokens, of which it keeps the last row and discards the rest. Count the work at
prefix length $t$ with width $D = 32$. Each block projects $t$ positions to queries,
keys and values, $3tD^2$ multiply-adds, forms a $t \times t$ score matrix per head,
and runs the FFN over $t$ positions, $8tD^2$ more. Step $t + 1$ does all of it
again for the first $t$ positions, although the causal mask guarantees that their
hidden states, keys and values have not changed: position $i$ never reads a
position after $i$. Per step the projections cost $O(t)$ and the scores $O(t^2)$;
summed over $T$ generated tokens that is $O(T^2)$ and $O(T^3)$. With a cache the
projections cost $O(1)$ per step, $O(T)$ in total, and only the scores keep
$O(t)$ per step, $O(T^2)$ in total: $O(T^2)$ against $O(T)$ for the term that
dominates at this width. At prefix 96 one
block's three projections cost $3 \cdot 96 \cdot 1024 = 294{,}912$ multiply-adds
for a step that only needs the newest token's 3,072.

## The KV cache [#the-kv-cache]

The cache keeps what the mask makes immutable. `KVCache` holds, per block, the keys
and values of every token fed so far, each `[heads, length, head_dim]`, here
`[4, t, 8]`. That is 64 floats per token per block, 128 for both blocks, 512 bytes
in float32; the full 24-token context is 3,072 floats, a tenth of the model's
28,288 parameters. Queries, hidden states and FFN outputs are not stored: each is
used once. `step_logits(model, cache, token_id, position)` embeds one token and its
position, then per block runs `norm_attention`, the block's own q/k/v projections on
that single token, `cache.append(index, key, value)`, and
`scaled_dot_product_attention(query, cache.keys[index], cache.values[index])`: one
query against $t$ keys, no mask, because only the past is stored. The output
projection, residual, `norm_ffn`, `ffn` and second residual follow as in
`DecoderBlock.forward`; `final_norm` and `lm_head` give the logits. `append` returns
a new `KVCache` and leaves the other blocks' tensors shared, so a caller that keeps
the old cache still has it. `prefill` feeds the prompt one token at a time,
`decode_cached` prefills `prompt_ids[:-1]` and then calls `step_logits` once per
generated token. Dropout is skipped; both decoders demand `model.eval()`.

Two paths through different code must give the same logits. The lab prints
`same ids under seed 1: True` and
`max |cached - full| logit difference over 24 positions: 2.92e-06`; the test
asserts a difference below $10^{-4}$ at each of 20 steps. It is not exactly zero because a batched $t \times t$ matmul and a
$1 \times t$ one sum in different orders in float32. The timing table measures a
random-init copy of the T05 architecture with `block_size=128`, because
`measure_decode` times prefixes up to 96 tokens plus 8 generated ones and the
values of the weights do not change the cost. Best of three runs, prefill not
timed, milliseconds per generated token on one laptop CPU:

| prefix | uncached ms | cached ms | speed-up |
| --- | --- | --- | --- |
| 8 | 0.617 | 0.191 | 3.23x |
| 32 | 0.657 | 0.208 | 3.16x |
| 64 | 0.711 | 0.208 | 3.42x |
| 96 | 0.911 | 0.197 | 4.63x |

The uncached column grows about 1.5× while the prefix grows 12×, and the cached
column stays flat. The speed-up at prefix 96 is about 4.6×, not the roughly 96×
that a pure count of recomputed positions would predict, because at width 32 each
step is dominated by the fixed overhead of a few dozen small tensor ops, and the
cached step runs the same number of ops. Your numbers will differ; the shape of
the two columns will not.

## Predict → Trace → Build → Break → Measure → Explain [#predict-trace-build-break-measure-explain]

1. **Predict:** After the shorter prompt `'the '`, how many of the 36 tokens will top-p = 0.9 keep: more or fewer than the four kept after `'the small model '`? Write the number down before running.
2. **Trace:** Run the lab, then `python examples/run_decoding_lab.py --prompt "the "`, and compare the `tokens kept` line with your prediction.
3. **Build:** Implement top-p from the description in the second section and compare with `distribution(logits, temperature=1.0, top_p=0.9)` on the five logits.
4. **Break:** In a copy of `_cached_block`, drop `block.norm_attention` and repeat the cache check. The 2.92e-06 becomes a difference of several units; the equality test is the only guard.
5. **Measure:** Call `measure_decode` on the `block_size=128` timing model (the T05 checkpoint has `block_size=24` and raises `ValueError`) with `max_new_tokens=16` and with `torch.set_num_threads(1)`. Which column moves?
6. **Explain:** Why does R01 decode greedily? Two sentences about reproducible answers and what R02 could not measure with sampling.

### Independent exercise [#independent-exercise]

Write `penalised(logits, seen_ids, penalty)` in `artifacts/mini-gpt/penalty.py`,
without editing `decoding.py`: for every id in `seen_ids`, divide a positive logit
by `penalty` and multiply a negative one by it, the rule of
[CTRL](https://arxiv.org/abs/1909.05858). Feed the result to `sample_next`. Report
the top-5 at $T = 1$ for the lab prompt with penalty 1.0 and 1.5, then the greedy
continuation with penalty 1.5 and 5.

<details>
<summary>Hint</summary>

`seen_ids` is `set(ids.tolist())` and includes the prompt. `'l'` occurs in
`small` and `model`, so it is penalised; `'f'`, `'p'` and `'r'` are not. A
negative logit divided by the penalty would rise, which is why the rule
multiplies it instead.


</details>

<details>
<summary>Reference answer</summary>

With penalty 1.5 the logit of `'l'` falls from 4.005 to 2.670 and its
probability from 0.1219 to 0.0354; `'f'` rises from 0.2858 to 0.3151. The
greedy continuation is still `follows`, because `'f'` was never penalised and
the penalised letters keep their margin at later steps. At penalty 5 it becomes
`frnicts`: on a character vocabulary the rule punishes letters already used,
which is spelling, not repetition. On a word or BPE vocabulary the same rule
discourages a repeated phrase, which is what it was designed for.


</details>

Checkpoint: compute both temperature columns and both masks by hand; state which
stop rule ended a sample; name what the cache stores per block and why the mask
makes it valid; read the timing table without claiming more than it measured.

[← T06 — Core gate](/llm-engineering-course-pages/mini-gpt-gate) · [E01 — Reproducible training →](/llm-engineering-course-pages/pretraining) · [Glossary](/llm-engineering-course-pages/glossary)
