# Bewegungsablauf: Neugierig (`curious`)

Status: draft

## Kontext
Die klassische Neugier-Geste: Reachy neigt den Kopf zur Seite, hält die Pose musternd, neigt zur anderen Seite und richtet sich dann wieder auf. Anwendungsfälle: Frage stellen, „Wie geht's weiter?", Reaktion auf neuen Input, Modus-Anzeige bei Listening / Aufmerksamkeit. Das Pollen-Notebook beschreibt diese Mimik direkt als „tilt head + asymmetric antennas".

## Charakteristik
- Deutlicher Roll-Tilt (±15° bis ±20°) als Hauptmerkmal — das ist die kanonische Curious-Geste
- **Asymmetrische Antennen**: die Antenne auf der „nach unten"-geneigten Seite leicht nach vorne, die andere zurück — verstärkt das musternde Bild
- Mittleres Tempo (~3,3 s); ausschließlich `MIN_JERK` und `EASE_IN_OUT`, kein `CARTOON`, kein `LINEAR`
- Body-Yaw leicht in die Tilt-Richtung — der Körper „lehnt sich mit"
- Mini-Yaw-Modulation während der Hold-Phasen — leichte musternde Suchbewegung

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Aufrichten (Anticipation) | 0,20 | (0, 0, +3, 0, +5, 0) | (+5, +5) | 0 | `MIN_JERK` | leichte Aufrichtung als Vorbereitung |
| 2 | Tilt links | 0,50 | (0, 0, +5, +20, +6, -8) | (-15, +25) | -5 | `EASE_IN_OUT` | Hauptbewegung — links-geneigt; rechte Antenne nach vorn, linke zurück |
| 3 | Hold links mit Mod | 0,80 | (0, 0, +5, +20, +6, -8 (±3°)) | (-15, +25) | -5 | `MIN_JERK` (Idle-Mod) | langsame Yaw-Modulation — „suchend" |
| 4 | Tilt rechts | 0,50 | (0, 0, +5, -20, +6, +8) | (+25, -15) | +5 | `EASE_IN_OUT` | Spiegelbild-Tilt zur anderen Seite |
| 5 | Hold rechts mit Mod | 0,60 | (0, 0, +5, -20, +6, +8 (±3°)) | (+25, -15) | +5 | `MIN_JERK` (Idle-Mod) | etwas kürzer als Phase 3 |
| 6 | Release | 0,60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Auflösen zur Neutralpose |

Gesamtdauer ≈ 3,20 s.

### Audio (optional)
Ein kurzer fragender Klang („Hmm?"-Sample, ≤ 400 ms), gestartet mit Phase 2. Lautstärke leise bis moderat (Volume 35–50). Eine zweite, leichtere Variante kann optional Phase 4 begleiten.

### Idle-Modulation während Hold (Phasen 3 und 5)
Langsame Sinus-Mod auf `yaw` (Amplitude 3°, Frequenz 0,4 Hz) — die Pose schwenkt minimal hin und her, als würde Reachy etwas mustern. Antennen bleiben statisch (die Asymmetrie ist die Aussage). Pitch und Roll halten ihre statischen Werte.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der leichte Body-Yaw-Versatz (-5 / +5°) wird durch IK glatt mit dem Head-Yaw verbunden. Asymmetrische Antennen brauchen keine IK-Behandlung, sie werden direkt gesetzt.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 3.20`, sechs Phasen.
- Die asymmetrischen Antennen sind das wichtigste Detail — `set_target_antenna_joint_positions([-15°, +25°])` (Phase 2/3) bzw. `[+25°, -15°]` (Phase 4/5). Konvention beachten: erstes Listen-Element ist linke, zweites rechte Antenne.
- Roll +20° / -20° liegen klar innerhalb des Pitch/Roll-Limits ±90°.
- Der `EASE_IN_OUT`-Easing in Phasen 2 und 4 sorgt für das organische „Nach-vorne-rollen" der Bewegung — `MIN_JERK` würde zu glatt wirken.
- Yaw-Modulation in Phasen 3 und 5 ist subtil (3° Amplitude) — sie sollte aufmerksam wirken, nicht nervös.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „neugierig" / „fragend" / „interessiert" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 3,2 ± 0,3 s
- [ ] Antennen sind in den Tilt-Phasen sichtbar asymmetrisch — Vorder- und Rück-Antenne unterschiedlich
- [ ] Roll-Tilt erreicht in Phase 2 sichtbare ±15°–20°
- [ ] Yaw-Modulation in den Hold-Phasen ist erkennbar als „mustern", nicht als „nervös"
- [ ] Phase 4 spiegelt Phase 2 sauber (vorzeichenwechsel auf Roll, Yaw, Body-Yaw, Antennen-Werte vertauscht)
- [ ] Audio (falls aktiviert) klingt fragend, nicht ausrufend

## Anti-Patterns
- Symmetrische Antennen — entwertet den Pollen-typischen Curious-Look
- Roll < ±10° — Tilt nicht mehr lesbar
- `LINEAR`- oder `CARTOON`-Easing — keine zur Geste passende Charakteristik
- Hold-Phasen ohne Idle-Mod — wirkt eingefroren, nicht musternd
- Audio mit ausrufendem Charakter (lautes „Ah!") — kollidiert mit der nachdenklichen Geste

## Offene Fragen
- Welche Antennen-Asymmetrie wirkt am stärksten als „neugierig"? -15°/+25° ist ein erster Vorschlag — empirisch zwischen 15° und 35° testen.
- Soll die Sequenz einseitig (nur Tilt links oder rechts) statt beidseitig gebaut werden? Pro: kürzer; Contra: weniger expressiv.
- Wie integriert sich `curious` mit `look_at_image()` aus dem SDK? Ein Blick mit Tilt-Bias wäre eine natürliche Zukunfts-Variante.
- Welche Audio-Datei eignet sich? Vorschlag: kurzes „Hmm?" mit ansteigendem Tonfall.
