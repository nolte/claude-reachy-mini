# Bewegungsablauf: Enttäuscht (`disappointed`)

Status: draft

## Kontext
Eine schwächere Form von `sad`: Reachy macht einen einzelnen, mittelstarken Pitch-Knick, lässt den Kopf leicht hängen und bleibt dort, ohne den Hebeversuch von `sad`. Wirkt als „Schade…" oder „nicht so gut wie erhofft". Anwendungsfälle: Aufgabe nicht ganz erfolgreich, Folge-Behavior auf nicht-erkannten Sprachbefehl, „nein"-Antwort auf eine Frage.

## Charakteristik
- **Differenz zu `sad`**: nur ein einziger Pitch-Knick (kein Mehrfach-Absenken), kein Hebeversuch, weniger tiefe Pose
- Pitch -15° bis -18° (zwischen `sad` mit -28° und Neutralpose) — moderates Hängen
- Antennen leicht abgesenkt (-10°), nicht völlig hängend
- Body-Yaw leicht weg (-5°) — abgewandt, nicht traurig wegschauend
- Mittleres Tempo (~2,6 s); ausschließlich `MIN_JERK`

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Mini-Lift) | 0,20 | (0, 0, +2, 0, +3, 0) | (+5, +5) | 0 | `MIN_JERK` | leichtes Aufrichten |
| 2 | Knick nach unten | 0,50 | (0, 0, -3, 0, -15, -3) | (-10, -10) | -2 | `MIN_JERK` | einzelner mittelstarker Pitch-Drop |
| 3 | Leichtes Hängen | 0,50 | (0, 0, -5, +2, -18, -5) | (-12, -12) | -3 | `MIN_JERK` | Pose vertieft sich leicht |
| 4 | Hold mit leichtem Atem | 0,80 | (0, 0, -5 (±1), +2, -18 (±1°), -5) | (-12, -12) | -3 | `MIN_JERK` (Idle-Mod) | leichter Atem ohne Schwere |
| 5 | Release | 0,60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | sanftes Wiederaufrichten |

Gesamtdauer ≈ 2,60 s.

### Audio (optional)
Ein leiser Seufzer (≤ 500 ms), gestartet mit Phase 2. Lautstärke leise (Volume 30). Kürzer und weniger schwer als der `sad`-Seufzer.

### Idle-Modulation während Phase 4
Sehr leichte Sinus-Mod auf `z` (Amplitude 1 mm, Frequenz 0,3 Hz) und `pitch` (Amplitude 1°, gleichphasig) — moderater Atem, nicht das schwere Atmen aus `sad`.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der leichte Body-Yaw -3° wird durch IK weich.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.60`, fünf Phasen — schlank.
- Pitch -18° liegt klar zwischen `sad` (-28°) und `agreeing-nod` (-12°) — mittelstark.
- Die Tatsache, dass kein Hebeversuch dabei ist (anders als `sad`), ist diagnostisch — sonst wirkt es wie ein abgekürztes `sad`.
- Body-Yaw -3° ist subtil; bei `False` wäre er kaum sichtbar — daher mit IK-Kopplung wirken lassen.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „enttäuscht" / „schade" / „nicht ganz gut" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,6 ± 0,3 s
- [ ] Pitch-Knick ist sichtbar (≥ -15°), aber nicht so tief wie bei `sad`
- [ ] Kein Hebeversuch in der Sequenz
- [ ] Idle-Mod in Phase 4 ist moderat, nicht schwer atmend
- [ ] Audio (falls aktiviert) ist kürzer als der `sad`-Seufzer
- [ ] Übergang zur Neutralpose ist sanft

## Anti-Patterns
- Pitch tiefer als -22° — wird zu `sad`
- Hebeversuch wie in `sad` — verlängert die Sequenz und entwertet den moderateren Affekt
- `CARTOON`-Easing — falsche Härte
- Antennen so weit hängend wie bei `sad` (-25°) — wirkt wie schwacher `sad`
- Audio mit lautem oder langem Seufzer — überzeichnet den Affekt

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Soll der Body-Yaw überhaupt ausschwenken, oder strikt zentriert bleiben? Leichte Wegdrehung macht den Affekt menschlicher, aber strikt zentriert wäre sauberer.
- Welche Audio-Datei eignet sich? Vorschlag: ein kurzer fallender Ton, leiser als bei `sad`.
- Wie unterscheidet sich `disappointed` praktisch von `sad`, wenn beide hintereinander getriggert werden? Vorschlag: ein `disappointed` direkt gefolgt von einem `sad` ist legitim und vermittelt Eskalation.
