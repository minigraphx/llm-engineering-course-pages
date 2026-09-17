---
title: "T05 — Mini-GPT: einen kleinen Decoder zusammensetzen"
sidebar:
  label: "T05 — Mini-GPT"
---

<span id="t05-mini-gpt-einen-kleinen-decoder-zusammensetzen" />


[← T04 — Self-Attention](/de/attention) · [T06 — Kerngate →](/de/mini-gpt-gate) · [Glossar](/de/glossary)

**Lernziel:** ein trainierbares Modell für das nächste Token bauen und jede
Operation von IDs bis zu Wahrscheinlichkeiten erklären. Plane 90–120 Minuten,
danach folgt die [Kernprüfung](/de/mini-gpt-gate). Voraussetzungen:
[Tensorformen](/de/foundations-02-shapes), [Autograd](/de/foundations-05-mlp-autograd),
[Dokumentpipeline](/de/data-pipeline) und [kausale Attention](/de/attention).
Sowohl der eigenständige Einstieg als auch die
[Godot-RL-Brücke](/de/foundations-godot-rl-bridge) führen hierher.

## Mit einer Vorhersage anfangen [#mit-einer-vorhersage-anfangen]

Aus `cat` erzeugt die Pipeline Eingaben `[<bos>, c, a, t]` und Ziele
`[c, a, t, <eos>]`. Position 1 liest `<bos>, c` und sagt `a` voraus. Der Datensatz
hat die Ziele bereits verschoben. Nochmaliges Verschieben würde `c` das Ziel `t`
zuordnen. Ein **Decoder** verwendet nur frühere Positionen und die aktuelle
Position. Ein **Logit** ist eine frei skalierte Bewertung eines möglichen Tokens;
erst Softmax macht daraus eine Wahrscheinlichkeit. Das **Vokabular** enthält alle
erlaubten Token-IDs.

Lies `MiniGPT.forward` in `src/llm_course/mini_gpt.py` parallel zu dieser Seite.
Der Decoder nutzt die sichtbare `MultiHeadSelfAttention` aus dem vorigen Labor;
Masken und Projektionen bleiben nachvollziehbar.

## Formen und Zahlen verfolgen [#formen-und-zahlen-verfolgen]

Für zwei Sequenzen mit je drei Tokens, Breite 4, zwei Köpfen und Spielvokabular 5:

| Schritt | Form | Bedeutung |
| --- | --- | --- |
| IDs | `[2,3]` | 6 ganze Zahlen, noch keine Merkmalsvektoren |
| Token- plus Positionsvektoren | `[2,3,4]` | 4 gelernte Merkmale je Position |
| Q, K, V je Kopf | `[2,2,3,2]` | 2 Köpfe mit jeweils 2 Merkmalen |
| Attention-Scores | `[2,2,3,3]` | jede Anfrage vergleicht sich mit 3 Schlüsseln |
| Zusammengeführte Köpfe | `[2,3,4]` | Merkmale aneinanderhängen und projizieren |
| FFN-Erweiterung | `[2,3,16]` | vierfache Breite je Position |
| Blockausgabe | `[2,3,4]` | Residualaddition erhält die Breite |
| Logits | `[2,3,5]` | 5 Bewertungen für das nächste Token |

1. **Nachschlagen und Position.** ID 2 wählt beispielsweise `[1,0,-1,2]` aus der
   Embedding-Tabelle. Der Positionsvektor `[0.1,0.2,0.3,0.4]` ergibt durch Addition
   `[1.1,0.2,-0.7,2.4]`. Gleiche Tokens bekommen so Ortsinformation. Padding wird
   durch Multiplikation mit null entfernt.
2. **Layer-Normalisierung.** Je Position den Merkmalsmittelwert abziehen und durch
   `sqrt(Varianz + 1e-5)` teilen, danach gelernte Skala und Verschiebung anwenden.
   Für `[1,3]` sind Mittelwert 2 und Varianz 1, das anfängliche Ergebnis ist etwa
   `[-1,1]`. Es wird weder über Zeitpositionen noch über Dokumente gemittelt.
3. **Attention.** Lineare Abbildungen erzeugen Q, K und V. Anfrage `[1,0]` und
   Schlüssel `[1,0]`, `[0,1]` ergeben nach Division durch `sqrt(2)` die Scores
   `[0.7071,0]`. Softmax liefert ungefähr `[0.670,0.330]`. Werte `[2,0]`, `[0,4]`
   werden zu `[1.340,1.320]` gemischt. Zukünftige und gepaddete Schlüssel werden
   schon vor Softmax gesperrt.
4. **Residualaddition.** Den Attention-Beitrag zum ursprünglichen Vektor addieren:
   `[1,2] + [0.3,-0.2] = [1.3,1.8]`. Dadurch bleiben direkte Pfade für Information
   und Gradienten erhalten. Normalisierung *vor* Attention heisst **Pre-Norm**.
   Ihre Position kann die Trainingsstabilität beeinflussen; die Motivation
   erläutern [Xiong et al.](https://arxiv.org/abs/2002.04745). Unser kleines Experiment
   ist eine eigene Messung, keine Reproduktion dieser Studie.
5. **Feed-forward-Netz (FFN).** An jeder Position `W1`, GELU und `W2` anwenden:
   Breite 4 → 16 → 4. Ein skalares lineares Beispiel: `2 × 0.5 + 0.1 = 1.1`.
   GELU ist `x × Φ(x)`: `GELU(1) ≈ 0.8413`, `GELU(-1) ≈ -0.1587`.
   Das ist eine glatte Nichtlinearität, keine Token-Wahrscheinlichkeitsverteilung.
   Siehe die [GELU-Definition von PyTorch](https://docs.pytorch.org/docs/2.14/generated/torch.nn.GELU.html).
   Es folgt eine weitere Residualaddition. Das FFN mischt Merkmale einer Position,
   Attention mischt Informationen zwischen erlaubten Positionen.
6. **Dropout.** Beim Training mit `p=0.25` werden zufällig Merkmale auf null gesetzt;
   Überlebende werden durch 0.75 geteilt: Wert 3 wird zu 4. Im Auswertungsmodus
   entfällt diese Zufälligkeit. Die Pflichtbaseline verwendet `p=0`.
7. **Abschliessende Norm und LM-Kopf.** Nach den Blöcken normalisieren und von Breite
   4 auf 5 Vokabularwerte projizieren. Gewichtszeile `[1,0,0,-1]` mal Zustand
   `[2,1,0,0.5]` liefert Logit 1.5. `forward` enthält kein abschliessendes Softmax.
8. **Loss.** Logits `[0,log(2),0]` ergeben Wahrscheinlichkeiten `[0.25,0.5,0.25]`.
   Für Zielindex 1 ist die negative Log-Likelihood (NLL) `-log(0.5)=0.6931`.
   Zwei gültige Verluste 0.6931 und 1.3863 ergeben im Mittel 1.0397. Zehn weitere
   Padding-Ziele verändern weder Mittelwert noch Gradienten. Perplexität ist
   `exp(mittlere NLL)`, hier etwa 2.828.

Die Blockgleichungen lauten `h = x + Attention(LN(x))` und
`y = h + FFN(LN(h))`. Dropout betrifft die Beiträge; Padding-Zustände werden nach
beiden Additionen auf null gesetzt. Softmax über ausschliesslich `-inf` ist
undefiniert. Gepaddete Anfragen bekommen daher einen bedeutungslosen diagonalen
Schlüssel, danach wird ihre Ausgabe genullt. Gültige Anfragen können weiterhin
kein Padding lesen. Probiere eine komplett gepaddete Zeile und prüfe
`torch.isfinite(logits).all()`.

## Die Referenz ausführen [#die-referenz-ausfuhren]

Nach dem [Setup](/de/setup) im Repository-Verzeichnis:

```bash
python -m pytest tests/test_mini_gpt.py -q
python examples/run_mini_gpt.py --steps 120 --seeds 7 19 42
```

`artifacts/mini-gpt/report.json` enthält Konfiguration, Dokument-Hashes,
Batch-Fingerabdrücke, Formen, Verluste, feste Stichproben und gepaarte Differenzen.
Der Lauf erzeugt den kleinen Kurskorpus im Arbeitsspeicher, verwendet CPU und
einen Thread und lädt nichts herunter. Der aktuelle Korpus liefert 12 Trainings-
und 2 Validierungsdokumente; für spätere grössere Versionen gelten Obergrenzen von
32/16. Jedes Dokument wird auf 25 IDs gekürzt, also höchstens 24 Vorhersageziele.
Batchgrösse 8, Breite 32, 4 Köpfe, 2 Blöcke, AdamW mit Lernrate 0.003,
Gewichtszerfall 0.01, Gradientennorm höchstens 1, insgesamt 120 Updates.
Die zweite Variante entfernt alle Layer-Normalisierungen. Split, Batchfolge,
Optimierer, Schrittzahl und Seed bleiben je Paar gleich.

Gemessen mit Python 3.12.13 / PyTorch 2.14.0 auf CPU:

| Seed | Norm Trainings-NLL | Norm Validierungs-NLL | Ohne Norm Validierungs-NLL | Differenz |
| --- | --- | --- | --- | --- |
| 7 | 0.130419 | 2.082053 | 4.471570 | +2.389516 |
| 19 | 0.127252 | 2.228553 | 3.334080 | +1.105527 |
| 42 | 0.131081 | 2.154143 | 3.479235 | +1.325092 |

Die mittlere Differenz „ohne Norm minus mit Norm“ beträgt **+1.606712 NLL**, ihre
Stichproben-Standardabweichung **0.686760**. Das beschreibt drei Seeds und ist kein
Konfidenzintervall. Alle Paare bevorzugen Normalisierung auf diesen zwei
Validierungsdokumenten. Die viel niedrigere Trainings-NLL zeigt Überanpassung;
sie belegt kein allgemeines Sprachverständnis. Mit den Normen entfallen auch ihre
320 gelernten Skalen-/Biasparameter, ein Bestandteil dieses Eingriffs. Der genaue
Vertrag steht in `docs/baselines/mini-gpt-v1.md`.

## Generieren, speichern, laden [#generieren-speichern-laden]

```python
import torch
from llm_course.data_pipeline import BOS_ID
from llm_course.mini_gpt import GPTConfig, MiniGPT, load_model, save_model

model = MiniGPT(GPTConfig())  # untrainiert: noch keine gelernte Sprachqualität
prompt = torch.tensor([[BOS_ID]], dtype=torch.long)
sample = model.generate(prompt, max_new_tokens=8, temperature=1.0, seed=7)
save_model(model, "artifacts/mini-gpt/model.pt")
restored = load_model("artifacts/mini-gpt/model.pt")  # CPU, Auswertungsmodus
assert torch.equal(model.eval()(prompt), restored(prompt))
```

Führe zuerst den Baseline-Befehl aus, damit das Artefaktverzeichnis existiert.
Der Zeichen-Encoder kann auch Sondertokens beschreiben; reines Zeichendekodieren
kennt BOS/EOS/PAD nicht. Sampling teilt Logits durch die Temperatur: `[0,2]` wird
bei Temperatur 2 zu `[0,1]`; die grössere Wahrscheinlichkeit sinkt von 0.881 auf
0.731. Ein lokaler Seed reproduziert Stichproben und erhält den vorherigen
Trainings-/Auswertungsmodus. Bei zu langem Kontext wird nur das letzte Fenster
gelesen, mit Positionsindizes wieder ab null. Der zurückgegebene Präfix bleibt
erhalten. Prompts sind ungepaddet. Es werden genau die angeforderten Tokens erzeugt,
ohne EOS-Stoppregel und ohne KV-Cache ([I01](/de/inference-decoding)). Checkpoints enthalten Architektur und
Gewichte; Fortsetzen inklusive Optimierer/Zufallszustand gehört zu E01.

## Absichtlich kaputtmachen und weitergehen [#absichtlich-kaputtmachen-und-weitergehen]

Setze in einem lokalen Versuch die Ziele gleich den Eingaben. Der Loss fällt
möglicherweise schneller, weil das Modell die Antwort an der aktuellen Position
sieht. Erkläre, warum das keine Vorhersage des nächsten Tokens mehr ist, und stelle
die verschobenen Ziele wieder her. Generierte Korpora und Checkpoints nicht
committen. Gehe zur [Kernprüfung](/de/mini-gpt-gate), sobald du jede Form verfolgen,
Attention-Maske und Loss-Maske unterscheiden und Tiny-Batch-Überanpassung als
Fehlersuche statt als Qualitätsbeleg erklären kannst.

[← T04 — Self-Attention](/de/attention) · [T06 — Kerngate →](/de/mini-gpt-gate) · [Glossar](/de/glossary)
