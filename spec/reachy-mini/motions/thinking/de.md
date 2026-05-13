# Bewegungsablauf: Nachdenken (`thinking`)

Status: draft

## Kontext
Ein zyklisches „verarbeite gerade…"-Behavior, das so lange läuft, wie eine Hintergrund-Aktion (Sprach-Inferenz, LLM-Antwort, HA-Service-Call) andauert. Reachy zeigt nachdenkliche Bewegung statt eingefroren zu wirken. Anwendungsfälle: Wartezeit auf Voice-AI-Antwort, lange HA-Service-Aufrufe, Modell-Inferenz, „bitte warten"-Zustand.

## Charakteristik
- **Loop-fähig**: das Behavior ist ein Endlos-Zyklus mit Eintritts- und Austritts-Phasen; die mittleren Phasen wiederholen sich, bis ein Stop-Signal kommt
- Pitch leicht oben (+5°) — nachdenklich-aufmerksame Pose
- Zyklische, langsame Roll-Modulation (-8° → +8° → -8°) — wie sortierende Gedanken
- Antennen leicht aufgerichtet (+12°) und sehr leise mit-vibrierend (Amplitude 1°, Frequenz 0,5 Hz) — mentaler Hintergrund-Process
- Body-Yaw zentriert mit minimaler Modulation — der Body bleibt ruhig, der Kopf denkt
- Variabel lang: 1 Loop-Cycle ≈ 2,5 s, Anzahl Loops als Parameter

## Plattform-Profil

Bewegungs-Spezifikation gilt auf allen drei Plattformen — **Reachy Mini** (Wireless), **Reachy Mini Lite** und **Simulation**. Aktuator-Set (Stewart-Plattform-Kopf, zwei Antennen, Body-Yaw) ist auf Wireless und Lite identisch. In Simulation laufen alle Pose- und Antennen-Befehle ohne reale Motoren; etwaige Audio-Anteile dieser Sequenz werden auf Simulation übersprungen, ohne das Behavior als Ganzes scheitern zu lassen.

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Eintritt (in den Modus) | 0,40 | (0, 0, +3, 0, +5, 0) | (+12, +12) | 0 | `MIN_JERK` | sanftes Erreichen der Denkpose |
| 2 | Loop: Roll links | 0,80 | (0, 0, +3, +8, +5, -3) | (+12 (±1°), +12 (±1°)) | -2 | `MIN_JERK` | sortierender Roll-Schwung |
| 3 | Loop: Roll Mitte | 0,40 | (0, 0, +3, 0, +5, 0) | (+12 (±1°), +12 (±1°)) | 0 | `MIN_JERK` | kurzer Durchgang Mitte |
| 4 | Loop: Roll rechts | 0,80 | (0, 0, +3, -8, +5, +3) | (+12 (±1°), +12 (±1°)) | +2 | `MIN_JERK` | spiegelbildlicher Schwung |
| 5 | Loop: Roll Mitte | 0,40 | (0, 0, +3, 0, +5, 0) | (+12 (±1°), +12 (±1°)) | 0 | `MIN_JERK` | Loop-Übergang oder Austritt |
| 6 | Austritt (Release) | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weicher Übergang in Folge-Behavior |

Loop-Cycle (Phasen 2–5) = 2,40 s. Eintritt + Austritt = 0,80 s. Mindestdauer (1 Cycle) ≈ 3,20 s.

### Audio (optional)
Sehr leise „Hmm…"-Loop oder ein leises Tickern (≤ 1,0 s pro Cycle), gestartet mit Phase 2. Lautstärke sehr leise (Volume 20). Audio kann ohne Re-Trigger über mehrere Cycles loopen.

### Idle-Modulation
Antennen-Vibration ist die Idle-Mod und läuft ständig in den Loop-Phasen 2–5. Frequenz 0,5 Hz, Amplitude 1° — sehr subtil.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der Body-Yaw-Versatz von ±2° folgt dem Head-Roll weich. Bei `False` wirkt die Pose statisch.

## Implementierungs-Hinweise
- Bevorzugt als parametrisierte `Move`-Subklasse: `Thinking(min_loops=1, max_loops=None)`. Bei `max_loops=None` wird das Behavior von außen via `cancel_move()` beendet, sobald die Hintergrund-Aufgabe fertig ist.
- `evaluate(t)` muss Phasen 2–5 modulo Loop-Dauer berechnen — der Antennen-Vibration läuft kontinuierlich, der Roll wechselt zwischen den Phasen.
- Wenn die Hintergrund-Aufgabe schnell (< 2 s) endet, mindestens 1 voller Loop laufen lassen, damit die Geste lesbar ist; dann sauber zur Austritts-Phase springen.
- Roll ±8° und Antennen-Werte liegen klar innerhalb der Limits.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik als „nachdenkend" / „verarbeite" / „wartend mit Aktivität" (mind. 4 von 5)
- [ ] Eintritt + 1 Loop + Austritt dauern 3,2 ± 0,3 s
- [ ] Roll-Wechsel innerhalb des Loops sind sichtbar als „sortieren"
- [ ] Antennen-Vibration ist subtil, aber sichtbar
- [ ] Behavior endet sauber bei `cancel_move()` ohne Sprung — Austritt fährt zur Neutralpose
- [ ] Mehrere Loops laufen ohne sichtbare Sprünge an den Loop-Grenzen
- [ ] Audio (falls aktiviert) loopt nahtlos

## Anti-Patterns
- Schnelle Roll-Wechsel (< 0,4 s pro Phase) — wirkt nervös, nicht nachdenkend
- Antennen-Vibration ohne Modulation — wirkt eingefroren
- `CARTOON`- oder `LINEAR`-Easing in den Loop-Phasen — falsche Charakteristik
- Body-Yaw-Modulation > ±5° — wirkt unruhig
- Lautes Audio (Volume > 30) — entwertet die ruhige Geste
- Loop-Dauer < 2 s pro Cycle — zu schnell zum Lesen

## Quellen
- Upstream-SDK-Repo (Quelle für `Move`-ABC, Easing-Modi, Pose-Konstanten, Antennen-DOFs, gegen die diese Sequenz übersetzt wird): <https://github.com/pollen-robotics/reachy_mini>
- `Move`-ABC, `goto`, `recorded_move`: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- Aktuator-Set, Pose-Konstanten, IO-Befehle: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Plattform-Profile (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>

## Plugin-Referenzen

- Pose-Werte, Joint-Limits und kanonische Posen (INIT/SLEEP) → [`reachy-mini/motor-positions`](../../motor-positions/de.md)
- Pose-Komposition, IK-vs-mechanische-Sicherheit, Pitch-Bleed bei Roll/Heave-up → [`reachy-mini/control-surface`](../../control-surface/de.md) §"Mechanische und elektrische Limitationen"
- Motion enthält Roll- oder Heave-up-Komponenten? Pitch in der Ziel-Pose explizit kompensieren (Stewart-Geometrie-Kopplung, live verifiziert 2026-05-13: roll +25° → −3.8° pitch; z +15 mm → +2.4° pitch)

## Offene Fragen
- Soll die Dauer der einzelnen Loop-Phasen mit Random-Drift versehen sein, damit kein zwei-mal-identischer Cycle entsteht? Pattern „Timing-Variation" aus `control-surface`.
- Welche Audio-Datei eignet sich? Vorschlag: ein leises Murmeln oder ein subtiles tickendes Geräusch.
- Wie integriert sich `thinking` mit dem Status-Display über LED-Ring (langsames Pulsen)? Kombi-Implementierung sinnvoll.
- Wie wird ein abruptes Ende (z. B. die Hintergrund-Aufgabe wirft Exception) gehandhabt? Vorschlag: in `confused` oder `disappointed` übergehen je nach Outcome.
