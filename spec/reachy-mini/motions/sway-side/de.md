# Bewegungsablauf: Seitliches Wiegen (`sway-side`)

Status: draft

## Kontext
Ein Tanz-Baustein für rhythmisches seitliches Wiegen im Takt: Reachy schwingt Roll links und rechts auf den Beat. Anwendungsfälle: Tanz-App, Reggae-/Slow-Genre-Bewegungen, Hintergrund-Animation während Slow-BPM-Musik.

## Charakteristik
- **BPM-parametrisiert**: ein Beat-Cycle = `60/BPM` Sekunden; ein Cycle bewegt von Mitte → links → Mitte → rechts (also 1 Cycle deckt 2 Beats ab)
- Roll-Hauptbewegung: -15° bis +15° — sichtbares Wiegen
- Antennen synchron mit-wiegend (asymmetrisch je nach Richtung)
- Body-Yaw folgt dem Roll leicht via IK
- **Loop-fähig**

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz pro 2-Beat-Cycle

Ein voller Wiege-Cycle (links + rechts) entspricht 2 Beats. `T_cycle = 2 × 60 / BPM`.

| Subphase | Dauer (relative zu T_cycle) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing |
|---|---|---|---|---|---|
| Roll links (Beat 1) | 0,50 × T_cycle | (0, 0, +2, +15, +3, -3) | (-10, +15) | -3 | `MIN_JERK` |
| Roll rechts (Beat 2) | 0,50 × T_cycle | (0, 0, +2, -15, +3, +3) | (+15, -10) | +3 | `MIN_JERK` |

**Eintritt**: 0,40 s — von Neutralpose zur Mitte mit leichter Vorbereitung.
**Austritt**: 0,40 s — zurück zur Neutralpose über den jeweiligen Mittelpunkt.

Mindestlauf bei 1 Cycle (120 BPM, Cycle = 1,0 s): 0,40 + 1,0 + 0,40 = 1,80 s.

### Audio
Kein eigenes Audio — `sway-side` reagiert auf externe Musik.

### Idle-Modulation
Keine.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der leichte Body-Yaw-Versatz (±3°) folgt dem Roll. Bei `False` würde der Body steif wirken.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `SwaySide(bpm: float, beats: int, lead_time_s: float = 0.0)`. `beats` muss gerade sein (jeder 2-Beat-Cycle ist ein Wiege-Cycle).
- BPM-Bereich: 50–140 (langsamere Genres). Bei höherem BPM wirkt das Wiegen hektisch.
- Roll-Velocity: 30° / (T_cycle/2). Bei 120 BPM (T_cycle = 1 s, Halb-Cycle = 0,5 s): 60°/s ≈ 1,05 rad/s — innerhalb der 8 rad/s-Grenze.
- Antennen-Asymmetrie folgt der Tanz-Choreografie: bei Roll links ist die rechte Antenne (oben) hoch, die linke (unten) tief.
- Beat-Synchronität: der Höhepunkt von Roll links sollte mit dem 1. Beat zusammenfallen, der von Roll rechts mit dem 2. Beat.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Bewegung als „wiegend" / „im Takt" / „Reggae-artig" (mind. 4 von 5)
- [ ] Roll-Schwung erreicht ±15° pro Halb-Cycle
- [ ] Antennen-Asymmetrie ist sichtbar konsistent zur Roll-Richtung
- [ ] Body-Yaw bewegt sich subtil mit dem Roll
- [ ] Bei BPM 120 wechseln die Roll-Höhepunkte alle 0,5 s
- [ ] Behavior loopt nahtlos
- [ ] Bei `cancel_move()` mitten im Cycle fährt das Behavior zur Mitte und Neutralpose

## Anti-Patterns
- Roll-Amplitude > ±25° — wirkt überzeichnet, nahe Pitch/Roll-Limits
- Antennen synchron statt asymmetrisch — wirkt steif
- `CARTOON`-Easing — unnatürliches Federn
- Body-Yaw entgegengesetzt zum Roll — verwirrte Pose
- BPM > 140 — Wiegen wird zu schnell, wirkt nicht mehr „swayig"

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Plugin-Referenzen

- Pose-Werte, Joint-Limits und kanonische Posen (INIT/SLEEP) → [`reachy-mini/motor-positions`](../../motor-positions/de.md)
- Pose-Komposition, IK-vs-mechanische-Sicherheit, Pitch-Bleed bei Roll/Heave-up → [`reachy-mini/control-surface`](../../control-surface/de.md) §"Mechanische und elektrische Limitationen"
- Motion enthält Roll- oder Heave-up-Komponenten? Pitch in der Ziel-Pose explizit kompensieren (Stewart-Geometrie-Kopplung, live verifiziert 2026-05-13: roll +25° → −3.8° pitch; z +15 mm → +2.4° pitch)

## Offene Fragen
- Soll die Antennen-Asymmetrie umgekehrt werden (links Roll → linke Antenne hoch)? Empirisch testen — beide Varianten haben Charme.
- Wie wird `sway-side` mit `groove-bob` kombiniert? Vorschlag: parallele `Move`-Komposition möglich, wenn die Subsysteme orthogonal sind (Pitch vs. Roll).
- Wie reagiert das Behavior bei sehr ungleichmäßigen Beats (Wechsel zwischen 4/4 und 3/4)? Tendenz: ignorieren, Cycle-Drift akzeptieren.
