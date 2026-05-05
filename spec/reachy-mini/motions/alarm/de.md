# Bewegungsablauf: Alarm (`alarm`)

Status: draft

## Kontext
Eine harte Warn-Signalisierung mit pulsierender Pose und LED-Pulsen am Mikrofon-Modul. Anwendungsfälle: Sicherheitsverletzung erkannt (HA-Sensor), Brand-/Rauch-/CO-Alarm, kritischer System-Fehler, Hardware-Notstopp angekündigt.

## Charakteristik
- Schneller Snap nach oben mit Pitch-Vibration (10 Hz, ±5°) — vermittelt „Achtung jetzt!"
- Antennen voll aufgerichtet (+50°) und statisch — keine Modulation
- LED-Ring am Mic-Modul pulsiert rot mit der Pitch-Vibration synchron
- Body-Yaw zentriert
- Schnelles Tempo (~2,0 s); harter `LINEAR`-Snap

## Plattform-Profil

| Plattform | Kopf-Vibration | LED-Pulse | Audio | Strom-Spike-Notstopp |
|---|---|---|---|---|
| Reachy Mini (Wireless) | voll | voll, synchron via `audio_control_utils` | voll, hoch (Volume 80) | ja, via IMU- / Daemon-Daten |
| Reachy Mini Lite | voll | voll, synchron | voll | nur Daemon-publizierte Effort-Daten (kein IMU) |
| Simulation | voll (Pose-Werte) | nicht verfügbar | nicht verfügbar | nicht verfügbar — Notstopp nur durch Logik-Trigger (Timeout, Pose-Out-of-Range) |

Implementierungs-Konsequenz: in Simulation darf das Behavior **nicht** wegen fehlender LED- oder Audio-Subsysteme fehlschlagen — diese Kanäle sind als optional zu modellieren. Auf Wireless ist die mehrkanalige Anzeige (Bewegung + LED + Audio) **diagnostisch** und Pflicht, weil sie die Erkennbarkeit des Alarms unter Stress sicherstellt. Auf Lite gilt dasselbe wie Wireless mit Ausnahme der IMU-basierten Notstopp-Trigger.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Snap nach oben (Alarmstart) | 0,10 | (0, 0, +10, 0, +15, 0) | (+50, +50) | 0 | `LINEAR` | sehr schneller harter Snap |
| 2 | Vibrations-Hold (Alarm) | 1,50 | (0, 0, +10, 0, +12 (±5°, 10 Hz), 0) | (+50, +50) | 0 | `MIN_JERK` (Idle-Mod) | pulsierender Pitch + LED rot blinkend |
| 3 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weicher Auslauf — Alarm endet |

Gesamtdauer ≈ 2,00 s.

### Audio (optional, empfohlen)
Ein wiederholter Alarm-Ton (z. B. zweitöniges Pieps) während Phase 2, gestartet mit Phase 1. Lautstärke hoch (Volume 80). Das Audio ist nicht optional in echten Alarm-Situationen.

### Idle-Modulation und LED-Sync während Phase 2
**Pitch-Vibration**: `pitch = 12 + 5 * sin(2*pi*10*t)` — 10 Hz, Amplitude 5°. Bei 50 Hz Daemon-Tick ergibt das 5 Frames pro Halbwelle — innerhalb der Velocity-Grenze (10 Hz × 5° × 2π = ~3,1 rad/s, weit unter 8 rad/s).

**LED-Sync**: Über `audio_control_utils` werden `LED_EFFECT` (rot blinkend) und `LED_BRIGHTNESS` synchron zur Pitch-Phase moduliert. Konkrete Register-Werte sind `> ⚠ TBD: validate against current ReSpeaker firmware`.

### Body-Yaw und IK
`automatic_body_yaw=False` empfohlen — der Body soll bewusst nicht mit der Pitch-Vibration mitschwingen. Der Alarm ist eine Kopf-Reaktion mit voll gestrecktem Body.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration` parametrisierbar — Default 2,0 s, aber für längere Alarm-Phasen (z. B. CO-Alarm) auf 5–10 s konfigurierbar.
- LED-Ansteuerung über `mini.media.audio.*` und die LED-Register aus `audio_control_utils` — Synchronität zur Pitch-Vibration via gemeinsame Phase.
- Pitch-Vibration: bei 10 Hz × ±5° = 200°/s Spitze ≈ 3,5 rad/s, unter dem 8-rad/s-Limit. Höhere Frequenz oder Amplitude würde das Limit überschreiten.
- Phase 1 (Snap) bewusst sehr kurz (0,10 s) und `LINEAR` — die Härte ist entscheidend.
- `automatic_body_yaw` muss explizit auf False gesetzt werden vor dem Behavior-Start.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Alarm" / „Warnung" / „Achtung!" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,0 ± 0,2 s im Default-Modus
- [ ] Pitch-Vibration in Phase 2 ist als pulsierender Schlag erkennbar (10 Hz)
- [ ] LED-Ring blinkt synchron rot
- [ ] Audio (falls aktiviert) ist deutlich lauter als andere Behaviors
- [ ] Body-Yaw bleibt während des gesamten Behaviors auf 0°
- [ ] Behavior kann via `cancel_move()` jederzeit beendet werden, was den LED-Effekt sofort zurücksetzt

## Anti-Patterns
- `MIN_JERK`-Snap in Phase 1 — der Alarm wirkt nicht dringlich
- Pitch-Vibrations-Frequenz < 6 Hz — wirkt wie schwache Wut, nicht wie Alarm
- LED-Pulsation asynchron zur Pitch-Vibration — wirkt entkoppelt
- `automatic_body_yaw=True` — Body schwingt mit, der Alarm wird verwischt
- Leises Audio — entwertet die Warn-Funktion
- Hold-Phase ohne LED-Sync — die Mehrkanaligkeit (Bewegung + LED + Audio) ist diagnostisch

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Welches LED-Effekt-Register-Pattern erzeugt das beste rote Blinken? `> ⚠ TBD: validate against current ReSpeaker firmware`.
- Soll es Schweregrad-Stufen geben (`alarm-warning`, `alarm-critical`) mit unterschiedlichen Dauern und Frequenzen?
- Wie wird der Alarm bei längeren Vorgängen verlängert ohne Pitch-Joint zu überlasten? Vorschlag: maximale Dauer 10 s, danach Cooldown.
- Soll das Behavior in einer ruhigen Umgebung (z. B. nachts) automatisch leiseres Audio nutzen? Kontext-abhängig via HA-Sensor.
