# Erkennung untypischer Bewegungsmuster am Reachy Mini

Status: draft

## Kontext

Wer eine Bewegung am Reachy Mini auslöst, muss vier Klassen untypischen Verhaltens vermeiden, bevor sie das Gerät erreichen, erkennen, während sie laufen, und nachvollziehen, wenn sie passiert sind: **Kopf-gegen-Körper-Selbstkollision** (am 2026-05-12 real beobachtet), **ruckartige Bewegungen**, die mechanisch in die Stewart-Aktuatoren schlagen, das **„Wacken"** der Antennen-Servos im Totband um `0°`, und **Stewart-Limit-Knocker** mit unlösbarem Inverse-Kinematik-Aufruf. Die Erkennungs-Logik dafür lebt aktuell verstreut: motor-positions Schicht 4 nennt die Konflikt-Konstellationen, control-surface die mechanischen Limits und das Antennen-Totband, app-logging die Triage-Klassen, und einzelne Skills haben jeweils eigene Ad-hoc-Checks. Diese Spec zentralisiert die Erkennungs-Methodologie, damit alle Konsumenten (Skills, Agents, später ein eigener Validator) dieselben Regeln teilen.

**Zielgruppe.** Skill- und Agent-Autoren der vier genannten Konsumenten-Skills (`reachy-mini-sdk`, `reachy-mini-inspect`, `app-log-triage`, `dance-choreography`) sowie Implementierer eines zukünftigen dedizierten `motion-validator`-Skills.

Die Spec wird konsumiert vom [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md)-Skill bei der Pre-Flight-Validierung von Code-Snippets, vom [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/de.md)-Skill bei Live-Telemetrie-Reads, vom [`app-log-triage`](../../claude/app-log-triage/de.md)-Skill bei der Post-hoc-Analyse von App-Logs und vom [`dance-choreography`](../../claude/dance-choreography/de.md)-Skill bei der Komposition extremer Posen-Folgen.

Wichtiger Befund vorweg: das Pollen-SDK macht **keinen Self-Collision-Check** (`engine: AnalyticalKinematics, collision check: false` aus `GET /api/kinematics/info`, siehe [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 2). Eine Pose, die das IK-Polytop akzeptiert, ist *kinematisch erreichbar* — nicht automatisch *mechanisch unbedenklich*. Die Erkennungs-Methodologie dieser Spec füllt genau diese Lücke, bis Pollen sie SDK-seitig schließt.

Verifikations-Basis: Reachy Mini Wireless, Firmware 1.7.1, live verifiziert am **2026-05-12** (Phase-B-Selbstkollisions-Vorfall) und **2026-05-13** (T1–T8 Bewegungs-Verifikation, Antennen-Totband-Messungen). Die Lite-Plattform ist explizit **noch nicht** verifiziert — siehe Offene Fragen.

## Ziele

- Drei orthogonale Erkennungs-Phasen sauber trennen: **Pre-Flight** (vor dem Befehl an den Daemon), **Live** (Telemetrie-Monitor während der Bewegung), **Post-hoc** (Log- und Telemetrie-Replay)
- Vier verbindliche Anomalie-Klassen katalogisieren — Kopf-Körper-Selbstkollision, ruckartige Bewegung, Antennen-Wacken, Stewart-Limit-Knocker — jede mit Härtegrad, bindender Regel und mindestens einem konkreten Detect-Signal pro Phase
- Pro Klasse die Recovery-Strategie nennen, falls die Anomalie nicht verhindert werden konnte (Power-Cycle, Pose-Korrektur, Telemetrie-Aufzeichnung)
- Cross-Links auf die kanonischen Wert-Quellen halten (motor-positions, control-surface, daemon-rest-api, app-logging), statt Werte zu duplizieren
- Ein verbindliches einheitliches Anomalie-Event-Record-Schema festlegen, damit konsumierende Skills strukturiert melden können und Aggregation über Plattformen hinweg möglich ist
- Hardware-Verifikations-Status pro Klasse explizit halten (Wireless 1.7.1 verifiziert, Lite noch nicht), damit eine spätere Lite-Verifikation gezielt nachgeführt werden kann

## Nicht-Ziele

- Implementierung der Erkennung in einem konkreten Skill oder Agent — diese Spec ist normative Wissensbasis, kein Operations-Skript. Die konsumierende Logik landet in den genannten Konsumenten-Skills bzw. einem späteren `motion-validator`-Skill
- Self-Collision-Algorithmus mit URDF-Mesh-Check oder Pose-Sampling-Solver — explizit nicht im Pollen-SDK und nicht in dieser Spec. Bis ein solcher Solver verfügbar ist, bleibt die Spec auf Pose-Range-Bounds (Pollen-Nominal) als binding layer
- Generelle Sicherheits-Architektur: Notstopp-Pfade, Brown-out-Schutz, Thermisches Budget, Cool-down — gehört zu [`reachy-mini/control-surface`](../control-surface/de.md) §"Sicherheits-Limits"
- Konkrete Bewegungs-Komposition (Easing, Anticipation, Beat-Sync) — gehört zu [`reachy-mini/control-surface`](../control-surface/de.md) §"Patterns für natürliche, flüssige Bewegung" und zum [`dance-choreography`](../../claude/dance-choreography/de.md)-Skill
- Werte für die Simulations-Variante, wenn sie von der Hardware abweichen — die Sim teilt die IK- und URDF-Limits, aber Phänomene wie Antennen-Totband und Stewart-Effort-Anschlag sind sim-irrelevant
- Hardware-Bringup, Kalibrierung, Firmware-Flash, IMU-Recovery — eigene Skills (geplant), nicht Teil dieser Spec
- Audio-, Vision- oder LED-Anomalien — andere Subsysteme

## Anforderungen

### Übersicht der Anomalie-Klassen

| Klasse | Name | Härtegrad | Bindende Regel | Primärer Detect-Pfad | Verifiziert |
|---|---|---|---|---|---|
| **A** | Kopf-gegen-Körper-Selbstkollision | hart | **MUSS NICHT** auftreten | Pre-Flight Pose-Range-Check | Wireless 1.7.1, 2026-05-12 |
| **B** | Ruckartige Bewegung (excessive Joint-Velocity) | weich/mittel | **SOLLTE NICHT** (>0,16 rad/sample) / **MUSS NICHT** (>0,30 rad/sample) | Pre-Flight Pose-Delta/dt | Wireless 1.7.1, partiell (siehe OQ2) |
| **C** | Antennen-„Wacken" (Mikro-Oszillation im Totband) | weich | **SOLLTE** vermieden werden | Pre-Flight Antennen-Setpoint | Wireless 1.7.1, 2026-05-13 |
| **D** | Stewart-Limit-Knocker + IK-unlösbar | mittel | **MUSS NICHT** versucht werden | Pre-Flight IK-Bisection | Wireless 1.7.1, 2026-05-13 |

### Klasse A — Kopf-gegen-Körper-Selbstkollision

**Was passiert.** Eine Pose-Target außerhalb der Pollen-Nominal-Operations-Range (±40° Pitch/Roll, ±60° Head-Yaw) ist *IK-mathematisch lösbar*, aber *mechanisch nicht sicher*. Der Daemon sendet die Bewegung; der Stewart-Mechanismus drückt den Kopf gegen den Körper-Korpus. Nach dem Vorfall meldet der Daemon `backend_status.ready: false`, `head_joints: null`, und `POST /api/motors/set_mode/gravity_compensation` antwortet mit HTTP 500. Kanonischer Präzedenzfall: Phase-B-Live-Vorfall **2026-05-12** an einem Reachy Wireless 1.7.1 ([`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 4).

**Bindende Regel.** Eine Live-Bewegung **MUSS** sich innerhalb der **Pollen-Nominal-Range** halten (±40° Pitch/Roll, ±60° Head-Yaw, ±155° Body-Yaw). Das IK-Polytop und die URDF-mechanischen-Limits sind als binding layer **nicht ausreichend**. Diese Regel ist die innerste der drei Validitäts-Schichten aus [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 2 §"Drei Schichten der Validität".

**Detect-Signale.**

| Phase | Signal | Kostendisposition |
|---|---|---|
| Pre-Flight | Soll-Pose außerhalb (±40°, ±40°, ±60°, ±155°) ⇒ **MUSS** abgelehnt werden | trivial, statisch |
| Live | `GET /api/daemon/status.backend_status.ready` flippt nach Pose-Befehl auf `false`, gleichzeitig `head_joints: null` und `state/full` liefert keinen aktuellen Head-Pose-Snapshot | Polling-Kost; Recovery-relevant |
| Post-hoc | App-Log enthält `ConnectionError: Could not connect to daemon on localhost` (siehe [`reachy-mini/app-logging`](../app-logging/de.md) Triage-Klasse `daemon-stale-state`) oder Daemon-Log zeigt `backend.ready: false` direkt nach `start-app` | Replay-Tauglich |

**Recovery.** Power-Cycle des Reachy. Der Daemon hat keinen Soft-Recovery-Pfad für diese Klasse, weil der Sensor-Backend-Status hängt. Vor dem nächsten Live-Versuch sollte die fehlerhafte Pose-Range im konsumierenden Skript korrigiert werden.

### Klasse B — Ruckartige Bewegung (excessive Joint-Velocity)

**Was passiert.** Der Daemon klippt eine Joint-Velocity-Anforderung beim URDF-Limit (8 rad/s pro Stewart-Joint), aber dieser Clip ist *zu grob* — er verhindert nur die absolute Spitze, nicht eine Sequenz von Pose-Targets, die in Summe einen mechanisch belastenden Ruck erzeugen. Symptome: hörbares Servo-Klacken, sichtbares Plattform-Vibrieren, in extremen Fällen Stewart-Arm-Resonanz. Mechanische Folge: Verkürzung der Servo-Lebensdauer, Plattform-Kalibrierung kann driften.

**Bindende Regel.** Eine Trajektorie **SOLLTE NICHT** zwischen zwei konsekutiven Pose-Befehlen einen Joint-Vektor-Diff erzeugen, der bei der Befehls-Frequenz das URDF-Velocity-Limit überschreitet. Praktisch: für die typische 50-Hz-Loop des Pollen-Daemons heißt das ein Joint-Delta pro Sample von höchstens `8 rad/s ÷ 50 Hz = 0.16 rad ≈ 9.2°`.

**Detect-Signale.**

| Phase | Signal | Kostendisposition |
|---|---|---|
| Pre-Flight | Pose-zu-Pose-Joint-Diff pro Befehl-Intervall > 0.16 rad (=9.2°) auf einem aktiven Stewart-Joint ⇒ Warnung; > 0.30 rad ⇒ Ablehnung | trivial, statisch wenn Trajektorie vorher bekannt |
| Live | `GET /api/state/full?with_head_joints=true` zwei konsekutive Reads, Berechnung von ‖Δjoints‖₂ pro Δt; oder `backend_status.nb_error` Spike | Polling-Kost; nb_error ist autoritativer |
| Post-hoc | Telemetrie-Replay: pro Stewart-Joint die maximale Sample-zu-Sample-Differenz; Smoothness-Score (mean von ‖Δjoints‖₂ pro Sekunde) | Replay-Tauglich, statisch |

**Recovery.** Eine reine Class-B-Anomalie führt selten zum Daemon-Hang; der Servo-Stress ist die primäre Folge. Korrektur durch sanfteres Easing oder niedrigere Befehls-Frequenz im Skript.

### Klasse C — Antennen-„Wacken" (Mikro-Oszillation im Totband)

**Was passiert.** Jeder Antennen-Servo (XL330-M077-T) zeigt ein vorzeichen-asymmetrisches Totband um `0°`, in dem der Soll-Wert nicht stabil gehalten wird: das Servo oszilliert ~±0,5° Peak-to-Peak. Empirisch verifiziert auf einem Wireless 1.7.1 (**2026-05-13**): Soll-Werte ab `|x| ≥ 15°` hielten felsenfest (stdev = 0,000° über mehrere Sekunden); im Bereich `|x| < 5°` wackelte mindestens eine der beiden Antennen sichtbar. Quelle: [`reachy-mini/control-surface`](../control-surface/de.md) §"Mechanische und elektrische Limitationen". Die SDK-eigene Konstante `INIT_ANTENNAS_JOINT_POSITIONS = [-10°, +10°]` ist eine implizite Anerkennung derselben Eigenschaft.

**Bindende Regel.** Antennen-Ruhe-Soll-Werte **SOLLTEN** bei `|setpoint| ≥ 5°` gehalten werden, idealerweise `≥ 10°` zur Übereinstimmung mit `INIT_ANTENNAS_JOINT_POSITIONS`. Eine Bewegung **SOLLTE NICHT** am Ende in eine Allnull-Antennen-Pose easen, wenn der Roboter danach im Idle steht — mindestens eine Antenne wackelt dann sichtbar.

**Detect-Signale.**

| Phase | Signal | Kostendisposition |
|---|---|---|
| Pre-Flight | Antennen-Setpoint im Rest-Frame mit `\|x\| < 5°` ⇒ Warnung; `\|x\| < 2°` ⇒ Ablehnung | trivial, statisch |
| Live | Standard-Abweichung der Antennen-Joint-Reads über ein 2-Sekunden-Rolling-Window > 0,2° trotz stabilem Soll-Wert ⇒ Antenne im Totband | Polling-Kost; braucht Telemetrie-Buffer |
| Post-hoc | Telemetrie-Replay: pro Antenne der Anteil der Samples mit `\|setpoint\| < 5°`; Stdev pro Setpoint-Bin | Replay-Tauglich |

**Recovery.** Class C beschädigt nichts und blockiert keine Folge-Bewegung. Sie ist primär ein Qualitäts-Befund („das wirkt nervös, obwohl der Roboter steht"). Korrektur: Antennen-Ruhe-Posen auf `≥ 10°` setzen.

### Klasse D — Stewart-Limit-Knocker + IK-unlösbar

**Was passiert.** Eine Pose-Target erfordert einen Stewart-Joint außerhalb seines URDF-Limits (z. B. `stewart_1 > +80°` oder `stewart_4 < −80°`, siehe [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 1). Der `AnalyticalKinematics.ik`-Solver wirft eine Exception, der Daemon antwortet mit HTTP 4xx oder 5xx, je nach Pfad. In Live-Telemetrie steigt `backend_status.nb_error`, und `head_joints` zeigt entweder den letzten gültigen Wert oder `null` (siehe `_status.ready`-Desync aus [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 5). Die Klasse ist mittlere Härte, weil sie keine mechanische Selbstkollision erzeugt, aber den IK-Solver in einen unklaren Folgestand bringen kann.

**Bindende Regel.** Eine Pose-Target, deren IK-Lösung außerhalb der URDF-Joint-Limits liegt, **MUSS NICHT** an den Daemon geschickt werden. Pre-Flight ist hier deutlich billiger als Live-Recovery.

**Detect-Signale.**

| Phase | Signal | Kostendisposition |
|---|---|---|
| Pre-Flight | Lokaler IK-Aufruf (z. B. `reachy_mini==1.7.2` in einem Helper-Venv) mit dem URDF-Limit-Tupel als Stop-Kriterium; Result außerhalb ⇒ Ablehnung | Modul-Aufruf-Kost; pinnt lokale SDK-Version |
| Live | `backend_status.nb_error` Spike (`> 0` und steigend) nach Pose-Befehl; `head_joints` vs. soll-Joints Diff > URDF-Limit-Toleranz | Polling-Kost; nb_error ist autoritativer |
| Post-hoc | App-Log zeigt `kinematics`-Exception-Traceback oder Daemon-Log zeigt eine `ik_failed`-Markierung; in App-Logs ist das eine Klasse `kinematics-unsolvable` (siehe [`reachy-mini/app-logging`](../app-logging/de.md), ggf. nachzupflegen) | Replay-Tauglich |

**Recovery.** Pose-Target auf eine Schicht-2-/Schicht-3-Validität bringen (URDF-Limits respektieren) und erneut senden. Class D braucht keinen Power-Cycle, sofern der IK-Solver sauber abbricht.

### Phase 1 — Pre-Flight (Erkennung vor dem Befehl)

Verpflichtende Checks vor jedem Pose- oder Trajektorien-Befehl an den Daemon:

- **MUSS [MUST]** die Soll-Pose-Range gegen die Pollen-Nominal-Operations-Range prüfen (Klasse A); außerhalb ⇒ Ablehnung mit Klassen-A-Markierung
- **MUSS [MUST]** für Trajektorien (mehrere konsekutive Pose-Targets) den Joint-Vektor-Diff pro Befehls-Intervall berechnen und gegen `0.16 rad / sample` prüfen (Klasse B); überschritten ⇒ Warnung bzw. Ablehnung
- **SOLLTE [SHOULD]** Antennen-Setpoints für Ruhe-Posen gegen `|x| ≥ 5°` prüfen (Klasse C)
- **MUSS [MUST]** für Posen, die nicht offensichtlich innerhalb der Pollen-Nominal-Range liegen, einen lokalen IK-Aufruf machen und die Stewart-Joint-Lösung gegen die URDF-Limits prüfen (Klasse D)
- **SOLLTE [SHOULD]** das Resultat strukturiert melden — Klasse, Schweregrad, betroffene Joints, vorgeschlagene Korrektur

Hinweis: Pre-Flight ersetzt nicht die Live- und Post-hoc-Schicht. Eine Pose-Range mag pre-flight in Ordnung sein, aber Stewart-Plattform-Kopplung (Pitch-Bleed aus [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 2 §"T1–T8 Live-Verifikation") kann zur Laufzeit eine andere Pose ergeben als gewünscht.

### Phase 2 — Live (Erkennung während der Bewegung)

Verpflichtende Checks während laufender Bewegungen, auf Basis der Daemon-REST-API (siehe [`reachy-mini/daemon-rest-api`](../daemon-rest-api/de.md)):

- **MUSS [MUST]** `GET /api/daemon/status` direkt nach einem Pose-Befehl polling, mit den drei Liveness-Cross-Checks aus [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 5: (a) `mean_control_loop_frequency > 40 Hz` + `nb_error == 0`, (b) `head_joints` populiert (nach `?with_head_joints=true`), (c) Pose-Mikro-Drift zwischen zwei konsekutiven `state/full`-Reads
- **MUSS [MUST]** `head_joints: null` plus `backend_status.ready: false` nach Pose-Befehl als Klasse-A-Anomalie behandeln und den Recovery-Pfad (Power-Cycle empfohlen, kein Auto-Restart aus dem Skill) eskalieren
- **SOLLTE [SHOULD]** `nb_error`-Spike (`> 0` und steigend) als Klasse-D-Indikator melden, zusammen mit dem letzten gesendeten Pose-Target
- **SOLLTE [SHOULD]** Antennen-Joint-Reads in einem 2-Sekunden-Rolling-Window halten und Standard-Abweichung als Klasse-C-Indikator berechnen, wenn der Soll-Wert nominal stabil ist
- **MUSS NICHT [MUST NOT]** auto-restart oder auto-`stop-current-app` ausführen, wenn eine Klasse A oder D detektiert wurde — solche Mutationen gehören in einen Recovery-Skill, nicht in den Erkennungs-Layer; statt dessen Ergebnis strukturiert an den User zurückmelden

Hinweis: Der Pollen-Daemon zeigt einen bekannten Desync-Bug (`_status.ready` und `_status.last_alive` werden nicht synchronisiert, siehe [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 5). Daher sind die drei Cross-Checks autoritativer als ein einzelner Bool-Read.

### Phase 3 — Post-hoc (Erkennung nach dem Vorfall)

Erkennung aus aufgezeichneten App-Logs und Telemetrie-Replays:

- **MUSS [MUST]** App-Logs gegen die Triage-Klassen aus [`reachy-mini/app-logging`](../app-logging/de.md) auswerten — insbesondere `daemon-stale-state` ist der Post-hoc-Fingerabdruck von Klasse A
- **SOLLTE [SHOULD]** Telemetrie-Replays auf Pro-Joint-Maximum-Sample-Differenz und Smoothness-Score auswerten (Klasse B)
- **SOLLTE [SHOULD]** Antennen-Joint-Reads pro Setpoint-Bin auf Stdev untersuchen (Klasse C)
- **SOLLTE [SHOULD]** Kinematics-Exception-Tracebacks in App-Logs als Klasse-D-Indikator markieren
- **MUSS [MUST]** den Verifikations-Datums-Marker pro detektierter Anomalie mitführen (Wireless 1.7.1 / Lite / Sim), damit eine spätere Klassen-Verfeinerung gezielt nachvollziehbar bleibt

### Einheitliches Anomalie-Event-Record

Konsumierende Skills (Pre-Flight aus `reachy-mini-sdk`, Live aus `reachy-mini-inspect`, Post-hoc aus `app-log-triage`) **MÜSSEN** strukturierte Anomalie-Records im folgenden JSON-Format emittieren, damit Aggregation und Replay über Plattformen und Skills hinweg konsistent möglich sind:

```jsonc
{
  "class": "A" | "B" | "C" | "D",
  "phase": "pre-flight" | "live" | "post-hoc",
  "severity": "hard" | "warn" | "info",
  "detected_at": "2026-05-13T17:42:11Z",
  "summary": "head pose pitch = +48° outside Pollen nominal range ±40°",
  "telemetry_snapshot": {
    "head_pose": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]],
    "head_joints": [0.0, 0.5, -0.3, ...],
    "backend_status": { "ready": false, "nb_error": 0, "mean_control_loop_frequency": 49.8 }
  },
  "suggested_correction": "clamp pitch to +40°",
  "verification_basis": "Reachy Mini Wireless firmware 1.7.1, 2026-05-13"
}
```

- **MUSS [MUST]** das Feld `class` einen der vier Klassen-Bezeichner (`A` / `B` / `C` / `D`) tragen
- **MUSS [MUST]** das Feld `phase` einen der drei Erkennungs-Phasen-Bezeichner (`pre-flight` / `live` / `post-hoc`) tragen
- **MUSS [MUST]** das Feld `severity` einen der Werte `hard` / `warn` / `info` tragen
- **MUSS [MUST]** das Feld `detected_at` einen ISO-8601-Timestamp mit Zeitzonen-Suffix führen
- **MUSS [MUST]** das Feld `verification_basis` Plattform + Firmware + Datum führen
- **SOLLTE [SHOULD]** `telemetry_snapshot` die relevanten Felder enthalten, ohne PII oder Auth-Tokens (PII-Klausel aus [`reachy-mini/app-logging`](../app-logging/de.md) gilt)
- **KANN [MAY]** `suggested_correction` leer bleiben, wenn keine deterministische Korrektur ableitbar ist

## Akzeptanzkriterien

- [ ] Die Spec trennt drei Erkennungs-Phasen (Pre-Flight / Live / Post-hoc) und nennt pro Klasse mindestens einen konkreten Detect-Signal-Quelle für jede Phase
- [ ] Jede Klasse trägt eine bindende Regel (MUSS / MUSS NICHT / SOLLTE / SOLLTE NICHT) und einen Recovery-Pfad
- [ ] Klasse A nennt die Pollen-Nominal-Range als binding layer und verweist auf [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 2 §"Drei Schichten der Validität"
- [ ] Klasse C nennt `|setpoint| ≥ 5°` (ideal `≥ 10°`) als Antennen-Ruhe-Soll-Wert und verweist auf [`reachy-mini/control-surface`](../control-surface/de.md) §"Mechanische und elektrische Limitationen" (Bullet "Antennen-Totband um 0°")
- [ ] Klasse B nennt `0.16 rad / sample` (entspricht 8 rad/s bei 50 Hz Loop) als Pre-Flight-Schwelle für Joint-Vektor-Diff
- [ ] Klasse D verlangt einen lokalen IK-Aufruf gegen URDF-Limits als Pre-Flight-Check, statt sich auf den Daemon zu verlassen
- [ ] Pro Erkennungs-Phase ist mindestens ein konkretes Detect-Signal benannt, das heute mit dem verifizierten REST-Vertrag aus [`reachy-mini/daemon-rest-api`](../daemon-rest-api/de.md) implementierbar ist (keine reinen TBD-Einträge)
- [ ] Hardware-Verifikations-Status ist pro Klasse explizit (Wireless 1.7.1 verifiziert 2026-05-12/13; Lite explizit noch nicht)
- [ ] Cross-Links zu konsumierenden Skills (`reachy-mini-sdk`, `reachy-mini-inspect`, `app-log-triage`, `dance-choreography`) sind im Body sichtbar und auflösbar
- [ ] Das verbindliche Anomalie-Event-Record ist mit den fünf Pflichtfeldern (`class`, `phase`, `severity`, `detected_at`, `verification_basis`) als MUSS-Regeln in den Anforderungen verankert
- [ ] Live-Konsumenten pollen `backend_status` unmittelbar nach jedem Pose-Befehl, nicht erst bei sichtbarer Anomalie
- [ ] Pro detektierter Anomalie wird der Verifikations-Datums-Marker (Plattform + Firmware + Datum) im Anomalie-Event-Record geführt
- [ ] Die Spec enthält keine Auto-Restart- oder Auto-Mutation-Anweisungen — Erkennung bleibt strikt von Recovery getrennt

## Quellen

- Pollen-SDK-Quelltext (IK-Solver, Posen-Konstanten, Daemon-Status-Loop): <https://github.com/pollen-robotics/reachy_mini>
- Phase-B-Live-Vorfall (Kopf-Körper-Selbstkollision, 2026-05-12): [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 4
- Drei Validitäts-Schichten + IK-Polytop: [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 2
- URDF-Joint-Limits und Stewart-Asymmetrie: [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 1
- Antennen-Totband + Pitch-Bleed: [`reachy-mini/control-surface`](../control-surface/de.md) §"Mechanische und elektrische Limitationen"
- Daemon-Liveness-Cross-Checks + `_status.ready`-Bug: [`reachy-mini/motor-positions`](../motor-positions/de.md) Schicht 5
- REST-Endpunkt-Inventar: [`reachy-mini/daemon-rest-api`](../daemon-rest-api/de.md)
- Triage-Klassen für Post-hoc-Analyse: [`reachy-mini/app-logging`](../app-logging/de.md)
- Konsumierende Skills: [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md), [`reachy-mini-inspect`](../../claude/reachy-mini-inspect/de.md), [`app-log-triage`](../../claude/app-log-triage/de.md), [`dance-choreography`](../../claude/dance-choreography/de.md)

## Offene Fragen

- Soll die Pre-Flight-Pose-Validierung im konsumierenden Plugin (heute) oder direkt im Pollen-SDK (Upstream-PR) landen? Empfehlung: zunächst in diesem Plugin als Skill-Logik, bis Pollen den Check SDK-seitig einbaut. Die Spec hält die Erkennungsregeln so, dass beide Wege denselben Vertrag teilen
- Welcher exakte Jerk-Schwellenwert für Klasse B (rad/s² oder rad/sample) ist verbindlich? Die `0,16 rad/sample`-SOLLTE-NICHT- und `0,30 rad/sample`-MUSS-NICHT-Schwellen sind aus dem URDF-Velocity-Limit abgeleitete Pose-Geschwindigkeits-Schranken, keine Beschleunigungs-Schranken. Eine empirische Mess-Kampagne (Trajektorien-Smoothness vs. Servo-Akustik / Plattform-Vibration) ist nötig, um die Schwellen zu validieren oder zu kalibrieren
- Für Klasse A Live-Detection: wie groß ist die maximale Latenz zwischen einem Pose-Befehl und dem `backend.ready=false`-Flip im Fehler-Fall? Brauchen eine gezielte Messung an einem zweiten Reachy Wireless, weil eine Wiederholung des Phase-B-Vorfalls am gleichen Gerät nicht ratsam ist, solange das Gerät noch im Recovery-Pfad steht
- Lite-Plattform: alle vier Klassen sind nur auf Wireless verifiziert. Insbesondere die Daemon-REST-Antworten könnten auf Lite einen anderen Schema-Pfad haben (USB-Host-getriebener Daemon vs. on-board), und das Antennen-Totband ist Servo-Hardware-spezifisch und sollte auf Lite mit derselben Methodik (`|x| < 5°` Setpoint-Sweep, stdev über Sekunden-Fenster) bestätigt werden
- Sollen die Wacken-/Smoothness-Schwellen (Klasse B `0,16 rad/sample` und `0,30 rad/sample`, Klasse C `stdev > 0,2°`) als Konfigurations-Tunables behandelt werden, oder als feste Konstanten dieser Spec? Empfehlung: feste Konstanten in der Spec; Konsumenten dürfen strenger sein, aber nicht laxer
- Soll `app-logging` um die Triage-Klasse `kinematics-unsolvable` für Klasse-D-Post-hoc-Erkennung ergänzt werden? Die vorliegende Spec referenziert diese Klasse in der Klasse-D-Detect-Tabelle, sie existiert aber noch nicht im `app-logging`-Spec. Folge-PR empfohlen
