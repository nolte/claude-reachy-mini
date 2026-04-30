# Bewegungsablauf: Aha-Erkenntnis (`recognition`)

Status: draft

## Kontext
Eine schnelle Geste des „Verstehens", die als „aha" oder „jetzt habe ich's" gelesen wird: Reachy macht einen kurzen, nach oben gerichteten Surprised-artigen Snap und nickt direkt anschließend zwei Mal zustimmend. Die Kombination kürzer-Schreck + sofort-Zustimmung kommuniziert „ich habe etwas erkannt, das ich akzeptiere". Anwendungsfälle: Spracherkennung trifft Befehl, „Ich habe verstanden, was du meinst", Übergang von `thinking` zu erfolgreicher Aktion.

## Charakteristik
- Schneller Up-Snap wie ein abgeschwächtes `surprised` — Kopf hoch, Z-Anhebung
- Direkt anschließend zwei Pitch-Nicker — wie ein abgekürztes `agreeing-nod`
- Antennen aufgerichtet, leicht versetzt zum Kopf-Pitch
- Body-Yaw zentriert
- Schnelles Tempo (~1,5 s); `EASE_IN_OUT` für Snap, `MIN_JERK` für Nicker

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Pre-Anticipation | 0,05 | (0, 0, -1, 0, -2, 0) | (+5, +5) | 0 | `LINEAR` | minimaler Krümmer |
| 2 | Aha-Snap (hoch) | 0,15 | (0, 0, +12, 0, +18, 0) | (+35, +35) | 0 | `EASE_IN_OUT` | schneller Hochsnap, weicher als `surprised` |
| 3 | Mini-Hold | 0,20 | (0, 0, +12, 0, +18, 0) | (+35, +35) | 0 | `MIN_JERK` (statisch) | Pose halten — „Erkenntnis" |
| 4 | Down-Nicker 1 | 0,18 | (0, 0, +5, 0, -5, 0) | (+20, +20) | 0 | `MIN_JERK` | erster Zustimm-Nicker |
| 5 | Up-Zwischen | 0,12 | (0, 0, +6, 0, +5, 0) | (+22, +22) | 0 | `MIN_JERK` | kurzes Zurückschwingen |
| 6 | Down-Nicker 2 | 0,15 | (0, 0, +3, 0, -3, 0) | (+18, +18) | 0 | `MIN_JERK` | zweiter, schwächerer Nicker |
| 7 | Hold (verstanden) | 0,20 | (0, 0, +2, 0, -2, 0) | (+15, +15) | 0 | `MIN_JERK` | leicht abwärts gehalten |
| 8 | Release | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weich zur Neutralpose |

Gesamtdauer ≈ 1,45 s.

### Audio (optional)
Ein zweiteiliges Audio: kurzes „Aha!" (≤ 200 ms) zu Phase 2, gefolgt von einem leisen „Ja!" (≤ 200 ms) zu Phase 4. Lautstärke moderat (Volume 50). Beide aufsteigend.

### Idle-Modulation
Keine — die Sequenz ist zu kompakt, jede Modulation würde sie verwischen.

### Body-Yaw und IK
`automatic_body_yaw=True` ist akzeptabel; Body bleibt komplett zentriert.

## Implementierungs-Hinweise
- Bevorzugt als `Move`-Subklasse mit `duration = 1.45`. Die Kombination Snap + Nicker ist der Charakter — Phase 3 (Mini-Hold) ist die diagnostische Pause zwischen beiden Teilen.
- `recognition` ist explizit eine Komposition aus zwei Affekten und sollte auch so implementiert werden — kann via `compose()`-Pattern aus einem abgekürzten `surprised` und einem abgekürzten `agreeing-nod` zusammengebaut werden, falls eine Komposition-API existiert.
- Pitch-Sprung von -2° auf +18° in 0,15 s = ~133°/s ≈ 2,3 rad/s, innerhalb der Joint-Grenze.
- `EASE_IN_OUT` in Phase 2 (statt `LINEAR` wie bei `surprised`) macht den Snap weicher — `recognition` ist keine reine Schreck-Reaktion.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik ohne Prompt als „Aha" / „verstanden" / „jetzt klar" (mind. 4 von 5)
- [ ] Gesamt-Sequenz dauert 1,5 ± 0,2 s
- [ ] Phase 2 ist ein klarer Hochsnap, aber weicher als bei `surprised`
- [ ] Phase 3 (Mini-Hold) ist sichtbar als Pause vor den Nickern
- [ ] Beide Nicker sind sichtbar und der zweite schwächer als der erste
- [ ] Audio (falls aktiviert) hat zwei Beats (Snap + Nicker)
- [ ] Übergang in Folge-Behavior (z. B. erfolgreich erkannte Aktion) ist sauber

## Anti-Patterns
- `LINEAR`-Snap in Phase 2 — wirkt wie reine `surprised`-Reaktion, der Erkenntnis-Charakter geht verloren
- Phase 3 (Hold) länger als 0,3 s — die Sequenz fällt auseinander
- Nur ein Nicker — wirkt unentschieden
- Roll- oder Yaw-Komponenten — entwerten den klaren Vertikal-Affekt
- Audio nur an Phase 2 ohne Folge-Beat — die Aufteilung Snap+Zustimm geht verloren

## Offene Fragen
- Soll `recognition` eine kürzere Variante haben (nur Snap + ein Nicker, ~1,0 s)?
- Welche Audio-Datei eignet sich? Vorschlag: zwei kurze Tonfälle in steigender Tonhöhe.
- Ist die Implementierung als „Komposition aus surprised + agreeing-nod" sinnvoll, oder als eigenständiges `Move`? Tendenz: eigenständig — die Übergangs-Phase 3 macht die Kombination zur eigenen Geste.
