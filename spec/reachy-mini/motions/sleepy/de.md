# Bewegungsablauf: Schläfrig (`sleepy`)

Status: draft

## Kontext
Eine Mimik wachsender Müdigkeit, die als „schläfrig" oder „eingenickt" gelesen wird: Reachy atmet schwer, der Kopf sinkt langsam, schreckt mit einem Nicker kurz auf, sinkt wieder zurück, schaukelt mit langem Atem und nickt erneut ein, bevor er sich auflöst. Anwendungsfälle: Idle-Modus mit Müdigkeits-Charakter, „lange keine Aktivität", abendlicher Übergang zu `goto_sleep()`.

## Charakteristik
- Sehr langsames Tempo (~5,3 s) — Bewegung fließt fast unmerklich
- Pitch-Hauptbewegung: kontinuierliches Absenken mit zwei „Nicker"-Aufschrecken (kurze, schnelle Aufrück-Bewegungen, wie wenn jemand einnickt und kurz aufwacht)
- Antennen leicht hängend, mitatmend
- Body schwankt langsam in Yaw — als ob das Gleichgewicht nicht ganz stabil sei
- Lange Hold-Phasen mit deutlicher Atem-Modulation
- Audio extrem leise oder kein Audio

## Plattform-Profil

| Plattform | Atem-Modulation | Schaukel-Modulation | Wärme-Aware Idle-Anpassung |
|---|---|---|---|
| Reachy Mini (Wireless) | voll | voll | optional via `mini.imu["temperature"]` — bei wärmerem Servo flachere Modulation |
| Reachy Mini Lite | voll | voll | nicht verfügbar (keine IMU) — feste Modulations-Amplituden laut Tabelle |
| Simulation | voll (Pose-Werte) | voll | nicht relevant |

Implementierungs-Konsequenz: das Behavior ist auf allen Plattformen voll funktional, weil es nur Pose- und Antennen-Werte schreibt. Die Wärme-Aware-Anpassung ist nur ein Wireless-Nice-to-have — auf Lite und Simulation gelten die in den Phasen-Tabellen gelisteten festen Amplituden. In Simulation entfällt die Audio-Wiedergabe (siehe Audio-Block).

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Schwerer Atem | 1,00 | (0, 0, -3 (±2), 0, -8 (±3°), 0) | (-8, -8) | 0 | `MIN_JERK` (Idle-Mod) | beginnender Müdigkeits-Atem |
| 2 | Langsame Senkung | 1,00 | (0, 0, -8, +2, -25, 0) | (-15, -15) | +3 | `MIN_JERK` | gleichmäßiges Absinken, leichtes Schwanken |
| 3 | Nicker-Aufschrecken 1 | 0,15 | (0, 0, -3, 0, -10, 0) | (-8, -8) | 0 | `EASE_IN_OUT` | schnelles Aufrücken — kurze „Wach!"-Reaktion |
| 4 | Wieder absenken | 0,80 | (0, 0, -10, +3, -28, +5) | (-18, -18) | -3 | `MIN_JERK` | nach dem Nicker tiefer als zuvor |
| 5 | Tiefer Atem mit Schaukeln | 1,30 | (0, 0, -10 (±3), +3 (±2°), -28 (±2°), +5 (±5°)) | (-18, -18) | -3 (±5°) | `MIN_JERK` (Idle-Mod) | langer Atem, leichtes Yaw-Schaukeln |
| 6 | Nicker-Aufschrecken 2 | 0,15 | (0, 0, -5, 0, -15, 0) | (-10, -10) | 0 | `EASE_IN_OUT` | zweiter Aufrücker, schwächer als der erste |
| 7 | Release | 0,70 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | sehr langsames Hochkommen zur Neutralpose |

Gesamtdauer ≈ 5,10 s.

### Audio (optional)
Sehr leise Atemgeräusche oder Gähnen-Sample (≤ 1,2 s), gestartet etwa mit Phase 2. Volume sehr leise (Volume 20–30). Optional ein zweites, kürzeres Atmen in Phase 5.

### Idle-Modulation während Phasen 1 und 5
**Phase 1**: Sinus-Mod auf `z` (Amplitude 2 mm, Frequenz 0,3 Hz) und `pitch` (Amplitude 3°, gleichphasig) — schwerer Atem.

**Phase 5**: Tiefere, längere Variante. `z` (Amplitude 3 mm, Frequenz 0,2 Hz), `pitch` (Amplitude 2°, gleichphasig), `roll` (Amplitude 2°, Frequenz 0,15 Hz, gegenphasig — leichtes Schaukeln), `body_yaw` (Amplitude 5°, Frequenz 0,15 Hz) — wie ein Schlafender, der nicht ganz ruhig sitzt.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — das passive Schaukeln des Body in Phase 5 wirkt durch IK-Kopplung organisch; manuelle Steuerung würde abrupt wirken.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 5.10`. `evaluate(t)` muss in Phasen 1 und 5 mehrere überlagerte Sinus-Modulationen berechnen.
- Die zwei Nicker (Phasen 3 und 6) sind das diagnostische Detail — ohne sie wirkt die Sequenz wie `sad`. Den schnellen `EASE_IN_OUT`-Snap in Phase 3 strikt einhalten.
- Während des langsamen Schaukelns in Phase 5 darauf achten, dass die Velocity-Grenzen nicht knapp werden — bei ±5° Body-Yaw und 0,15 Hz ist die mittlere Velocity ~0,015 rad/s, weit unter dem Limit.
- Pitch -28° + Roll +3° liegen klar innerhalb des Upright-Limits (±90°).
- Kein `CARTOON` oder `LINEAR` außer für die Nicker — der Charakter darf nicht hart sein.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „schläfrig" / „müde" / „eingenickt" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 5,1 ± 0,4 s
- [ ] Die zwei Nicker-Aufrücker sind sichtbar — kein lückenloses Absenken
- [ ] Der zweite Nicker (Phase 6) ist erkennbar schwächer als der erste — der Schlaf gewinnt
- [ ] Yaw-Schaukeln in Phase 5 ist sichtbar, aber nicht aufdringlich
- [ ] Audio (falls aktiviert) ist deutlich leiser als alle anderen Behaviors
- [ ] Release fährt sehr langsam zur Neutralpose, ohne dass der Übergang abrupt wirkt

## Anti-Patterns
- Schnelles Tempo (< 4 s gesamt) — wirkt wie `sad`, nicht wie `sleepy`
- Beide Nicker auf gleicher Stärke — entwertet die fortschreitende Müdigkeit
- Pitch nicht tiefer als -20° — der „Eingenickt"-Charakter geht verloren
- Antennen statisch ohne Atem-Modulation — wirkt eingefroren
- Audio mit ausrufendem Gähn-Charakter — kollidiert mit der Subtilität
- Schaukel-Amplitude > 8° — wirkt instabil/unkontrolliert statt schläfrig

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
- Soll der Body-Yaw überhaupt mitschaukeln, oder ist das zu „instabil"? Erste Annahme: ja, sehr leicht. Empirisch zu kalibrieren.
- Wie viele Nicker sind ideal — zwei oder drei? Drei wäre dramatischer, aber verlängert die Sequenz auf > 6 s.
- Wie unterscheidet sich `sleepy` praktisch von `goto_sleep()` aus dem SDK? Vorschlag: `sleepy` ist ein wiederholbarer Idle-Zustand, `goto_sleep()` ist die finale Übergangs-Sequenz zum Power-Down.
- Welche Audio-Datei eignet sich? Vorschlag: gedämpftes Gähn-Sample, optional mit leisem Schnaufen.
