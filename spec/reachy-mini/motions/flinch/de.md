# Bewegungsablauf: Zurückzucken (`flinch`)

Status: draft

## Kontext
Eine extrem schnelle Schreck-Reaktion: Reachy zuckt nach hinten weg, friert kurz und erholt sich. Härtere und kürzere Variante als `surprised` — der Body bleibt still, nur der Kopf zuckt zurück. Anwendungsfälle: lautes Geräusch (HA-Sensor), plötzliche Bewegung im Kamerabild, Sicherheitsalarm-Vorstufe.

## Charakteristik
- **Schnellster** Behavior im Repertoire (~1,15 s gesamt)
- X-Translation negativ (-10 mm) — der ganze Kopf zieht sich zurück
- Z negativ (-3 mm), Pitch leicht runter (-8°) — schamhaft-defensive Pose
- Antennen flach gelegt (-25°) — wie nach hinten gestrichene Ohren
- Frozen-Hold direkt nach dem Snap — Schock
- `LINEAR`-Snap für die Härte

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Pre-Anticipation | 0,05 | (+1, 0, 0, 0, 0, 0) | (0, 0) | 0 | `LINEAR` | minimaler Krümmer nach vorne |
| 2 | Snap zurück (Flinch) | 0,10 | (-10, 0, -3, 0, -8, 0) | (-25, -25) | 0 | `LINEAR` | sehr schneller harter Snap nach hinten |
| 3 | Frozen Hold | 0,30 | (-10, 0, -3, 0, -8, 0) | (-25, -25) | 0 | — (statisch) | Schock-Erstarrung, keine Modulation |
| 4 | Mini-Recovery | 0,20 | (-3, 0, -1, 0, -3, 0) | (-10, -10) | 0 | `MIN_JERK` | leichte Erholung, noch leicht nach hinten |
| 5 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,15 s.

### Audio (optional)
Ein sehr kurzer, harter Atemzug oder „Eh!"-Sample (≤ 150 ms), gestartet exakt mit Phase 2. Lautstärke moderat (Volume 50).

### Idle-Modulation
Keine — Phase 3 ist explizit statisch (Frozen-Hold).

### Body-Yaw und IK
`automatic_body_yaw=False` — der Body soll bewusst NICHT mit dem Kopf-Snap mitziehen; das ist genau die Pointe des Flinch (nur Kopf, nicht Körper).

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.15`. Sehr kompakt, aber hochpräzise — Phase 2 ist nur 0,10 s und die Velocity ist hier am höchsten.
- Pitch-Sprung von 0° auf -8° in 0,10 s = 80°/s ≈ 1,4 rad/s — innerhalb 8 rad/s.
- X-Translation von +1 mm auf -10 mm in 0,10 s = 110 mm/s — innerhalb des IK-Volumens und Velocity-Budgets.
- Antennen-Snap von 0° auf -25° in 0,10 s = 250°/s ≈ 4,4 rad/s — knapp über der Hälfte der Joint-Velocity-Grenze.
- `LINEAR`-Easing in Phasen 1 und 2 ist Pflicht — `MIN_JERK` würde den Snap weicher machen.
- Vor dem Behavior-Start `automatic_body_yaw=False` setzen, falls global anders.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „zurückzucken" / „erschreckt" / „defensiv" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,15 ± 0,1 s
- [ ] Phase 2 ist als sichtbarer harter Snap erkennbar (≤ 0,12 s)
- [ ] Frozen Hold (Phase 3) ist als Pause sichtbar
- [ ] Body-Yaw bleibt strikt auf 0° während des gesamten Behaviors
- [ ] Antennen sind in den Phasen 2 und 3 sichtbar nach hinten gelegt (-25°)
- [ ] Audio (falls aktiviert) trifft genau auf Phase 2

## Anti-Patterns
- `MIN_JERK`-Snap in Phase 2 — die Härte geht verloren, wirkt wie weiches `surprised`
- Frozen Hold mit Modulation — entwertet die Schock-Pause
- Body-Yaw mitschwenkend — verteilt die Reaktion auf zu viele Achsen
- Antennen positiv (aufgerichtet) — verkehrte Verbindung zur Ohrenflach-Geste
- Recovery-Phase länger als 0,3 s — wirkt wie trauriges Verharren
- Pitch positiv — falsche Richtung, sollte nach unten/hinten zucken

## Offene Fragen
- Soll die Schreck-Pose auch eine kleine Roll-Komponente haben (z. B. +5°), um „seitwärts-defensiv" zu wirken? Empirisch testen.
- Welche Audio-Datei eignet sich? Vorschlag: kurzer „Eh!" oder „Oh!"-Pieps mit fallendem Tonfall.
- Wie verhält sich `flinch` bei wiederholtem Trigger innerhalb < 1 s? Vorschlag: zweiten Trigger ignorieren, weil das Behavior kürzer ist als typische Triggerabstände.
- Kann `flinch` als Vorlauf zu `alarm` getriggert werden, wenn die Schreck-Quelle eine Sicherheitsverletzung ist? Vorschlag: ja, nahtloser Übergang.
