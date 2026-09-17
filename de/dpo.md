---
title: "A03 — DPO und die kleine RLHF-Brücke"
sidebar:
  label: "A03 — DPO"
---

<span id="a03-dpo-und-die-kleine-rlhf-brucke" />


[← A02 — Adaptationsvergleich](/llm-engineering-course-pages/de/adaptation-comparison) · [A04 — RLHF-Brücke →](/llm-engineering-course-pages/de/rlhf-bridge) · [Glossar](/llm-engineering-course-pages/de/glossary)

Voraussetzungen: [A01](/llm-engineering-course-pages/de/sft-lora), [C03](/llm-engineering-course-pages/de/company-evaluation). Eine Sitzung.
Der Code steht in `src/llm_course/alignment.py`; lies `response_log_probs` und
`dpo_loss` neben dieser Seite und hebe `_fit_reward_model` und `_ppo_policy` für
[A04](/llm-engineering-course-pages/de/rlhf-bridge) auf. Jede Funktion validiert ihre Tensoren und gibt einen
neuen Wert zurück; die Referenzpolicy wird nie verändert.

## Antwort-Logwahrscheinlichkeiten summieren [#antwort-logwahrscheinlichkeiten-summieren]

`response_log_probs` summiert nur Antworttokens; Prompt und Padding tragen Label
`-100`. Anders als A01s `sft_loss` erwartet es bereits verschobene Labels:
`labels[b, t]` ist das Token, das `logits[b, t]` vorhersagen soll. Nimm eine
Sequenz aus drei Positionen über einem Vokabular von drei Tokens, deren
Softmax-Zeilen `[0.5, 0.3, 0.2]`, `[0.25, 0.5, 0.25]` und `[0.1, 0.1, 0.8]`
sind, mit Labels `[-100, 1, 2]`. Die erste Position ist Prompt und trägt nichts
bei; die Summe ist `log 0,5 + log 0,8 = −0,6931 − 0,2231 = −0,9163`. Zusätzliche
Promptpositionen können diese Summe nicht ändern; deshalb ist die DPO-Marge eine
Aussage über Antworten. Eine Sequenz ohne Antworttoken oder ein Label außerhalb
des Vokabulars wirft `ValueError`, bevor gerechnet wird.

## Ein DPO-Paar nachrechnen [#ein-dpo-paar-nachrechnen]

Direct Preference Optimization (DPO) lernt aus einem bevorzugten und einem
abgelehnten Paar. Eine eingefrorene Referenz liefert die relative Baseline. Die
Loss ist `-log sigmoid(beta * (Policy-Marge − Referenz-Marge))`. Nimm das erste
Paar aus `compare_dpo_reference.py`: Policy-Logwahrscheinlichkeiten −0,7
(bevorzugt) und −1,1 (abgelehnt), Referenz −0,8 und −1,0, `beta = 0.2`. Die
Policy-Marge ist `0,4`, die Referenz-Marge `0,2`, das Argument also
`0,2 × (0,4 − 0,2) = 0,04` und die Loss `-log sigmoid(0,04) = 0,6733`. Mit
`sigmoid(0,04) = 0,5100` ist der Gradient auf `policy_chosen`
`-beta × (1 − 0,5100) = −0,0980`, oder `−0,0245`, sobald das Skript über seine
vier Paare mittelt; die abgelehnte Logwahrscheinlichkeit erhält `+0,0245`. Sind
beide Margen gleich, ist das Argument null und die Loss `log 2 = 0,6931`: Das
Paar hat noch nichts gelehrt. `dpo_loss` ruft `.detach()` auf beiden
Referenztensoren auf; darum kann das Skript sie mit `requires_grad` markieren
und zeigen, dass ihr `.grad` `None` bleibt.

## Mit TRL vergleichen, dann den Bandit lesen [#mit-trl-vergleichen-dann-den-bandit-lesen]

Die endliche Aktionsbrücke macht Bradley–Terry-Reward, kategorische KL und den
geclippten PPO-Surrogat sichtbar. Sie ist ein Contextual-Bandit-Lehrbeispiel,
kein Token-RLHF für Produktion.

```bash
.venv/bin/python examples/run_alignment.py --dpo-steps 80 --ppo-steps 60
.venv/bin/python examples/compare_dpo_reference.py
```

Der DPO-Lauf stimmt mit TRL 0.26.1 bei Loss und Policy-Gradienten überein. Das
Skript ruft das installierte `DPOTrainer.dpo_loss` mit `loss_type="sigmoid"` auf
denselben vier Paaren auf; beide Seiten liefern die Losses
`0,6733, 0,7032, 0,8429, 0,6163`, und `max_loss_absolute_difference` sowie
`max_policy_gradient_absolute_difference` sind `0.0`. Die PPO-Variante zeigt
Reward-Hacking: naive PPO erhöht Proxy-Reward, senkt aber Qualität und erhöht die
Hackrate; KL-Kontrolle verbessert sie auf der kleinen Tabelle. Das ist kein
allgemeiner Sicherheitsbeweis.

Der Bandit hat vier Nordlicht-Kontexte (Billing, Kontozugang, Export, eine
Secret-Anfrage) mit je vier Aktionen, immer in der Reihenfolge richtig, teilweise,
nutzlos, schädliche Abkürzung. Die Referenzpolicy ist die Logit-Zeile
`[1.4, 0.5, -0.6, -1.0]`, Wahrscheinlichkeiten `0,6125, 0,2490, 0,0829, 0,0556`,
echte Qualität `0,713387` und Hackrate `0,0556`. `_dpo_policy` startet von dieser
Zeile und trainiert die Paare `(0,1)`, `(0,2)` und `(0,3)` mit Adam über die
angeforderten Updates:

| DPO-Lauf (80 Updates) | Echte Qualität | Hackrate | KL zur Referenz |
| --- | ---: | ---: | ---: |
| `beta = 0.05` | 0,9998 | 4,0e-5 | 0,4878 |
| `beta = 0.2` | 0,9995 | 9,5e-5 | 0,4850 |
| `beta = 1.0` | 0,9903 | 1,9e-3 | 0,4262 |
| `beta = 0.2`, nur Paar `(0,2)` | 0,9891 | 2,9e-3 | 0,4188 |

Ein größeres `beta` hält die Policy näher an der Referenz und lässt mehr Masse
auf der Abkürzung; das Paar `(0,3)` wegzulassen, das die Abkürzung als abgelehnt
benennt, wirkt genauso. Jeder Lauf wählt greedy in allen vier Kontexten weiterhin
die richtige Aktion. Vergleiche die PPO-Zeilen in [A04](/llm-engineering-course-pages/de/rlhf-bridge), bevor du
schließt, dass Präferenzdaten allein sicher sind.

## Vorhersagen → Nachvollziehen → Bauen → Beschädigen → Messen → Erklären [#vorhersagen-nachvollziehen-bauen-beschadigen-messen-erklaren]

1. **Vorhersagen:** Notiere Loss und Vorzeichen des `policy_rejected`-Gradienten für das obige Paar, bevor du etwas ausführst.
2. **Nachvollziehen:** Führe `compare_dpo_reference.py` aus und ordne `ours_losses[0]` und `ours_policy_chosen_gradients[0]` der Handrechnung zu.
3. **Bauen:** Reproduziere die Summe −0,9163 mit `response_log_probs(torch.log(probabilities)[None], labels[None])`.
4. **Beschädigen:** `compare_dpo_reference.py` importiert `dpo_loss`; entferne also beide `.detach()`-Aufrufe vorübergehend in `alignment.py` und mache es danach rückgängig, oder ersetze `llm_course.alignment.dpo_loss` per Monkeypatch durch eine Kopie ohne `detach`, bevor du sein `main()` aufrufst; `reference_inputs_frozen` wird falsch, weil die Referenztensoren nun einen Gradienten tragen, den jeder Optimizer mit Zugriff anwenden würde.
5. **Messen:** Führe `run_alignment.py --dpo-steps 20 --ppo-steps 60` aus und notiere, wie weit die echte Qualität bei `beta = 0.2` nach einem Viertel der Updates von 0,9995 entfernt ist.
6. **Erklären:** Begründe, warum eine DPO-Policy mit `hack_rate` unter 1e-4 auf dieser Tabelle keine Evidenz über das Eskalationsverhalten eines Sprachmodells ist.

### Eigenständige Aufgabe [#eigenstandige-aufgabe]

Wie groß ist die DPO-Loss, wenn Policy- und Referenzmarge gleich sind? Warum muss
die Referenz abgetrennt werden?

<details>
<summary>Hinweis</summary>

Setze das Argument von `sigmoid` auf null und werte `-log sigmoid(0)` aus.
Frage dann, welche Tensoren einen Gradienten erhielten, wenn die Referenz Teil
des Graphen wäre, und was der Optimizer damit täte.


</details>

<details>
<summary>Referenzantwort</summary>

Die Loss ist `log(2)`. Durch `detach` erhält nur die zu trainierende Policy
Gradienten.


</details>

Checkpoint: Summiere eine Antwort-Logwahrscheinlichkeit, finde das PPO-Clipping
und erkläre Proxy-Reward gegenüber Aufgabenqualität.

[← A02 — Adaptationsvergleich](/llm-engineering-course-pages/de/adaptation-comparison) · [A04 — RLHF-Brücke →](/llm-engineering-course-pages/de/rlhf-bridge) · [Glossar](/llm-engineering-course-pages/de/glossary)
