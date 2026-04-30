# Bewegungsablauf: Traurig (`sad`)

Status: draft

## Kontext
Eine ausgedrückte Niedergeschlagenheit, die als „traurig" oder „enttäuscht" gelesen wird: Reachy senkt den Kopf, lässt die Antennen hängen, neigt den Kopf leicht zur Seite, hält die Pose mit schwerem Atem, versucht ein kurzes Aufrichten und sinkt wieder zurück. Anwendungsfälle: Fehlermeldung, „Aufgabe gescheitert", negativer HA-Trigger, Reaktion auf „Nein".

## Charakteristik
- Klar abwärts gerichteter Kopf (negativer Pitch) und leicht eingesunkene Z-Höhe — vermittelt „Energie weg"
- Antennen hängen lassen (negative Joint-Winkel) — wie geknickte Ohren
- Leichter Seiten-Roll plus Body-Yaw weg von der Mittellage — schiefes „Wegschauen"
- Sehr langsames Tempo (gesamt ~4–5 s); ausschließlich `MIN_JERK`-Easing, kein `CARTOON`
- Während Hold ein langsames, schweres Atmen (geringe Frequenz, größere Amplitude als bei `happy`)

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

Pose-Konvention wie in `control-surface` und `happy`. Alle Werte respektieren Pitch/Roll ≤ ±90°, Body-Yaw ≤ ±160°.

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Mini-Anticipation (Aufseufzen) | 0,20 | (0, 0, +2, 0, +3, 0) | (+5, +5) | 0 | `MIN_JERK` | sehr kleines Anheben — wie ein letztes Aufrichten |
| 2 | Hauptabsenkung Kopf | 0,80 | (0, 0, -8, 0, -25, 0) | (-25, -25) | 0 | `MIN_JERK` | langsam und gleichmäßig nach unten |
| 3 | Seitliches Hängen | 0,40 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | leichter Schräg-Hang, Body folgt |
| 4 | Hold mit schwerem Atmen | 1,50 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | tiefe, langsame Idle-Modulation |
| 5 | Vergeblicher Hebeversuch | 0,30 | (0, 0, -7, +4, -20, -8) | (-22, -22) | -6 | `MIN_JERK` | schwacher Versuch, sich aufzurichten |
| 6 | Zurücksinken | 0,50 | (0, 0, -10, +6, -28, -10) | (-30, -28) | -8 | `MIN_JERK` | resigniertes Absinken |
| 7 | Release | 0,70 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | sehr langsame Rückkehr zur Neutralpose |

Gesamtdauer ≈ 4,40 s.

### Audio (optional)
Ein leises, abfallendes Atemgeräusch oder leiser Seufzer (≤ 800 ms), gestartet mit Beginn von Phase 2. Lautstärke deutlich unterhalb von `happy`-Sounds (Vorschlag: Volume 30 von 100). Kein perkussives Element.

### Idle-Modulation während Hold (Phase 4)
Tiefe, langsame Sinus-Modulation auf `z` (Amplitude 2 mm, Frequenz 0,15 Hz — also alle ~6,7 s) und `pitch` (Amplitude 1,5°, Frequenz 0,15 Hz, gleichphasig) — simuliert „schwere Brust hebt und senkt sich". Antennen leicht mit (Amplitude 1°, gleichphasig).

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen; der schräge Hang (Body-Yaw -8°, Head-Yaw -10°) ist konsistent — Body folgt Head leicht versetzt, kein Verdrehen.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 4.40`, `evaluate(t)` interpoliert sieben Phasen.
- Alternativ als Folge von `mini.goto_target(...)` mit `method=InterpolationTechnique.MIN_JERK` durchgehend; Phase 4 mit eigenem Idle-Loop (kontinuierlich `set_target` mit modulierten Werten bei 50 Hz Tick-Frequenz, da der Daemon nicht schneller publiziert).
- Pose -28° pitch und +6° roll liegen klar innerhalb der ±90°-Upright-Grenze.
- Body-Yaw -8° liegt deutlich innerhalb von ±160°.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „traurig" / „enttäuscht" / „niedergeschlagen" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 4,4 ± 0,3 s
- [ ] Kein einziger `CARTOON`-Easing-Aufruf in der Sequenz
- [ ] Hold-Phase 4 zeigt sichtbares „Atmen" ohne in „nervös" zu kippen
- [ ] Audio (falls aktiviert) ist deutlich leiser als `happy` und endet vor Phase 6
- [ ] Phase 5 wirkt als „schwacher Versuch", nicht als „beschwingt" — Pitch geht maximal auf -20° hoch, nicht auf 0°
- [ ] Übergang in nachfolgendes Behavior: Release-Phase fährt sauber bis zur Neutralpose

## Anti-Patterns
- Kurze Phasen (< 0,3 s) in der Hauptabsenkung — wirkt eilig, untergräbt die Schwere
- `CARTOON`- oder `EASE_IN_OUT`-Easing — beide tragen zu viel Energie ein
- Antennen bleiben auf 0° — entwertet den Effekt; das Hängen muss sichtbar sein
- Body-Yaw in beide Richtungen wackeln (rechts und links) — wirkt unentschieden statt traurig
- Schnelles Audio mit hohem Tempo — kollidiert mit dem schweren Tempo

## Offene Fragen
- Welche Audio-Datei dient als Referenz für den Seufzer? Vorschlag: kurzer absteigender Sinus oder ein generisches „aw"-Sample.
- Wie tief darf der Pitch hängen, ohne dass das Bild „kaputt" wirkt? Pitch -25° bis -30° empirisch zu prüfen.
- Soll Phase 5 (Hebeversuch) optional sein, oder fester Teil des Patterns? Argument für „fest": macht den Affekt menschlicher.
- Wie reagiert das Behavior, wenn die Stewart-Plattform für -28° Pitch + +6° Roll nahe der Singularität läuft? Im Zweifel Werte um 5° abrunden.
