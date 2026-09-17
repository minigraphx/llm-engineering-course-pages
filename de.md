---
title: "LLM-Engineering-Kurs"
sidebar:
  label: "Start"
---

<span id="llm-engineering-kurs" />


Verstehe Sprachmodelle, indem du ihre wichtigen Bestandteile selbst baust.
Später verwendest du sie für einen realistischen Firmenfall.

Beginne mit der [Einrichtung](/llm-engineering-course-pages/de/setup) und dem [Lernleitfaden](/llm-engineering-course-pages/de/learning-guide).
Er nennt Voraussetzungen und Checkpoints. Das [Glossar](/llm-engineering-course-pages/de/glossary) erklärt
unbekannte Begriffe. Vorkenntnisse in Machine Learning sind nicht erforderlich.

## Zwei Einstiegspfade [#zwei-einstiegspfade]

- **Standalone:** Python, Mathematik, neuronale Netze und PyTorch sind enthalten.
- **Godot-RL-Brücke:** Eine Diagnose erkennt bereits belegte Fähigkeiten aus dem
  Godot-RL-Kurs an und vermeidet unnötige Wiederholung.

Beide Wege beginnen mit [Einheit 0](/llm-engineering-course-pages/de/unit-00) und der
[Diagnose](/llm-engineering-course-pages/de/diagnostic). Neueinsteiger bearbeiten [F01–F06](/llm-engineering-course-pages/de/foundations-01-python-numpy);
Godot-RL-Absolventen nutzen die [kompakte Brücke](/llm-engineering-course-pages/de/foundations-godot-rl-bridge).
Jede übersprungene Grundlage behält einen Rücksprunglink. Alle Wege führen über
das gemeinsame F06-Gate zum selben Kernkurs.

Danach folgen [T01 Bigramm-Baseline](/llm-engineering-course-pages/de/bigram-baseline), [T02 Byte-Level-BPE](/llm-engineering-course-pages/de/bpe-tokenizer),
[T03 Datenpipeline und Embeddings](/llm-engineering-course-pages/de/data-pipeline), [T04 Self-Attention](/llm-engineering-course-pages/de/attention),
[T05 Mini-GPT](/llm-engineering-course-pages/de/mini-gpt), das [T06 Kerngate](/llm-engineering-course-pages/de/mini-gpt-gate) und
[I01 Decoding, Sampling und KV-Cache](/llm-engineering-course-pages/de/inference-decoding). E01–E04 ergänzen
[reproduzierbares Training](/llm-engineering-course-pages/de/pretraining), [Datenqualität](/llm-engineering-course-pages/de/data-quality),
[Evaluation](/llm-engineering-course-pages/de/evaluation) und [Profiling](/llm-engineering-course-pages/de/profiling). Die Firmenstrecke
[C01–C04](/llm-engineering-course-pages/de/company-strategy) führt von der Anforderung zur Modellentscheidung,
zu prüfbaren Daten, geschützter Evaluation und Continued Pretraining; die
Adaptationsstrecke [A01–A04](/llm-engineering-course-pages/de/sft-lora) behandelt SFT/LoRA, den
Adaptationsvergleich, DPO und die RLHF-Brücke; der [Capstone](/llm-engineering-course-pages/de/company-capstone)
verbindet alles zu einer umkehrbaren Firmenmodell-Entscheidung. Der Track
[„Angewandt: RAG“](/llm-engineering-course-pages/de/rag-build) (R01–R02, derzeit nur auf Englisch) ist ein
eigenständiger Einstieg, der nur Python und einen LLM-Aufruf braucht.

Alle Pflichtziele haben einen CPU-Pfad. Prüfe vor größeren Experimenten die
[Hardware- und Kostenleitplanken](/llm-engineering-course-pages/de/hardware).

## Was du im gesamten Kurs bauen wirst [#was-du-im-gesamten-kurs-bauen-wirst]

- eine Bigramm-Sprachmodellbaseline;
- einen BPE-Tokenizer;
- Self-Attention und einen Decoder-Transformer;
- ein kleines GPT-Modell mit Training und Evaluation;
- Instruktions- und Präferenz-Finetuning;
- eine synthetische Datenpipeline mit Qualitätsprüfungen;
- einen optimierten lokalen Inferenzdienst;
- einen reproduzierbaren Firmenmodell-Capstone.

<aside className="course-note" aria-label="Entwicklungsstand">
<strong>Entwicklungsstand</strong>

Alle Kurseinheiten von Einheit 0 bis zum Capstone sind ausgeliefert und auf
CPU ausführbar: die Grundlagen F01–F06 mit der B01-Brücke, der Kern T01–T06
und I01, die Engineering-Einheiten E01–E04, die Firmeneinheiten C01–C04, die
Adaptationseinheiten A01–A04 und der Capstone sowie der Track „Angewandt:
RAG“ R01–R02. Die deutsche Fassung ist vollständig bis auf R01/R02 (MIN-129).
Noch offen: die größeren bereitgestellten Datenpakete (M1A), die
Interpretierbarkeitserweiterung (M2A), die Einheiten zu Quantisierung,
Batching und Serving (M6), die menschliche Freigabe des versiegelten
Gold-Sets und der synthetischen Stichprobe (MIN-95/96/100) und der echte
Pilot mit fünf Lernenden. Automatische Prüfungen belegen weder das
Verständnis aller Lernenden noch die Ausstellung eines Zertifikats.

</aside>
