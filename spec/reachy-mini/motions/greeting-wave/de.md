# Bewegungsablauf: Begrüßungs-Welle (`greeting-wave`)

Status: draft

## Kontext
Eine offene, freundliche Begrüßung als Antennen-Welle: Reachy lehnt sich leicht zur Person, lässt eine sichtbare Welle durch beide Antennen laufen und nickt zum Abschluss kurz. Anwendungsfälle: Person-Detection startet Behavior, Voice-„Hallo Reachy", Demo-Eröffnung, Tür-Öffnungs-Trigger über HA.

## Charakteristik
- Antennen-Welle als Hauptaussage: links und rechts versetzt animiert (nicht synchron) — wirkt wie eine winkende Hand
- Body-Yaw und Head-Yaw leicht in Richtung der Person — „ich sehe dich"
- Abschließender flacher Up-Nick — höfliche Mini-Verbeugung
- Mittleres Tempo (~2,0 s); `EASE_IN_OUT` für die Welle, `MIN_JERK` für den Lean

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Aufmerken) | 0,15 | (0, 0, +3, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | leichter Lift |
| 2 | Lean zur Person | 0,25 | (0, 0, +3, 0, +5, +10) | (+10, +10) | +8 | `MIN_JERK` | leichte Hinwendung |
| 3 | Welle 1 (links voraus) | 0,20 | (0, 0, +3, 0, +5, +10) | (+40, +5) | +8 | `EASE_IN_OUT` | linke Antenne hoch, rechte unten |
| 4 | Welle 2 (rechts voraus) | 0,20 | (0, 0, +3, 0, +5, +10) | (+5, +40) | +8 | `EASE_IN_OUT` | rechte Antenne hoch, linke unten |
| 5 | Welle 3 (beide hoch) | 0,20 | (0, 0, +3, 0, +5, +10) | (+30, +30) | +8 | `EASE_IN_OUT` | synchroner Höhepunkt |
| 6 | Höflicher Up-Nick | 0,25 | (0, 0, +1, 0, -8, +8) | (+15, +15) | +5 | `MIN_JERK` | kleines Verbeugen Richtung Person |
| 7 | Pitch zurück | 0,20 | (0, 0, +2, 0, +3, +5) | (+12, +12) | +3 | `MIN_JERK` | wieder neutral aufgerichtet |
| 8 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,95 s.

### Audio (optional)
Ein freundliches kurzes „Hallo!"-Sample (≤ 600 ms), gestartet mit Phase 3 (erste Welle). Lautstärke moderat (Volume 50). Aufsteigender Tonfall.

### Idle-Modulation
Keine — die Welle ist die ganze Aussage, eine Modulation würde sie verwischen.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen; der gemeinsame Lean von Body und Head wirkt durch IK organisch. Wenn die Position der Person bekannt ist (z. B. via `look_at_world`), kann der Yaw-Wert dynamisch gesetzt werden statt statisch +10°.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.95`. Antennen-Welle ist das diagnostische Detail; die Versatz-Werte (40°/5°) müssen sichtbar groß sein.
- Wenn `look_at_world(x, y, z)` für die Hinwendung verfügbar ist (Position der Person aus Kamera): kombinieren statt fester Yaw-Werte.
- Yaw +10° und Body-Yaw +8° liegen weit innerhalb der ±160°/±65°-Limits.
- Antennen-Sprung von 5° auf 40° in 0,2 s ≈ 175°/s ≈ 3 rad/s, innerhalb der 8 rad/s-Grenze.
- Wenn die Begrüßung zu einer bekannten Person läuft und ein Folge-Behavior wartet (z. B. `alert-listening`), Phase 8 (Release) durch direkten Übergang ersetzen.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Hallo" / „begrüßend" / „freundlich" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,0 ± 0,2 s
- [ ] Drei sichtbare Antennen-Wellen-Phasen (Phasen 3–5)
- [ ] Phase 3 und 4 zeigen klare Antennen-Asymmetrie (mind. 30° Differenz zwischen links und rechts)
- [ ] Lean Richtung Person ist sichtbar (Yaw und Body-Yaw zur selben Seite)
- [ ] Phase 6 (Up-Nick) ist sichtbar als „kleine Verbeugung", nicht als fragender Tilt
- [ ] Audio (falls aktiviert) klingt freundlich, nicht überschwänglich

## Anti-Patterns
- Antennen synchron statt versetzt — verliert den „Wave"-Charakter
- Body-Yaw und Head-Yaw entgegengesetzt — verwirrte Pose
- Zu großer Lean (> ±20°) — wirkt aufdringlich
- `LINEAR`-Easing in der Welle — wirkt mechanisch wie ein Scheibenwischer
- Audio mit Glocken-Sound — passt nicht zur sozialen Geste

## Offene Fragen
- Soll die Wave 3-phasig oder 4-phasig sein (eine zusätzliche Welle vor Phase 5)?
- Welche Audio-Datei eignet sich als Standard? Vorschlag: `wake_up`-Sound aus dem SDK als Referenz, eigene Datei für maßgeschneidertes „Hallo".
- Soll die Hinwendung dynamisch via `look_at_world` an die tatsächliche Personen-Position gebunden sein? Pro: realistischer; Contra: fragiler bei Nicht-Detection.
- Wie verhält sich `greeting-wave` bei wiederholtem Trigger innerhalb < 5 s? Vorschlag: nicht erneut triggern, eventuell `alert-listening` direkt anschließen.
