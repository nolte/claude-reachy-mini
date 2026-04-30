# Bewegungsablauf: Warten (`waiting-idle`)

Status: draft

## Kontext
Der Standard-Idle-Zustand zwischen anderen Behaviors: kein affektiver Inhalt, nur eine subtile Atemzug-Modulation, damit Reachy nicht „eingefroren" wirkt. Anwendungsfälle: Default-State zwischen Triggern, ruhige Zwischenzeit nach abgeschlossenem Behavior, Demo-Pause. Implementiert das Pattern „Idle-Atemzug" aus `control-surface`.

## Charakteristik
- **Loop-fähig**: läuft kontinuierlich bis ein Event ein anderes Behavior triggert
- Pose ist die Neutralpose mit sehr leichter Atemzug-Modulation
- Z- und Pitch-Modulation in Phase, Frequenz 0,25 Hz (Atemzyklus von 4 s) — wie ruhiges menschliches Atmen
- Antennen leicht mitatmend (sehr kleine Amplitude)
- Body-Yaw stationär
- Sehr langsames Tempo, sehr geringe Amplitude — die Pose darf NICHT als Affekt gelesen werden

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Eintritt (in Idle) | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | Übergang von Folge-Behavior in Idle |
| 2 | Atemzyklus (Loop) | 4,0 / Cycle | (0, 0, 0 (±2 mm), 0, 0 (±1°), 0) | (0 (±2°), 0 (±2°)) | 0 | `MIN_JERK` (Idle-Mod) | Sinus-Atmen: alle 4 s ein Zyklus |
| 3 | Austritt | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weicher Übergang in Folge-Behavior |

Loop-Body-Cycle = 4,0 s. Mindestlauf (1 Cycle inkl. Eintritt + Austritt) ≈ 4,8 s. Tatsächlich loopt das Behavior unbegrenzt.

### Audio
**Kein Audio** — Idle ist still.

### Idle-Modulation während Phase 2
**Sinus-Mod**:
- `z = 2 * sin(2*pi*0.25*t)` — Amplitude 2 mm, Frequenz 0,25 Hz
- `pitch = 1 * sin(2*pi*0.25*t)` — Amplitude 1°, gleichphasig zu z
- `antenna_left = 2 * sin(2*pi*0.25*t)` — Amplitude 2°, gleichphasig
- `antenna_right = 2 * sin(2*pi*0.25*t)` — Amplitude 2°, gleichphasig

Alle Modulationen sind gleichphasig — das ist der Atemzug. Roll, Yaw, Body-Yaw bleiben strikt auf 0°.

### Body-Yaw und IK
Beliebig — Body-Yaw bleibt sowieso konstant 0°.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration=None` (unbegrenzte Laufzeit). Wird durch `cancel_move()` extern beendet, sobald ein anderes Behavior triggert.
- `evaluate(t)` berechnet die Sinus-Modulationen modulo Loop-Dauer (4,0 s) — der Atem ist kontinuierlich, ohne Loop-Naht.
- Velocity bei z = 2 mm × 2π × 0,25 = ~3,1 mm/s — winzig, weit unter allen Limits.
- Die Amplitude darf nicht erhöht werden, ohne den „affekt-freien" Charakter zu verlieren. ±2 mm und ±1° sind Maxima.
- Wenn das Plugin keine ständige Befehlsverbindung halten will (Energie-Sparen auf Wireless), kann `waiting-idle` auch als „periodisches Atmen alle ~10 s" implementiert werden, statt kontinuierlich.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik als „ruhig" / „lebendig" / „wartend" (mind. 4 von 5) — nicht als spezifische Emotion
- [ ] Atem-Modulation ist sichtbar, aber so subtil, dass sie nicht ablenkt
- [ ] Behavior loopt unbegrenzt ohne sichtbare Naht zwischen den Cycles
- [ ] Behavior endet sauber bei `cancel_move()` ohne Sprünge
- [ ] Roll, Yaw und Body-Yaw bleiben strikt auf 0°
- [ ] Kein Audio
- [ ] Übergang zu einem aktiven Behavior (z. B. `alert-listening`) ist nahtlos

## Anti-Patterns
- Atem-Frequenz > 0,5 Hz — wirkt nervös, nicht ruhig
- Atem-Amplitude > ±3 mm oder ±2° auf Pitch — wirkt wie ein affektives Behavior
- Roll- oder Yaw-Modulation einführen — entwertet die affektfreie Idle-Charakteristik
- Antennen statisch ohne Modulation — wirkt eingefroren
- Audio jeglicher Art — Idle muss still sein

## Offene Fragen
- Soll die Atem-Frequenz langsam driften (z. B. 0,2–0,3 Hz statt fix 0,25 Hz), damit der Idle nicht mechanisch wirkt? Pattern „Timing-Variation" aus `control-surface`.
- Wie integriert sich `waiting-idle` mit `mini.disable_motors()` für Energie-Spar-Modus? Vorschlag: nach 5 min Idle automatisch in `goto_sleep()` übergehen.
- Soll bei sehr kühlen Umgebungstemperaturen die Atem-Frequenz reduziert werden, um Servo-Wärme zu sparen?
