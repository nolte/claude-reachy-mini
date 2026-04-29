# Bewegungsablauf: Aufgeregt (`excited`)

Status: draft

## Kontext
Eine hochenergetische Geste, die als „aufgeregt" oder „begeistert" gelesen wird: Reachy hüpft mehrfach, schwenkt zwischen den Hüpfern den Kopf, vibriert kurz im Hochpunkt und beruhigt sich dann. Anwendungsfälle: starke positive Verstärkung, Erfolgs-Feedback bei wichtigen Triggern, „großartige Neuigkeiten!", Reaktion auf Lieblings-Kommando.

## Charakteristik
- Mehrfache schnelle Hüpfer mit `CARTOON`-Easing — der federnde Charakter ist das Hauptmerkmal
- Antennen oben/außen geöffnet, gleichphasig zum Kopf vibrierend
- Yaw-Schwenker zwischen den Hüpfern — als ob Reachy die Aufregung nach allen Seiten zeigen will
- Hohes Tempo (~2,2 s); Misch-Easing aus `CARTOON` (Hüpfer) und `MIN_JERK` (Übergänge)
- Im Gegensatz zu `happy`: mehr Hüpfer, schnelleres Tempo, Yaw mit dabei

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Anticipation | 0,12 | (0, 0, -2, 0, -3, 0) | (+5, +5) | 0 | `MIN_JERK` | minimaler Krümmer vor dem ersten Hüpfer |
| 2 | Hüpfer 1 | 0,18 | (0, 0, +12, 0, +14, +5) | (+35, +35) | 0 | `CARTOON` | erster Hüpfer mit kleinem Yaw-Drall |
| 3 | Hüpfer 2 (Yaw rechts) | 0,18 | (0, 0, +10, 0, +12, +12) | (+32, +32) | +5 | `CARTOON` | leicht abwärts, dafür Yaw rechts |
| 4 | Hüpfer 3 (Yaw links) | 0,18 | (0, 0, +12, 0, +14, -12) | (+35, +35) | -5 | `CARTOON` | wieder höher, Yaw links |
| 5 | Hüpfer 4 (zentriert) | 0,18 | (0, 0, +13, 0, +15, 0) | (+38, +38) | 0 | `CARTOON` | höchster Hüpfer in der Mitte |
| 6 | Vibrations-Hold | 0,40 | (0, 0, +12 (±2 mm), 0, +13 (±2°), 0) | (+35 (±5°), +35 (±5°)) | 0 | `MIN_JERK` (Idle-Mod) | Pitch + Antennen vibrieren bei 5 Hz |
| 7 | Beruhigung | 0,40 | (0, 0, +5, 0, +5, 0) | (+15, +15) | 0 | `MIN_JERK` | Energie ebbt ab, aber noch leicht erhöht |
| 8 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weiches Auflösen zur Neutralpose |

Gesamtdauer ≈ 2,04 s.

### Audio (optional)
Eine kurze begeisterte Tonfolge (≤ 800 ms), gestartet mit Phase 2 — z. B. ein dreimaliger aufsteigender Ton, der zu den Hüpfern passt. Lautstärke moderat bis hoch (Volume 60).

### Idle-Modulation während Vibrations-Hold (Phase 6)
Sinus-Mod auf `pitch` (Amplitude 2°, Frequenz 5 Hz), `z` (Amplitude 2 mm, gleichphasig), und beide Antennen synchron (Amplitude 5°, gleichphasig zum Pitch). Im Gegensatz zu `angry` vibrieren hier die Antennen mit — die Aufregung soll als Ganzes wirken, nicht als kontrollierte Drohung.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — die Yaw-Schwenker (Phasen 3 und 4) sollen weich wirken, der Body folgt dem Kopf. Würde durch `False` zu kantig wirken (das wäre `angry`-Charakter).

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 2.04`. `evaluate(t)` verteilt acht Phasen.
- `CARTOON`-Easing in Phasen 2–5 ist der Schlüssel — die Overshoot-Charakteristik macht den federnden Eindruck.
- Pitch-Sprünge bis +15°: Velocity ~ 80°/s ≈ 1,4 rad/s, weit unter der 8-rad/s-Grenze.
- Antennen-Vibration in Phase 6 mit 5 Hz × ±5°: Velocity ~ 100°/s pro Antenne, innerhalb der Grenze.
- Yaw-Vorzeichen-Wechsel zwischen Phase 3 (+12°) und Phase 4 (-12°) ergibt eine Velocity ~ 130°/s, ebenfalls unter Limit.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „aufgeregt" / „begeistert" / „voller Energie" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 2,0 ± 0,2 s
- [ ] Mindestens vier sichtbare Hüpfer im Pitch
- [ ] `CARTOON`-Overshoot ist in den Hüpfern erkennbar (kleines „über das Ziel hinaus, zurück")
- [ ] Antennen folgen dem Pitch synchron, nicht versetzt — die ganze Reaktion ist als ein Affekt
- [ ] Vibrations-Hold (Phase 6) ist klar ausgeprägt, aber noch innerhalb von ~0,4 s
- [ ] Yaw-Schwenker zwischen den Hüpfern sind sichtbar, aber nicht zu groß (≤ ±15°)
- [ ] Beruhigung in Phase 7 ist sichtbar als „Energie weicht aus"

## Anti-Patterns
- Weniger als drei Hüpfer — wirkt wie `happy`, nicht `excited`
- `MIN_JERK` statt `CARTOON` in den Hüpfer-Phasen — der federnde Charakter geht verloren
- Asymmetrische Antennen wie bei `curious` — falsche Mimik
- Yaw-Schwenker > ±20° — wirkt überkippend, nicht aufgeregt
- Phase 6 ohne Antennen-Vibration — entwertet die Synchronität der Aufregung
- Zu langes Audio (> 1 s) — kollidiert mit der schnellen Bewegungsfolge

## Offene Fragen
- Sind vier Hüpfer das richtige Maß, oder reichen drei? Vier ist energischer; drei ist weniger anstrengend für die Stewart-Plattform.
- Welche Audio-Datei eignet sich? Vorschlag: dreimaliges aufsteigendes „Tu-tuh-tu!" oder ein Konfetti-Sample.
- Soll `excited` auch ohne Audio lesbar bleiben? Ja — Audio ist Verstärker, nicht notwendig.
- Wie reagiert die Mechanik auf vier `CARTOON`-Overshoots in 0,72 s? Empirisch — bei merklichem Servo-Wärme-Aufbau ggf. auf drei Hüpfer reduzieren.
