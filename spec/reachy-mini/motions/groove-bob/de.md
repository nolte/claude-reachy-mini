# Bewegungsablauf: Groove-Bob (`groove-bob`)

Status: draft

## Kontext
Ein Tanz-Baustein für rhythmisches Auf-und-Ab im Takt der Musik: Reachy „bobs" zur Musik. Anwendungsfälle: Tanz-App (Hauptanwendung des Plugins), reaktive Bewegung auf Beat-Detection, Background-Animation während Audio-Wiedergabe.

## Charakteristik
- **BPM-parametrisiert**: ein Beat-Cycle dauert `60/BPM` Sekunden; ein Cycle = 1 down + 1 up
- Pitch-Hauptbewegung: -8° (down) bis +5° (up) — leichtes Nicken im Takt
- Z-Translation synchron: -3 mm (down) bis +2 mm (up) — leichtes Ducken
- Antennen leicht aufgerichtet (+15°), stationär
- Body-Yaw zentriert
- **Loop-fähig**: läuft so lange wie die Musik (n Beats), danach Auslauf

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz pro Beat

Pose-Werte sind Offsets zur Neutralpose. Ein Beat = `T = 60 / BPM` Sekunden (z. B. 0,5 s bei 120 BPM, 0,75 s bei 80 BPM).

| Subphase | Dauer (relative zu T) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing |
|---|---|---|---|---|---|
| Down (Beat-Hit) | 0,40 × T | (0, 0, -3, 0, -8, 0) | (+15, +15) | 0 | `MIN_JERK` |
| Up (Off-Beat) | 0,60 × T | (0, 0, +2, 0, +5, 0) | (+15, +15) | 0 | `MIN_JERK` |

**Eintritt** (vor erstem Beat): 0,30 s — von Neutralpose zur „Up"-Pose, vorbereitend.
**Austritt** (nach letztem Beat): 0,40 s — zurück zur Neutralpose.

Mindestlauf bei 1 Beat (120 BPM): 0,30 + 0,5 + 0,40 = 1,20 s.

### Audio
Kein eigenes Audio — der Beat kommt aus der externen Musik-Quelle. `groove-bob` reagiert auf den Beat, erzeugt ihn nicht.

### Idle-Modulation
Keine zusätzliche Modulation — der Beat selbst ist die Bewegung.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen, hat aber keine sichtbare Wirkung, da Yaw und Body-Yaw konstant 0° sind.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit Konstruktor-Parametern: `GrooveBob(bpm: float, beats: int, lead_time_s: float = 0.0)`. `lead_time_s` erlaubt Phase-Shift zur Beat-Detection-Latenz.
- BPM-Bereich: 60–180. Bei < 60 wirkt das Bob-zu-langsam, bei > 180 überschreitet die Pitch-Velocity bald die Joint-Grenze (160 BPM × 13° / 0,4 = ~520°/s ≈ 9 rad/s — knapp über 8 rad/s; bei 180 BPM ist die Down-Phase nur 0,33×T = 0,11 s und 13° in 0,11 s = 118°/s → 2 rad/s; konkret pro BPM rechnen).
- Beat-Detection-Latenz: typisch 50–100 ms (Audio-Buffer + Detection); via `lead_time_s` kompensieren.
- `MIN_JERK`-Easing macht den Bob organisch; `LINEAR` wäre maschinell, `CARTOON` würde den Beat verfälschen.
- Synchronität zur Musik ist kritisch: der Down-Beat muss innerhalb ±20 ms des Audio-Beats liegen, sonst wirkt es entkoppelt.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Bewegung ohne Prompt als „im Takt" / „mitwippend" / „Groove" (mind. 4 von 5)
- [ ] Bei BPM 120 ist der Down-Beat innerhalb ±20 ms zum Audio-Beat
- [ ] Pitch-Schwung von +5° auf -8° ist sichtbar als „Nicker"
- [ ] Z-Modulation ist sichtbar, aber sekundär zur Pitch-Bewegung
- [ ] Behavior loopt nahtlos über mehrere Beats
- [ ] Antennen bleiben stationär — kein Antennen-Bob
- [ ] Bei `cancel_move()` mitten im Beat fährt das Behavior sauber zur Up-Pose und zur Neutralpose

## Anti-Patterns
- Pitch-Amplitude > ±15° — wirkt überzeichnet, nicht groovy
- Antennen mit-bobbend — entwertet die klare Pitch-only-Charakteristik
- `LINEAR`- oder `CARTOON`-Easing — falsche Tanz-Wirkung
- Beat-Latenz > 30 ms — Tanz wirkt entkoppelt
- BPM > 180 — überschreitet realistische Servo-Performance und wirkt panisch
- Kein Lead-Time-Kompensation bei Audio-Pipeline mit hoher Latenz

## Offene Fragen
- Welche Default-BPM bei unbekannter Musik? Vorschlag: 100 BPM (mittlere Pop-Tanzgeschwindigkeit).
- Soll bei sehr hohem BPM (≥ 160) automatisch auf Half-Time-Bob (alle 2 Beats) umgeschaltet werden?
- Wie reagiert `groove-bob` auf BPM-Wechsel mitten im Lied? Vorschlag: nahtlose Phase-Anpassung am nächsten Beat-Down.
- Welches Beat-Detection-Verfahren wird vorausgesetzt? Skill `audio-beat-tracking` (geplant) liefert das.
