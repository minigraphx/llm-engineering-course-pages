---
title: "F02 — Tensoren und Shapes"
sidebar:
  label: "F02 — Tensoren & Shapes"
---

<span id="f02-tensoren-und-shapes" />


[← F01 — Python & NumPy](/llm-engineering-course-pages/de/foundations-01-python-numpy) · [F03 — Wahrscheinlichkeit & Loss →](/llm-engineering-course-pages/de/foundations-03-probability) · [Glossar](/llm-engineering-course-pages/de/glossary)

## Lernziel [#lernziel]

Du sagst Matrix- und Batch-Shapes vor dem Lauf voraus, prüfst sie mit
Assertions und diagnostizierst einen absichtlichen Broadcasting-Fehler.

## Beginne mit Zeilen und Spalten [#beginne-mit-zeilen-und-spalten]

Ein Skalar ist eine Zahl, ein Vektor eine Zahlenreihe, eine Matrix eine Tabelle;
ein Tensor verallgemeinert Arrays auf mehr Achsen. `shape` nennt die Achsenlängen:
`[2,3]` bedeutet zwei Zeilen mit drei Einträgen. Bei der Matrixmultiplikation ist
ein Ausgabeeintrag ein **Skalarprodukt**: Passende Einträge multiplizieren und
summieren. Links oben ergibt sich unten `1×1 + 2×0 + 3×1 = 4`; rechts oben
`1×0 + 2×1 + 3×1 = 5`.

`reshape` packt dieselbe Wertefolge in neue Dimensionen; `transpose` vertauscht
Achsen. Aus 2×3 wird beim Transponieren 3×2. **Broadcasting** verwendet ein
kleineres Array über passende Achsen mehrfach: `[10,20,30]` wird zu jeder Zeile
einer 2×3-Tabelle addiert. Die Achsengrößen müssen übereinstimmen oder eine muss
1 sein. `*` multipliziert elementweise; `@` verrechnet Matrixachsen. Auch bei
gleicher Ausgabeform sind das unterschiedliche Operationen.

## Eine konkrete Matrixmultiplikation [#eine-konkrete-matrixmultiplikation]

Seien

```text
X = [[1, 2, 3],
     [0, 1, -1]]
W = [[1, 0],
     [0, 1],
     [1, 1]]
```

`X` hat Shape `[2,3]`, `W` hat `[3,2]`; die gemeinsame Dimension 3 wird
kontrahiert. `X @ W` hat deshalb `[2,2]` und ergibt konkret
`[[4,5],[-1,0]]`.

```
import numpy as np

x = np.array([[1, 2, 3], [0, 1, -1]])
w = np.array([[1, 0], [0, 1], [1, 1]])
result = x @ w
assert x.shape == (2, 3)
assert w.shape == (3, 2)
assert result.shape == (2, 2)
assert np.array_equal(result, [[4, 5], [-1, 0]])
```

## Batch, Zeit, Kanal [#batch-zeit-kanal]

LLM-Aktivierungen verwenden meist `[B,T,C]`: Beispiele, Tokenpositionen,
Kanäle. Aus `X:[4,8,32]` und `W:[32,64]` entsteht `[4,8,64]`; nur die letzte
Achse wird kontrahiert. Ein Bias `[64]` broadcastet über Batch und Zeit. Ein
Bias `[8]` beschreibt keine Kanäle:

```
activations = np.zeros((4, 8, 64))
bias = np.zeros(64)
assert bias.shape == (activations.shape[-1],)
assert (activations + bias).shape == (4, 8, 64)
```

## Bauen und brechen [#bauen-und-brechen]

Schreibe `linear(x, weight, bias)` mit expliziten Shape-Prüfungen. Teste 2D und
`[B,T,C]`. Transponiere danach versehentlich das Gewicht und erkläre, welche
Dimensionen nicht mehr übereinstimmen.

## Abschlussnachweis [#abschlussnachweis]

- jeder Zwischen-Shape steht vor der Ausführung fest;
- Assertions decken Batch-, Zeit-, Feature-, Hidden- und Klassenachsen ab;
- du unterscheidest Reshape, Transpose und Broadcasting;
- der Fehler stoppt an deiner Schnittstelle, nicht tief im Training.

## Wie geht es weiter? [#wie-geht-es-weiter]

Weiter mit [F03 — Wahrscheinlichkeit und Loss](/llm-engineering-course-pages/de/foundations-03-probability).

[← F01 — Python & NumPy](/llm-engineering-course-pages/de/foundations-01-python-numpy) · [F03 — Wahrscheinlichkeit & Loss →](/llm-engineering-course-pages/de/foundations-03-probability) · [Glossar](/llm-engineering-course-pages/de/glossary)
