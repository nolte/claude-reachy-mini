# Bewegungsablauf: Raum scannen (`scanning`)

Status: draft

## Kontext
Eine ruhige, kontinuierliche Such-Bewegung des Kopfes von links nach rechts (und zurück) — wie ein Sicherheitskamera-Schwenker. Anwendungsfälle: Vision-basierte Personen-Suche, „wo ist…?"-Antwort-Modus, Demo-Modus zum Beobachten der Umgebung, Initialisierung von `look_at_world` durch Raumerkundung.

## Charakteristik
- Großer, kontinuierlicher Yaw-Sweep (-40° → +40° → -40° → 0°) — kein abruptes Wechseln, kein „Schauen-und-Halten"
- Pitch leicht oben (+5°) — Kamera-Blickfeld wird leicht angehoben
- Antennen aufgerichtet (+25°), aber stationär
- Body-Yaw folgt dem Head-Yaw weich (IK-gekoppelt) — der ganze Körper schwenkt
- Langes Tempo (~4,1 s); ausschließlich `EASE_IN_OUT` für die Sweeps

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Aufmerken) | 0,20 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `MIN_JERK` | erreichen der Scan-Pose |
| 2 | Sweep nach links | 1,20 | (0, 0, +3, 0, +5, -40) | (+25, +25) | -25 | `EASE_IN_OUT` | langsamer Schwenk nach links |
| 3 | Sweep nach rechts | 1,80 | (0, 0, +3, 0, +5, +40) | (+25, +25) | +25 | `EASE_IN_OUT` | doppelt so weiter Schwenk durch die Mitte |
| 4 | Sweep zur Mitte | 0,90 | (0, 0, +3, 0, +5, 0) | (+25, +25) | 0 | `EASE_IN_OUT` | zentrierender Auslauf |
| 5 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 4,50 s.

### Audio (optional)
Sehr leiser kontinuierlicher Such-Pieps (≤ 100 ms pro Pieps, alle 0,5 s) während Phasen 2–4. Lautstärke sehr leise (Volume 20). Optional auch ohne Audio.

### Idle-Modulation
Keine zusätzliche Modulation — der Sweep selbst ist die Bewegung.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — Body folgt Head smooth durch IK. Da Head-Yaw bis ±40° geht und Body-Yaw bis ±25°, bleibt die relative Yaw bei ±15° (innerhalb max_relative_yaw=±65°). Bei `False` würde der Kopf gegen den Body „verdreht" wirken.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 4.50`. Ideal kompatibel mit `look_at_world`-Aufrufen, die einen erkannten Punkt anvisieren — der Scan wird dann durch ein Tracking-Behavior ersetzt.
- Yaw-Geschwindigkeit Phase 3: 80°/1,8 s = 44°/s ≈ 0,77 rad/s, weit unter dem 8-rad/s-Joint-Limit.
- Phase 3 ist bewusst länger als Phase 2 (1,8 s vs. 1,2 s), weil sie 80° überstreicht, nicht nur 40°.
- Bei aktiver Vision-Pipeline: das Behavior kann via `cancel_move()` unterbrochen werden, sobald die Kamera ein Ziel erkennt; dann sofort `look_at_world(target)`.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik als „suchend" / „scannend" / „beobachtend" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 4,5 ± 0,4 s
- [ ] Sweep ist gleichmäßig, keine Zwischenstopps
- [ ] Body-Yaw bewegt sich sichtbar mit dem Head-Yaw
- [ ] Pitch bleibt während des Sweeps konstant bei +5°
- [ ] Antennen statisch auf +25°
- [ ] Bei `cancel_move()` mitten im Sweep fährt das Behavior sauber zur aktuellen Pose und gibt frei

## Anti-Patterns
- Pausen zwischen den Sweeps — bricht das kontinuierliche Such-Gefühl
- Pitch-Modulation während des Sweeps — wirkt unsicher
- `LINEAR`-Easing — wirkt mechanisch wie eine Kamera-Pan-Bewegung
- Body-Yaw nicht mit-bewegen — wirkt verdreht
- Antennen-Modulation — entwertet die ruhige Scan-Charakteristik
- Sweep-Geschwindigkeit > 60°/s — wirkt panisch statt ruhig

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
- Soll der Scan immer in derselben Richtung starten (links zuerst), oder Random?
- Soll bei einer aktiven Vision-Pipeline der Scan automatisch durch `look_at_world` ersetzt werden, sobald ein Ziel erkannt wird? Pro: kontextrelevant; Contra: erhöht Komplexität der Behavior-Komposition.
- Welche Audio-Datei eignet sich? Vorschlag: leises Sonar-artiges Pieps oder ganz ohne Audio.
- Soll es eine kürzere Variante (`scanning-quick`) geben, die nur ±25° schwenkt und ~2 s dauert?
