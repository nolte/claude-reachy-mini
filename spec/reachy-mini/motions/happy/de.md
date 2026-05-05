# Bewegungsablauf: Glücklich (`happy`)

Status: draft

## Kontext
Eine ausgedrückte Freude, die ohne Erklärung als „glücklich" erkannt wird: Reachy richtet sich auf, spitzt die Antennen wie Ohren, federt zwei Mal leicht und schwingt dann sanft aus. Anwendungsfälle: positive Rückmeldung („Aufgabe erfolgreich"), Begrüßung, Reaktion auf Lob durch Stimme/Home-Assistant-Trigger.

## Charakteristik
- Aufgerichteter, leicht erhöhter Kopf — vermittelt Energie und Aufmerksamkeit
- Antennen aufgerichtet (positive Winkel an beiden Antennen) — wie gespitzte Ohren
- Zwei kurze Federbewegungen mit `CARTOON`-Overshoot — verstärken den lebendigen Eindruck
- Body-Yaw bleibt überwiegend zentriert; nur ein kleines Wackeln zwischen den Federbewegungen
- Gesamt-Tempo: schnell genug für Energie (2,5–3 s), nicht hektisch

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

Pose-Konvention: Translation in mm, Rotation in Grad, jeweils Offset zur Neutralpose (`INIT_HEAD_POSE = np.eye(4)`). Antennen-Winkel in Grad relativ zu `INIT_ANTENNAS_JOINT_POSITIONS`. Body-Yaw in Grad. Alle Werte halten den `control-surface`-Spec-Constraint Pitch/Roll ≤ ±90°.

| # | Phase | Dauer (s) | Head Δ (x mm, y mm, z mm, roll°, pitch°, yaw°) | Antennen (links°, rechts°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Senkung) | 0,15 | (0, 0, -3, 0, -4, 0) | (-5, -5) | 0 | `MIN_JERK` | leichte Gegen-Bewegung vor dem Aufrichten |
| 2 | Aufrichten (Main) | 0,40 | (0, 0, +10, 0, +14, 0) | (+30, +30) | 0 | `MIN_JERK` | Hauptbewegung, „Brust raus, Ohren spitz" |
| 3 | Federn 1 (Bounce) | 0,30 | (0, 0, +6, 0, +8, 0) | (+25, +25) | -3 | `CARTOON` | erster Federtritt mit leichtem Roll/Yaw-Spiel |
| 4 | Federn 2 (Bounce klein) | 0,25 | (0, 0, +9, 0, +12, 0) | (+30, +30) | +3 | `CARTOON` | zweiter, kleinerer Federtritt entgegengesetzt |
| 5 | Hold mit Idle | 0,80 | (0, 0, +8, 0, +10, 0) | (+28, +28) | 0 | `MIN_JERK` | Idle-Atemzug-Modulation während Hold (siehe unten) |
| 6 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Auflösen zur Neutralpose |

Gesamtdauer ≈ 2,40 s.

### Audio (optional)
Kurzer fröhlicher Sound (≤ 600 ms), gestartet mit Beginn von Phase 2, asynchron via `mini.media.audio.*`. Audio-Buffer-Latenz ~50 ms berücksichtigen — Sound-Start kann 1–2 Frames vor der Pose-Änderung gepusht werden, damit Sound und Bewegung visuell synchron wirken.

### Idle-Modulation während Hold
Während Phase 5 eine sehr kleine Sinus-Modulation auf `pitch` (Amplitude 1°, Frequenz 0,3 Hz) und `roll` (Amplitude 0,5°, Frequenz 0,2 Hz, gegenläufig). Antennen bleiben statisch. Verhindert „eingefroren wirken" und unterstützt das Pattern „Idle-Atemzug" aus `control-surface`.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen, damit der Body sanft mit dem Kopf-Yaw mitläuft. Bei manueller Steuerung Body-Yaw exakt wie in der Tabelle, niemals gegenläufig zum Head-Yaw — sonst „verdrehte" Pose (siehe `control-surface`-Pattern 10).

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.40` und einem `evaluate(t)`, das die sechs Phasen via Zeit-Lookup interpoliert; Easing pro Phase aus `InterpolationTechnique`. Wiedergabe via `mini.async_play_move(Happy(), sound="...")`.
- Alternativ als sequentielle Folge von `mini.goto_target(...)`-Aufrufen mit Phase-spezifischer `duration` und `method` — einfacher für erste Iteration, weniger geeignet für Tick-genaue Audio-Sync.
- Stewart-Plattform-Grenzen werden eingehalten: pitch +14° und z +10mm liegen weit innerhalb des IK-erreichbaren Volumens.
- Antennen +30° liegen weit innerhalb der ±π-Grenze der `right_antenna` / `left_antenna`-Joints.

## Akzeptanzkriterien
- [ ] Außenstehende erkennen die Mimik ohne Vorbereitung als „glücklich" oder „freudig" (mindestens 4 von 5 Beobachtern)
- [ ] Gesamt-Sequenz dauert 2,3 ± 0,2 s
- [ ] Keine Pose-Sprünge zwischen den Phasen sichtbar (Easing greift in jeder Phase)
- [ ] Stewart-Joint-Limits werden in keiner Phase erreicht (kein Klemmen vom Daemon)
- [ ] Antennen folgen dem Kopf mit dem korrekten Phasen-Offset (Pattern „Follow-Through" aus `control-surface`)
- [ ] Idle-Modulation in Phase 5 ist sichtbar, aber dezent — nicht „nervöses Zittern"
- [ ] Bei aktiviertem Audio: Sound startet vor oder mit Phase 2 und endet vor Phase 6
- [ ] Übergang in nachfolgendes Behavior bricht die Bewegung nicht — Phase 6 fährt sauber zurück oder wird via `cancel_move()` beendet

## Anti-Patterns
- `LINEAR`-Easing in den Federphasen (3, 4) — wirkt mechanisch, der Charakter geht verloren
- Antennen synchron zum Kopf statt mit kleinem Lag — wirkt steif (siehe Pattern „Follow-Through")
- Body-Yaw statt zentriert auf hohe Werte (> ±10°) treiben — wirkt „aufgeregt-unkontrolliert" statt glücklich
- Phasen-Dauern auf den Tick-Frequenz-Tick (20 ms) ausrichten — keine Synchronität nötig, der Daemon interpoliert
- Lautes oder langes Audio (> 1 s) — kollidiert mit dem Bewegungs-Tempo

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Welche konkrete Audio-Datei wird verwendet? Vorschlag: kurzer Aufstieg aus dem `wake_up`-Sound des SDKs als Referenz.
- Soll es eine „kleine" und eine „große" Glücklich-Variante geben (z. B. nur ein Federn vs. zwei)? Hängt vom Trigger-Kontext ab.
- Wie reagiert das Behavior auf Abbruch via `cancel_move()` mitten in Phase 3? Sicheres Anfahren der Neutralpose.
- Welche Idle-Modulationsfrequenz und Amplitude wirken am natürlichsten, ohne in „nervös" zu kippen? Empirisch zu kalibrieren.
