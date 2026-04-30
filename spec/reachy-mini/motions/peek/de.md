# Bewegungsablauf: Hervorlugen (`peek`)

Status: draft

## Kontext
Eine kleine spielerische Geste, in der Reachy seitlich hervorlugt — Body und Kopf drehen leicht zur selben Seite, Z hebt sich, der Kopf neigt sich kurz „suchend" und kehrt dann zurück. Anwendungsfälle: spielerische Reaktion auf Person, Versteck-Modus in einer Demo, „guck mal!"-Trigger, Aufmerksamkeit aus dem Idle holen ohne große Geste.

## Charakteristik
- Body- und Head-Yaw zur selben Seite (gleichgerichtet, keine Verdrehung) — zeigt eine klare Schaurichtung
- Z leicht angehoben (+5 mm) — wie „auf die Zehen stellen"
- Kleiner Pitch +5° und Roll 0° — neugierig, aber nicht so stark wie `curious`
- Mini-Yaw-Suchbewegung im Hold — leicht nach links und rechts in der ausgesehenen Richtung
- Mittleres Tempo (~2,0 s)

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (klein) | 0,15 | (0, 0, +2, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | minimaler Lift |
| 2 | Peek-Out (zur Seite) | 0,40 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `EASE_IN_OUT` | gleichgerichtetes Hervorlugen |
| 3 | Mini-Pause | 0,30 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `MIN_JERK` (statisch) | kurzer Halt zum Sehen |
| 4 | Suchbewegung | 0,30 | (0, 0, +5, 0, +5, +20 (±5°)) | (+15, +15) | +20 | `MIN_JERK` (Idle-Mod) | leichtes Yaw-Schwanken zum Suchen |
| 5 | Mini-Hold | 0,20 | (0, 0, +5, 0, +5, +25) | (+15, +15) | +20 | `MIN_JERK` (statisch) | kurzer Halt — gefunden? |
| 6 | Zurückziehen | 0,40 | (0, 0, +2, 0, +3, +5) | (+8, +8) | +5 | `MIN_JERK` | Yaw und Z klingen ab |
| 7 | Release | 0,30 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 2,05 s.

### Audio (optional)
Ein leiser fragender Pieps oder „Hm?"-Ton (≤ 300 ms), gestartet mit Phase 2. Lautstärke leise (Volume 30).

### Idle-Modulation während Phase 4
Sinus-Mod auf `yaw` (Amplitude 5°, Frequenz 1,0 Hz) — schnelles, kurzes Suchen. Pitch und Roll halten ihre Werte; Antennen statisch.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — die gleichgerichtete Drehung von Body und Head wirkt durch IK weicher. Die `+25° Head-Yaw` in Kombination mit `+20° Body-Yaw` ergibt eine relative Yaw von +5° — innerhalb des `max_relative_yaw=65°`.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.05`. `evaluate(t)` mit Sinus-Mod in Phase 4.
- Body-Yaw + Head-Yaw beide auf dieselbe Seite — anders als bei `curious` (wo Body in Tilt-Richtung geht und Head in Yaw-Richtung versetzt) und anders als bei `disagreeing-shake` (wo Body fix bleibt).
- Yaw +25° für Head und +20° für Body sind beide weit innerhalb der ±65° / ±160°-Limits.
- Z-Translation +5 mm liegt im IK-Volumen.
- Variante: spiegelbildliche Version (negative Yaw-Werte) für Hervorlugen zur anderen Seite — sinnvoll als Random-Wahl bei wiederholtem Trigger.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „lugen" / „guckend" / „hervorschauen" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,0 ± 0,2 s
- [ ] Body-Yaw und Head-Yaw zeigen sichtbar in dieselbe Richtung
- [ ] Z-Anhebung in Phase 2 ist sichtbar
- [ ] Phase 4 zeigt die kleine Suchbewegung deutlich
- [ ] Phasen 3 und 5 sind statisch ohne Modulation
- [ ] Audio (falls aktiviert) klingt fragend, nicht ausrufend

## Anti-Patterns
- Body-Yaw und Head-Yaw entgegengesetzt — wirkt verdreht, nicht peek
- Pitch < 0° — wirkt traurig, nicht neugierig
- Roll-Komponente — wird zu `curious`
- Suchbewegung in Phase 4 mit Frequenz > 2 Hz — wirkt nervös
- `CARTOON`-Easing in Phase 2 — wirkt federnd, nicht heimlich

## Offene Fragen
- Soll der Peek immer auf dieselbe Seite gehen oder Random links/rechts?
- Welche Audio-Datei eignet sich? Vorschlag: ein leiser fragender Pieps.
- Wie verhält sich `peek` mit `look_at_world` für eine erkannte Person? Zusammenführen wäre kontextrelevant.
- Soll bei keinem Such-Treffer (z. B. vorgegebener Punkt nicht in Sicht) ein Folge-`confused` getriggert werden?
