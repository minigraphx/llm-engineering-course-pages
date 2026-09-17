---
title: "F01 — Python und NumPy"
sidebar:
  label: "F01 — Python & NumPy"
---

<span id="f01-python-und-numpy" />


[← B01 — Godot-RL-Brücke](/de/foundations-godot-rl-bridge) · [F02 — Tensoren & Shapes →](/de/foundations-02-shapes) · [Glossar](/de/glossary)

## Lernziel [#lernziel]

In 45–60 Minuten verwandelst du eine kleine Datentransformation in ein
getestetes NumPy-Programm und erklärst jeden erzeugten Wert. Dieser Block ist
Pflicht, wenn die Diagnose Python und Entwicklung als Pflicht markiert hat.

Ist Programmieren ganz neu, teile diese Einheit in mehrere Sitzungen auf; die
45–60 Minuten sind eine Orientierung für den Kernteil, keine Frist.

## Wenn dies dein erstes Python-Programm ist [#wenn-dies-dein-erstes-python-programm-ist]

Schließe zuerst die [Einrichtung](/de/setup) ab. Erstelle mit dem Dateimanager
`artifacts/my-work` und darin eine reine Textdatei `hello.py` (nicht `hello.py.txt`).
Schreibe diesen Python-Code hinein:

```python
name = "tiny model"
print("Hello", name)
```

Starte im Terminal aus dem Kursordner `python artifacts/my-work/hello.py`.
Erwartet: `Hello tiny model`. `=` weist einem Namen einen Wert zu; Text in
Anführungszeichen ist ein String; `print(...)` gibt seine Argumente aus.
`#` beginnt einen Kommentar. Fehler nennen Datei und Zeile: Behebe den ersten
relevanten Fehler, bevor du weitergehst.

### Listen, Positionen, Ausschnitte und Schleifen [#listen-positionen-ausschnitte-und-schleifen]

```python
tokens = ["a", "b", "c", "d"]
print(tokens[0])       # a: Python zählt Positionen ab null
print(tokens[1:3])     # ['b', 'c']: Start inklusive, Ende exklusiv
print(tokens[:-1])     # ['a', 'b', 'c']: letztes Element weglassen
print(tokens[1:])      # ['b', 'c', 'd']: erstes Element weglassen
for token in tokens:
    print(token)
```

Eckige Klammern erzeugen Listen oder wählen Elemente aus. Ein Doppelpunkt wählt
einen Ausschnitt (**Slice**); `-1` zählt vom Ende. Eingerückte Zeilen gehören zur
Schleife. `for` wiederholt einmal pro Element. `zip(left, right)` besucht
zusammengehörige Elemente; `list(...)` sammelt sie. Eine **Funktion** benennt eine
wiederverwendbare Berechnung:

```python
def next_item_pairs(sequence):
    if len(sequence) < 2:
        return []
    return list(zip(sequence[:-1], sequence[1:]))

assert next_item_pairs(["a", "b", "c"]) == [("a", "b"), ("b", "c")]
assert next_item_pairs([]) == []
assert next_item_pairs(["a"]) == []
```

`def` definiert die Funktion, `sequence` ist ihre Eingabe, `return` gibt das
Ergebnis zurück. `if` führt einen Zweig bei wahrer Bedingung aus; `<` vergleicht
Zahlen. `==` prüft Gleichheit und ist keine Zuweisung. `assert` stoppt mit einem
`AssertionError`, wenn die Bedingung falsch ist. Paare in runden Klammern sind
**Tupel**, also feste Folgen. Ändere ein erwartetes Paar und prüfe, dass die
Assertion den Fehler findet.

Speichere deine Variante in `artifacts/my-work/test_pairs.py`. Setze Assertions
in eine Funktion namens `test_pairs`. Starte
`python -m pytest artifacts/my-work/test_pairs.py`. Grün heißt, dass diese
Assertions bestehen, und beweist nichts über ungeprüfte Eingaben. Typannotationen
wie `sequence: list[str]` erklären erwartete Typen für Menschen und Werkzeuge;
Python erzwingt sie nicht automatisch. Ein Docstring ist eine Erklärung in
dreifachen Anführungszeichen direkt innerhalb einer Funktion.

### NumPy-Begriffe vor der nächsten Aufgabe [#numpy-begriffe-vor-der-nachsten-aufgabe]

`import numpy as np` lädt die Zahlenbibliothek unter dem Kurznamen `np`.
Ein **Array** enthält Werte mit gemeinsamem Zahlentyp (**dtype**): Ganzzahlen für
IDs, Gleitkommazahlen für Wahrscheinlichkeiten. `shape` zählt Elemente entlang
jeder Achse. Bei `[[1,4,2],[3,0,2]]` bedeutet `(2,3)` zwei Zeilen mit drei Zahlen.
`np.all(condition)` prüft, ob alle Elemente die Bedingung erfüllen. Eine Funktion
kann mit `raise ValueError("reason")` ungültige Eingaben erklären, statt mit ihnen
weiterzurechnen. Beginne mit dem Beispiel unten und baue dann den Validator.

## Vorhersagen → Verfolgen → Bauen → Brechen → Messen → Erklären [#vorhersagen-verfolgen-bauen-brechen-messen-erklaren]

Beginne mit benachbarten Paaren:

```
tokens = ["a", "b", "c", "d"]
pairs = list(zip(tokens[:-1], tokens[1:]))
assert pairs == [("a", "b"), ("b", "c"), ("c", "d")]
```

Sage die drei Paare vor dem Lauf voraus und verfolge beide Slices. Kapsle die
Transformation anschließend in einer Funktion und teste leere, einteilige und
normale Eingaben. Die Schleife besucht jedes Element höchstens einmal; Laufzeit
und Ausgabespeicher wachsen linear mit der Eingabelänge.

## NumPy macht numerische Verträge sichtbar [#numpy-macht-numerische-vertrage-sichtbar]

```
import numpy as np

token_ids = np.array([[1, 4, 2], [3, 0, 2]], dtype=np.int64)
assert token_ids.shape == (2, 3)
assert token_ids.dtype == np.int64
assert np.all(token_ids >= 0)
```

Der Shape beschreibt zwei Beispiele mit je drei Positionen. Brich den Vertrag
mit -1, beobachte die fehlschlagende Assertion und repariere die Daten statt
die Assertion zu entfernen.

## Bauen [#bauen]

Implementiere `next_item_pairs(sequence)` ohne Kurscode zu importieren. Ergänze
Typangaben, Docstring und drei Tests. Implementiere danach
`validate_token_batch(array)`, das nicht-ganzzahlige, negative oder nicht-2D
Eingaben ablehnt. Jede Fehlermeldung nennt den fehlerhaften Wert.

## Abschlussnachweis [#abschlussnachweis]

- Tests decken leere, einteilige, normale und ungültige Eingaben ab;
- die Ausgabe ist deterministisch;
- du erklärst Slicing, Iteration, dtype und jede Assertion;
- `ruff check .` und `pytest` bestehen.

## Wie geht es weiter? [#wie-geht-es-weiter]

Weiter mit [F02 — Tensoren und Shapes](/de/foundations-02-shapes). Kehre zu F01
zurück, wenn Python-Ablauf, Tests, Exceptions oder NumPy-dtypes Verhalten
verdecken.

[← Diagnose](/de/diagnostic) · [→ F02](/de/foundations-02-shapes)

<details>
<summary>Hinweis zum Validator</summary>

Prüfe `array.ndim != 2`, `np.issubdtype(array.dtype, np.integer)` und
`np.any(array < 0)`. Das prüft getrennt Rang, Typ und Werte.


</details>

<details>
<summary>Referenzvalidator</summary>

```python
import numpy as np

def validate_token_batch(array):
    if array.ndim != 2:
        raise ValueError(f"expected 2D, got shape {array.shape}")
    if not np.issubdtype(array.dtype, np.integer):
        raise ValueError(f"expected integer IDs, got {array.dtype}")
    if np.any(array < 0):
        raise ValueError(f"negative IDs: {array[array < 0].tolist()}")
```


</details>

[← B01 — Godot-RL-Brücke](/de/foundations-godot-rl-bridge) · [F02 — Tensoren & Shapes →](/de/foundations-02-shapes) · [Glossar](/de/glossary)
