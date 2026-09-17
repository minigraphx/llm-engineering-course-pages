---
title: "A01 — SFT und LoRA: Antworten lernen, Basismodell erhalten"
sidebar:
  label: "A01 — SFT & LoRA"
---

<span id="a01-sft-und-lora-antworten-lernen-basismodell-erhalten" />


[← C04 — Continued Pretraining](/llm-engineering-course-pages/de/continued-pretraining) · [A02 — Adaptationsvergleich →](/llm-engineering-course-pages/de/adaptation-comparison) · [Glossar](/llm-engineering-course-pages/de/glossary)

Voraussetzungen: [Decoder](/llm-engineering-course-pages/de/mini-gpt), [Training](/llm-engineering-course-pages/de/pretraining) sowie die Daten-
und Evaluationsverträge der fiktiven Firma. Zuerst den expliziten CPU-Versuch
verstehen, danach den optionalen Versuch mit einem vortrainierten Modell ausführen.
Beide Wege benötigen weder private Daten noch eine kostenpflichtige Lehrer-API.

## Antwortmaske nachvollziehen [#antwortmaske-nachvollziehen]

Der Prompt konditioniert die Antwort, wird aber nicht als gewünschte Antwort
bewertet. Aus Token-IDs `[11, 12 | 21, 22]` entstehen unverschobene Labels
`[-100, -100, 21, 22]`. `logits[1]` sagt 21 voraus, `logits[2]` sagt 22 voraus.
Die Loss-Funktion verschiebt genau einmal. Bereits verschobene Labels würden den
falschen Übergang trainieren. Padding erhält ebenfalls −100 und Attention-Maske 0.

Bei Zielwahrscheinlichkeiten 0,5 und 0,25 beträgt die mittlere Antwort-NLL
`(-log(0,5) - log(0,25))/2 = 1,0397`. Zehn zusätzliche Prompt-Tokens dürfen diesen
Mittelwert nicht verdünnen. Bei festen Logits ändert ein maskiertes Ziel den Loss
nicht; veränderte Prompt-Eingaben können trotzdem andere Antwort-Logits erzeugen.

Prompt und Antwort werden getrennt tokenisiert, damit die Grenze explizit bleibt.
Der Assistant-Header gehört zum Prompt, das End-of-turn-Token zur Antwort.
Zu lange Beispiele werden abgelehnt: Abschneiden könnte die Sicherheitsregel oder
Antwort entfernen.

## Niedrigrang-Update vor PEFT bauen [#niedrigrang-update-vor-peft-bauen]

Vollständiges SFT verändert alle Gewichte. LoRA berechnet
`y = Wx + (alpha/r)BAx`, mit A der Form `[r, Eingang]` und B `[Ausgang, r]`.
W und Bias bleiben eingefroren. Bei 512×512 Gewichten und Rang 4 enthalten die
Faktoren `4×512 + 512×4 = 4.096` Parameter statt 262.144, also 1,5625%.
Andere Schichten, Bias, Optimizer-Zustand und Aktivierungen fehlen in diesem Anteil.

A startet zufällig, B mit null. Das angepasste Modell reproduziert zunächst exakt
das Basismodell. Beim ersten Rückwärtslauf kann B einen Gradienten haben, A wegen
B=0 aber noch nicht. Nachdem B verändert wurde, lernt auch A. Ein Test auf einen
von null verschiedenen A-Gradienten im ersten Schritt wäre falsch.

`LoRALinear.merged()` kopiert `W + (alpha/r)BA` in eine neue lineare Schicht.
Adapterdateien enthalten Version und Hash der exakten Basisgewichte. Eine andere
Basis gleicher Dimension wird abgelehnt. Der Kern lädt mit
`torch.load(weights_only=True)`, PEFT verwendet safetensors. Nichtendliche Werte
werden beim Merge abgelehnt. Die unveränderte Basis ermöglicht den Rollback.

## Ausführen und messen [#ausfuhren-und-messen]

```bash
.venv/bin/pytest tests/test_post_training.py -q
.venv/bin/python examples/run_sft_lora.py --steps 120
.venv/bin/pip install -e '.[post-training]'
.venv/bin/python examples/run_open_model_lora.py --download --device cpu
.venv/bin/python examples/run_open_model_lora.py --device mps --output artifacts/open-model-lora-mps
```

Der MiniGPT-Versuch vergleicht vollständiges SFT und manuelles LoRA ab derselben
zufälligen Initialisierung: gleiche Splits, Seed, Optimizer, Lernrate, Schritte
und Batchgröße. Der Bibliotheksversuch lädt das echte Apache-2.0-Modell
[SmolLM2-135M-Instruct](https://huggingface.co/HuggingFaceTB/SmolLM2-135M-Instruct),
Revision `12fd25f77366fa6b3b4b768ec3050bf629380bac`. Nur freigegebene Konfigurations-
und Tokenizerdateien sowie `model.safetensors` werden heruntergeladen.
`trust_remote_code=False`; Rang-4-Adapter an q/v trainieren in float32.

Alle Methoden nutzen Trainingsanweisungen und separat verfasste Entwicklungsfälle,
niemals Gold. Derselbe Evaluator bewertet generierte Antworten vor/nach Training.
Antwort-NLL und exakte/Format-/Sicherheitsmetriken messen Verschiedenes: Bessere NLL
unter Teacher Forcing garantiert keine brauchbaren Antworten. Reload und Merge
werden gegen den trainierten Adapter geprüft. Messwerte und Grenzen stehen im
[Anpassungsvergleich](/llm-engineering-course-pages/de/adaptation-comparison) und in `docs/baselines/sft-lora-v1.md`.

## Vorhersagen → Nachvollziehen → Bauen → Beschädigen → Messen → Erklären [#vorhersagen-nachvollziehen-bauen-beschadigen-messen-erklaren]

1. **Vorhersagen:** Welches Token erhält den ersten Antwort-Loss? Wie groß ist A's erster Gradient?
2. **Nachvollziehen:** IDs, Labels und Attention-Masken unterschiedlich langer Beispiele ausgeben.
3. **Bauen:** `response_batch`, `sft_loss` und `LoRALinear` aus den Formeln implementieren.
4. **Beschädigen:** W freigeben, Verschiebung entfernen oder falsche Basis laden; Tests prüfen.
5. **Messen:** Train/Dev-NLL, Formatfehler, allgemeine Kontroll-NLL, Laufzeit und Parameter zählen.
6. **Erklären:** Adapter behalten oder zurückrollen? Regressionen sind Entscheidungsevidenz.

### Eigenständige Aufgabe [#eigenstandige-aufgabe]

Eine Schicht hat 12 Eingänge, 8 Ausgänge, Rang 2 und alpha 6. Parameterzahl und
Skalierung berechnen. Bei festen Logits ein Prompt- und ein Antwortziel getrennt
ändern und den Loss testen. Nach zwei Optimizer-Schritten speichern, laden und
mergen; die Basis darf keine Gradienten erhalten.

<details>
<summary>Hinweis</summary>

Beide rechteckigen Faktoren zählen. Das Ziel an Position t wird aus Logits an
t−1 vorhergesagt. Maske und Ziel müssen gemeinsam verschoben werden.


</details>

<details>
<summary>Referenzantwort</summary>

A: 24, B: 16, zusammen 40 statt 96 Basisgewichte. Skalierung: 3.
Maskierte Promptziele ändern den Loss nicht. Antwortziele ändern ihn bei
verschiedenen Ziel-Logwahrscheinlichkeiten. A lernt, nachdem B verändert wurde.


</details>

Checkpoint: Loss-Ausrichtung ohne Trainer erklären; anfängliches Null-Update,
eingefrorene Basis sowie sicheres Laden/Mergen belegen; verfehlte Qualitätsziele
klar benennen. Quellen: [LoRA](https://arxiv.org/abs/2106.09685),
[PEFT](https://huggingface.co/docs/peft/v0.18.0/index).

[← C04 — Continued Pretraining](/llm-engineering-course-pages/de/continued-pretraining) · [A02 — Adaptationsvergleich →](/llm-engineering-course-pages/de/adaptation-comparison) · [Glossar](/llm-engineering-course-pages/de/glossary)
