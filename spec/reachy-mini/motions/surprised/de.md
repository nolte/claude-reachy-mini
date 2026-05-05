# Bewegungsablauf: Überrascht (`surprised`)

Status: draft

## Kontext
Eine schlagartige Reaktion, die als „überrascht" oder „erstaunt" gelesen wird: Reachy macht einen kurzen, kleinen Krümmer, schnellt dann scharf nach oben, friert kurz im Schreckmoment ein, zittert leicht und orientiert sich dann fragend zur Seite. Anwendungsfälle: unerwarteter Trigger (Türsensor, Geräusch), „Da ist was!", Reaktion auf plötzliche Bewegung in der Kamera.

## Charakteristik
- Schnelles, scharfes Hochschnellen (positive Z + positiver Pitch) — vermittelt „Aufschreck"
- Antennen nach oben/außen geöffnet (stark positive Werte) — wie aufgerichtete Ohren
- Frozen-Hold direkt nach dem Snap — der Schreck friert die Pose kurz ein
- Optionales Mikro-Zittern (kleine Amplitude, kurze Dauer)
- Anschließendes Yaw-Orientieren — „wo war das?"
- Sehr schnelles Tempo (~1,8 s); harter `LINEAR`-Snap für den Schock-Moment

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Pre-Anticipation (Krümmer) | 0,05 | (0, 0, -2, 0, -3, 0) | (0, 0) | 0 | `LINEAR` | minimaler Sprungvor-Krümmer, fast unsichtbar |
| 2 | Snap nach oben | 0,15 | (0, 0, +15, 0, +18, 0) | (+40, +40) | 0 | `LINEAR` | scharfer, schneller Hochschnell-Snap |
| 3 | Frozen Hold (Schreck) | 0,40 | (0, 0, +15, 0, +18, 0) | (+40, +40) | 0 | — (statisch) | Pose völlig statisch, keine Idle-Modulation |
| 4 | Mikro-Zittern | 0,20 | (0, 0, +14, 0 (±0,5°), +18 (±0,8°), 0) | (+40, +40) | 0 | `MIN_JERK` (Idle-Mod) | sehr kleines Zittern auf Roll und Pitch |
| 5 | Yaw-Orientieren links | 0,25 | (0, 0, +12, 0, +15, -25) | (+38, +35) | -8 | `EASE_IN_OUT` | „wo war das?" — fragender Schwenk |
| 6 | Yaw-Orientieren rechts | 0,25 | (0, 0, +12, 0, +15, +25) | (+35, +38) | +8 | `EASE_IN_OUT` | spiegelbildlicher Such-Schwenk |
| 7 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Auflösen zur Neutralpose |

Gesamtdauer ≈ 1,80 s.

### Audio (optional)
Ein kurzer scharfer Klang (≤ 200 ms) wie ein „Oh!"- oder „Hm?"-Sample, getriggert exakt mit Phase 2 (Snap). Lautstärke moderat (Volume 50). Audio-Buffer-Latenz ~50 ms berücksichtigen.

### Idle-Modulation während Mikro-Zittern (Phase 4)
Sehr kleine, schnelle Sinus-Mod auf `pitch` (Amplitude 0,8°, Frequenz 6 Hz) und `roll` (Amplitude 0,5°, Frequenz 5 Hz, leicht versetzt). Antennen statisch.

### Body-Yaw und IK
`automatic_body_yaw=True` für die Yaw-Schwenker (Phasen 5–6) — Body folgt dem fragenden Suchen weich. Bei Phase 2 (Snap) bleibt Body bei 0° — der Schreck ist eine Kopf-Reaktion, nicht eine Ganzkörper-Reaktion.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.80`, kompakter `evaluate(t)`.
- Phase 2 ist der wichtigste Frame — Velocity-Check: pitch von -3° auf +18° in 0,15 s = ~140°/s ≈ 2,4 rad/s, weit innerhalb des 8-rad/s-Joint-Limits.
- Die Antennen-Bewegung von 0° auf +40° in 0,15 s = ~267°/s ≈ 4,7 rad/s, ebenfalls innerhalb der Grenzen.
- Phase 3 (Frozen Hold) explizit ohne Idle-Mod — der Schock ist die Stille zwischen Snap und Zittern.
- `LINEAR`-Easing in Phasen 1 und 2 ist Pflicht für den Schreck-Charakter.
- Z-Translation +15 mm liegt komfortabel im IK-erreichbaren Volumen.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „überrascht" / „erschrocken" / „aufmerksam" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,8 ± 0,2 s
- [ ] Phase 2 wirkt als sichtbarer harter Snap, nicht als weicher Move (`LINEAR` ist Pflicht)
- [ ] Phase 3 zeigt einen sichtbaren statischen Halt (≥ 0,3 s)
- [ ] Mikro-Zittern in Phase 4 ist erkennbar, aber nicht aufdringlich
- [ ] Yaw-Schwenker (Phasen 5–6) wirken fragend, nicht kontrolliert — der `EASE_IN_OUT` macht den Unterschied
- [ ] Audio (falls aktiviert) trifft den Snap exakt; verspätetes Audio entwertet den Schreck-Effekt
- [ ] Release fährt sauber zur Neutralpose

## Anti-Patterns
- `MIN_JERK` in Phase 2 — der Snap wird zu weich, der Überraschungs-Charakter verschwindet
- Phase 1 (Pre-Anticipation) länger als 0,1 s — ruiniert den Schock, weil er angekündigt wird
- Frozen Hold (Phase 3) kürzer als 0,3 s — der Beobachter sieht keinen Halt
- Body-Yaw beim Snap (Phase 2) abweichend von 0° — verteilt die Reaktion auf zu viele Achsen
- Audio später als Phase 2 — entkoppelt Klang von Bewegung

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Offene Fragen
- Reicht ein einzelner Yaw-Schwenker (nur Phase 5) oder sind beide nötig? Empirisch testen — beide sind dramatischer.
- Sollte das Mikro-Zittern (Phase 4) optional sein? Pro: macht den Affekt menschlicher. Contra: macht das Behavior länger.
- Welche Audio-Datei eignet sich? Vorschlag: kurzes scharfes „Oh!" oder ein synthetisches Aufmerk-Signal.
- Wie reagiert das Behavior bei sehr kurzem Trigger-Abstand (zwei Überraschungen direkt nacheinander)? Cancel-and-restart oder Queue?
