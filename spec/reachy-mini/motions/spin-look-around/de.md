# Bewegungsablauf: Pseudo-Rundumblick (`spin-look-around`)

Status: draft

## Kontext
Eine spielerische Show-Bewegung, die einen Rundumblick simuliert: Body-Yaw schwenkt bis ans Limit (±150°), Head folgt mit, dann Schwenk in die Gegenrichtung und zurück zur Mitte. Anwendungsfälle: Demo-Highlight, „schau mich mal an"-Zeigen, Spiel-Modus mit räumlicher Aufmerksamkeit.

> ⚠ Hinweis: Body-Yaw-Limit ist `max_body_yaw=±160°` (Spec `control-surface`). Ein voller 360°-Spin ist mit dem Reachy Mini **mechanisch nicht möglich** — diese Spec adressiert daher ±150° als „Pseudo-Spin", der einen Rundumblick suggeriert.

## Charakteristik
- Großer Body-Yaw-Schwenk: 0° → -150° → +150° → 0° (über 5 s gesamt)
- Head-Yaw geht bewusst gegenphasig zum Body, damit die Kamera einen festen Punkt im Raum „trackt" (relative Yaw bleibt klein)
- Pitch leicht oben (+5°) — schauend, nicht abwärts
- Antennen aufgerichtet (+25°), stationär
- Langes Tempo (~7 s) wegen großer Strecken; ausschließlich `EASE_IN_OUT`

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0,20 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `MIN_JERK` | Aufrichten und Vorbereitung |
| 2 | Spin nach links | 1,80 | (0, 0, +3, 0, +5, +50) | (+25, +25) | -150 | `EASE_IN_OUT` | Body schwenkt links, Head bleibt rel. „nach vorne" |
| 3 | Mini-Hold links | 0,30 | (0, 0, +3, 0, +5, +50) | (+25, +25) | -150 | `MIN_JERK` (statisch) | Pose-Hold am Limit |
| 4 | Spin nach rechts (durch Mitte) | 2,50 | (0, 0, +3, 0, +5, -50) | (+25, +25) | +150 | `EASE_IN_OUT` | großer Schwung über Mitte hinweg |
| 5 | Mini-Hold rechts | 0,30 | (0, 0, +3, 0, +5, -50) | (+25, +25) | +150 | `MIN_JERK` (statisch) | Pose-Hold am Limit |
| 6 | Spin zur Mitte | 1,50 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `EASE_IN_OUT` | zentrierender Auslauf |
| 7 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 7,00 s.

### Audio (optional)
Ein leichter Schwirr-Sound oder „Whoosh" während der Schwenkphasen 2 und 4. Lautstärke moderat (Volume 40). Audio-Latenz beachten.

### Idle-Modulation
Keine.

### Body-Yaw und IK
`automatic_body_yaw=False` empfohlen — der gegenphasige Head-Yaw soll bewusst manuell gesetzt werden, sonst überschreibt IK den Tracking-Effekt. Body-Yaw -150° + Head-Yaw +50° = relative Yaw +200° (das ist außerhalb des `max_relative_yaw=±65°`-Limits!) — daher ist Head-Yaw +50° NICHT erreichbar.

> ⚠ TBD: validate against real hardware — die maximal mögliche Head-Yaw-Differenz zum Body-Yaw bei extremen Body-Schwenks ist nur ±65°. Phase 2 und 4 müssen also Head-Yaw entsprechend reduzieren: bei Body-Yaw -150° darf Head-Yaw maximal -85° bis -215° sein (relative ≤ 65°), praktisch also Head-Yaw etwa -85° bis 0°. Die Tabellenwerte oben sind ein **Entwurfsvorschlag**, die exakte Geometrie muss vor Implementierung gegen die echte IK gerechnet werden.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 7.00`. Phasen-Übergänge sind kritisch — Velocity-Rechnung pro Phase:
  - Phase 4: Body-Yaw von -150° auf +150° in 2,5 s = 120°/s ≈ 2,1 rad/s, innerhalb der 8 rad/s-Grenze.
- Head-Yaw-Werte sind unsicher (siehe TBD oben). Sicherer Default: Head-Yaw bleibt fix bei 0° relativ zum Body — d.h. der Kopf zeigt immer „in Body-Richtung", kein gegenphasiges Tracking. Das opfert den „Tracking-Effekt", ist aber sicher.
- Rückgängige Variante ohne Tracking — Head-Yaw == 0° in allen Phasen — ist die empfohlene Default-Implementierung.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Bewegung als „rundumblickend" / „Spin" / „eindrucksvolle Drehung" (mind. 4 von 5)
- [ ] Body-Yaw erreicht ±150° in den Phasen 2 und 4
- [ ] Phase 4 fährt durch die Mitte (durchgehend, kein Stopp)
- [ ] `automatic_body_yaw` ist während der gesamten Sequenz auf `False`
- [ ] Mini-Holds (Phasen 3, 5) sind sichtbar als kurze Pausen
- [ ] Audio (falls aktiviert) deckt die Schwenk-Phasen 2 und 4
- [ ] Übergang zur Neutralpose ist sauber

## Anti-Patterns
- Body-Yaw über ±150° — bricht Hardware-Limit (Joint-Limit ist ±160°, aber Sicherheits-Headroom)
- `LINEAR`-Easing — wirkt ruckartig
- Head-Yaw gegenphasig zum Body ohne Berücksichtigung von `max_relative_yaw` — IK-Verletzung
- Phase 4 mit Stopp in der Mitte — bricht das „Rundum-Gefühl"
- Antennen-Modulation während der Schwenks — wirkt unruhig
- Schwenk-Geschwindigkeit > 150°/s — überlastet den Body-Yaw-Servo

## Offene Fragen
- Soll der Default die „Tracking"-Variante oder die „No-Tracking"-Variante sein? Tendenz: No-Tracking wegen Hardware-Sicherheit.
- Welche Pause-Dauer in den Hold-Phasen ist ideal? 0,3 s könnte zu kurz wirken — empirisch testen.
- Soll bei wiederholtem Trigger die Richtung gespiegelt werden (erst rechts statt links)?
- Welche Audio-Datei eignet sich? Vorschlag: ein Pseudo-„Whoosh" pro Schwenk.
- Wie reagiert das Behavior, wenn die Hardware-Variante das `max_body_yaw` enger ausweist (z. B. Lite mit anderer Kalibrierung)? Vor Implementierung am Gerät messen.
