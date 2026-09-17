---
title: "C02 — Prüffähige Firmendaten"
sidebar:
  label: "C02 — Firmendaten"
---

<span id="c02-pruffahige-firmendaten" />


[← C01 — Firmenstrategie](/llm-engineering-course-pages/de/company-strategy) · [C03 — Geschützte Evaluation →](/llm-engineering-course-pages/de/company-evaluation) · [Glossar](/llm-engineering-course-pages/de/glossary)

Voraussetzungen: [C01 Strategie](/llm-engineering-course-pages/de/company-strategy) und [Datenqualität](/llm-engineering-course-pages/de/data-quality).
Nordlicht Workspace ist erfunden; die gesamte Quellenliste kann ohne Kundendaten
geprüft werden.

## Erst einen Snapshot bauen, dann Beispiele erzeugen [#erst-einen-snapshot-bauen-dann-beispiele-erzeugen]

`source_inventory()` enthält aktuelle Handbuchfakten und absichtlich gesetzte
Prüffälle: einen gelöschten Preis, einen Secret-Marker und einen PII-Marker.
`build_snapshot()` schließt diese Zeilen aus, protokolliert den Grund, ersetzt
E-Mail-Muster und liefert einen stabilen SHA-256-Hash. Der Hash bezeichnet genau
den freigegebenen Snapshot.

Der Generator ist deterministisch und führt die Quellenherkunft mit. Jede
Instruktion besitzt eine Aufgabe (`billing`, `access`, `export` oder `escalate`),
ein Split-Feld und Quellen-IDs. Der Filter lehnt unbekannte Aktionen, doppelte oder
geschützte Prompts, Secrets, PII und unbekannte Quellen ab. Es gibt 16 Trainings-
und 8 Entwicklungszeilen. Geschützte Goldfälle werden nicht importiert.

Bevorzugte Antworten werden mit einer klar unsicheren Alternative gepaart. Die
Ablehnung ist ein Lernsignal, niemals eine Produktionsantwort. Roh- und generierte
Daten bleiben in ignorierten `artifacts/`; versioniert werden Manifest und Code.

## Eine Zeile verfolgen [#eine-zeile-verfolgen]

1. Quellen-ID, Lizenz und Zugriffsentscheidung prüfen.
2. Prüfen, dass die Aktion zur Aufgabe passt und exakt zwei String-Felder besitzt.
3. Vor dem Training Kontaminationsprüfungen ausführen.
4. Datensatz-Hash sowie Code-/Modellrevision notieren.

### Aufgabe [#aufgabe]

Gib `filter_instruction_candidates` erst eine Zeile mit `[SECRET_001]`, dann einen
doppelten Prompt. Sage die beiden Audit-Entscheidungen voraus. Warum muss eine
gelöschte Quelle auch dann ausgeschlossen werden, wenn ihr Text nützlich wirkt?

<details>
<summary>Hinweis</summary>

Lies die Audit-Zeile, die der Filter für jeden abgelehnten Kandidaten
schreibt: Die Entscheidung nennt die Regel, die gegriffen hat. Prüfe dann,
was `build_snapshot()` mit dem gelöschten Preis bereits getan hat, bevor
eine Instruktion erzeugt wurde.


</details>

<details>
<summary>Referenzantwort</summary>

Der Secret-Marker wird ausgeschlossen, der zweite Prompt als Duplikat. Eine
gelöschte Quelle ist nicht mehr freigegeben; ihre Nutzung bricht Herkunft und
Richtlinienprüfung.


</details>

Checkpoint: Erkläre den Unterschied zwischen Quellensnapshot, Trainingszeile,
Entwicklungsfall und geschütztem Goldfall. Weiter mit
[Firmen-Evaluation](/llm-engineering-course-pages/de/company-evaluation).

[← C01 — Firmenstrategie](/llm-engineering-course-pages/de/company-strategy) · [C03 — Geschützte Evaluation →](/llm-engineering-course-pages/de/company-evaluation) · [Glossar](/llm-engineering-course-pages/de/glossary)
