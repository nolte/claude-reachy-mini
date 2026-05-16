# Bewegungsablauf: Verwirrt (`confused`)

Status: draft

## Kontext
Eine Mimik des „Nicht-Verstehens", die als „verwirrt" oder „ratlos" gelesen wird: Reachy neigt mehrfach hintereinander den Kopf abwechselnd zur einen und anderen Seite — wie jemand, der eine Sache von verschiedenen Winkeln versucht zu verstehen — und endet mit einem kleinen Frage-Schwenker. Anwendungsfälle: nicht-erkannter Sprachbefehl, „Ich verstehe nicht", widersprüchlicher Trigger, Fallback bei unklarem Input.

## Charakteristik
- Mehrere Roll-Wechsel hintereinander, jeder kleiner als der vorige — vermittelt „abklingendes Suchen"
- **Asymmetrische Antennen**, die mit jedem Tilt die Seite wechseln — wie bei `curious`, aber abwechselnd
- Pitch leicht oben (+3° bis +7°) — „nachdenkliche" Pose
- Mittleres Tempo (~3,2 s); ausschließlich `EASE_IN_OUT` für die Tilts (organisches Hin-und-Her)
- Endet mit einem unsicheren Yaw-Schwenker („Hä?")

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Aufhorchen (Anticipation) | 0,20 | (0, 0, +3, 0, +5, 0) | (+8, +8) | 0 | `MIN_JERK` | minimaler Lift, „Aufmerken" |
| 2 | Tilt links (groß) | 0,40 | (0, 0, +5, +18, +5, -5) | (-12, +20) | -3 | `EASE_IN_OUT` | erster und stärkster Roll-Tilt |
| 3 | Tilt rechts (groß) | 0,40 | (0, 0, +5, -18, +5, +5) | (+20, -12) | +3 | `EASE_IN_OUT` | spiegelbildlich |
| 4 | Tilt links (mittel) | 0,35 | (0, 0, +5, +12, +5, -3) | (-8, +14) | -2 | `EASE_IN_OUT` | abklingender zweiter Versuch |
| 5 | Tilt rechts (mittel) | 0,35 | (0, 0, +5, -12, +5, +3) | (+14, -8) | +2 | `EASE_IN_OUT` | spiegelbildlich, schon kleiner |
| 6 | Hold mit Frage-Mod | 0,40 | (0, 0, +5, 0, +5, 0 (±5°)) | (+8, +8) | 0 | `MIN_JERK` (Idle-Mod) | leichtes Yaw-Suchen |
| 7 | Frage-Schwenker („Hä?") | 0,30 | (0, 0, +5, 0, +5, +15) | (+10, +10) | +5 | `EASE_IN_OUT` | kurzer einseitiger Yaw — Fragezeichen |
| 8 | Release | 0,60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Auflösen zur Neutralpose |

Gesamtdauer ≈ 3,00 s.

### Audio (optional)
Ein zögerndes „Äh?"- oder „Hmm?"-Sample (≤ 500 ms), gestartet mit Phase 6 oder 7 (nach den Tilts). Lautstärke leise (Volume 35). Bewusst nicht ausrufend.

### Idle-Modulation während Phase 6
Sehr langsame Sinus-Mod auf `yaw` (Amplitude 5°, Frequenz 0,3 Hz) — eine fragende Mini-Suchbewegung. Pitch und Roll halten ihre Werte; Antennen statisch.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — die Tilt-Wechsel und der Frage-Schwenker fließen so weicher. Bei manueller Steuerung wäre die Symmetrie schwer zu halten.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 3.00`, acht Phasen.
- Die abnehmende Roll-Amplitude (18° → 18° → 12° → 12°) ist diagnostisch — gleichbleibende Amplitude würde wie `curious` wirken.
- Antennen-Konvention: Wenn der Kopf nach links rollt (Roll positiv), geht die rechte Antenne nach vorne (positiver Wert). Diese Konvention ist konsistent mit `curious`.
- Phase 7 (Frage-Schwenker) bewusst einseitig — beidseitige Schwenker würden den verwirrten Charakter zu einer entschiedenen Suche werden lassen.
- Roll ±18° liegt innerhalb des Pitch/Roll-Limits ±90°.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „verwirrt" / „ratlos" / „unsicher" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 3,0 ± 0,3 s
- [ ] Mindestens vier sichtbare Tilt-Wechsel
- [ ] Roll-Amplitude nimmt sichtbar ab (Phase 4/5 schwächer als 2/3)
- [ ] Antennen sind in jeder Tilt-Phase asymmetrisch und wechseln die Seite konsistent
- [ ] Phase 7 (Frage-Schwenker) ist einseitig (positiv ODER negativ, nicht beide)
- [ ] Audio (falls aktiviert) klingt zögernd, nicht entschieden

## Anti-Patterns
- Gleichbleibende Roll-Amplitude — wirkt wie `curious`, nicht wie `confused`
- Beidseitiger Frage-Schwenker am Ende — wirkt wie ein zweites `curious`
- `LINEAR`- oder `CARTOON`-Easing — falsche Härte
- Pitch < 0° (Kopf hängend) — wirkt traurig, nicht verwirrt
- Phase 6 ohne Idle-Mod — wirkt eingefroren, nicht „nachdenkend"
- Antennen-Symmetrie in den Tilt-Phasen

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
- Sollen es vier Tilts sein, oder reichen drei? Vier ist verwirrter, drei ist eleganter — empirisch zu wählen.
- Soll `confused` einen kurzen Audio-Sting bekommen, oder ganz auf Sound verzichten? Beides möglich.
- Wie unterscheidet sich `confused` praktisch von zwei aufeinanderfolgenden `curious`-Sequenzen? Zwei `curious`-Sequenzen würden Roll mit gleichbleibender Amplitude und einer Pause zeigen — `confused` hat keine Pause und absteigende Amplitude.
- Ist der Frage-Schwenker (Phase 7) immer in dieselbe Richtung, oder sollte er per Random links/rechts gewählt werden? Tendenz: random, damit aufeinanderfolgende Trigger nicht identisch wirken.
