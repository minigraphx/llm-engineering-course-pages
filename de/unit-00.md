---
title: "Einheit 0 — Erster Tiny-LM-Lauf"
sidebar:
  label: "Einheit 0 — Erster Tiny-LM-Lauf"
---

<span id="einheit-0-erster-tiny-lm-lauf" />


[← Lernleitfaden](/de/learning-guide) · [Fehlerbehebung für Einheit 0 →](/de/unit-00-troubleshooting) · [Glossar](/de/glossary)

## Lernziel [#lernziel]

In 45–60 Minuten startest du ein vorbereitetes Tiny-LM, generierst mit einer
eigenen Eingabe, untersuchst Token-IDs, Wahrscheinlichkeiten, Shapes und Loss,
änderst einen Logit und erklärst die gemessene Wirkung. Du benötigst keine
ML-Vorkenntnisse, eigenen Daten, Cloud oder Beschleuniger.

Du musst Befehle in einem Terminal eintippen können. Alles andere — auch jedes
Fachwort auf dieser Seite — wird dort erklärt, wo es zuerst vorkommt. Jeder
Begriff ist zugleich ein Vorgriff: [F03](/de/foundations-03-probability) und
[T01](/de/bigram-baseline) bauen dieselben Ideen später sauber auf. Hier brauchst
du sie nur so weit, dass du eine Änderung vorhersagen, messen und erklären
kannst.

## 0–10 Min. · Vorhersagen und Einrichtung prüfen [#010-min-vorhersagen-und-einrichtung-prufen]

Bevor du etwas vorhersagen kannst, brauchst du eine Grundlage dafür. Das ist das
ganze Modell in drei Sätzen:

1. Es hat wenig Text gelesen und merkt sich für jedes Zeichen, welche Zeichen
   darauf gefolgt sind.
2. Zum Schreiben schaut es auf das Zeichen, das es gerade erzeugt hat, und wählt
   das Zeichen, das am häufigsten darauf folgte.
3. Dieses Zeichen wird zur neuen Eingabe, und das Ganze wiederholt sich.

Mehr ist nicht darin.

Notiere jetzt zwei Vorhersagen. Beide prüfst du noch in dieser Stunde gegen
Messwerte.

1. Was schreibt das Modell, ausgehend von der Eingabe `the `?
2. Wenn du `m` nach einem Leerzeichen zum wahrscheinlichsten Zeichen machst:
   ändert sich der geschriebene Text?

Prüfe danach deine Einrichtung:

```
python examples/diagnose_unit0.py
```

Jede Prüfung soll `pass` melden. Der Befehl prüft Python, CPU/MPS/CUDA,
beschreibbaren Artefaktspeicher, das Laden des Checkpoints und eine
Testgenerierung. Bei einem Fehler hilft die
[Fehlerbehebung](/de/unit-00-troubleshooting).

## 10–20 Min. · Erste Generierung [#1020-min-erste-generierung]

```
python examples/run_unit0.py
```

<aside className="course-note" aria-label="Was du sehen solltest">
<strong>Was du sehen solltest</strong>

Der erzeugte Text wirkt eintönig — etwa `the te te te te`. Das ist das
richtige Ergebnis und kein kaputter Lauf. Ein Modell, das sich nur *ein*
Zeichen Kontext merkt, kann es nicht besser; genau dieses Scheitern
anzusehen ist der Zweck dieser Einheit. Der ganze restliche Kurs dreht sich
darum, einem Modell mehr Kontext zu geben.


</aside>

Der erste Lauf erzeugt zwei von Git ignorierte Dateien unter `artifacts/unit0/`:

- einen deterministischen synthetischen 596-Byte-Korpus unter CC0-1.0;
- einen vorbereiteten Zeichen-Bigramm-Checkpoint aus benachbarten Zeichen.

Hashes und Herkunft stehen in `manifest.json`. Die Dateien entstehen lokal,
weil Datensätze und Checkpoints nicht in Git gehören.

Der Befehl gibt einen JSON-Bericht aus. Lies einen Namen mit Punkt als Pfad
hinein: `baseline.corpus_loss` meint das Feld `corpus_loss` innerhalb von
`baseline`. Suche diese Felder:

- `backend` — welcher Prozessor gerechnet hat: cpu, mps oder cuda;
- `prompt_token_ids` — deine Eingabe, jedes Zeichen durch seine Nummer ersetzt;
- `baseline.top_next_tokens` — die fünf Zeichen, die das Modell als Nächstes am
  wahrscheinlichsten hält, mit je einer Wahrscheinlichkeit;
- `baseline.corpus_loss` — eine Zahl dafür, wie überrascht das Modell vom
  gelernten Text war; kleiner ist besser;
- `baseline.generation` — was das Modell tatsächlich geschrieben hat.

Vergleiche `baseline.generation` jetzt mit deiner ersten Vorhersage.

## 20–30 Min. · Modell verfolgen [#2030-min-modell-verfolgen]

Das Modell kennt 33 verschiedene Zeichen. Diese Menge heißt **Vokabular**, ihre
Größe schreibt man `V`, also `V=33`.

Für das zuletzt gesehene Zeichen hält das Modell einen Score für jedes Zeichen
bereit, das folgen könnte — 33 Scores. Ein solcher Score heißt **Logit**: ein
unnormierter Wert, bei dem größer „wahrscheinlicher“ bedeutet, der aber noch
kein Prozentwert ist. Alle zusammen bilden eine **Übergangsmatrix** mit Shape
`[V,V]`: eine Zeile je aktuellem Zeichen, eine Spalte je möglichem Folgezeichen,
also 33 × 33 = 1089 Zahlen, und das aktuelle Zeichen wählt genau eine Zeile aus.

Eine Zeile mit 33 Logits in 33 Wahrscheinlichkeiten zu verwandeln, die sich zu 1
addieren, heißt **Softmax**. Den höchsten Wert zu nehmen und dieses Zeichen
wieder einzuspeisen heißt **Greedy Decoding** — genau die Schleife, aus der du
in den ersten zehn Minuten vorhergesagt hast.

`corpus_loss` ist die **Cross-Entropy**: Für jedes Zeichen im Korpus misst sie,
wie viel Wahrscheinlichkeit das Modell dem tatsächlich folgenden Zeichen gegeben
hat, und mittelt das. Ein kleinerer Wert heißt: weniger überrascht.

Wenn du bereits programmierst, prüfe die Shapes für eine vierstellige Eingabe
selbst:

```
from llm_course import run_vertical_slice

report = run_vertical_slice("the ", preferred_device="cpu")
assert len(report["prompt_token_ids"]) == 4
assert len(report["baseline"]["top_next_tokens"]) == 5
assert 0.0 <= report["baseline"]["target_probability"] <= 1.0
```

Wenn du noch nicht programmierst, überspringe diesen Schritt nicht: Lies die
drei `assert`-Zeilen als drei Behauptungen und prüfe sie von Hand im
JSON-Bericht, den du schon hast — die vierstellige Eingabe ergab vier Token-IDs,
das Modell nannte fünf Kandidaten, und die Zielwahrscheinlichkeit liegt zwischen
0 und 1. Diesen Code schreibst du in [F01](/de/foundations-01-python-numpy)
selbst.

## 30–42 Min. · Kontrollierte Änderung bauen [#3042-min-kontrollierte-anderung-bauen]

Der Referenzlauf addiert `4.0` auf genau einen Übergangslogit: Nach dem letzten
Leerzeichen des Prompts wird `m` wahrscheinlicher. Weil die Änderung an einem
Logit und nicht an einer Wahrscheinlichkeit passiert, bedeutet `+4.0` nicht
„vier Prozent mehr“ — Softmax entscheidet, wie viel Wahrscheinlichkeit
tatsächlich wandert, und nimmt sie den anderen Kandidaten weg.

Vergleiche `baseline` und `controlled_change`:

- die Zielwahrscheinlichkeit vorher und nachher;
- ob sich die erzeugte Fortsetzung geändert hat;
- ob der Korpus-Loss besser oder schlechter wurde.

Hier prüfst du deine zweite Vorhersage.

Nutze dann eine eigene Eingabe:

```
python examples/run_unit0.py --prompt "a " --target-character t --boost 3
```

Sage die Richtung vorher. Nur eine Zelle in `[V,V]` ändert sich; das kann einen
Übergang verbessern und gleichzeitig den gesamten Korpus-Loss verschlechtern.

## 42–50 Min. · Annahme brechen [#4250-min-annahme-brechen]

Kehre den Eingriff um:

```
python examples/run_unit0.py --prompt "a " --target-character t --boost -4
```

Die Zielwahrscheinlichkeit soll fallen. Behauptet deine Erklärung weiter das
Gegenteil, ist deine Interpretation defekt. Probiere danach einen leeren Prompt
und lies die Grenzfehlermeldung, statt die Prüfung zu entfernen.

## 50–55 Min. · Reproduzierbarkeit messen [#5055-min-reproduzierbarkeit-messen]

Greedy Decoding nimmt immer den höchsten Score und liefert deshalb auf jedem
Backend dasselbe. Sampling wählt stattdessen zufällig unter den Kandidaten und
braucht darum einen Seed — einen festen Startpunkt für den Zufall —, um sich zu
wiederholen. Sampling läuft hier auf CPU, damit derselbe Seed denselben Pfad
nimmt:

```
python examples/run_unit0.py --prompt "the " --sample --seed 11
```

Starte zweimal und vergleiche. Auch mit Beschleuniger bleibt `--device cpu` der
verbindliche Pflichtnachweis.

## 55–60 Min. · Erklären [#5560-min-erklaren]

Vervollständige in deinen Notizen:

1. Der Prompt wird zu Token-IDs durch ...
2. Das aktuelle Zeichen wählt eine Zeile mit Scores, und diese Zeile enthält ...
   Zahlen, weil ...
3. Softmax wird benötigt, weil ...
4. Die Änderung verschob Wahrscheinlichkeit von ... auf ...
5. Die Generierung änderte sich/nicht, weil ...
6. Der Korpus-Loss änderte sich, weil der Übergang ...

## Abschluss-Gate [#abschluss-gate]

Du bestehst, wenn die Diagnose bereit ist, dein eigener Prompt Text generiert,
du Token-IDs, Wahrscheinlichkeiten und Loss in deinem eigenen Bericht zeigen
kannst und Erklärung sowie Vorhersage zur Messung passen. Eine Erklärung in
eigenen Worten zählt; die Worte dieser Seite zu wiederholen zählt nicht. Bewahre
den JSON-Bericht als Nachweis auf.

Die Kursfreigabe verlangt zusätzlich den dokumentierten Pilot mit fünf
Lernenden.

## Wie geht es weiter? [#wie-geht-es-weiter]

Weiter mit [Diagnose und Lernpfad](/de/diagnostic). Dort bestimmst du sicher
überspringbare Grundlagen und behältst für jede Abkürzung einen Re-Entry-Link.

[← Lernleitfaden](/de/learning-guide) · [Fehlerbehebung für Einheit 0 →](/de/unit-00-troubleshooting) · [Glossar](/de/glossary)
