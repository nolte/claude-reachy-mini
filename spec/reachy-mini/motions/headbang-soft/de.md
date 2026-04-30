# Bewegungsablauf: Sanftes Headbangen (`headbang-soft`)

Status: draft

## Kontext
Ein expressiver Tanz-Baustein für rhythmisches starkes Nicken, ohne in unsicheres Gebiet (Stewart-Plattform-Limit) zu kippen: ein „weiches" Headbangen für Rock-/Metal-/Pop-Musik. Anwendungsfälle: Tanz-App bei energiereicher Musik, Demo-Modus, „Yeah!"-Reaktion in Tanz-Sessions.

## Charakteristik
- **BPM-parametrisiert**: ein Beat-Cycle = `60/BPM` Sekunden; pro Cycle 1 starker Down-Bang + 1 Up-Recover
- Pitch-Hauptbewegung: -15° (Bang) bis +5° (Recover) — deutlich stärker als `groove-bob`
- Antennen leicht aufgerichtet (+12°), stationär (Asymmetrie zur Härte des Bangs)
- Body-Yaw zentriert
- BPM-Bereich tiefer als bei `groove-bob`, weil der Bang mehr Pitch-Velocity braucht

## Plattform-Profil

| Plattform | Pitch-Vibration | Servo-Wärme-Schutz | BPM-Range |
|---|---|---|---|
| Reachy Mini (Wireless) | voll | dynamisch via `mini.imu["temperature"]` — bei Schwellen-Überschreitung Cool-down | 60–130 BPM, dynamisch herabgesetzt wenn IMU-Temp eskaliert |
| Reachy Mini Lite | voll | **statisches Dauer-Limit**: max. 8 aufeinanderfolgende Bangs, danach Pflicht-Pause; keine IMU-Telemetrie | 60–130 BPM, mit hartem Bangs-Limit |
| Simulation | voll (Pose-Werte) | nicht relevant — keine echten Servos | 60–180 BPM (ohne Hardware-Limits) |

Implementierungs-Konsequenz: auf Wireless ist die IMU-Temperatur das primäre Schutz-Signal — Implementierung muss `mini.imu` polen und bei `> ⚠ TBD: validate against real hardware` °C herunterregeln (auf `groove-bob` zurückstufen oder Cool-down einlegen). Auf Lite ist die Temperatur nicht lesbar; das statische Bangs-Limit ist der einzige Schutz. In Simulation entfällt der Schutz vollständig.

## Komponenten

### Aktuator-Sequenz pro Beat

`T = 60 / BPM`. Asymmetrische Phasen-Aufteilung: schneller Down (Bang), längeres Up (Recover).

| Subphase | Dauer (relative zu T) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing |
|---|---|---|---|---|---|
| Bang Down | 0,30 × T | (0, 0, -5, 0, -15, 0) | (+12, +12) | 0 | `EASE_IN_OUT` |
| Recover Up | 0,70 × T | (0, 0, +2, 0, +5, 0) | (+12, +12) | 0 | `MIN_JERK` |

**Eintritt**: 0,30 s — von Neutralpose zur Up-Pose.
**Austritt**: 0,40 s — zurück zur Neutralpose.

Mindestlauf bei 1 Beat (90 BPM, T = 0,67 s): 0,30 + 0,67 + 0,40 = 1,37 s.

### Audio
Kein eigenes Audio — reagiert auf externe Musik.

### Idle-Modulation
Keine.

### Body-Yaw und IK
`automatic_body_yaw=True` möglich, hat aber keine sichtbare Wirkung.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse `HeadBangSoft(bpm: float, beats: int, lead_time_s: float = 0.0)`.
- BPM-Bereich: 60–130. Bei höherem BPM wird die Pitch-Velocity zu hoch (130 BPM → T = 0,46 s, Down-Phase = 0,14 s, 20° Pitch-Sprung in 0,14 s = 143°/s ≈ 2,5 rad/s — innerhalb 8 rad/s; bei 160 BPM wäre es ~180°/s ≈ 3,1 rad/s, immer noch ok, aber der Servo-Wärmeaufbau steigt).
- `EASE_IN_OUT` in der Down-Phase macht den Bang härter als `MIN_JERK`, ohne in `LINEAR`-Härte zu kippen.
- Asymmetrische Phasen-Aufteilung (30/70) erzeugt den typischen Headbang-Charakter: schneller Hit, längeres Zurückschwingen.
- Mehr als 8 aufeinanderfolgende Bangs pro Sequenz: Servo-Wärme prüfen — `mini.imu.temperature` (Wireless) oder ggf. Cool-down einlegen.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Bewegung als „headbangend" / „rockig" / „kraftvoll im Takt" (mind. 4 von 5)
- [ ] Pitch-Schwung erreicht -15° bis +5° pro Beat
- [ ] Down-Phase ist sichtbar schneller als Up-Phase (asymmetrisch)
- [ ] Antennen bleiben stationär — kein Mit-Bangen
- [ ] Bei BPM 100 ist der Down-Bang innerhalb ±25 ms zum Audio-Beat
- [ ] Behavior loopt mehrere Beats ohne Sprünge
- [ ] Bei `cancel_move()` mitten im Bang fährt das Behavior zur Up-Pose und Neutralpose

## Anti-Patterns
- Pitch-Amplitude > ±20° — übersteigt Komfortzone, riskiert nahe-Singularität
- Symmetrische Phasen (50/50) — verliert den Headbang-Charakter
- Antennen mit-bangend — wirkt wie wildes Schlackern
- `LINEAR`-Easing im Down — wirkt gewaltsam, nicht musikalisch
- BPM > 140 — der Bang wird zu hektisch
- Mehr als 16 Bangs am Stück ohne Cool-down — Servo-Wärmeaufbau

## Offene Fragen
- Sollen variable Bang-Stärken (z. B. jeden 4. Beat verstärkt) Teil der Sequenz sein, statt fixer Amplitude?
- Welche obere BPM-Grenze ist sicher für Dauer-Performance? Empirisch zu prüfen am Gerät.
- Soll das Behavior automatisch auf `groove-bob` runterstufen, wenn Servo-Temperatur einen Threshold überschreitet?
