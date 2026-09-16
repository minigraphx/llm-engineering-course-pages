---
title: "M4 — Continued Pretraining ohne Verlust der Baseline"
sidebar:
  label: "M4 — Continued Pretraining"
---

<span id="m4-continued-pretraining-ohne-verlust-der-baseline" />


Continued Pretraining (DAPT) setzt das Sprachmodellziel auf freigegebenem
Fachtext fort. Es lernt Wortschatz und Stil, aber keinen JSON-Antwortvertrag.

Der Lauf nutzt den Firmensnapshot, Byte-Fallback-Tokenisierung, getrennte
Fach-Holdouts und zwei allgemeine Kontrollen. Ein unveränderliches Basismodell,
unterbrochene Fortsetzung und Rollback werden geprüft.

```bash
.venv/bin/python examples/run_continued_pretraining.py --seeds 7 19 43
```

Die Messungen verbessern Domain-NLL um 0,359–0,411 Nats, verschlechtern die
allgemeine Kontroll-NLL aber um 0,032–0,109 Nats. Deshalb bleibt der Kandidat in
Prüfung. Exakte Fortsetzung und Rollback sind Engineering-Nachweise, kein
Qualitätsbeweis.

### Aufgabe [#aufgabe]

Warum werden Domaingewinn und allgemeine Regression getrennt berichtet? Was macht
den Versuch ungültig: einen allgemeinen Testsatz in den Trainingstext aufnehmen
oder nur den Lernraten-Seed ändern?

<details>
<summary>Referenzantwort</summary>

Getrennte Werte zeigen Vergessen. Testsatztext im Training ist Kontamination;
ein geänderter Seed ist ein neuer, zu dokumentierender Lauf.


</details>

Checkpoint: Lege eine Freigabeschwelle fest, benenne das Rollback-Artefakt und
begründe die anschließende Instruktions-Evaluation.
