---
title: "E02 — Datenqualität und Provenienz"
sidebar:
  label: "E02 — Datenqualität"
---

<span id="e02-datenqualitat-und-provenienz" />


[Kursstart](/llm-engineering-course-pages/de/) · [← Training](/llm-engineering-course-pages/de/pretraining) · [→ Evaluation](/llm-engineering-course-pages/de/evaluation)

## Bevor du beginnst [#bevor-du-beginnst]

Schließe [T03](/llm-engineering-course-pages/de/data-pipeline) ab: Unterscheide Dokument, Tokenfenster und
Split. Wiederhole [Loss](/llm-engineering-course-pages/de/foundations-03-probability), falls ein kleinerer Wert
noch automatisch besser wirkt. Plane 60–90 Minuten. Das Lab braucht nur CPU
und erzeugt erfundene Beispiele im Speicher.

Du erklärst die Herkunft eines Trainingsbeispiels, findest Kontamination,
reparierst Trainingsdaten ohne Änderung des Benchmarks und misst eine scheinbare
Verbesserung. **Provenienz** ist die dokumentierte Herkunft und Verarbeitung.
**Governance** regelt, wer eine Quelle wofür und mit welchen Zugriffs- und
Aufbewahrungsregeln verwenden darf.

## Predict: Können 100% Genauigkeit wertlos sein? [#predict-konnen-100-genauigkeit-wertlos-sein]

Das Training enthält acht Paare aus willkürlichen Karten-IDs und Labels. Die
Evaluation enthält vier andere IDs. Ein Programm, das nur Trainingsantworten
speichert, liegt bei 0/4 richtig. Kopiere die vier Evaluationspaare ins Training:
Jetzt erreicht es 4/4. Der Algorithmus wurde nicht besser; seine Eingabe enthielt
die Antworten. Das nennt man **Leakage**.

```bash
python examples/run_data_quality.py
```

Der Bericht landet zusätzlich in `artifacts/data-quality/report.json`. Erwartet:

| Feld | Vorher | Nach Reparatur |
| --- | --- | --- |
| Dokumentanzahl | 9 | 4 |
| Exakte Duplikatpaare | 2 | 0 |
| Near-Duplicate-Paare | 2 | 0 |
| Exakte Duplikatpaare über Splitgrenzen | 1 | 0 |
| Datensätze mit sensiblen Markern | 1 | 0 |
| Datensätze ohne Provenienz | 1 | 0 |

In `leakage_experiment` beträgt die saubere Accuracy 0, die kontaminierte 1 und
die reparierte wieder 0. Das sind Anteile: 1 bedeutet 100%. Der Runner evaluiert
einen einfachen Nachschlage-Speicher mit willkürlichen Labels. Das ist ein
bewusst einfaches Gegenbeispiel, kein Sprachmodellbenchmark. Ein niedrigerer
LM-Loss auf zurückgehaltenen Daten braucht denselben Schutz.

## Trace: Herkunft → Audit → Entscheidung → Version [#trace-herkunft-audit-entscheidung-version]

Ein `QualityDocument` enthält `id`, `text`, `split`, `source` und `license`.
Die ersten Duplikate haben unterschiedliche IDs und identischen Text: Ein
ID-Vergleich reicht nicht. Die Normalisierung von Großschreibung und Leerraum
findet oberflächliche Unterschiede. Für ähnliche Texte vergleichen wir Mengen
aus fünf Zeichen langen Stücken (**Shingles**). Teilen zwei Mengen 8 Stücke bei
10 Stücken in der Vereinigung, beträgt die Jaccard-Ähnlichkeit `8/10 = 0.8`.
Unser Lehr-Schwellenwert ist 0.8. Er kann auch legitime wiederholte Formulierungen
markieren; prüfe die Belege, bevor du diese Regel auf reale Daten überträgst.

```python
from llm_course.data_quality import (
    generate_quality_documents, audit_documents, repair_documents,
)

raw = generate_quality_documents()
before = audit_documents(raw)
clean, decisions = repair_documents(raw)
assert before["cross_split_duplicates"] == [["leaked", "test"]]
assert audit_documents(clean)["ready"]
for decision in decisions:
    print(decision["id"], decision["action"], decision["reason"])
```

Die Reparatur behält die zurückgehaltene Kopie und entfernt das Trainingsduplikat.
Validation/Test werden niemals still geändert. Ein Defekt darin löst einen
Fehler aus: Es braucht unabhängige Prüfung und eine neue Benchmarkversion.
Sensible Marker wie `[EMAIL_001]` und `[SECRET_001]` werden zu `[REDACTED]`;
Einträge mit `[TOXIC]` kommen in Quarantäne. Diese Regeln demonstrieren
Mechanismen. Sie erkennen nicht alle personenbezogenen Daten, Secret-Formate,
problematischen Inhalte oder fehlenden Berechtigungen.

## Build: Eine Datenmischung wählen und erklären [#build-eine-datenmischung-wahlen-und-erklaren]

Ein **Samplinggewicht** bestimmt, wie oft eine Quelle gezogen wird. Bei 20
Beispielen und 75% Original / 25% synthetisch ziehen wir 15 und 5. Ziehen mit
Zurücklegen kann Datensätze wiederholen; Wiederholungen sind kein neues Wissen.

```python
from llm_course.data_quality import QualityDocument, deterministic_mixture

original = [doc for doc in clean if doc.split == "train"]
synthetic = [QualityDocument("new", "new authored teaching example", "train")]
mix = deterministic_mixture(
    {"original": original, "synthetic": synthetic},
    {"original": 0.75, "synthetic": 0.25}, size=20, seed=7,
)
assert sum(doc.id == "new" for doc in mix) == 5
```

Probiere 50/50 mit gleicher Größe und gleichem Seed. Sage die Anzahl vorher.
Berichte einzigartige Dokumente getrennt von gezogenen Beispielen. Wähle
Gewichte anhand der Validation, nicht anhand des geschützten finalen Tests.

## Selbst brechen, messen und reparieren [#selbst-brechen-messen-und-reparieren]

Kopiere eine Validation-Zeile mit neuer ID und `split="train"`. Erkläre vorab,
warum das weiterhin Leakage ist. Führe den Audit aus, entferne die Trainingskopie
und zeige, dass Text und Mitgliedschaft des zurückgehaltenen Datensatzes gleich
bleiben. Ändere dann das letzte Zeichen: Findet der Near-Duplicate-Test die Kopie
noch? Notiere Ähnlichkeitswert und mögliche Fehlalarme.

<details>
<summary>Hinweis</summary>

Nutze `dataclasses.replace(doc, id="copied", split="train")`. Der Split muss
vor der Fensterbildung erfolgen. Vergleiche Inhalte und nicht nur IDs.


</details>

<details>
<summary>Referenzbegründung</summary>

Eine neue ID macht den Text nicht neu. Der exakte Vergleich findet die Kopie;
Shingles können kleine Änderungen erkennen. Entferne die Trainingskopie und
bewahre den ursprünglichen Benchmark. Die Demonstration fällt von 100% auf
0% zurück und zeigt ehrlich die fehlende Generalisierung des Speichers.


</details>

## Quellen- und Lizenzinventar [#quellen-und-lizenzinventar]

Erfasse vor realer Beschaffung Eigentümer, Quell-URL/Version, Lizenz/Erlaubnis,
Zweck, Sprache, Zugriffsgruppe, personenbezogene Kategorien, Aufbewahrung und
Lösch-/Neubauverfahren. Öffentlich sichtbar bedeutet nicht für Training freigegeben.
Unklare Rechte bleiben bis zur Prüfung außerhalb des Trainingssnapshots.
Automatische Ersetzung verringert mögliche Offenlegung; sie erteilt keine
Berechtigung und beweist keine Anonymisierung.

Der Kurs liefert Generatoren und Manifeste für Einheit 0, BPE und dieses Lab.
Alle drei haben Data Cards unter `docs/data-cards/`. Jedes Manifest nennt
Generator, Lizenz, UTF-8-Bytezahl, SHA-256 und Splitregeln. Ein **Hash** bezeichnet
exakte Bytes: Ein geändertes Leerzeichen ändert die Identität. Ändere einen Hash
nicht, um eine ungeklärte Abweichung zu verstecken. Eine neue Version braucht
eine neue Baseline und eine dokumentierte Migration. Die [Datenrichtlinie](/llm-engineering-course-pages/de/data-policy)
unterscheidet heutige Mini-Fixtures von geplanten größeren Datenpaketen. Erzeugte
Korpora, reale personenbezogene Daten und Gewichte gehören nicht ins Git.

## Abschlusskriterium [#abschlusskriterium]

Führe `pytest tests/test_data_quality.py` aus. Reiche Vorher-/Nachher-Bericht,
einen reparierten Kontaminationsfehler, eine erklärte Mischung und eine Data Card
für eine erfundene Quelle ein. Erkläre, warum `ready=true` nur bestandene
Lehrprüfungen bedeutet und warum die Wiederherstellung der Testintegrität einen
berichteten Wert verschlechtern kann.
