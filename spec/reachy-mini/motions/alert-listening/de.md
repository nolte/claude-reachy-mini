# Bewegungsablauf: Aufmerksam Hörend (`alert-listening`)

Status: draft

## Kontext
Ein loop-fähiges State-Behavior, das aktive Aufmerksamkeit signalisiert: Reachy ist aufgerichtet, Antennen sind voll aufgestellt, ein leiser Yaw-Idle zeigt aktive Suche/Aufmerksamkeit. Anwendungsfälle: nach Wake-Word-Erkennung („Alexa"-artiger Trigger), während Voice-Input-Capture, „bereit für Befehl", Listening-Mode-Anzeige.

## Charakteristik
- **Loop-fähig**: läuft so lange, bis ein Stop-Signal kommt (Voice-Recording endet, Befehl erkannt)
- Antennen voll aufgerichtet (+45°) — das stärkste „Listening"-Signal
- Pitch leicht oben (+5°) — aufmerksam, nicht entspannt
- Body-Yaw zentriert
- Mini-Yaw-Idle (Suchbewegung) — der Kopf scannt subtil
- Mittleres Tempo bei Eintritt/Austritt (jeweils ~0,4 s); Loop-Body läuft kontinuierlich

## Komponenten

### Aktuator-Sequenz

| # | Phase | Dauer (s) | Head Δ (x, y, z mm; roll, pitch, yaw °) | Antennen (l°, r°) | Body-Yaw (°) | Easing | Hinweis |
|---|---|---|---|---|---|---|---|
| 1 | Eintritt (Aufmerken) | 0,30 | (0, 0, +3, 0, +5, 0) | (+45, +45) | 0 | `MIN_JERK` | erreichen der Listening-Pose |
| 2 | Loop-Body (Yaw-Idle) | variabel | (0, 0, +3, 0, +5, 0 (±8°, 0,4 Hz)) | (+45, +45) | 0 | `MIN_JERK` (Idle-Mod) | langsame Yaw-Suche |
| 3 | Austritt (Release) | 0,40 | (0, 0, 0, 0, 0, 0) | (0, 0) | 0 | `MIN_JERK` | weicher Übergang in Folge-Behavior |

Eintritt + Austritt = 0,70 s. Loop-Body Standardzeit (1 Yaw-Cycle bei 0,4 Hz) = 2,5 s. Mindestdauer (1 Cycle) ≈ 3,2 s.

### Audio (optional)
Optional ein subtiler einmaliger „Bereit"-Ton (≤ 200 ms) zu Eintritt — kein kontinuierliches Audio. Lautstärke leise (Volume 30).

### Idle-Modulation während Phase 2
Sinus-Mod auf `yaw` (Amplitude 8°, Frequenz 0,4 Hz) — subtile Yaw-Suchbewegung, scannt nach links und rechts. Pitch und Roll halten ihre Werte; Antennen bleiben auf +45°.

### Body-Yaw und IK
`automatic_body_yaw=True` empfohlen — der Body folgt dem Head-Yaw mit IK weich, was die Wirkung „aufmerksam scannend" verstärkt. Bei `False` würde der Body zu starr wirken.

## Implementierungs-Hinweise
- Bevorzugt als parametrisierte `Move`-Subklasse: `AlertListening(timeout_s=None)`. `timeout_s=None` heißt: läuft, bis `cancel_move()` aufgerufen wird; wenn ein Timeout gesetzt ist, terminiert die Phase nach Ablauf automatisch.
- `evaluate(t)` für Phase 2 berechnet `yaw = 8 * sin(2*pi*0.4*t)` — kontinuierliche Modulation.
- Antennen-Wert +45° ist nahe dem Maximum, das als „aufmerksam" liest, ohne überzogen zu wirken. Höher (+60°) wäre theoretisch möglich, aber visuell aufdringlich.
- Pitch +5° ist subtil, aber wichtig — bei Pitch 0° wirkt die Pose neutraler.

## Akzeptanzkriterien
- [ ] Außenstehende lesen die Mimik als „aufmerksam" / „lauschend" / „bereit" (mind. 4 von 5)
- [ ] Eintritt + 1 Loop-Cycle + Austritt dauern 3,2 ± 0,3 s
- [ ] Antennen sind auf +45° voll aufgerichtet
- [ ] Yaw-Idle in Phase 2 ist sichtbar als „leichtes Scannen", nicht als Ruckeln
- [ ] Behavior endet sauber bei `cancel_move()` ohne sichtbare Sprünge
- [ ] Pose wirkt aufmerksam, nicht angespannt — Pitch und Antennen sind gehalten, nicht zitternd
- [ ] Audio (falls aktiviert) ist einmalig zu Eintritt, nicht kontinuierlich

## Anti-Patterns
- Yaw-Idle-Frequenz > 1 Hz — wirkt nervös statt aufmerksam
- Antennen < +30° — die Listening-Wirkung geht verloren
- Body-Yaw-Modulation > ±5° — wirkt unsicher
- `LINEAR`- oder `CARTOON`-Easing in Eintritt/Austritt — falsche Charakteristik
- Kontinuierliches Audio (z. B. Pieps-Loop) — entwertet die ruhige Aufmerksamkeit

## Offene Fragen
- Soll bei Erkennung eines Triggers der Übergang automatisch in `recognition` oder `agreeing-nod` gehen?
- Soll die Yaw-Idle-Amplitude konfigurierbar sein (z. B. größer in „Wo bist du?"-Modus)?
- Wie integriert sich `alert-listening` mit dem LED-Ring am Mic-Modul (Standard-Listening-LED)? Vorschlag: weiches Pulsen, synchron zur Yaw-Modulation.
- Welche Audio-Datei eignet sich für den Eintritts-Ton? Vorschlag: kurzer aufsteigender Pieps oder leises „Bereit".
