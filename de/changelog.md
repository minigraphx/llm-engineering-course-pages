---
title: "Neu im Kurs"
sidebar:
  label: "Neu im Kurs"
---

<span id="neu-im-kurs" />


[Kursstart](/de/)

Datierte Änderungen, die Lernende bemerken, neueste zuerst. Jeder Eintrag
verlinkt die geänderte Seite. Interne Arbeit — Tests, Werkzeuge,
Abhängigkeiten — steht nicht hier; die Git-Historie hat sie.

## 2026-09-24 — Einheit 0: einen Logit selbst verstärken [#2026-09-24-einheit-0-einen-logit-selbst-verstarken]

Vor „Annahme brechen“ hat Einheit 0 jetzt ein interaktives Element: Wähle den Prompt
(`"the "` oder `"a "`), ein Zielzeichen und eine Verstärkung von −6 bis +6 und sieh
zu, wie sich beim Ziehen die Wahrscheinlichkeit des Ziels, die fünf
wahrscheinlichsten nächsten Zeichen vorher und nachher, beide Greedy-Texte, der
Korpus-Loss und das Urteil ändern. Es baut dieselbe gezählte Bigram-Tabelle wie das
Skript nach, deshalb stimmen seine Zahlen mit den Zusammenfassungen der Seite
überein; drei Schaltflächen spielen die Läufe der Seite nach (+4 auf m, +3 auf t,
−4 auf t).
→ [Einheit 0 — Erster Tiny-LM-Lauf](/de/unit-00)

## 2026-09-23 — R01 und R02 jetzt auf Deutsch [#2026-09-23-r01-und-r02-jetzt-auf-deutsch]

Beide RAG-Lektionen liegen jetzt vollständig auf Deutsch vor, von der Vorbereitung
des Korpus und der Embedding-Rechnung über die Guardrails bis zum Referenzlauf und zu
den eigenständigen Übungen. Befehle, Code und Programmausgaben bleiben wie auf den
anderen deutschen Seiten englisch; die Fragen an den Index bleiben englisch, weil der Index
aus den englischen Seiten gebaut wird.
→ [R01 — Vom Dokument zur Antwort](/de/rag-build) · [R02 — Damit es nicht lügt](/de/rag-quality)

## 2026-09-23 — Einheit 0: zehn Zeilen Zusammenfassung statt eines JSON-Blocks [#2026-09-23-einheit-0-zehn-zeilen-zusammenfassung-statt-eines-json-blocks]

`run_unit0.py` gibt jetzt zehn Zeilen aus — Eingabe und Token-IDs, die
wahrscheinlichsten Kandidaten, beide Generierungen, die Zielwahrscheinlichkeit und
den Korpus-Loss vor → nach der Änderung sowie ein berechnetes Urteil — und speichert
den vollständigen Bericht unter `artifacts/unit0/report.json`; `--json` gibt ihn aus.
Die Seite erklärt jede Zeile, geht den vollständigen Bericht einmal durch und zeigt,
warum `--boost 3` eine Wahrscheinlichkeit verschiebt, ohne den Text zu ändern.
`diagnose_unit0.py` gibt eine Zeile pro Prüfung aus, und ein leerer Prompt liefert
jetzt eine einzeilige Fehlermeldung statt eines Tracebacks. AG01 startet
`run_unit0.py` jetzt ohne Umleitung der Ausgabe. Zeichen mit gleicher
Wahrscheinlichkeit stehen jetzt nach Token-ID sortiert, sodass die Kandidatenlisten
in den Berichten von Einheit 0 und T01 auf jeder Plattform gleich aussehen.
→ [Einheit 0 — Erster Tiny-LM-Lauf](/de/unit-00)

## 2026-09-23 — R01 und R02: jede Zahl auf dem heutigen Kurs gemessen [#2026-09-23-r01-und-r02-jede-zahl-auf-dem-heutigen-kurs-gemessen]

Die RAG-Lektionen zitierten einen Referenzlauf von vor der Zeit, als die Agenten-Seiten
in den Korpus kamen. Jede Zahl in R01, R02 und im RAG-README stammt jetzt aus einem
neuen Lauf auf den heutigen 41 Seiten: `improved` schlägt `naive` beim Retrieval
weiterhin (Recall@1 0,878 gegenüber 0,561), aber seine lexikalische Groundedness ist
niedriger (0,557 gegenüber 0,631), und R02 sagt das jetzt, statt sie unverändert zu
nennen. Das Offline-Beispiel in R01 zeigt eine andere erste Passage. Ein Test meldet
jetzt, wenn eine neue Seite diese Zahlen verschiebt. Die Lektionen sind weiterhin nur
auf Englisch verfügbar.
→ [R02 — Damit es nicht lügt](/de/rag-quality) · [R01 — Vom Dokument zur Antwort](/de/rag-build)

## 2026-09-23 — AG03: messen, was eine Abwehr gegen Prompt Injection bringt [#2026-09-23-ag03-messen-was-eine-abwehr-gegen-prompt-injection-bringt]

Ein Angriffs-Set und dasselbe präparierte Dokument laufen gegen R02s
Retrieval-Pipeline und AG02s Agenten durch eine kumulative Verteidigungsleiter —
delimiters, escaping, spotlighting, ein pattern filter, ingest cleaning, ein
confirmation-Schritt und zuletzt das Einschränken des Sende-Tools, sodass kein
Aufruf einen Empfänger wählen kann. Jede Messung läuft nur auf lokalen
Qwen-Modellen, nie auf einer gehosteten API. Beim kleinen
Modell fällt die Erfolgsquote beim Retrieval von 7 von 8 auf 3 von 8 und erreicht
nie null; der 4B-Agent geht auf keiner Stufe auf den Köder ein, doch dieselben
Abwehrmaßnahmen kosten ihn trotzdem die Aufgabe — von 6 von 8 erledigten Läufen
auf 0 von 8.
→ [AG03 — Prompt-Injection](/de/agent-injection)

## 2026-09-23 — R02: den Injection-Filter überlisten [#2026-09-23-r02-den-injection-filter-uberlisten]

Die eigenständige Übung verlangt kein siebtes Regex-Muster mehr. Du schreibst jetzt
zwei Formulierungen des 999-Euro-Angriffs, die an allen sechs Mustern vorbeikommen,
siehst zu, wie der Filter den Guardrails-Abschnitt von R02 selbst verwirft, und
erklärst, welcher Guardrail dich trotzdem erwischt. Die Passage-Tags werden jetzt
escaped, sodass ein Dokument sein eigenes Tag nicht mehr schließen und eine Passage
fälschen kann. Die Lektion ist weiterhin nur auf Englisch verfügbar.
→ [R02 — Damit es nicht lügt](/de/rag-quality)

## 2026-09-22 — AG02: derselbe Agent mit nativem Tool-Use [#2026-09-22-ag02-derselbe-agent-mit-nativem-tool-use]

Lass den Agenten aus AG01 auf drei Wegen an einer Frage laufen — Textprotokoll auf
Claude, Claudes natives Tool-Use und Qwen3s eigenes `<tool_call>`-Format — und lies,
was die API übernommen hat (Schema, Parsen, Aufruf-IDs) gegen das, was deine Sache
bleibt. Mit der ersten Brechen-Übung, in der die API eine kaputte Konversation ablehnt.
→ [AG02 — Native Tool-Use](/de/agent-native)

## 2026-09-21 — F01 nimmt die Python-Grundlagen eine Idee nach der anderen [#2026-09-21-f01-nimmt-die-python-grundlagen-eine-idee-nach-der-anderen]

Der Abschnitt, der Ausschnitte, Schleifen, `zip`, Funktionen, `assert` und
pytest auf einmal einführte, besteht jetzt aus fünf kurzen Schritten mit je
einer Sache zum Vorhersagen. Ein Ausschnitt-Visualizer auf der Seite lässt dich
einen Ausschnitt tippen und zeigt, welche Positionen von `["a", "b", "c", "d"]`
aufleuchten, und legt zwei Ausschnitte nebeneinander, damit du die Paare siehst,
die `zip` erzeugt — die Bigramm-Idee im Kleinen.
→ [F01 — Python & NumPy](/de/foundations-01-python-numpy)

## 2026-09-21 — AG01: eine Agentenschleife von Hand bauen [#2026-09-21-ag01-eine-agentenschleife-von-hand-bauen]

Die erste Einheit der neuen Gruppe **Angewandt: Agenten** ist da. Du baust einen
Agenten, in dem nichts versteckt ist — ein Klartext-Tool-Protokoll, das du selbst
parst, ein Schrittbudget und ein Trace, der zeigt, was das Modell tatsächlich
gesehen hat —, vergleichst zwei lokale Modelle an derselben Frage und siehst
einem kleinen Modell dabei zu, wie es eine Quelle erfindet. Zwei weitere
Agenten-Einheiten sind geplant und auf der Startseite mit *(in Kürze)* markiert.
→ [AG01 — Die Agentenschleife, von Hand](/de/agent-loop)

## 2026-09-20 — Einheit 0 setzt nichts mehr voraus, was der Kurs erst später lehrt [#2026-09-20-einheit-0-setzt-nichts-mehr-voraus-was-der-kurs-erst-spater-lehrt]

Das ganze Modell steht in drei Sätzen vor der ersten Vorhersage, jeder Fachbegriff
wird dort erklärt, wo er zuerst vorkommt, und die eintönige Ausgabe wird als
erwartetes Ergebnis erklärt, nicht als kaputter Lauf. Die Python-Prüfung bekommt
einen Weg von Hand für alle, die noch nicht programmieren.
→ [Einheit 0 — Erster Tiny-LM-Lauf](/de/unit-00)

## 2026-09-19 — Die Startseite sagt, was du danach kannst [#2026-09-19-die-startseite-sagt-was-du-danach-kannst]

Neun Fähigkeiten in klarer Sprache, jede mit der Einheit verlinkt, die sie
belegt, und eine Liste geplanter Einheiten mit *(in Kürze)*, damit du siehst,
was noch nicht geschrieben ist.
→ [Kursstart](/de/)

## 2026-09-19 — „Angewandt: RAG“ unter die Kursgruppe verschoben [#2026-09-19-angewandt-rag-unter-die-kursgruppe-verschoben]

Der Retrieval-Track steht in der Seitenleiste jetzt nach dem Kernkurs. Inhalt und
Lernweg im Lernleitfaden sind unverändert.
→ [R01 — Vom Dokument zur Antwort](/de/rag-build)

## 2026-09-17 — Der Kurs läuft unter llm.onlinekurs.training [#2026-09-17-der-kurs-lauft-unter-llmonlinekurstraining]

Die veröffentlichte Seite ist auf eine eigene Domain umgezogen. Alte Links
leiten weiter.

[Kursstart](/de/)
