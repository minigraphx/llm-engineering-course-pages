---
title: "Capstone — From requirement to a reversible company model"
sidebar:
  label: "Capstone — Company model"
---

<span id="capstone-from-requirement-to-a-reversible-company-model" />


[← A04 — RLHF bridge](/rlhf-bridge) · [Course home →](/) · [Glossary](/glossary)

Prerequisites: [C01](/company-strategy), [A02](/adaptation-comparison), [C04](/continued-pretraining).
Allow two sessions: one to rerun and collect the evidence, one to write the
record. You reuse `src/llm_course/company_strategy.py` (`Requirements`,
`score_candidates`, `total_cost`) and every module from C02 to A04; nothing new is trained.

## Scope and the seven steps [#scope-and-the-seven-steps]

Use only the committed code, manifests and baseline reports. Do not add real
customer data or tune on the sealed gold cases.

1. Write five must/can requirements for Nordlicht support.
2. Rebuild the approved snapshot and prove its hash and contamination report.
3. Score both transparent baselines on development cases.
4. Compare prompting/retrieval, DAPT, full SFT/LoRA and DPO using the measured
   reports; record trainable parameters, latency/memory scope and failure cases.
5. Select a candidate only after hard safety, provenance and local-processing
   gates pass.
6. Run the sealed review with a human, then record the decision and corrections.
7. Specify monitoring, a rollback checkpoint and an owner for each alert.

## Gates before ranking [#gates-before-ranking]

`score_candidates(candidates, requirements)` applies every must-gate before it
sorts: offline processing, a reviewed license, `quality >= min_quality`,
`latency_ms` and `memory_gib` within their budgets. Ineligible
candidates keep their `reasons` list and sort last; eligible ones sort by
quality, then latency, then memory. Trace C01's hypothetical pair with
`Requirements(need="format")`: an online candidate at quality 0.95 and a local
one at 0.80. The online row returns `eligible: false` with the single reason
`offline processing required`, and the local row ranks first although its
quality is lower. Averaging the two numbers would have picked the wrong system.

Now feed the same function the course's measured candidates with development
exact accuracy as `quality`: `keyword_router` 0/8, full SFT 0/8 and rank-4 LoRA
0/8. Every row fails the default 0.7 gate with the reason `quality gate failed`,
the eligible list is empty, and the transparent baseline stays deployed. That is
the committed outcome in `docs/company-model-decision.md`; the capstone asks you
to rebuild it from the reports rather than to improve on it.

## Evidence inventory [#evidence-inventory]

Each step has a committed artifact and a number you can regenerate on CPU.

| Step | Command or artifact | What it must reproduce |
| --- | --- | --- |
| 2 | `build_snapshot()` from C02 | 12 approved documents, SHA-256 `d220da8e…acc8288` |
| 2 | `contamination_report(instruction_records("train"))` | no prompt and no response overlap with the 8 sealed gold cases (`d0164978…2aacc6`) |
| 3 | C03's `score_predictions` on both baselines | `keyword_router` route rate 1.0, exact 0.0, 0 critical failures; `always_escalate` route rate 0.25 |
| 4 | `examples/run_continued_pretraining.py --seeds 7 19 43` | domain gain 0.359–0.411, general regression 0.032–0.109, `keep_for_further_review` |
| 4 | `examples/run_sft_lora.py --steps 120`, `docs/baselines/sft-lora-v1.md` | full SFT train/dev NLL 0.488/3.816, LoRA 3.768/4.469, 0/8 correct for both |
| 4 | `examples/run_open_model_lora.py --download --device cpu`, `docs/model-cards/company-adapter-v1.md` | 230,400 of 134,745,408 trainable parameters, response NLL 4.246→3.124, JSON still unreliable |
| 4 | `examples/compare_dpo_reference.py`, `examples/run_alignment.py --dpo-steps 80 --ppo-steps 60` | TRL difference 0.0; naive PPO hack rate 0.108824 against controlled 0.041577 |
| 5 | `examples/run_company_strategy.py` | primary `sft_lora` for `need="format"`, four rejected alternatives, TCO 359 units |
| 6 | `docs/reviews/company-gold-v1.md` | reviewer, date and decision fields are **pending** until a person fills them |
| 7 | `general-reference.pt` from C04, the pre-DAPT MiniGPT base | `logits_exact: true` after reload: the DAPT candidate rolls back to this checkpoint |
| 7 | the SmolLM2 base at revision `12fd25f7…` reloaded without the adapter (`docs/model-cards/company-adapter-v1.md`) | a rejected adapter rolls back by one reload of the pinned base, not a retrain |

Latency is the one column the course has not measured for these candidates.
Your record must contain it with the device, thread count and input length, or
must say that the latency gate was not evaluated.

## The decision record [#the-decision-record]

Your submission is a short decision record linking every claim to a command,
hash, report or review row. Include two rejected alternatives and the evidence
that would change the decision. The model may suggest a route; application code
enforces authorization, secrets handling and export permissions. Follow the shape
of `docs/company-model-decision.md`: scope, decision, reasons, rejected
alternatives and the evidence that would change it. Step 6 is the only step a
person must perform: copy the review table, fill in reviewer, date, decision and
corrections yourself, and if a gold answer changes, recompute the seal and rerun
every baseline before you cite either.

Exit criteria: another learner can run the CPU recipes from a clean checkout,
explain every metric, reproduce the baseline hashes and identify the exact step
that rolls back a rejected adapter.

## Predict → Trace → Build → Break → Measure → Explain [#predict-trace-build-break-measure-explain]

1. **Predict:** Before scoring, write which of the four candidates (`keyword_router`, full SFT, rank-4 LoRA, the SmolLM2 adapter) would pass the 0.7 quality gate; then check the reports.
2. **Trace:** Follow one development case, `dev-escalate-02`, from C03's rubric through the `keyword_router` prediction to its row in `case_results`.
3. **Build:** Call `score_candidates` with the measured candidates and `Requirements(need="format")`; print every `reasons` list.
4. **Break:** Set `min_quality=0.0`: every row becomes eligible although `score_candidates` never reads `critical_failure_count`; the safety gate is yours to add to the record.
5. **Measure:** Time `baseline_predictions(cases, "keyword_router")` and one model candidate on the eight development prompts; record milliseconds, device and threads.
6. **Explain:** Name the artifact each rejected alternative would need in order to change the decision.

### Independent exercise [#independent-exercise]

Write the `score_candidates` call for the same four candidates, `keyword_router`,
full SFT, rank-4 LoRA and the SmolLM2 adapter, with the development exact
accuracy as quality, the latency you measured and the memory scope each report declares. Decide with `Requirements(need="format")`, then
state which single new artifact would make one candidate eligible.

<details>
<summary>Hint</summary>

`quality` is a fraction in `[0, 1]`; use correct cases over eight. Memory
values must name their scope, parameter bytes or process peak, and a
candidate without a measured latency cannot claim to pass the latency gate.


</details>

<details>
<summary>Reference answer</summary>

The router, full SFT and LoRA rows fail on `quality gate failed` at 0/8, so
latency and memory never get to decide; the SmolLM2 adapter's quality comes
from your own rerun, and its model card records unreliable JSON and no
approval. The artifact that changes the decision is a development score at
or above the gate from a frozen evaluator with zero critical failures,
followed by the sealed human review.


</details>

[← A04 — RLHF bridge](/rlhf-bridge) · [Course home →](/) · [Glossary](/glossary)
