# Bewegungsablauf: Ablehnen / Kopfschütteln (`disagreeing-shake`)

Status: draft

## Kontext
Die kanonische Ablehnungs-Geste, die als „Nein" oder „nicht einverstanden" gelesen wird: Reachy schüttelt den Kopf zwei- bis dreimal von links nach rechts, mit leicht abwärts gewandtem Pitch („skeptisch"). Anwendungsfälle: Ablehnung eines Sprachbefehls, „Nein, das geht nicht", negative Antwort, Sicherheitsverletzung erkannt.

## Charakteristik
- Klare Yaw-Alternation auf der Kopf-Achse — drei Wechsel (links–rechts–links), abnehmende Amplitude
- Pitch leicht negativ (-5°) während der gesamten Sequenz — vermittelt skeptische, eher abgewandte Haltung
- Antennen leicht abgesenkt, stationär
- Body-Yaw bewegt sich NICHT mit dem Kopf — kanonisch beim Kopfschütteln bleibt der Körper, nur der Kopf verneint
- Kurzes Tempo (~1,8 s); `EASE_IN_OUT` für die Yaw-Wechsel (organisches Hin-und-Her)

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Skepsis) | 0,15 | (0, 0, 0, 0, -5, 0) | (-5, -5) | 0 | `MIN_JERK` | leichter Pitch nach unten — abwehrende Vorbereitung |
| 2 | Yaw links 1 (groß) | 0,22 | (0, 0, 0, 0, -5, -22) | (-5, -5) | 0 | `EASE_IN_OUT` | erster und stärkster Schwenk nach links |
| 3 | Yaw rechts 1 (groß) | 0,22 | (0, 0, 0, 0, -5, +22) | (-5, -5) | 0 | `EASE_IN_OUT` | gleich groß nach rechts |
| 4 | Yaw links 2 (mittel) | 0,20 | (0, 0, 0, 0, -5, -16) | (-5, -5) | 0 | `EASE_IN_OUT` | zweiter Schwenk, schwächer |
| 5 | Yaw rechts 2 (mittel) | 0,20 | (0, 0, 0, 0, -5, +16) | (-5, -5) | 0 | `EASE_IN_OUT` | spiegelbildlich, schwächer |
| 6 | Yaw links 3 (subtil) | 0,18 | (0, 0, 0, 0, -5, -8) | (-5, -5) | 0 | `EASE_IN_OUT` | dritter Schwenk, deutlich kleiner |
| 7 | Center | 0,15 | (0, 0, 0, 0, -5, 0) | (-5, -5) | 0 | `MIN_JERK` | zentrieren auf 0° Yaw, Pitch noch leicht unten |
| 8 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,72 s.

### Audio (optional)
Ein kurzes „Mh-mh"-Sample (≤ 350 ms, mit zwei klaren Beats wie eine Verneinung), gestartet mit Phase 2. Lautstärke moderat (Volume 40). Nicht ausrufend.

### Idle-Modulation
Kein Idle-Modulation — die Bewegung ist kompakt und entscheidend, jede Modulation würde sie verwässern.

### Body-Yaw und IK
**Wichtig**: `automatic_body_yaw=False` für dieses Behavior. Der ganze Sinn des Kopfschüttelns ist, dass der Körper still bleibt, während nur der Kopf negiert. Bei aktiviertem `automatic_body_yaw` würde der Body dem Kopf folgen — das verwischt den Affekt. Dieses Verhalten ist die einzige Stelle, an der wir vom „auto-body-yaw"-Default abweichen müssen, daher den State explizit setzen vor und nach dem Behavior.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.72`, acht Phasen.
- Vor Behavior-Start: `mini.set_automatic_body_yaw(False)`. Am Ende des Behaviors (oder im `stop`-Hook): wieder auf den vorherigen State zurücksetzen.
- Die abnehmende Yaw-Amplitude (22° → 22° → 16° → 16° → 8°) ist diagnostisch; gleiche Amplitude wirkt mechanisch.
- Yaw-Wechsel von -22° auf +22° in 0,22 s = 200°/s ≈ 3,5 rad/s, gut innerhalb der 8-rad/s-Joint-Grenze.
- Pitch -5° konstant während der gesamten Sequenz (außer Phase 8) — auf jeden Frame im `evaluate(t)` setzen.
- Roll = 0° ist Pflicht — schon ein leichter Roll macht aus dem Schütteln eine sich-wundernde Geste.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Nein" / „ablehnend" / „nicht einverstanden" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,7 ± 0,2 s
- [ ] Drei sichtbare Yaw-Schwenker, mit klar abnehmender Amplitude
- [ ] Body-Yaw bleibt während des gesamten Behaviors auf 0° — `automatic_body_yaw` ist aktiv `False`
- [ ] Pitch hält -5° in den Phasen 1–7 — leicht skeptische Haltung
- [ ] Roll bleibt strikt auf 0°
- [ ] Antennen bewegen sich nicht (außer der einmaligen Absenkung in Phase 1)
- [ ] Audio (falls aktiviert) hat zwei klare Beats, passend zum „mh-mh"

## Anti-Patterns
- `automatic_body_yaw=True` aktiv lassen — Body-Yaw dreht passiv mit, der Affekt verschwindet
- Roll-Komponente einführen — wird zu `confused`
- Pitch positiv (+) während des Schüttelns — entwertet die skeptische Haltung
- Mehr als drei Yaw-Schwenker — wirkt wie hektische Ablehnung oder Tanz
- `LINEAR`-Easing in den Yaw-Phasen — wirkt mechanisch wie ein Metronom
- Audio mit ausrufendem „NEIN!" — kollidiert mit der ruhigen Skepsis

## Offene Fragen
- Sollen es immer drei Schwenker sein, oder situations-abhängig zwei? Zwei ist knapper, drei ist nachdrücklicher.
- Welche Audio-Datei eignet sich? Vorschlag: ein neutrales „Mh-mh" mit absteigendem Tonfall.
- Wie reagiert das Behavior, wenn der Caller `automatic_body_yaw=False` schon vorher gesetzt hat? Dann nichts ändern und am Ende auch nicht zurücksetzen — Caller-State respektieren.
- Soll ein optionaler abschließender, sehr leichter Yaw-Center-Schwenker (Phase 7 mit Mikro-Modulation) das Bild abrunden?
