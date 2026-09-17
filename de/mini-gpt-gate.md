---
title: "T06 — Kernprüfung: programmieren, Fehler finden, erklären"
sidebar:
  label: "T06 — Kerngate"
---

<span id="t06-kernprufung-programmieren-fehler-finden-erklaren" />


[← T05 — Mini-GPT](/de/mini-gpt) · [I01 — Decoding, Sampling & KV-Cache →](/de/inference-decoding) · [Glossar](/de/glossary)

Bearbeite zuerst [Decoder](/de/mini-gpt), [Datenpipeline](/de/data-pipeline) und
[Attention](/de/attention). Plane 60–90 Minuten. Zeige eine eigene Umsetzung;
das blosse Ausführen der Musterlösung reicht nicht. Speichere eigene Versuche in
`artifacts/mini-gpt/`; generierte Korpora und Gewichte gehören nicht in Git.

## 1. Einen Pre-Norm-Residualblock programmieren (4 Punkte) [#1-einen-pre-norm-residualblock-programmieren-4-punkte]

Speichere dieses unvollständige Gerüst als `artifacts/mini-gpt/exercise.py`.
Ersetze den `NotImplementedError` durch deine Berechnung. Nutze die vorhandenen
Module des Blocks, rufe aber in deiner Funktion nicht `block.forward` auf.
Erhalte `[B,T,D]`, beide Vor-Normalisierungen, beide Residualpfade und die
Padding-Isolation. Sage zuerst voraus, was passiert, wenn beide Zweige nur null
zurückgeben.

```python
import torch
from llm_course.attention import causal_mask
from llm_course.mini_gpt import DecoderBlock, GPTConfig


def learner_forward(block, inputs, allowed, valid):
    # TODO: Norm -> Attention -> Dropout -> Residual -> Padding nullen.
    # TODO: Norm -> FFN -> Dropout -> Residual -> Padding nullen.
    raise NotImplementedError("beide Residualzweige implementieren")


torch.set_num_threads(1)
config = GPTConfig(vocab_size=9, block_size=3, embedding_dim=4, heads=2, layers=1)
block = DecoderBlock(config, index=0).eval()
valid = torch.tensor([[[1], [1], [0]]], dtype=torch.bool)
inputs = torch.tensor([[[1., 2., 3., 4.], [4., 3., 2., 1.], [0., 0., 0., 0.]]])
allowed = causal_mask(3)[None, None] & valid.squeeze(-1)[:, None, None, :]
allowed |= ~valid[:, None] & torch.eye(3, dtype=torch.bool)[None, None]
actual = learner_forward(block, inputs, allowed, valid)
expected = block(inputs, allowed, valid)
assert actual.shape == (1, 3, 4)
assert torch.count_nonzero(actual[:, 2]) == 0
assert torch.isfinite(actual).all()
torch.testing.assert_close(actual, expected)
print("Residualblock stimmt; jetzt den Kausalitätstest ergänzen")
```

```bash
python artifacts/mini-gpt/exercise.py
python -m pytest tests/test_mini_gpt.py -q
```

Ergänze einen eigenen Test: Verändere das letzte **gültige** Token einer Sequenz
mit drei Tokens und prüfe, dass frühere Ausgaben gleich bleiben. Verwende dafür
nur gültige Positionen und `eval()`, damit Dropout nicht stört. Erkläre, warum ein
reiner Formvergleich Attention entlang der falschen Achse nicht erkennen würde.

Je ein Punkt: Pre-Norm-Attention, erster Residualpfad, Pre-Norm-FFN mit zweitem
Residualpfad sowie eigene Kausalitäts-/Padding-Prüfung. Der anfängliche Fehler ist
beabsichtigt. Blosses Entfernen der Ausnahme ohne Berechnungen zählt nicht.
Halte Code und fehlgeschlagene/erfolgreiche Ausgaben fest, bevor du Antworten öffnest.

## 2. Eingebaute Fehler diagnostizieren (3 Punkte) [#2-eingebaute-fehler-diagnostizieren-3-punkte]

Nenne für jeden Defekt ein Gegenbeispiel, ein erwartetes beobachtbares Symptom und
eine Korrektur. Nur alle drei zusammen ergeben den jeweiligen Punkt.

| Defekt in einer lokalen Kopie | Erwarteter Nachweis |
| --- | --- |
| Datensatz setzt `labels = input_ids.clone()` | Sehr kleiner Loss kann nur sichtbare Tokens kopieren messen |
| `allowed` komplett True **und** `block.attention.causal=False` | Frühere Logits reagieren auf einen geänderten zukünftigen Suffix |
| Gültige Verluste zusammen mit Padding-Nullen mitteln | Zusätzliches Padding senkt den angezeigten Mittelwert künstlich |

Beim dritten Defekt: zwei gültige Verluste 2 und 4 sowie zwei Padding-Plätze
ergeben fälschlich `(2+4+0+0)/4=1.5`; korrekt ist 3. Warum braucht eine komplett
gepaddete Zeile Behandlung schon vor Softmax und zusätzlich beim Loss? Stelle nach
den Messungen alle korrekten Varianten wieder her.

## 3. Erklären und vergleichen (3 Punkte) [#3-erklaren-und-vergleichen-3-punkte]

Gib eine kurze mündliche oder schriftliche Erklärung mit eigenen Zahlen:

- Verfolge IDs `[2,3]`, Embeddings `[2,3,4]`, zwei Köpfe `[2,2,3,2]`, Scores
  `[2,2,3,3]`, FFN `[2,3,16]` und Logits `[2,3,5]`. Unterscheide Merkmals- von
  Positionsmischung und begründe, weshalb die Ziele genau einmal verschoben werden.
- Warum müssen reproduzierbare Blöcke unterschiedliche Anfangsgewichte haben?
  Warum nutzt Sampling vorübergehend `eval()`? Warum reichen gespeicherte Gewichte
  nicht zum exakten Fortsetzen von AdamW? Nenne zusätzliche Informationen für E01.
- Führe die drei Seeds der Lektion aus. Berichte jede gepaarte Differenz der
  Validierungs-NLL „ohne Norm minus mit Norm“, Mittelwert und Stichproben-
  Standardabweichung. Warum beweisen zwei Validierungsdokumente und drei Seeds
  keine allgemeine Überlegenheit?

Jede vollständige Erklärung ergibt einen Punkt. **Bestanden:** mindestens 8/10,
darunter alle 4 Programmierpunkte und alle 3 Diagnosepunkte, erfolgreiche
mitgelieferte Tests, Tiny-Batch-NLL &lt;0.1 und exakt gleiche Logits nach Neuladen.
Die letzten beiden Prüfungen sind in den Tests enthalten; bewahre ihre Ausgabe
auf. Bei Lücken die passende Voraussetzung wiederholen und neu erklären.

## Hinweise — erst nach einem eigenen Versuch öffnen [#hinweise-erst-nach-einem-eigenen-versuch-offnen]

<details>
<summary>Hinweis 1: die Residualpfade trennen</summary>

Lasse `inputs` unverändert, während du den normalisierten Attention-Beitrag
berechnest. Attention mit `mask=allowed` liefert `(output, weights)`; nur die erste
Komponente wird addiert. Der zweite normalisierte Zweig verwendet den aktualisierten
Zustand statt der ursprünglichen Eingabe. `valid` hat Form `[B,T,1]` und wird über
Merkmale verteilt.

</details>

<details>
<summary>Hinweis 2: die zwei Masken unterscheiden</summary>

Die Attention-Maske bestimmt vor Softmax, welche Schlüssel beitragen dürfen. Die
Loss-Maske legt fest, welche Ziele im Lernziel zählen. Eine ersetzt nicht die
andere. Eine Score-Zeile nur aus `-inf` ergibt eine undefinierte Verteilung, auch
wenn ihr Ziel später ignoriert wird.

</details>

## Referenzantworten — nach dem eigenen Nachweis vergleichen [#referenzantworten-nach-dem-eigenen-nachweis-vergleichen]

<details>
<summary>Referenzimplementierung des Blocks</summary>

```python
def learner_forward(block, inputs, allowed, valid):
    attended, _ = block.attention(block.norm_attention(inputs), mask=allowed)
    hidden = (inputs + block.dropout(attended)) * valid
    update = block.ffn(block.norm_ffn(hidden))
    return (hidden + block.dropout(update)) * valid
```

Bei Null-Zweigen entspricht die gültige Ausgabe der Eingabe; Padding bleibt null.
GELU und FFN wirken je Position auf Merkmale; maskierte Attention erlaubt
Information früherer Positionen. Verschiedene Block-Seeds vermeiden gleiche
Anfangsprojektionen bei trotzdem reproduzierbarem Gesamtlauf.

</details>

<details>
<summary>Referenzdiagnose und Erklärung</summary>

1. Sichtbare Eingabe `c` mit Ziel `c` belohnt Kopieren. Ziel sollte `a` sein.
   Im Datensatz einmal `tokens[:-1]` und `tokens[1:]` verwenden.
2. Für `[1,2,3]` und `[1,2,8]` müssen die ersten beiden Logits gleich bleiben.
   Kausales Dreieck und gültige Schlüssel wieder kombinieren. Beide Schutzstellen
   müssen für den absichtlichen Defekt dieser Implementierung ausgeschaltet sein.
3. Nur NLL für `labels != -100` summieren und durch deren Anzahl teilen. Komplett
   ignorierte Ziele liefern differenzierbare null. Vor Softmax gepaddeten Anfragen
   einen sicheren diagonalen Platz geben, danach deren Ausgaben nullen.

Sampling verwendet den Auswertungsmodus, damit Dropout den lokalen Seed nicht
unterläuft, und stellt danach jeden Modulmodus wieder her. Exaktes Fortsetzen
braucht Optimierermomente, gegebenenfalls Scheduler-/Scaler-Zustand, Schritt- und
Batchposition, Zufallszustände, Konfiguration und Datenidentität. Die Norm-Differenzen
sind 2.389516, 1.105527 und 1.325092; Mittelwert 1.606712, Stichproben-
Standardabweichung 0.686760. Das beschreibt den kleinen festen Validierungsversuch,
keinen Sicherheits-, Fakten- oder allgemeinen Sprachbenchmark.

</details>

[← T05 — Mini-GPT](/de/mini-gpt) · [I01 — Decoding, Sampling & KV-Cache →](/de/inference-decoding) · [Glossar](/de/glossary)
