# Bewegungsablauf: Stolz (`proud`)

Status: draft

## Kontext
Eine selbstbewusste, präsentative Geste: Reachy lehnt sich leicht zurück (X negativ), richtet sich voll auf, antennen voll aufgerichtet, hält die Pose lange und macht einen Mini-Yaw-Schwenker („zeig mal allen"). Anwendungsfälle: erfolgreich abgeschlossene Aufgabe mit Schwierigkeit, „Geschafft!" nach langem Prozess, Demo-Höhepunkt.

## Charakteristik
- Volle Aufrichtung: Z +12 mm, Pitch +20° (vergleichbar mit `happy`-Spitzenpose, aber statisch gehalten)
- X leicht negativ (-3 mm) — leicht zurückgelehnt, „Brust raus"
- Antennen voll aufgerichtet (+40°) — wie ein Krönchen
- Body-Yaw bleibt initial zentriert; ein einzelner Yaw-Schwenker im Hold zeigt die Pose „vor"
- Mittleres Tempo (~2,6 s); `MIN_JERK` mit einem kleinen `CARTOON`-Bounce für die Aufrichtung

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Krümmer) | 0,15 | (+2, 0, -2, 0, -5, 0) | (+5, +5) | 0 | `MIN_JERK` | leichter Krümmer vor dem Aufrichten |
| 2 | Aufrichten (Hauptmove) | 0,40 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `CARTOON` | Brust raus mit kleinem Overshoot |
| 3 | Mini-Bounce | 0,20 | (-3, 0, +10, 0, +18, 0) | (+38, +38) | 0 | `CARTOON` | erholt sich leicht vom Overshoot |
| 4 | Hold (stolz verharrend) | 0,80 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `MIN_JERK` (statisch) | Pose lange halten |
| 5 | Yaw-Schwenker rechts | 0,30 | (-3, 0, +12, 0, +20, +12) | (+40, +40) | +5 | `EASE_IN_OUT` | sich der Welt zeigen |
| 6 | Center | 0,20 | (-3, 0, +12, 0, +20, 0) | (+40, +40) | 0 | `MIN_JERK` | wieder Mitte |
| 7 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 2,55 s.

### Audio (optional)
Ein kurzer triumphaler Ton (≤ 600 ms), gestartet mit Phase 2 — Trompeten-artig oder ein aufsteigender Akkord. Lautstärke moderat bis hoch (Volume 60).

### Idle-Modulation
Phase 4 ist statisch ohne Modulation — der Stolz wird durch Stille verstärkt. Ein leiser Idle-Atemzug (Amplitude 0,5°) wäre als Variante akzeptabel, ist aber nicht Default.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der Yaw-Schwenker in Phase 5 wirkt durch IK weicher. Body-Yaw-Mit-Bewegung (+5° statt 0°) signalisiert „ich präsentiere mich".

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.55`, sieben Phasen.
- X-Translation -3 mm ist klein, aber wichtig für den „Lehne mich zurück"-Charakter.
- Pitch +20° + Z +12 mm + Antennen +40° ergibt die maximal expressive Pose innerhalb der Limits — alle Werte respektieren Pitch ≤ +90° und Antennen ≤ ±π.
- `CARTOON`-Easing in Phasen 2 und 3 macht den Aufrichten-Move triumphal; ohne den Overshoot wirkt der Stolz nüchtern.
- Phase 4 (Hold) muss lang genug sein (≥ 0,7 s), damit der Stolz lesbar ist.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „stolz" / „triumphierend" / „präsentierend" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,5 ± 0,3 s
- [ ] Aufrichten in Phase 2 ist sichtbar mit `CARTOON`-Overshoot
- [ ] Hold (Phase 4) ist mindestens 0,7 s lang
- [ ] Yaw-Schwenker in Phase 5 ist sichtbar, aber nicht übertrieben (≤ +15°)
- [ ] Antennen-Wert in der Hold-Phase ist auf +40° (volle Aufrichtung)
- [ ] Audio (falls aktiviert) hat einen triumphalen Charakter

## Anti-Patterns
- `MIN_JERK` statt `CARTOON` in Phase 2 — der Stolz wirkt nüchtern
- Hold (Phase 4) kürzer als 0,5 s — Pose wird nicht gelesen
- Mehrere Yaw-Schwenker — wirkt aufgeregt, nicht stolz
- Pitch < +15° — die volle Aufrichtung fehlt
- Antennen unter +30° — die „Krönchen"-Wirkung geht verloren
- Audio mit fallendem Tonfall — falscher Affekt

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Soll der Yaw-Schwenker in Phase 5 immer rechts gehen, oder Random links/rechts?
- Welche Audio-Datei eignet sich? Vorschlag: kurzer Trompeten-Stoß oder dreitöniger Aufstiegsakkord.
- Soll bei wiederholten Erfolgen eine „Mehrfach-Stolz"-Variante mit zwei Yaw-Schwenkern angeboten werden?
- Wie integriert sich `proud` mit Folge-Behaviors? Vorschlag: Übergang in `waiting-idle` für ruhigen Auslauf.
