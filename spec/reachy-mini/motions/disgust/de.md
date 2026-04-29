# Bewegungsablauf: Ekel (`disgust`)

Status: draft

## Kontext
Die sechste Ekman-Basisemotion: ein abwehrendes Wegdrehen mit kleinem Schauer. Reachy zieht sich leicht zurück, dreht den Kopf zur Seite, neigt ihn ekel-typisch leicht und lässt einen kurzen Schauer über die Pose laufen. Anwendungsfälle: „Igitt!", Rückmeldung auf unangenehmen Sensor-Input (z. B. starker Geruch via HA-Luftqualitäts-Sensor), spielerische Reaktion auf falsche Eingabe.

## Charakteristik
- X-Translation leicht negativ (-5 mm) — Zurückziehen
- Yaw zur Seite (+25°) — Wegdrehen
- Roll +6° — typische Ekel-Pose-Komponente (leichte Schräglage)
- Pitch leicht zurück (-3°) — abwehrend, nicht traurig
- Antennen flach (-15°) — angelegt, abwehrend
- Mini-Schauer in Phase 4 — vibration auf roll
- Schnelles Tempo (~1,5 s)

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Mini-Krümmer) | 0,10 | (+1, 0, 0, 0, +1, 0) | (-3, -3) | 0 | `LINEAR` | sehr klein, kaum sichtbar |
| 2 | Ekel-Reaktion | 0,30 | (-5, 0, -2, +6, -3, +25) | (-15, -15) | +5 | `EASE_IN_OUT` | Hauptbewegung — Wegdrehen mit Roll |
| 3 | Hold | 0,40 | (-5, 0, -2, +6, -3, +25) | (-15, -15) | +5 | `MIN_JERK` (statisch) | Pose halten |
| 4 | Mini-Schauer | 0,20 | (-5, 0, -2, +6 (±2°, 8 Hz), -3, +25) | (-15, -15) | +5 | `MIN_JERK` (Idle-Mod) | kleine Roll-Vibration — „pfui" |
| 5 | Release | 0,50 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,50 s.

### Audio (optional)
Ein kurzer abgekauter „Igitt"-Ton oder „Bä!"-Sample (≤ 250 ms), gestartet mit Phase 2. Lautstärke moderat (Volume 45).

### Idle-Modulation während Phase 4
Sinus-Mod auf `roll`: Amplitude 2°, Frequenz 8 Hz — kurzer Schauer, ähnlich zur `angry`-Vibration aber kürzer und auf Roll statt Pitch. 4 Schauer-Cycles in 0,2 s.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der leichte Body-Yaw (+5°) folgt dem Head-Yaw (+25°) durch IK weich. Relative Yaw +20°, weit innerhalb der ±65°-Grenze.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.50`, fünf Phasen.
- Roll-Schauer-Frequenz 8 Hz × ±2° = 32°/s ≈ 0,56 rad/s — weit innerhalb der 8 rad/s-Grenze.
- Pitch -3° (nur leicht negativ) ist diagnostisch — bei tieferem Pitch wirkt es wie `disappointed`.
- Roll +6° ist die Ekel-typische Komponente und sollte nicht weggelassen werden — sie unterscheidet `disgust` von `disagreeing-shake`.
- Variante: gespiegelte Version (Roll -6°, Yaw -25°) — Ekel zur anderen Seite.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Ekel" / „igitt" / „abgewiesen" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,5 ± 0,2 s
- [ ] Roll-Komponente +6° ist sichtbar
- [ ] Schauer in Phase 4 ist als kurze Vibration erkennbar
- [ ] Antennen flach (-15°) während Phasen 2–4
- [ ] Audio (falls aktiviert) hat einen abgewiesenen Charakter

## Anti-Patterns
- Pitch tiefer als -10° — wird zu `disappointed`
- Schauer-Frequenz < 5 Hz — wirkt zu langsam
- Antennen aufgerichtet — falsche Verbindung
- Roll = 0° — verliert den ekel-typischen Schräglage-Aspekt
- Gesamt-Dauer > 2 s — entwertet die spontane Reaktion
- `LINEAR`-Easing in Phase 2 — wirkt aggressiv, nicht abwehrend

## Offene Fragen
- Sollen es zwei Schauer (in Phase 4) oder nur einer sein? Mehrere Schauer wirken stärker, aber riskieren in „nervös" zu kippen.
- Welche Audio-Datei eignet sich? Vorschlag: kurzes abgekautes „Bä!" oder „Igitt".
- Wie unterscheidet sich `disgust` praktisch von `shy`? `shy` ist sozial-zurückhaltend, `disgust` ist sensorisch-abwehrend; der Roll-Schauer ist diagnostisch für `disgust`.
- Soll die Wegdreh-Richtung Random links/rechts sein, oder vom Trigger-Kontext abhängig?
