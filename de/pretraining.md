---
title: "T05 — Reproduzierbares Vortraining"
sidebar:
  label: "E01 — Reproduzierbares Training"
---

<span id="t05-reproduzierbares-vortraining" />


[← Mini-GPT](/llm-engineering-course-pages/de/mini-gpt) · [Kursübersicht](/llm-engineering-course-pages/de/)

## Ziel und Voraussetzungen [#ziel-und-voraussetzungen]

Plane 90–120 Minuten ein. Du brauchst die [Datenpipeline](/llm-engineering-course-pages/de/data-pipeline),
[Mini-GPT](/llm-engineering-course-pages/de/mini-gpt), Gradienten und die Bedeutung der Kreuzentropie.
Arbeite zuerst das [Setup](/llm-engineering-course-pages/de/setup) durch. Alle Pflichtversuche laufen ohne
Downloads auf der CPU. Danach kannst du einen Optimiererschritt erklären, einen
Lauf in derselben CPU-Laufzeit exakt fortsetzen und Qualität von Geschwindigkeit
unterscheiden.

**Vortraining** minimiert die negative logarithmische Wahrscheinlichkeit (NLL)
des nächsten Tokens auf Trainingstext. Ein **Mikrobatch** ist ein Vorwärts- und
Rückwärtsdurchlauf. Ein **Optimiererschritt** aktualisiert Parameter nach einem
oder mehreren Mikrobatches. Hier materialisiert ein gesetzter Loader-Seed einmal
eine Batchliste. Der Trainer kopiert sie und besucht sie zyklisch. Es gibt keine
Worker, zufällige Datenaugmentation oder neue Mischung pro Epoche. Der Fingerabdruck
enthält Reihenfolge, Formen, Datentypen und Werte.

## Vorhersagen und reproduzieren [#vorhersagen-und-reproduzieren]

Im Repository-Hauptverzeichnis mit aktivierter virtueller Umgebung:

```bash
python examples/run_pretraining.py --profile cpu-small --steps 20 --unstable
pytest tests/test_training.py -q
```

Der Befehl rechnet 20 Schritte ohne Unterbrechung. Ein separater Lauf speichert
nach 10 Schritten `artifacts/pretraining-v1/resume.pt`; ein neu erzeugter Trainer
lädt den Stand und setzt bis Schritt 20 fort. `final.pt` und `report.json` werden
ebenfalls gespeichert. Diese erzeugten Artefakte gehören nicht in Git. Beide Läufe
starten mit demselben Modell: 28.544 Parameter, Kontext 32, Breite 32, vier Köpfe,
zwei Schichten, Dropout 0,1 und Seed 7. Nur Trainingsdokumente gelangen in die Batches.
Der Bericht enthält Herkunftsmanifest und Fingerabdruck der geordneten Batches.

Überlege vorher: Reicht gleiche Initialisierung, damit Dropout nach dem Neustart
übereinstimmt? Die Referenz ergibt **Loss 3,71458 → 2,61039** und **2.553 überwachte
Tokens** in 20 Updates. `parameter_max_abs_difference` muss **0** sein;
`optimizer_exact` und `quality_metrics_exact` müssen in derselben CPU-Laufzeit
**true** sein. Wechselnde Batches und Dropout bedeuten, dass nicht jeder einzelne
Loss sinkt. Das ist eine technische Referenz, kein Beleg für Generalisierung.

## Ein Update mit konkreten Zahlen [#ein-update-mit-konkreten-zahlen]

`language_model_loss` bekommt Logits `[B,T,V]` und bereits verschobene Labels
`[B,T]`. Nur Labels ungleich `-100` zählen. Padding wird weder im Loss noch in der
Tokenanzahl berücksichtigt; die Attention-Maske verhindert zusätzlich den Blick
auf Padding.

Mikrobatch A hat 4 überwachte Tokens und mittleren Loss 2,0; B hat 1 Token mit
Loss 5,0. Der Mittelwert der beiden Mittelwerte wäre 3,5 und gewichtet B zu stark.
Korrekt ist `(4×2 + 1×5) / 5 = 2,6`. Berechne Gradienten für `0,8 * loss_A` und
`0,2 * loss_B`, aktualisiere danach genau einmal. Der Trainer zählt das gesamte
Akkumulationsfenster **vor** dem Rückwärtsdurchlauf. Unterschiedliche Zeilenzahlen,
Paddingmengen und Sequenzlängen erhalten dadurch die richtigen Gewichte. Ein
vollständig leeres Fenster ist ein Fehler; ein leerer Mikrobatch in einem sonst
nichtleeren Fenster wird übersprungen.

```python
# Kern der Akkumulation; der vollständige Trainer prüft auch endliche Werte.
optimizer.zero_grad(set_to_none=True)
counts = [(batch.labels != -100).sum().item() for batch in window]
for batch, count in zip(window, counts, strict=True):
    if count:
        loss = language_model_loss(model(batch.input_ids, batch.attention_mask), batch.labels)
        (loss * count / sum(counts)).backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

Ohne Dropout vergleichen Tests dieses Update mit einem zusammengefügten Batch
mit 4 und 3 gültigen Tokens. `atol=1e-7, rtol=1e-6` berücksichtigen unterschiedliche
Reihenfolgen von Gleitkomma-Summationen. Mit Dropout hängen Zufallsziehungen von
Tensorformen ab; Mikrobatches und ein großer Batch müssen trotz korrekter
Gewichtung nicht bitweise identisch sein.

## AdamW, Lernratenplan und Clipping [#adamw-lernratenplan-und-clipping]

**AdamW** speichert pro Parameter zwei gleitende Mittel: den Gradienten `m` und
den quadrierten Gradienten `v`. Die PyTorch-Standardfaktoren sind 0,9 und 0,999.
Eine Korrektur berücksichtigt die anfänglichen Nullwerte. Das adaptive Update
teilt das korrigierte `m` durch `sqrt(v) + 1e-8`. Entkoppelter Gewichtszerfall
multipliziert den Parameter zusätzlich mit `(1 - Lernrate × Zerfall)`.
Für Gewicht 2, Lernrate 0,01 und Zerfall 0,1 ergibt allein dieser Zerfall 1,998.
`adamw_parameter_groups` wendet Zerfall auf Matrizen einschließlich Embeddings an,
aber nicht auf Bias- und Normalisierungsvektoren. Das ist eine bewusste Lehrkonvention.

**Warmup** beginnt mit kleineren Updates; **Kosinusabfall** reduziert anschließend
die Lernrate sanft. `learning_rate_at` zählt Schritte ab null. Bei fünf Schritten,
zwei Warmup-Schritten, maximaler Lernrate 0,01 und Minimalanteil 0,1 lautet der Plan
`[0.005, 0.010, 0.010, 0.0055, 0.001]`. Der Standardlauf startet bei 0,0015, erreicht
0,003 und endet bei 0,0003. Der gesamte Plan steht vor dem Start fest: Eine Änderung
von `total_steps` beim Fortsetzen würde das Experiment ändern und wird abgelehnt.
Bleibt nach Warmup nur ein Update übrig, verwendet es die minimale Lernrate.
Ein Lauf mit genau einem Schritt und ohne Warmup verwendet deshalb
`learning_rate * min_lr_ratio`. Bei drei Schritten mit zwei Warmup-Schritten
lautet das Beispiel `[0.005, 0.010, 0.001]`; für einen zusätzlichen Schritt am
Maximum ist dann kein Platz.

**Gradienten-Clipping** skaliert den gesamten Gradientenvektor, wenn seine
euklidische Norm zu groß wird. `[3,4]` hat Norm 5; die Grenze 1 ergibt ungefähr
`[0.6,0.8]`. `gradient_norm` protokolliert die Norm **vor** dem Clipping. Clipping
kommt einmal nach der akkumulierten Rückwärtsrechnung und vor AdamW. Es begrenzt
die Gradientennorm; es repariert keine nichtendlichen Parameter und macht keine
beliebig hohe Lernrate sicher.

## Einen Neustart selbst bauen [#einen-neustart-selbst-bauen]

```python
from llm_course.data_pipeline import PaddedDocumentDataset, build_split, make_batch_loader
from llm_course.mini_gpt import GPTConfig, MiniGPT
from llm_course.training import TrainConfig, Trainer

batches = list(make_batch_loader(
    PaddedDocumentDataset(build_split().train, max_length=33), batch_size=2, seed=7))
config = TrainConfig(total_steps=20, accumulation_steps=2)
trainer = Trainer(MiniGPT(GPTConfig(dropout=0.1)), batches, config)
trainer.train(10)  # zusätzliche Optimiererschritte
trainer.save_checkpoint('artifacts/my-pretraining/resume.pt')
restarted = Trainer(MiniGPT(GPTConfig(dropout=0.1)), batches, config)
restarted.load_checkpoint('artifacts/my-pretraining/resume.pt')
new_rows = restarted.train()  # restliche Schritte; history enthält die ersten zehn
```

Der Checkpoint speichert Modell und Architektur, AdamW-Momente und Schrittzähler,
Präzisionsskalierer, Trainingskonfiguration, nächste absolute Batchposition,
Tokenzähler, Metrikhistorie sowie Torch-Zufallszustände von CPU/Gerät. Konfiguration
und abgeschlossener Schritt bestimmen den Lernratenplan vollständig; ein weiterer
veränderlicher Scheduler ist unnötig. Innerhalb des Trainers benutzt nur Torch
Zufall. Der Trainer verwaltet seinen eigenen Zufallsstrom und stellt den Strom des
Aufrufers danach wieder her. Python-/NumPy-Zufall wird in dieser Schleife nicht
benutzt. Die Daten sind bereits materialisiert: Ein DataLoader-Iterator muss beim
Laden nicht rekonstruiert werden.

Gespeichert wird nur zwischen abgeschlossenen Optimiererschritten, nie mit
teilweise akkumulierten Gradienten. Modell, Trainingskonfiguration, Datenfingerabdruck,
aufgelöstes Gerät, Präzision, Torch-Version, Threadzahl und Einstellung für
deterministische Algorithmen werden vor dem Wiederherstellen geprüft. Eine Änderung
dieser Identität erfordert einen neuen Lauf. Der Loader ist für kursinterne
Checkpoints gedacht, nicht zur Reparatur beliebig beschädigter Dateien.

PyTorch garantiert keine exakte Reproduzierbarkeit über Releases, Plattformen oder
CPU/GPU hinweg. Unsere Zusage gilt für dieselbe CPU-Laufzeit. Die
[PyTorch-Hinweise zur Reproduzierbarkeit](https://docs.pytorch.org/docs/2.14/notes/randomness.html)
beschreiben diese Grenzen.

## Messen und gezielt kaputtmachen [#messen-und-gezielt-kaputtmachen]

Das JSON protokolliert Loss, Norm vor Clipping, Lernrate, überwachte Tokens,
kumulierte Tokens, Sekunden, Tokens/Sekunde und Parameterbytes. Die CPU-Referenz
hat **114.176 Parameterbytes** (28.544 FP32-Parameter). Das ist kein gesamter
Trainingsspeicher: Gradienten, AdamW-Momente, Aktivierungen und Framework benötigen
zusätzlichen Platz. `peak_device_memory_bytes` erfasst auf CUDA den maximal
allozierten Gerätespeicher pro Update; CPU/MPS liefern `null`, weil diese Messung
dort nicht erhoben wird. Die Zeit enthält Transfer und Update mit Gerätesynchronisation,
aber weder Loader-/Modellaufbau noch Checkpoint-I/O. Aufwärmen und andere Systemlast
verändern den Durchsatz; er ist kein Qualitätskriterium zum Bestehen.

`--unstable` trainiert ein separates Modell mit Lernrate `1e20`, ohne Warmup und
in FP32. Die Referenz stoppt bei nullbasiertem Schritt 1 mit `nonfinite loss`, nach
einem abgeschlossenen Update. Das ist ein kontrollierter Überlauf, kein realistischer
Tuningwert. Ein großer endlicher Loss ist noch kein automatischer Fehler. Prüfe
Loss **und** Gradientennorm, Labels und Lernrate; lade einen guten Checkpoint.
`TrainingDiverged` stoppt vor dem aktuellen Optimiererupdate bei nichtendlichem Loss
oder Norm. Nach dieser Ausnahme vom guten Checkpoint neu starten; ein fehlgeschlagenes
AMP-Update nicht einfach fortsetzen.

Optionale Profile mit CPU-Ersatz, falls das Gerät fehlt:

```bash
python examples/run_pretraining.py --profile mps-small --precision float16
python examples/run_pretraining.py --profile cuda --precision bfloat16
```

Der Trainer verlangt **FP32-Masterparameter**: die gespeicherten Gewichte, die der
Optimierer aktualisiert. Übergebene `.double()`- oder `.half()`-Modelle werden vor
dem Training abgelehnt, statt falsch bezeichnet oder still umgewandelt zu werden.
Nutze einen normalen FP32-`MiniGPT` und fordere die Rechenpräzision über
`TrainConfig.precision` an; ändere nach dem Erzeugen des Trainers keinen
Parameterdatentyp. Autocast kann Operationen mit geringerer Präzision ausführen,
während die Masterparameter FP32 bleiben.

CUDA nutzt bei Verfügbarkeit Kontext 64, Breite 128, vier Schichten und Batchgröße 8.
CPU/MPS bleiben in dieser Lektion FP32, auch wenn niedrigere Präzision angefordert
wird; der Bericht erklärt den Rückfall. CUDA-FP16 nutzt Autocast und
Gradientenskalierung, BF16 Autocast ohne Skalierung. Während der Akkumulation bleibt
der Skalierungsfaktor gleich; erst danach wird einmal entskaliert und geclippt.
Die Reihenfolge entspricht den [PyTorch-AMP-Beispielen](https://docs.pytorch.org/docs/2.14/notes/amp_examples.html).
Unterschiede beim Accelerator-Neustart werden ausgewiesen; der Pflichtnachweis ist CPU.

## Übung, Hinweise und Referenz [#ubung-hinweise-und-referenz]

1. Implementiere die Tokengewichtung in einer Arbeitskopie: A hat 2 gültige Tokens
   mit Loss 1, B hat 6 mit Loss 3. Berechne den erwarteten Loss vor dem Lauf.
2. Speichere nach Schritt 4, ersetze ein Label und versuche fortzusetzen. Stelle das
   Label wieder her und ändere nur `total_steps`. Warum müssen beide Versuche scheitern?
3. Warum beweist gleicher End-Loss keinen exakten Neustart? Nenne zwei weitere
   Checkpointbestandteile, deren Fehlen das nächste Update verändert.

**Hinweis 1:** Zähle überwachte Tokens, nicht Zeilen oder Kapazität.
**Hinweis 2:** Ein Lernratenplan hängt vom ursprünglichen Horizont ab.
**Hinweis 3:** Denke an Optimierergedächtnis und Dropouts nächste Zufallszahlen.

**Referenz:** `(2×1+6×3)/8 = 2,5`; die Gewichte sind 1/4 und 3/4. Das geänderte Label
ändert den Fingerabdruck, der Horizont die Konfiguration. Vergleiche jeden Parameter,
AdamW-Momente und Schrittzähler, Zufallsstrom, Batchposition und deterministische
Qualitätsmetriken. Ein zufällig gleicher Skalar-Loss kann das nicht beweisen.

Gehe weiter, wenn du die Gewichte ohne Code herleitest, die Position von Clipping
erklärst, CPU-Neustart exakt reproduzierst und den absichtlichen Fehler erkennst.
Erkläre außerdem, warum sinkender Trainings-Loss noch nichts über zurückgehaltene
Daten aussagt. Darum geht es im Evaluationslabor.
