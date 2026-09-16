---
title: "M5 — DPO und die kleine RLHF-Brücke"
sidebar:
  label: "M5 — DPO"
---

<span id="m5-dpo-und-die-kleine-rlhf-brucke" />


Direct Preference Optimization (DPO) lernt aus einem bevorzugten und einem
abgelehnten Paar. Eine eingefrorene Referenz liefert die relative Baseline. Die
Loss ist `-log sigmoid(beta * (Policy-Marge − Referenz-Marge))`.
`response_log_probs` summiert nur Antworttokens; Prompt und Padding tragen Label
`-100`.

Die endliche Aktionsbrücke macht Bradley–Terry-Reward, kategorische KL und den
geclippten PPO-Surrogat sichtbar. Sie ist ein Contextual-Bandit-Lehrbeispiel,
kein Token-RLHF für Produktion.

```bash
.venv/bin/python examples/run_alignment.py --dpo-steps 80 --ppo-steps 60
.venv/bin/python examples/compare_dpo_reference.py
```

Der DPO-Lauf stimmt mit TRL 0.26.1 bei Loss und Policy-Gradienten überein. Die
PPO-Variante zeigt Reward-Hacking: naive PPO erhöht Proxy-Reward, senkt aber
Qualität und erhöht die Hackrate; KL-Kontrolle verbessert sie auf der kleinen
Tabelle. Das ist kein allgemeiner Sicherheitsbeweis.

### Aufgabe [#aufgabe]

Wie groß ist die DPO-Loss, wenn Policy- und Referenzmarge gleich sind? Warum muss
die Referenz abgetrennt werden?

<details>
<summary>Referenzantwort</summary>

Die Loss ist `log(2)`. Durch `detach` erhält nur die zu trainierende Policy
Gradienten.


</details>

Checkpoint: Summiere eine Antwort-Logwahrscheinlichkeit, finde das PPO-Clipping
und erkläre Proxy-Reward gegenüber Aufgabenqualität.
