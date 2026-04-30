# Bewegungsablauf: Abschieds-Welle (`farewell-wave`)

Status: draft

## Kontext
Eine offene, leicht melancholische Abschiedsgeste: wie `greeting-wave`, aber endet nicht aufgerichtet, sondern in einer abfallenden Pose. Anwendungsfälle: Person verlässt Raum (Kamera erkennt Tür), „Auf Wiedersehen" als Sprachbefehl, HA-Trigger „Person außer Reichweite", Übergang zum `goto_sleep()`.

## Charakteristik
- Antennen-Welle wie bei `greeting-wave` — gleiche linke-rechts-versetzte Animation
- Body-Yaw und Head-Yaw zur Person geneigt
- **Differenz zu `greeting-wave`**: nach der dritten Welle senkt sich der Pitch ab statt auf Up-Nick zu gehen — die Pose endet wegschauend statt anlachend
- Mittleres Tempo (~2,2 s), etwas länger als `greeting-wave` durch das Verharren

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0,15 | (0, 0, +3, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | leichter Lift |
| 2 | Lean zur Person | 0,25 | (0, 0, +3, 0, +5, +10) | (+10, +10) | +8 | `MIN_JERK` | Hinwendung |
| 3 | Welle 1 | 0,20 | (0, 0, +3, 0, +5, +10) | (+40, +5) | +8 | `EASE_IN_OUT` | linke Antenne hoch |
| 4 | Welle 2 | 0,20 | (0, 0, +3, 0, +5, +10) | (+5, +40) | +8 | `EASE_IN_OUT` | rechte Antenne hoch |
| 5 | Welle 3 | 0,25 | (0, 0, +3, 0, +5, +10) | (+30, +30) | +8 | `EASE_IN_OUT` | beide hoch — Höhepunkt der Welle |
| 6 | Pitch-Absenkung | 0,30 | (0, 0, -2, 0, -10, +8) | (+15, +15) | +5 | `MIN_JERK` | Kopf sinkt — Abschieds-Charakter |
| 7 | Hold (verabschiedend) | 0,40 | (0, 0, -3, 0, -12, +5) | (+10, +10) | +3 | `MIN_JERK` | leichtes Verharren in der gefallenen Pose |
| 8 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 2,25 s.

### Audio (optional)
Ein leiserer fallender Ton („Tschüüss" — ≤ 700 ms), gestartet mit Phase 3. Lautstärke leise bis moderat (Volume 40). Im Gegensatz zu `greeting-wave`: fallender, nicht aufsteigender Tonfall.

### Idle-Modulation
Keine.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen; identisch zu `greeting-wave`. Wenn Personen-Position bekannt: dynamisch via `look_at_world`.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.25`. Code-seitig kann `farewell-wave` als Variante von `greeting-wave` implementiert werden — gleiche Wellenphasen, andere End-Phasen.
- Pitch -12° in Phase 7 ist nicht so tief wie bei `sad`, gerade genug für die Abschiedsstimmung.
- Antennen-Werte folgen denselben Velocity-Limits wie bei `greeting-wave`.
- Phase 8 (Release) idealerweise in den `waiting-idle` oder `goto_sleep()` übergehen lassen.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Tschüss" / „Abschied" / „verabschiedend" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,2 ± 0,2 s
- [ ] Drei Antennen-Wellen-Phasen sind sichtbar
- [ ] Pitch fällt sichtbar in Phase 6 unter 0° — kein Up-Nick
- [ ] Hold-Phase 7 zeigt eine sichtbare gesunkene Pose ohne in `sad` zu kippen
- [ ] Audio (falls aktiviert) hat fallenden Tonfall

## Anti-Patterns
- Pitch in der Endpose ≥ 0° — wirkt wie greeting-wave, nicht wie farewell
- Pitch < -20° in Phase 7 — wirkt wie `sad`, nicht wie ruhiger Abschied
- Antennen synchron statt versetzt in Phasen 3/4
- Audio aufsteigend — falscher Affekt
- Hold-Phase 7 länger als 0,6 s — wirkt wehmütig statt verabschiedend

## Offene Fragen
- Soll der Pitch in Phase 7 noch tiefer (z. B. -18°) für stärkeren Abschieds-Charakter? Empirisch testen.
- Wie wird der Übergang zu `waiting-idle` oder `goto_sleep()` gekoppelt? Skill-Layer-Entscheidung.
- Gemeinsame Implementierung mit `greeting-wave` als parametrisierte `Move`-Subklasse `WaveMove(direction="hello"|"goodbye")`?
