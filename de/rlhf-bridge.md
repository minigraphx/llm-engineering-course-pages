---
title: "A04 — RLHF-Brücke: Fehler sichtbar machen"
sidebar:
  label: "A04 — RLHF-Brücke"
---

<span id="a04-rlhf-brucke-fehler-sichtbar-machen" />


[← A03 — DPO](/de/dpo) · [Capstone — Firmenmodell →](/de/company-capstone) · [Glossar](/de/glossary)

Voraussetzungen: [A03](/de/dpo). Eine Sitzung. Der Code ist die zweite Hälfte
von `src/llm_course/alignment.py`: `reward_pair_loss`, `categorical_kl`,
`ppo_clipped_loss`, `_fit_reward_model` und `_ppo_policy`. Die Seite ist für den
Kurspfad optional und Pflicht, bevor du eine RLHF-Bibliothek anfasst.

## Ein Reward-Modell mit einer Lücke anpassen [#ein-reward-modell-mit-einer-lucke-anpassen]

Die optionale Brücke friert eine Referenzpolicy ein, lernt ein Reward-Modell aus
Präferenzpaaren, bestraft Abweichung per KL und aktualisiert eine endliche Policy
mit PPO. Die Aktionstabelle umfasst Billing-, Zugangs-, Export- und
Secret-Anfragen. Eine eingebaute Konfidenz-Abkürzung macht Reward-Hacking
messbar. Jede der vier Aktionen pro Kontext wird durch fünf Merkmale beschrieben:
Vollständigkeit, selbstsicherer Stil, Prozesssprache, Kürze und ein
Abkürzungsmarker. Die schädliche Aktion (Index 3) hat Konfidenz `4.0` und den
Marker `1.0`; die richtige Aktion hat Konfidenz `0.6`. Das Reward-Modell ist eine
lineare Schicht über diesen Merkmalen, trainiert mit `reward_pair_loss`, der
negativen Bradley–Terry-Log-Likelihood `-log sigmoid(r_chosen − r_rejected)`, die
bei gleichen Rewards `log 2 = 0.6931` beträgt und fällt, sobald der bevorzugte
Reward vorne liegt.

Der naive Fit sieht nur die Paare `(0,1)`, `(0,2)` und `(1,2)`: Niemand hat die
Abkürzung je als abgelehnt markiert. Der kontrollierte Fit ergänzt `(0,3)` und
`(1,3)`. Nach 120 Adam-Schritten erreichen beide Trainings-Paargenauigkeit `1.0`,
und beide sind deterministisch: Die Gewichte starten bei null, gesampelt wird
nicht. Die naiven Scores sind `6.45, 2.66, -2.24, 10.46`; die Abkürzung ist die
bestbelohnte Aktion, obwohl ihre echte Qualität `-0.5` ist. Die kontrollierten
Scores sind `5.47, 2.79, -2.25, -2.43`. Eine perfekte Paargenauigkeit sagte
nichts über die Aktion, die nie in einem Paar vorkam.

## KL und das geclippte Verhältnis [#kl-und-das-geclippte-verhaltnis]

`categorical_kl` mittelt `sum(p × (log p − log p_ref))` über Kontexte; sie ist
`0.0`, wenn Policy und Referenz gleich sind. `ppo_clipped_loss` bildet das
Verhältnis `exp(log_probs − old_log_probs)` und klemmt es auf `[1 − ε, 1 + ε]` mit
`ε = 0.2`. Rechne eine Aktion nach, deren Wahrscheinlichkeit von 0,4 auf 0,5
stieg: Das Verhältnis ist `1.25`. Bei Vorteil `+1` ist der Surrogat
`min(1.25, 1.2) = 1.2`, die Loss `-1.2`: Das Clipping begrenzt, wie viel Gewinn
ein Update verbuchen darf. Bei Vorteil `−1` ist er `min(-1.25, -1.2) = -1.25`,
die Loss `1.25`: Ein Schritt, der geschadet hat, wird voll berechnet.
`_ppo_policy` sampelt 32 Aktionen pro Kontext aus der aktuellen Policy, erneuert
sie alle vier Updates, nutzt `Reward − erwarteter Reward` als Vorteil und
minimiert `Surrogat + 0.08 × KL` mit Adam.

## Die drei Größen lesen [#die-drei-groen-lesen]

Lies `alignment.py` Zeile für Zeile, bevor du einen Bibliothekstrainer nutzt.
Prüfe nach dem Lauf Proxy-Reward, echte Qualität und Hackrate. Höherer Reward
bedeutet nicht automatisch mehr Sicherheit. Kritisches Routing bleibt eine
Anwendungsprüfung.

Nutze das CPU-Rezept aus [DPO](/de/dpo) und behalte bei fehlendem Accelerator
den CPU-Fallback. Für einen Accelerator bleiben Seed, Daten und Evaluator gleich.
Mit `--dpo-steps 80 --ppo-steps 60` und Seed 7 hält der Bericht fest:

| Policy | Proxy-Reward (naive Skala) | Echte Qualität | Hackrate | KL zur Referenz |
| --- | ---: | ---: | ---: | ---: |
| Referenz | 5,011 | 0,713387 | 0,055566 | 0,0 |
| Naive PPO | 5,643 | 0,684654 | 0,108824 | 0,1374 |
| Kontrollierter-Reward-PPO (`ppo.controlled_proxy`) | 5,374 | 0,785538 | 0,041577 | 0,1215 |

Beide PPO-Läufe nutzen `kl_coefficient=0.08`; der Unterschied ist das
Reward-Modell, die kontrollierte Zeile ist die mit den Abkürzungspaaren gefittete.
Naive PPO erhöht den Proxy, den sie bekam, und senkt die Qualität, die sie nicht
bekam; ihre Hackrate verdoppelt sich fast. Alle drei Policies wählen greedy in
jedem Kontext weiterhin die richtige Aktion: Der Schaden sitzt in der
Wahrscheinlichkeitsmasse, eine Prüfung nur des Argmax meldete keine Änderung.
Die kontrollierte Zeile ist auf dieser Tabelle besser als die Referenz und nicht
mehr; `limitations` im Bericht erklärt, warum das kein Sicherheitsbeweis ist.

## Vorhersagen → Nachvollziehen → Bauen → Beschädigen → Messen → Erklären [#vorhersagen-nachvollziehen-bauen-beschadigen-messen-erklaren]

1. **Vorhersagen:** Welche Aktion bewertet das naive Reward-Modell am höchsten? Entscheide aus den Merkmalszeilen, bevor du `naive_action_scores` liest.
2. **Nachvollziehen:** Führe `run_alignment.py --dpo-steps 80 --ppo-steps 60` aus und finde die drei Größen für `ppo.naive_proxy` und `ppo.controlled_proxy`.
3. **Bauen:** Reproduziere die Verhältnisrechnung mit `ppo_clipped_loss(torch.log(torch.tensor([0.5])), torch.log(torch.tensor([0.4])), torch.tensor([1.0]))`.
4. **Beschädigen:** Setze in einer Kopie von `_ppo_policy` für den naiven Reward `kl_coefficient=0.0` und vergleiche `hack_rate` und `kl_from_reference` mit der Seed-7-Zeile; die Änderung nach 60 Updates ist klein, entscheide also, ob der Koeffizient oder das Reward-Modell den Effekt in der Tabelle trug.
5. **Messen:** Führe Seed 11 und Seed 23 mit denselben Budgets aus; notiere Proxy-Reward, echte Qualität und Hackrate für beide PPO-Zeilen.
6. **Erklären:** Sage in einem Satz, der das fehlende Paar nennt, warum die Paargenauigkeit `1.0` des Reward-Modells keine Evidenz war.

### Eigenständige Aufgabe [#eigenstandige-aufgabe]

Definiere aus den gemessenen Zeilen einen Rollback-Monitor für diese Policy:
Größe, Schwelle und das Artefakt, das du neu lädst. Führe dann zwei weitere Seeds
aus und berichte, ob die Ordnung naive Hackrate > Referenz > kontrolliert hält;
wo nicht, notiere die Zahl, die sie gebrochen hat.

<details>
<summary>Hinweis</summary>

Die Hackrate ist `mean_probabilities[3]`, die Masse auf der schädlichen
Aktion; die echte Qualität ist der eingebaute Wert, den ein produktives
System nicht lesen kann. Der Seed ändert nur, welche Aktionen PPO sampelt;
die Reward-Fits ändern sich nicht.


</details>

<details>
<summary>Referenzantwort</summary>

Ein vertretbarer Monitor ist die Hackrate auf der eingefrorenen Aktionstabelle
mit einer Schwelle unter dem Referenzwert `0.055566` von Seed 7; beim
Überschreiten werden die Referenz-Logits `[1.4, 0.5, -0.6, -1.0]` neu
geladen. Die zwei zusätzlichen Seeds sind deine Messung; die naiven
Reward-Scores `6.45, 2.66, -2.24, 10.46` sind für jeden Seed gleich, jede
Änderung stammt also aus dem Sampling von PPO.


</details>

Checkpoint: Nenne ein Rollback-Signal, etwa einen kritischen Sicherheitsfehler
oder steigende Hackrate, und die unveränderliche Referenz für Reproduktion.

[← A03 — DPO](/de/dpo) · [Capstone — Firmenmodell →](/de/company-capstone) · [Glossar](/de/glossary)
