# Bewegungsablauf: Wütend (`angry`)

Status: draft

## Kontext
Eine klare, aufgebrachte Geste, die als „wütend" oder „verärgert" gelesen wird: Reachy zieht kurz zurück, stößt dann scharf nach vorn, vibriert in der Drohgeste und schwenkt anschließend ruckartig nach links und rechts. Anwendungsfälle: Nachdruck bei abgelehnter Eingabe, „darf nicht!", harte Fehlermeldung, Spielmodus.

## Charakteristik
- Vorgerecktes Vorstoßen (positiver Pitch + positive X-Translation) — vermittelt „Konfrontation"
- Antennen stark zurückgelegt (negative Joint-Winkel, ~-25°) — wie Katzen-Ohren in der Drohung
- Schnelle Vibration auf Pitch im Hold — pulsierender Charakter, nicht kontinuierliche Bewegung
- Scharfe, ruckartige Body-Yaw-Schwenker mit `LINEAR`-Easing — bewusst nicht weich
- Insgesamt schnelles Tempo (~2,2 s); klar härtere Übergänge als bei `happy`

## Komponenten

### Aktuator-Sequenz

Pose-Konvention wie in `control-surface`. Werte respektieren Pitch ≤ ±90°, Body-Yaw ≤ ±160°, Stewart-Joint-Limit ±80° (effektiv enger durch IK).

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Zurückziehen) | 0,15 | (-5, 0, 0, 0, -8, 0) | (-10, -10) | 0 | `EASE_IN_OUT` | leichtes Zurücklehnen vor dem Stoß |
| 2 | Vorstoß (Drohung) | 0,20 | (+12, 0, +5, 0, +20, 0) | (-25, -25) | 0 | `LINEAR` | schneller, harter Snap nach vorn — `LINEAR` macht ihn kantig |
| 3 | Vibrations-Hold | 0,50 | (+12, 0, +5, 0, +18 (±2°), 0) | (-25, -25) | 0 | `MIN_JERK` (Idle-Mod) | pulsierende Vibration auf Pitch — siehe Idle-Modulation |
| 4 | Yaw-Schwenker links | 0,20 | (+10, 0, +5, 0, +18, -15) | (-25, -22) | -10 | `LINEAR` | scharfer Schwenk nach links |
| 5 | Yaw-Schwenker rechts | 0,20 | (+10, 0, +5, 0, +18, +15) | (-22, -25) | +10 | `LINEAR` | spiegelbildlicher Schwenk |
| 6 | Center-Snap | 0,15 | (+8, 0, +3, 0, +15, 0) | (-22, -22) | 0 | `LINEAR` | kurz zentrieren — Vorbereitung Release |
| 7 | Release | 0,60 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Zurückgleiten — Wut ebbt ab |

Gesamtdauer ≈ 2,00 s.

### Audio (optional)
Ein kurzer, harter Klangimpuls (≤ 300 ms): tiefes Knurren oder perkussives „Hmpf!". Gestartet exakt mit Phase 2 (Vorstoß). Lautstärke höher als `happy`-Sounds, bis zu Volume 70.

### Idle-Modulation während Vibrations-Hold (Phase 3)
Schnelle Sinus-Modulation auf `pitch`: Amplitude ±2°, Frequenz 8 Hz. Bei 50 Hz Daemon-Tick ergibt das ~6 Frames pro Halbwelle — innerhalb der Velocity-Grenze (8 rad/s pro Joint). Antennen vibrieren NICHT mit; die Asymmetrie (starre Antennen, vibrierender Kopf) macht den aggressiven Charakter aus.

### Body-Yaw und IK
`automatic_body_yaw=False` empfohlen — die scharfen, gegenphasigen Schwenker (Phasen 4 und 5) sollen den IK-glättenden Effekt nicht haben; der Body soll bewusst „später" als der Kopf-Yaw mitziehen. Bei aktiviertem `automatic_body_yaw` würden die Schwenker weicher wirken — was den Effekt entwertet.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.00`. `evaluate(t)` muss in Phase 3 die Sinus-Modulation implementieren — kein Verlassen auf SDK-Easing für die Vibration.
- Alternativ als `goto_target`-Folge plus eigener Idle-Loop für die Vibration. Bei Tick-Frequenz 50 Hz reicht `set_target_head_pose` mit zeitabhängigen Werten.
- `LINEAR`-Easing in Phasen 2, 4, 5, 6 ist bewusst — `MIN_JERK` würde die Härte der Bewegung kosten.
- Vor- und nach dem Behavior `automatic_body_yaw`-State explizit setzen, falls global anders eingestellt.
- Antennen-Wert -25° liegt komfortabel innerhalb des ±π-Limits.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „wütend" / „verärgert" / „streng" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,0 ± 0,2 s
- [ ] Vibration in Phase 3 ist sichtbar als „pulsierender Druck", nicht als „klappern"
- [ ] Antennen vibrieren NICHT mit — Asymmetrie zum Kopf bleibt erhalten
- [ ] Yaw-Schwenker (Phasen 4–5) wirken hart und ruckartig, nicht geschmeidig
- [ ] Audio (falls aktiviert) trifft genau auf den Vorstoß (Phase 2), nicht später
- [ ] Body-Yaw schwenkt nicht versehentlich passiv durch IK mit — es bleibt bei den dokumentierten Werten
- [ ] Release in Phase 7 fährt sauber zur Neutralpose, ohne dass Restvibration sichtbar bleibt

## Anti-Patterns
- `MIN_JERK` in den Schwenkphasen — verliert die Härte
- Antennen mit dem Kopf vibrieren lassen — wirkt wie ein elektrisches Klappern statt Drohung
- `automatic_body_yaw=True` aktiv lassen — glättet die Schwenker
- Vibrations-Frequenz > 10 Hz — schnurrt, nicht droht; auch in Konflikt mit der Velocity-Grenze
- Phase 1 (Anticipation) weglassen — der Vorstoß wirkt dann weniger lesbar
- Audio-Knurren länger als 500 ms — überdauert den Vorstoß und verwischt das Timing

## Offene Fragen
- Welche Vibrations-Frequenz wirkt am ehesten als „kontrollierte Wut" und nicht als „nervös"? Empirisch zwischen 6 und 10 Hz testen.
- Ist die Drohgeste (Phase 2 + 3) auch ohne die Yaw-Schwenker (Phasen 4–5) lesbar? Ja vermutlich — kürzere „mild-wütend"-Variante als Option.
- Wie weit darf Pitch +20° gehen, bevor die Stewart-Plattform in die Nähe ihrer Effort-Grenze kommt? Effort 10 N·m laut URDF.
- Wie reagiert ein durch HA getriggertes Behavior auf eine Wut-Reaktion in einem ruhigen Raum? Kontextabhängige Volume-Senkung empfehlen.
