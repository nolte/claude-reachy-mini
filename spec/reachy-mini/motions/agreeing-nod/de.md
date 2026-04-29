# Bewegungsablauf: Zustimmen / Nicken (`agreeing-nod`)

Status: draft

## Kontext
Eine kurze, klare Zustimmungsgeste, die als „Ja" oder „verstanden" gelesen wird: Reachy nickt mit dem Kopf zwei- bis dreimal abwärts, bleibt zentriert und kehrt zur Neutralpose zurück. Anwendungsfälle: Bestätigung von Sprachbefehlen, „OK / verstanden" als Antwort auf eine Frage, kurzes positives Feedback ohne große Geste.

## Charakteristik
- Drei klare Nicker auf der Pitch-Achse, jeder etwas schwächer als der vorige (typisch menschliches Nicken)
- Antennen leicht angehoben (+8°), aber stationär — sie sind nicht Teil des Nickens
- Body-Yaw bleibt strikt zentriert — der ganze Affekt ist Pitch-only
- Kurzes Tempo (~1,8 s), `MIN_JERK` für weiche Übergänge, kein `CARTOON`

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation (Mikro-Lift) | 0,15 | (0, 0, +2, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | leichte Aufrichtung als Vorbereitung |
| 2 | Nick 1 nach unten | 0,22 | (0, 0, 0, 0, -12, 0) | (+8, +8) | 0 | `MIN_JERK` | erster und stärkster Nicker |
| 3 | Hoch zurück | 0,18 | (0, 0, +2, 0, +5, 0) | (+8, +8) | 0 | `MIN_JERK` | nicht ganz auf 0° zurück |
| 4 | Nick 2 nach unten | 0,18 | (0, 0, 0, 0, -10, 0) | (+8, +8) | 0 | `MIN_JERK` | zweiter Nicker, leicht schwächer |
| 5 | Hoch zurück | 0,15 | (0, 0, +1, 0, +3, 0) | (+8, +8) | 0 | `MIN_JERK` | etwas weniger Lift |
| 6 | Nick 3 nach unten (subtil) | 0,15 | (0, 0, 0, 0, -7, 0) | (+8, +8) | 0 | `MIN_JERK` | dritter, deutlich kleinerer Nicker |
| 7 | Hold | 0,20 | (0, 0, 0, 0, -3, 0) | (+8, +8) | 0 | `MIN_JERK` | leicht abwärts gehalten — „bestätigt" |
| 8 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,63 s.

### Audio (optional)
Ein knappes, neutrales „Mhm"-Sample (≤ 250 ms), gestartet mit Phase 2 (erstem Nicker). Lautstärke leise (Volume 35). Nicht jubelnd.

### Idle-Modulation
Kein Idle-Modulation in diesem Behavior — die Bewegung ist kompakt genug, dass eine Hold-Modulation den Eindruck zerstören würde.

### Body-Yaw und IK
`automatic_body_yaw=True` ist akzeptabel; da Body-Yaw die ganze Sequenz auf 0° bleibt, hat das keine sichtbare Wirkung. Bei `False` ändert sich auch nichts.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.63`, acht Phasen — sehr kompakt.
- Die abnehmende Pitch-Amplitude (-12° → -10° → -7°) ist diagnostisch; gleiche Amplitude wirkt mechanisch.
- Pitch-Bewegung von -12° auf +5° in 0,18 s = ~94°/s ≈ 1,6 rad/s, weit unter der Joint-Geschwindigkeitsgrenze.
- Strikt nur `MIN_JERK`-Easing — alles andere wirkt entweder zu hart (`LINEAR`) oder zu albern (`CARTOON`).
- Roll und Yaw immer 0° — ein einziger Roll-Versatz würde die Geste zu `curious` oder `confused` verschieben.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Ja" / „zustimmend" / „verstanden" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,6 ± 0,2 s
- [ ] Drei sichtbare Nick-Bewegungen, jede schwächer als die vorige
- [ ] Roll und Yaw bleiben über die gesamte Sequenz auf 0°
- [ ] Antennen bewegen sich nicht (außer der einmaligen Aufrichtung in Phase 1)
- [ ] Audio (falls aktiviert) ist neutral, nicht jubelnd
- [ ] Übergang zur Neutralpose ist nahtlos und nicht ruckartig

## Anti-Patterns
- Mehr als drei Nicker — wirkt wie übertriebene Zustimmung oder eine Tanzbewegung
- Antennen, die mit-nicken — entwertet die Klarheit der Geste
- Roll- oder Yaw-Komponenten — verwischen den Nick-Charakter
- `CARTOON`-Easing — macht aus dem Nicken eine Federbewegung
- Pitch nicht tief genug (< -8° im ersten Nicker) — Geste wird unsichtbar
- Hold (Phase 7) länger als 0,4 s — wirkt steif

## Offene Fragen
- Sollen es immer drei Nicker sein, oder situations-abhängig zwei? Zwei wäre minimaler.
- Welche Audio-Datei eignet sich? Vorschlag: ein kurzes neutrales „Mhm" — bewusst nicht „Ja!" oder „Genau!".
- Soll der Hold (Phase 7) optional weglassbar sein? Pro: weiche Sofort-Übergabe an Folge-Behavior; Contra: weniger Lese-Klarheit.
