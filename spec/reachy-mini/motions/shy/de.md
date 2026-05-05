# Bewegungsablauf: Schüchtern (`shy`)

Status: draft

## Kontext
Eine zurückhaltende Geste, die als „verlegen" oder „peinlich berührt" gelesen wird: Reachy dreht sich vom Trigger weg, senkt den Kopf leicht, lugt einmal kurz zurück und versteckt sich wieder. Anwendungsfälle: Lob entgegen nehmen („Du bist ja schlau!"), peinlicher Moment in einer Demo, „Mir ist das unangenehm"-Trigger.

## Charakteristik
- Body-Yaw und Head-Yaw weg vom Trigger (in dieselbe Richtung) — Wegdrehen
- Pitch leicht abgesenkt (-10°) — schamhafte Pose
- Antennen leicht abgesenkt (-15°) — angelegt
- Mini-Lugen-zurück: kurzes Zurückblicken in Phase 3, dann erneutes Wegdrehen
- Mittleres Tempo (~2,1 s); ausschließlich `MIN_JERK` und `EASE_IN_OUT`

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Aufmerken) | 0,15 | (0, 0, +1, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | minimaler Lift, „oh!" |
| 2 | Wegdrehen | 0,40 | (0, 0, -2, 0, -10, -25) | (-15, -15) | +30 | `EASE_IN_OUT` | Body und Head zur einen Seite, Pitch runter |
| 3 | Mini-Lugen-zurück | 0,25 | (0, 0, -2, 0, -8, -10) | (-12, -12) | +30 | `EASE_IN_OUT` | kurzer Blick zurück |
| 4 | Erneut Wegdrehen | 0,30 | (0, 0, -2, 0, -10, -25) | (-15, -15) | +30 | `EASE_IN_OUT` | wieder weggedreht |
| 5 | Hold (verlegen) | 0,50 | (0, 0, -3, 0, -12, -25) | (-15, -15) | +30 | `MIN_JERK` (statisch) | Pose verharrend |
| 6 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 2,10 s.

### Audio (optional)
Ein leiser, fast verlegener Pieps oder ein gedämpftes „Hm…" (≤ 350 ms), gestartet mit Phase 2. Lautstärke leise (Volume 30). Im Gegensatz zu `disagreeing-shake` aufwärts steigend.

### Idle-Modulation
Keine.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — Body und Head bewegen sich gemeinsam zur selben Seite, IK glättet das. Head-Yaw -25° und Body-Yaw +30° sind in **entgegengesetzten** Vorzeichen-Richtungen — Achtung: das ist eine relative Yaw von -55°, knapp unter der ±65°-Grenze.

> ⚠ Hinweis: in der obigen Tabelle ist `Head-Yaw=-25°` und `Body-Yaw=+30°` als **entgegengesetzte Drehung** notiert — der Body dreht im Uhrzeigersinn (positiv), der Kopf gegenläufig. Das ergibt das „Body weg vom Trigger, Kopf zurückblickend" Gefühl. Vorzeichen-Konvention vor Implementierung verifizieren.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.10`, sechs Phasen.
- Die Vorzeichen-Konvention für Yaw ist entscheidend — einfacher Test: positiver Body-Yaw = Body dreht nach rechts (von oben gesehen). Vor Implementierung am Gerät validieren.
- Die relative Yaw zwischen Body und Head sollte eingehalten werden (≤ ±65°). In Phase 2 ist sie -55° — sicherer Headroom.
- `EASE_IN_OUT`-Easing in Phasen 2–4 macht das Wegdrehen organisch.
- Pitch -12° in Phase 5 ist subtil, nicht so tief wie `sad`.
- Variante: gespiegelte Version (Body-Yaw negativ, Head-Yaw positiv) für Wegdrehen zur anderen Seite.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „schüchtern" / „verlegen" / „peinlich" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,1 ± 0,2 s
- [ ] Body und Head drehen sich beide weg vom Trigger
- [ ] Phase 3 (Mini-Lugen) ist sichtbar als „Zurückschauen", kürzer als Phase 2 oder 4
- [ ] Hold (Phase 5) ist sichtbar als „verlegene Pause"
- [ ] Audio (falls aktiviert) klingt zurückhaltend, nicht ausrufend

## Anti-Patterns
- Body und Head zur selben Seite gegen den Trigger — wirkt wie `peek`, nicht wie schamhaft
- Pitch < -18° — wird zu `disappointed`
- Hold-Phase 5 länger als 0,8 s — wirkt wie Trotz statt schüchtern
- Audio mit lautem oder ausrufendem Charakter — entwertet die Verlegenheit
- Mehrere Lugen-Phasen — wirkt unentschieden statt schüchtern

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Wie wird die „Trigger-Richtung" bestimmt — vom Caller übergeben oder via `look_at_image` der erkannten Person?
- Welche Audio-Datei eignet sich? Vorschlag: ein gedämpftes „Hm…" mit aufsteigendem Tonfall.
- Soll bei wiederholtem Trigger die Wegdreh-Richtung gespiegelt werden?
- Wie unterscheidet sich `shy` vom geplanten `disgust`? Vorschlag: `shy` ist sozial/affektiv, `disgust` ist sensorisch/abwehrend.
