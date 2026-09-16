---
title: "Glossar"
sidebar:
  label: "Glossar"
---

<span id="glossar" />


[Kursstart](/llm-engineering-course-pages/de/) · [Lernleitfaden](/llm-engineering-course-pages/de/learning-guide)

Die Beispiele und Herleitungen stehen in den verlinkten Lektionen. Diese Seite hilft beim Nachschlagen.

| Begriff | Bedeutung |
| --- | --- |
| Token / Token | Eine diskrete Texteinheit: Zeichen, Byte oder gelerntes Teilstück; die ID ist ihre Ganzzahlkennung. |
| Vocabulary / Vokabular | Alle Token-IDs, die das Modell vorhersagen kann; V ist ihre Anzahl. |
| Scalar, vector, matrix, tensor / Skalar, Vektor, Matrix, Tensor | Eine Zahl; eine Zahlenliste; eine Tabelle; ein Zahlenarray mit beliebig vielen Achsen. |
| Shape, B/T/D/H/V / Shape, B/T/D/H/V | Achsenlängen: Beispiele im Batch, Tokenpositionen, versteckte Breite, Attention-Heads und Vokabular. |
| Parameter / Parameter | Eine gespeicherte trainierbare Zahl, z. B. ein Gewicht. Das Training verändert sie. |
| Activation / Aktivierung | Ein Zwischenwert für eine bestimmte Eingabe; etwas anderes als ein gespeichertes Gewicht. |
| Embedding / Embedding | Eine Tabelle, die eine Token- oder Positions-ID in einen gelernten Vektor übersetzt. |
| Logit, softmax / Logit, Softmax | Ein unbeschränkter Score; die Operation, die eine Scorezeile in positive Wahrscheinlichkeiten mit Summe eins verwandelt. |
| Loss, NLL, cross-entropy / Loss, NLL, Kreuzentropie | Vorhersagestrafe. Für ein wahres nächstes Token ist die negative Log-Likelihood -ln(zugewiesene Wahrscheinlichkeit). |
| Perplexity / Perplexität | exp(mittlere NLL). Gleichverteilte Wahl unter vier Tokens hat NLL ln(4) und Perplexität 4; nur bei gleichem Tokenizer und gleichen Daten vergleichen. |
| Gradient, backpropagation / Gradient, Backpropagation | Empfindlichkeiten des Loss gegenüber Parametern; der Kettenregel-Algorithmus berechnet sie rückwärts durch den Graphen. |
| Optimizer, learning rate / Optimizer, Lernrate | Die Regel zum Aktualisieren der Parameter anhand von Gradienten; die Steuerung ihrer Schrittweite. |
| Batch, microbatch, accumulation / Batch, Microbatch, Accumulation | Gemeinsam verarbeitete Beispiele; ein kleinerer Teil; Gradienten mehrerer Teile vor einem Update sammeln. |
| Epoch, step / Epoche, Schritt | Ein Durchlauf des Datensatzes; ein Optimizer-Update. Mehrere Microbatches können einen Schritt bilden. |
| Causal mask / Kausale Maske | Eine Karte erlaubter Positionen, die Zukunftsinformation für eine Vorhersage sperrt. |
| Padding, loss mask / Padding, Lossmaske | Füll-Tokens für gleich lange Sequenzen; eine Auswahl der Ziele, die zum Loss zählen. |
| Residual, normalization, FFN / Residual, Normalisierung, FFN | Blockeingabe wieder zum Ergebnis addieren; Featurewerte umskalieren; ein Feed-forward-Netz pro Position. |
| Pretraining / Pretraining | Lernen der nächsten Tokens aus Text. Das allein lehrt keine zuverlässige Instruktionsbefolgung. |
| Train / validation / test | Daten für Updates / Konfigurationswahl / finale Evaluation nach Festlegung aller Entscheidungen. |
| Overfitting / Überanpassung | Training wird besser, Qualität unbekannter Daten stagniert oder sinkt; Tiny-Batch-Overfit dient absichtlich als Mechanikprüfung. |
| Baseline, ablation / Baseline, Ablation | Das unveränderte Referenzexperiment; ein kontrollierter Vergleich mit Änderung einer Komponente. |
| Seed, checkpoint, resume / Seed, Checkpoint, Resume | Startwert eines Zufallsgenerators; gespeicherter Experimentzustand; Fortsetzung mit diesem Zustand. |
| Leakage, memorization / Leakage, Memorierung | Zurückgehaltene Information gelangt ins Training/die Auswahl; Gesehenes wiedergeben statt zu generalisieren. |
| Paired seeds, uncertainty / Gepaarte Seeds, Unsicherheit | Baseline und Änderung mit passenden Seeds ausführen; Streuung zeigt mögliche Unsicherheit der Differenz. |
| FLOP, throughput, memory / FLOP, Durchsatz, Speicher | Eine Gleitkommaoperation; verarbeitete Tokens pro Sekunde; Bytes für Parameter, Gradienten, Optimizer und Aktivierungen. |
| Mixed precision / Gemischte Präzision | Einige Operationen nutzen kleinere Zahlenformate, um Aufwand zu senken; im Kurs gibt es einen Float32-Fallback. |
| RAG, SFT, LoRA / RAG, SFT, LoRA | Spätere Einheiten: externe Texte abrufen; überwachtes Instruktions-Finetuning; Anpassung mit kleinen trainierbaren Matrizen niedrigen Rangs. |

[F01](/llm-engineering-course-pages/de/foundations-01-python-numpy) · [F02](/llm-engineering-course-pages/de/foundations-02-shapes) · [F03](/llm-engineering-course-pages/de/foundations-03-probability) · [F04](/llm-engineering-course-pages/de/foundations-04-neuron) · [F05](/llm-engineering-course-pages/de/foundations-05-mlp-autograd) · [F06](/llm-engineering-course-pages/de/foundations-06-pytorch-checkpoint)
