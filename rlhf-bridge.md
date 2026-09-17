---
title: "A04 — RLHF bridge: inspect the failure mode"
sidebar:
  label: "A04 — RLHF bridge"
---

<span id="a04-rlhf-bridge-inspect-the-failure-mode" />


[← A03 — DPO](/dpo) · [Capstone — Company model →](/company-capstone) · [Glossary](/glossary)

Prerequisites: [A03](/dpo). Allow one session. The code is the second half of
`src/llm_course/alignment.py`: `reward_pair_loss`, `categorical_kl`,
`ppo_clipped_loss`, `_fit_reward_model` and `_ppo_policy`. The page is optional
for the course path and required before touching any RLHF library.

## Fit a reward model with a hole in it [#fit-a-reward-model-with-a-hole-in-it]

The optional bridge fixes a reference policy, fits a reward model from preference
pairs, adds a KL penalty and updates a finite-action policy with PPO. The action
table includes billing, access, export and secret-request routes. A planted
confidence shortcut makes reward hacking observable. Each of the four actions per
context is described by five features: completeness, confident style, process
language, brevity and a shortcut marker. The harmful action (index 3) has
confidence `4.0` and the marker `1.0`; the correct action has confidence `0.6`.
The reward model is a linear layer over those features trained with
`reward_pair_loss`, the Bradley–Terry negative log-likelihood
`-log sigmoid(r_chosen − r_rejected)`, which is `log 2 = 0.6931` for equal
rewards and falls as the chosen reward pulls ahead.

The naive fit sees only the pairs `(0,1)`, `(0,2)` and `(1,2)`: nobody ever
labelled the shortcut as rejected. The controlled fit adds `(0,3)` and `(1,3)`.
After 120 Adam steps both reach training pair accuracy `1.0`, and both are
deterministic: the weights start at zero and no sampling is involved. The naive
scores are `6.45, 2.66, -2.24, 10.46`, so the shortcut is the best-rewarded
action although its true quality is `-0.5`; the controlled scores are
`5.47, 2.79, -2.25, -2.43`. A perfect pair accuracy told you nothing about the
action that was never in a pair.

## KL and the clipped ratio [#kl-and-the-clipped-ratio]

`categorical_kl` averages `sum(p × (log p − log p_ref))` over contexts; it is
`0.0` when the policy equals the reference. `ppo_clipped_loss` forms the ratio
`exp(log_probs − old_log_probs)` and clamps it to `[1 − ε, 1 + ε]` with
`ε = 0.2`. Trace one action whose probability rose from 0.4 to 0.5: the ratio is
`1.25`. With advantage `+1` the surrogate is `min(1.25, 1.2) = 1.2`, loss
`-1.2`: the clip caps how much credit one update can take. With advantage `-1`
it is `min(-1.25, -1.2) = -1.25`, loss `1.25`: a move that hurt is charged in
full. `_ppo_policy` samples 32 actions per context from the current policy,
refreshes them every four updates, uses `reward − expected reward` as the
advantage and minimises `surrogate + 0.08 × KL` with Adam.

## Read the three quantities [#read-the-three-quantities]

Read `alignment.py` line by line before using a library trainer. Check three
quantities after the run: proxy reward, true quality and hack rate. A policy is
not safer merely because its learned reward increased. Critical routing remains
an application gate and must be evaluated separately.

Run the CPU recipe from [DPO](/dpo). For an accelerator, keep the same seed,
data and evaluator; retain the CPU fallback when the accelerator is unavailable.
With `--dpo-steps 80 --ppo-steps 60` and seed 7 the report records:

| Policy | Proxy reward (naive scale) | True quality | Hack rate | KL from reference |
| --- | ---: | ---: | ---: | ---: |
| Reference | 5.011 | 0.713387 | 0.055566 | 0.0 |
| Naive PPO | 5.643 | 0.684654 | 0.108824 | 0.1374 |
| Controlled-reward PPO (`ppo.controlled_proxy`) | 5.374 | 0.785538 | 0.041577 | 0.1215 |

Both PPO runs use `kl_coefficient=0.08`; what differs is the reward model, so
the controlled row is the one fitted with the shortcut pairs. Naive PPO raises
the proxy it was given and lowers the quality it was not, and its hack rate
nearly doubles. All three policies still pick the correct action
greedily in every context: the damage sits in probability mass, so a check that
only reads the argmax would report no change. The controlled row is better than
the reference on this table and nothing more; `limitations` in the report says
why it is not a safety claim.

## Predict → Trace → Build → Break → Measure → Explain [#predict-trace-build-break-measure-explain]

1. **Predict:** Which action will the naive reward model score highest? Decide from the feature rows before reading `naive_action_scores`.
2. **Trace:** Run `run_alignment.py --dpo-steps 80 --ppo-steps 60` and find the three quantities for `ppo.naive_proxy` and `ppo.controlled_proxy`.
3. **Build:** Reproduce the ratio trace with `ppo_clipped_loss(torch.log(torch.tensor([0.5])), torch.log(torch.tensor([0.4])), torch.tensor([1.0]))`.
4. **Break:** In a copy of `_ppo_policy`, set `kl_coefficient=0.0` for the naive reward and compare `hack_rate` and `kl_from_reference` with the seed-7 row; the change after 60 updates is small, so decide whether the coefficient or the reward model carried the effect in the table.
5. **Measure:** Run seed 11 and seed 23 with the same budgets; record proxy reward, true quality and hack rate for both PPO rows.
6. **Explain:** Say why the reward model's pair accuracy of `1.0` was not evidence, in one sentence that names the missing pair.

### Independent exercise [#independent-exercise]

Define one rollback monitor for this policy from the measured rows: name the
quantity, its threshold and the artifact you would reload. Then run two more
seeds and report whether the ordering naive hack rate > reference > controlled
holds; where it does not, write down the number that broke it.

<details>
<summary>Hint</summary>

Hack rate is `mean_probabilities[3]`, the mass on the harmful action; true
quality is the planted value, not something a deployed system can read. The
seed changes only which actions PPO samples; the reward fits do not change.


</details>

<details>
<summary>Reference answer</summary>

A defensible monitor is the hack rate on the frozen action table with a
threshold below the seed-7 reference value `0.055566`, reloading the reference
logits `[1.4, 0.5, -0.6, -1.0]` when it is crossed. The two extra seeds are
your measurement; the naive reward scores `6.45, 2.66, -2.24, 10.46` are the
same for every seed, so any change you see comes from PPO's sampling.


</details>

Checkpoint: describe one monitor that would trigger rollback (for example, a
critical safety failure or a rising hack rate) and name the immutable reference
needed to reproduce it.

[← A03 — DPO](/dpo) · [Capstone — Company model →](/company-capstone) · [Glossary](/glossary)
