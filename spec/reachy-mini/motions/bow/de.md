# Bewegungsablauf: Verbeugung (`bow`)

Status: draft

## Kontext
Eine formelle, einzelne Verbeugung: Reachy senkt den Kopf in einer würdevollen, langsamen Bewegung, hält die Pose kurz und richtet sich wieder auf. Anwendungsfälle: Demo-Eröffnung oder -Abschluss, formelle Begrüßung eines Gastes, „Danke" als Antwort, „bitte" als Höflichkeitsgeste.

## Charakteristik
- Ein einziger, langsamer Pitch-Sweep nach unten (-25°) — kein Mehrfach-Nicken wie `agreeing-nod`
- Antennen leicht angelegt (-10°) — zurückhaltend, nicht aufgerichtet
- Body-Yaw und Head-Yaw zentriert — formelle, gerade Pose
- Z-Translation -5 mm während der Verbeugung — der ganze Kopf senkt sich, nicht nur der Pitch
- Mittleres Tempo (~2,4 s); ausschließlich `MIN_JERK` für die Würde

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Aufrichten) | 0,20 | (0, 0, +3, 0, +5, 0) | (-5, -5) | 0 | `MIN_JERK` | leichtes Aufrichten als Vorbereitung |
| 2 | Tiefe Verbeugung | 0,80 | (0, 0, -5, 0, -25, 0) | (-10, -10) | 0 | `MIN_JERK` | langsame, gleichmäßige Senkung |
| 3 | Hold | 0,40 | (0, 0, -5, 0, -25, 0) | (-10, -10) | 0 | `MIN_JERK` (statisch) | Pose bewusst halten |
| 4 | Aufrichten | 0,60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | langsames Wiederaufrichten |
| 5 | Release (subtil) | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | Übergang in Folge-Behavior |

Gesamtdauer ≈ 2,40 s.

### Audio (optional)
Optional ein leiser dezenter Ton mit Phase 2 (z. B. ein gehauchtes „Danke") — Lautstärke leise (Volume 25). In den meisten Anwendungsfällen ist `bow` audiolos.

### Idle-Modulation
Keine — die Stille während des Holds (Phase 3) trägt die Würde der Geste.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen; da Body und Head komplett zentriert bleiben, hat der Modus keine sichtbare Wirkung.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.40`, fünf Phasen — sehr schlank.
- Pitch -25° / 0,80 s = 31°/s ≈ 0,54 rad/s, weit unter der 8-rad/s-Joint-Grenze.
- Z-Translation -5 mm liegt komfortabel innerhalb des IK-erreichbaren Volumens.
- Pitch -25° liegt unter der ±90°-Grenze und nahe am Pitch-Limit von -28° in `sad`. Für eine tiefere „japanische" Verbeugung kann auf -30° gegangen werden, dann aber Translation und Pitch kombinieren statt Pitch alleine zu maximieren.
- Antennen-Wert -10° ist subtil und macht keinen affektiven Eindruck — bewusst neutral.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Verbeugung" / „formell" / „höflich" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,4 ± 0,2 s
- [ ] Phase 2 ist eine durchgehende, gleichmäßige Senkung — kein Zwischenstopp
- [ ] Phase 3 (Hold) ist statisch ohne Modulation
- [ ] Roll und Yaw bleiben über die gesamte Sequenz auf 0°
- [ ] Aufrichten in Phase 4 ist gleichmäßig, ohne Ruckler
- [ ] Audio (falls aktiviert) ist leise und unaufgeregt

## Anti-Patterns
- Mehrere Pitch-Wechsel — wirkt wie `agreeing-nod`, nicht wie `bow`
- Roll- oder Yaw-Komponenten — entwertet die formale Pose
- `CARTOON`- oder `LINEAR`-Easing — falsche Charakteristik
- Pitch tiefer als -30° — riskiert Stewart-Joint-Grenze und sieht überlastet aus
- Hold (Phase 3) kürzer als 0,3 s — der formelle Charakter geht verloren
- Antennen aufgerichtet (positive Werte) — wirkt freudig statt formell

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Soll es Varianten geben (`bow-deep` mit -30° für besondere Ehrungen, `bow-light` mit -15° für Alltagshöflichkeit)?
- Welche Audio-Datei ist passend? Vorschlag: leises „Danke" oder „bitte" als optionales Audio.
- Soll die Translation Z bei einer tiefen Variante stärker abgesenkt werden, statt nur den Pitch zu erhöhen?
