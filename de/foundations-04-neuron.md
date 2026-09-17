---
title: "F04 — Neuron und Gradient Descent"
sidebar:
  label: "F04 — Neuron & Gradient Descent"
---

<span id="f04-neuron-und-gradient-descent" />


[← F03 — Wahrscheinlichkeit & Loss](/de/foundations-03-probability) · [F05 — MLP, Backprop & Autograd →](/de/foundations-05-mlp-autograd) · [Glossar](/de/glossary)

## Lernziel [#lernziel]

Du berechnest Forward Pass und Gradienten eines Neurons und beweist, dass das
gewählte Update seinen Loss senkt.

## Was Neuron und Ableitung hier bedeuten [#was-neuron-und-ableitung-hier-bedeuten]

Unser Neuron ist eine kleine Funktion mit veränderbaren Zahlen `w` (Gewicht) und
`b` (Bias). Der Vorwärtslauf berechnet aus `x` eine Vorhersage. Ziel `y` ist die
gewünschte Antwort. Das Quadrat der Differenz ergibt einen nichtnegativen Fehler.

Eine Ableitung misst lokale Empfindlichkeit: Steigt der Loss, wenn `w` minimal
steigt, ist `dL/dw` positiv. Zur Senkung gehen wir in die Gegenrichtung. Die
**Lernrate** steuert die Schrittgröße. Ein Gradient sammelt eine Ableitung je
veränderbarem Parameter. Für `u²` ist die lokale Ableitung `2u`; bei `wx+b`
ändert sich das Ergebnis beim Ändern von `w` mit Rate `x`. Multipliziere die
lokalen Raten: `dL/dw = 2(ŷ-y)x` (Kettenregel). Die exakte Null im Beispiel ist
eine günstige Zahlenwahl. Ein allgemeiner Datensatz braucht viele Updates und
muss keinen Loss von null erreichen.

## Ein vollständiges Update mit Zahlen [#ein-vollstandiges-update-mit-zahlen]

Ein skalares Neuron berechnet `ŷ=wx+b`. Nutze `x=2`, `w=0.5`, `b=0.1`, Ziel
`y=3` und quadratischen Loss `L=(ŷ-y)²`:

1. `ŷ = 0.5 × 2 + 0.1 = 1.1`.
2. Fehler `1.1 - 3 = -1.9`; Loss `(-1.9)² = 3.61`.
3. `dL/dw = 2(ŷ-y)x = 2 × -1.9 × 2 = -7.6`.
4. `dL/db = 2(ŷ-y) = -3.8`.
5. Bei Lernrate `0.1`: Gradienten subtrahieren, also `w=1.26`, `b=0.48`.
6. Neue Vorhersage `1.26 × 2 + 0.48 = 3.0`; Loss `0`.

Das Vorzeichen ist sichtbar: Das Subtrahieren eines negativen Gradienten erhöht
die Parameter. Addieren bewegt sie in die Gegenrichtung.

## Trainieren und untersuchen [#trainieren-und-untersuchen]

```
import numpy as np
from llm_course import train_scalar_neuron

x = np.array([-1.0, 0.0, 1.0, 2.0])
y = 2.0 * x + 1.0
trace = train_scalar_neuron(x, y)
assert trace.losses[-1] < trace.losses[0]
```

Schreibe die Gradienten zuerst selbst. Protokolliere Gewicht, Bias und Loss pro
Schritt; nur der Endwert erklärt keinen Fehler.

## Brechen und diagnostizieren [#brechen-und-diagnostizieren]

Nutze dieselben Daten mit `fault="wrong_gradient_sign"`. Schaue noch nicht in
die Referenz. Bilde aus Losskurve und Parameterbewegung eine Hypothese, nenne
die fehlerhafte Gleichung und sage die Reparatur voraus.

## Abschlussnachweis [#abschlussnachweis]

- Handrechnung und Code stimmen überein;
- Shapes von Input, Ziel und Vorhersage stimmen;
- gesunder Loss fällt, defekter Loss steigt;
- die Diagnose nennt Mechanismus, Nachweis und Reparatur.

## Wie geht es weiter? [#wie-geht-es-weiter]

Weiter mit [F05 — MLP, Backpropagation und Autograd](/de/foundations-05-mlp-autograd).

[← F03 — Wahrscheinlichkeit & Loss](/de/foundations-03-probability) · [F05 — MLP, Backprop & Autograd →](/de/foundations-05-mlp-autograd) · [Glossar](/de/glossary)
