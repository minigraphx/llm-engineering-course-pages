---
title: "M5 — RLHF-Brücke: Fehler sichtbar machen"
sidebar:
  label: "M5 — RLHF-Brücke"
---

<span id="m5-rlhf-brucke-fehler-sichtbar-machen" />


Die optionale Brücke friert eine Referenzpolicy ein, lernt ein Reward-Modell aus
Präferenzpaaren, bestraft Abweichung per KL und aktualisiert eine endliche Policy
mit PPO. Eine eingebaute Konfidenz-Abkürzung macht Reward-Hacking messbar.

Prüfe nach dem Lauf Proxy-Reward, echte Qualität und Hackrate. Höherer Reward
bedeutet nicht automatisch mehr Sicherheit. Kritisches Routing bleibt eine
Anwendungsprüfung.

Nutze das CPU-Rezept aus [DPO](/llm-engineering-course-pages/de/dpo) und behalte bei fehlendem Accelerator
den CPU-Fallback.

Checkpoint: Nenne ein Rollback-Signal, etwa einen kritischen Sicherheitsfehler
oder steigende Hackrate, und die unveränderliche Referenz für Reproduktion.
