# Motion-Monitor-Agent für Reachy Mini

Status: draft

## Kontext

Wenn ein Reachy Mini eine Behavior-Session läuft, kann eine der vier Anomalie-Klassen aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) jederzeit auftreten — Kopf-Körper-Selbstkollision (Klasse A), ruckartige Bewegung (Klasse B), Antennen-Wacken (Klasse C), Stewart-Limit-Knocker (Klasse D). Die Spec definiert *was* zu erkennen ist, aber nicht *wer* es zur Laufzeit beobachtet. Diese Lücke füllt der `motion-monitor`-Agent: er pollt die Daemon-REST-API mit 10 Hz, korreliert die Reads mit `journalctl --user`-Output des Pollen-Daemons und der aktuell laufenden App, klassifiziert jede Anomalie verbindlich nach den vier Klassen, und schreibt ein strukturiertes Anomalie-Event-Record-Log unter `~/.cache/reachy-mini-monitor/`. Am Ende einer bounded 5–15-Minuten-Session liefert er eine knappe Zusammenfassung an den Hauptthread zurück — Findings nach Klasse aufgeschlüsselt, mit Verweis auf den Volltext-Log-Artefakt.

Diese Aufgabe wird als **Agent** modelliert, weil sie lang läuft, viele tausende Telemetrie-Punkte verarbeitet und der Hauptthread weder das Polling-Detail noch die Rohausgabe sehen muss. Der Agent ist read-only über die REST-API (nur `GET`), er löst keine Recovery-Aktionen aus (keine Daemon-Restarts, keine `stop-current-app`-Calls), und er bewegt den Roboter nicht — die strikte Trennung Erkennung ↔ Recovery aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) wird durchgehalten.

Konsumierte Wissens-Specs: [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) (Klassifikation, Detect-Signale, Event-Record-Schema), [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/de.md) (Endpunkt-Inventar), [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md) (URDF-Limits, Drei-Schichten-Validität, `_status.ready`-Bug aus Schicht 5), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) (Triage-Klassen für Post-hoc-Patterns).

## Ziele

- Eine bounded Live-Überwachungs-Session läuft mit einem einzigen Agent-Aufruf und liefert eine strukturierte Zusammenfassung zurück
- Die vier Anomalie-Klassen aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) werden zur Laufzeit erkannt und im verbindlichen Event-Record-Format gemeldet
- Drei Log-Quellen werden parallel ausgewertet: Pollen-Daemon-Journal (`journalctl --user -u reachy-mini-daemon.service`), App-Log (`/api/apps/current-app-status.error` plus Daemon-Output zur App), und ein eigenes Audit-Log unter `~/.cache/reachy-mini-monitor/<session-id>.jsonl`
- Der Agent bleibt strikt read-only und mutiert weder den Daemon noch die App — Erkennung ist sauber von Recovery getrennt
- Eine Session läuft bounded zwischen 5 und 15 Minuten; nach Timeout wird kontrolliert beendet und der Audit-Log abgeschlossen
- Pro Session wird genau **eine** Audit-Log-Datei unter `~/.cache/reachy-mini-monitor/<YYYY-MM-DDTHH-MM-SS>.jsonl` geschrieben, ein Event pro Zeile, JSON-Lines-Format

## Nicht-Ziele

- Daueranlauf / produktiver Watchdog — der Agent ist ein bounded Test-Lifecycle, kein systemd-Daemon. Eine "always-on"-Überwachung gehört in ein separates Tool (z. B. eine Variante des `reachy-mini-mcp-server` mit Streaming-Endpunkt)
- Auto-Recovery oder Auto-Mitigation — der Agent meldet, der Agent restartet nicht. Restart-Logik gehört zu einem eigenen Recovery-Skill, nicht hierher
- Mutationen am Daemon: kein `POST /api/daemon/start`, kein `POST /api/apps/stop-current-app`, kein `set_mode/*`, kein App-Lock-Force-Release. Auch nicht im Fehlerfall
- Bewegungs-Komposition oder Pose-Targeting — der Agent observiert nur, er triggert keine Bewegung
- App-Lifecycle-Management — App-Start gehört zu [`reachy-mini-start`](../reachy-mini-start/de.md), App-Deploy zu [`reachy-mini-deploy`](../reachy-mini-deploy/de.md), Live-Trial-Test zu [`reachy-mini-on-device`](../reachy-mini-on-device/de.md)
- Hardware-Bringup, Kalibrierung, Firmware-Flash — eigene Skills (geplant)
- Audio-, Vision- oder LED-Anomalien — andere Subsysteme; diese Spec ist motorisch / motion-fokussiert
- Definition der Anomalie-Klassen selbst — die kommen aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md), nicht von hier

## Skill-vs-Agent-Begründung

Diese Aufgabe wird als **Agent** und nicht als Skill modelliert, weil mehrere Begründungen aus dem Skill-vs-Agent-Trade-off simultan zutreffen:

- **Lange Laufzeit** — Eine Session läuft bounded 5–15 Minuten, mit Polling alle 100 ms (10 Hz). Skills sind für interaktive Inline-Workflows optimiert; ein langer, observe-then-report-Lifecycle gehört in eine eigene Tool-Session.
- **Hohe Telemetrie-Volumina** — 10 Hz × 900 s = bis zu 9.000 State-Reads pro 15-Minuten-Session, plus parallele `journalctl --follow`-Streams aus zwei Quellen. Im Hauptthread würde das den Kontext verschlingen; der Agent reduziert das auf eine Zusammenfassung plus Datei-Artefakt.
- **Fire-and-forget-Lifecycle** — der Caller setzt einmal die Parameter (Dauer, Plattform, Klassen-Filter), der Agent läuft, der Agent meldet. Mid-flow Tweaks während des Laufs sind nicht vorgesehen.
- **Eigenständiges Tool-Set** — der Agent braucht Bash für `journalctl`, HTTP-Aufrufe (via `curl` oder `httpx`), und JSON-Lines-Schreiben auf Disk. Diese Tools gehören nicht in den Hauptthread, der typischerweise Code-Editing dominiert.
- **Spezialisiertes Verhalten** — Anomalie-Klassifikation nach vier Klassen, drei-Wege-Liveness-Cross-Check (siehe [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md) Schicht 5), strukturiertes Event-Record-Schema. Ein dedizierter Agent kanonisiert das.
- **Distribution: `plugin`** — der Agent gehört zum Plugin und wird mit ihm verteilt.

## Anforderungen

### Eingaben

- **MUSS [MUST]** die Plattform annehmen (`platform`: `wireless` / `lite` / `simulation`); auf `simulation` läuft der Agent gegen einen Daemon im selben Python-Prozess und kann den App-Log-Pfad nicht über `systemd journalctl` lesen, sondern über App-Process-Stdout
- **MUSS [MUST]** die Daemon-Adresse annehmen (`daemon_url`, Default `http://127.0.0.1:8000`); auf Wireless mit mDNS-Default `http://reachy-mini.local:8000`
- **MUSS [MUST]** die Session-Dauer annehmen (`duration_seconds`, Default `300` = 5 Min, Maximum `900` = 15 Min); Werte außerhalb [60, 900] werden abgelehnt
- **MUSS [MUST]** die Polling-Cadence annehmen (`poll_interval_ms`, Default `100` = 10 Hz); Werte außerhalb [50, 1000] werden abgelehnt — schneller als 20 Hz erzeugt unnötige Daemon-Last, langsamer als 1 Hz verfehlt Klasse-A-Detektion
- **SOLLTE [SHOULD]** eine Klassen-Filter-Liste annehmen (`watch_classes`, Default `[A, B, C, D]`); leere Liste wird abgelehnt
- **SOLLTE [SHOULD]** ein optionales `audit_log_dir` annehmen (Default `~/.cache/reachy-mini-monitor/`); fehlende Schreibrechte abbrechen, niemals stillschweigend in `/tmp/` ausweichen
- **SOLLTE [SHOULD]** ein optionales `severity_floor` annehmen (`hard` | `warn` | `info`, Default `info`); Events unterhalb der Schwelle werden geschrieben, aber nicht in die Zusammenfassung aufgenommen
- **KANN [MAY]** den Daemon-Journal-Service-Namen annehmen (`daemon_service`, Default `reachy-mini-daemon.service`) — Wireless-Default; auf Lite kann der Name abweichen, abhängig vom Host-Setup
- **DARF NICHT [MUST NOT]** unbegrenzte Laufzeit erlauben — `duration_seconds=0` oder Werte > 900 sind verbotene Eingaben

### Plattform-Profile

| Plattform | Daemon-Log-Quelle | App-Log-Quelle | Polling-Endpunkt | Einschränkungen |
|---|---|---|---|---|
| **`wireless`** | `journalctl --user -u <daemon_service> --follow` per SSH oder lokal auf dem Reachy | `/api/apps/current-app-status` plus App-Stdout via Daemon-Forward | `http://reachy-mini.local:8000/api/state/full?with_head_joints=true` | Volle Klasse-A-Detektion via `backend.ready`-Flip |
| **`lite`** | `journalctl --user -u <daemon_service>` auf dem Host-PC | `/api/apps/current-app-status` plus App-Stdout via Daemon | `http://127.0.0.1:8000/api/state/full?with_head_joints=true` | Klassen A–D verifizierbar; **bisher nicht hardware-validiert** (siehe Offene Fragen) |
| **`simulation`** | App-Process-Stdout direkt (kein systemd) | App-Process-Stdout (gleiche Quelle) | `http://127.0.0.1:8000/api/state/full?with_head_joints=true` | Klasse A und C sind im Sim **nicht** detektierbar (keine echte Stewart-Mechanik, keine Servo-Hardware); nur Klassen B und D laufen sinnvoll |

Anforderungen:

- **MUSS [MUST]** beim Session-Start die Plattform aus der Eingabe gegen `GET /api/daemon/status.wireless_version` validieren — Mismatch ist ein FAIL und beendet die Session vor dem ersten Poll
- **MUSS [MUST]** plattform-spezifische Filter auf die Klassen-Detektion anwenden: Klasse A und C werden auf `simulation` automatisch übersprungen und im Report als `not_applicable_in_simulation` markiert
- **DARF NICHT [MUST NOT]** auf Lite ein Fehlen von IMU-Telemetrie als Defekt werten — das ist erwartet (siehe [`reachy-mini-on-device`](../reachy-mini-on-device/de.md) §Plattform-Profile)

### Lifecycle

- **MUSS [MUST]** den Lifecycle in dieser Reihenfolge ausführen: pre-flight → session-start → poll-loop → log-tail → graceful-stop → report → cleanup
- **MUSS [MUST]** in der **pre-flight**-Phase fünf Gates prüfen und bei erstem Fehlschlag abbrechen:
  1. **Daemon erreichbar** — `GET <daemon_url>/api/daemon/status` HTTP 200 in 3 s; mDNS-Cold-Start-Retry einmal erlaubt
  2. **Plattform-Match** — wie oben
  3. **Audit-Log-Pfad schreibbar** — `audit_log_dir/<session-id>.jsonl` muss erzeugbar sein
  4. **Daemon-Journal erreichbar** — `systemctl --user is-active <daemon_service>` muss `active` zurückgeben; bei `simulation` entfällt dieses Gate
  5. **Aktuelle App-Sitzung sichtbar** — `GET /api/apps/current-app-status` darf nicht HTTP 5xx liefern; `null` (keine App läuft) ist OK und wird im Report vermerkt
- **MUSS [MUST]** in der **session-start**-Phase eine eindeutige Session-ID generieren (`YYYY-MM-DDTHH-MM-SS_<random>`), das Audit-Log mit einem Session-Header-Event eröffnen (Plattform, Daemon-URL, Konfiguration, Verifikations-Datum aus `daemon/status.version`) und einen Async-Tail auf das Daemon-Journal starten
- **MUSS [MUST]** in der **poll-loop**-Phase alle `poll_interval_ms` einen State-Snapshot lesen, gegen die vier Klassen prüfen und detektierte Anomalien ins Audit-Log schreiben
- **MUSS [MUST]** in der **log-tail**-Phase parallel zum Polling die Log-Streams (`journalctl --follow` und App-Stdout) auf bekannte Patterns prüfen (Klasse A: `ConnectionError: Could not connect to daemon on localhost`; Klasse D: `kinematics`-Exception-Traceback; siehe [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md))
- **MUSS [MUST]** in der **graceful-stop**-Phase nach Ablauf von `duration_seconds` (oder bei explizitem Abbruch des Callers) den Poll-Loop und die Log-Tails sauber beenden, alle gepufferten Events in das Audit-Log flushen und einen Session-Footer-Event schreiben (Gesamtdauer, Event-Counts pro Klasse)
- **MUSS [MUST]** in der **report**-Phase eine knappe Zusammenfassung an den Caller zurückgeben (Schema siehe "Ausgabe-Schema")
- **MUSS [MUST]** in der **cleanup**-Phase sicherstellen, dass alle Subprocesses beendet sind und keine Datei-Handles offen bleiben

### Live-Polling und Klassen-Detektion

Pro Poll werden folgende Felder gelesen und gegen die vier Klassen geprüft (Klassen-Details und Schwellen aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md)):

- **Klasse A** — `GET /api/daemon/status.backend_status.ready` flippt auf `false` plus `GET /api/state/full?with_head_joints=true.head_joints == null` plus Pose keine Mikro-Drift zwischen zwei konsekutiven Reads ⇒ Drei-Wege-Cross-Check positiv ⇒ Event `class=A, severity=hard`
- **Klasse B** — `‖Δjoints‖₂` zwischen zwei konsekutiven `head_joints`-Reads pro `Δt` größer als die URDF-Velocity-Schwelle (`0,16 rad/sample` bei 10 Hz → ~1,6 rad/s, also nicht jeder Sample ist die volle URDF-Geschwindigkeit ausreizbar; bei `poll_interval_ms=100` ist die Schwelle `0,16 rad`) ⇒ `severity=warn`; > `0,30 rad/sample` ⇒ `severity=hard`
- **Klasse C** — Standard-Abweichung der Antennen-Joint-Reads über ein 2-Sekunden-Rolling-Window (= 20 Samples bei 10 Hz) `> 0,2°` trotz nominal stabilem Soll-Wert ⇒ Event `class=C, severity=warn`; Klasse C ist auf `simulation` skipped
- **Klasse D** — `backend_status.nb_error` spike (`> 0` und steigend) nach dem letzten gesendeten Pose-Befehl, oder `head_joints` weicht von `target_head_joints` um mehr als URDF-Limit-Toleranz ab ⇒ Event `class=D, severity=warn`

Anforderungen:

- **MUSS [MUST]** für jede detektierte Anomalie das verbindliche Event-Record-Schema aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) verwenden (Felder `class`, `phase`, `severity`, `detected_at`, `verification_basis` sind Pflicht)
- **MUSS [MUST]** `phase` im Record auf `live` setzen (für Polling-Detektionen) bzw. auf `post-hoc` (für Log-Pattern-Matches)
- **MUSS [MUST]** `verification_basis` aus `daemon/status.version` und der Plattform-Eingabe befüllen, nicht erraten
- **DARF NICHT [MUST NOT]** auf einen `backend_status.ready=false`-Read alleine als Klasse A klassifizieren — die drei-Wege-Cross-Check-Regel aus [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md) Schicht 5 ist zwingend
- **SOLLTE [SHOULD]** ein Pollen-Daemon-Bug (`_status.ready`-Desync, siehe [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md) Schicht 5) im Audit-Log markieren, sobald `backend_status.ready=false` länger als 10 s ansteht ohne dass die anderen zwei Cross-Checks (mean_freq, head_joints) Liveness widerlegen

### Log-Quellen und Tail-Mechanik

Drei parallele Log-Quellen werden während der Session ausgewertet:

1. **Pollen-Daemon-Journal** — `journalctl --user -u <daemon_service> --follow --since <session-start>` (auf `wireless` per SSH, auf `lite` lokal, auf `simulation` entfällt). Pattern-Match auf Klasse-A-Indikatoren (`backend.ready: false` direkt nach `start-app`), Klasse-D-Indikatoren (Kinematics-Exception-Tracebacks)
2. **App-Log** — `/api/apps/current-app-status.error` (Status-Polling) plus Daemon-Forward des App-Stdout; Pattern-Match auf `ConnectionError: Could not connect to daemon on localhost` (= Klasse A post-hoc per [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) Triage-Klasse `daemon-stale-state`), Kinematics-Tracebacks (= Klasse D)
3. **Eigenes Audit-Log** — `audit_log_dir/<session-id>.jsonl`, JSON-Lines-Format, ein Event pro Zeile

Anforderungen:

- **MUSS [MUST]** jedes Pattern-Match aus Quellen 1 und 2 als Event ins Audit-Log (Quelle 3) schreiben, niemals separate Logfiles für die einzelnen Quellen
- **MUSS [MUST]** das Audit-Log JSON-Lines-Format verwenden, eine vollständige Event-Zeile pro `\n`, nie partielle Schreibvorgänge zwischenspeichern
- **MUSS [MUST]** ein Session-Header-Event als erste Zeile und ein Session-Footer-Event als letzte Zeile schreiben — Header trägt Konfiguration und Plattform, Footer trägt Event-Counts pro Klasse und Gesamt-Dauer
- **SOLLTE [SHOULD]** Audit-Log-Files älter als 30 Tage NICHT automatisch löschen — Aufräumen ist Caller-Verantwortung
- **DARF NICHT [MUST NOT]** PII oder Auth-Tokens ins Audit-Log schreiben (PII-Klausel aus [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md))

### Notstopp und Grenzen

- **MUSS [MUST]** bei eigenem internem Fehler (z. B. HTTP-Timeout dauerhaft, `journalctl`-Subprocess crasht) den Session-Footer mit `result=aborted` schreiben und im Report `aborted_reason` füllen
- **MUSS [MUST]** bei Caller-Abbruch (z. B. SIGTERM, Kontext-Cancel) graceful-stop ausführen, nicht hart abbrechen
- **DARF NICHT [MUST NOT]** auch im Fehlerfall mutierende Daemon-Calls absetzen — kein `stop-current-app`, kein `restart`, kein `set_mode/*`
- **DARF NICHT [MUST NOT]** den Reachy bewegen — der Agent ist Erkennung, keine Korrektur

### Ausgabe-Schema

Der Agent gibt eine knappe Zusammenfassung zurück (Markdown), zusätzlich den Pfad zum Audit-Log-Artefakt:

```text
# Motion Monitor — Session <session-id>

Platform: <wireless|lite|simulation>
Daemon: <daemon_url>  (firmware <version>)
Duration: <actual_seconds> s of <duration_seconds> s budget
Result: <pass|warn|fail|aborted>

## Findings per class

- Class A (head-against-body): <count> events (<severities>)
- Class B (jerky motion):      <count> events
- Class C (antenna wobble):    <count> events  [skipped on simulation]
- Class D (Stewart-limit):     <count> events

## Notable patterns

- <one-line summary per pattern, e.g. "Class A live + Class A post-hoc match — daemon hung after start-app at 17:42:11">

## Audit log

Path: ~/.cache/reachy-mini-monitor/<session-id>.jsonl
Events: <total_event_count>
```

Anforderungen:

- **MUSS [MUST]** `result` einer der Werte `pass` (keine Hard-Events), `warn` (nur Warn-Events), `fail` (mindestens ein Hard-Event), `aborted` (Session vorzeitig beendet) sein
- **MUSS [MUST]** Plattform, Daemon-URL, Firmware-Version, Session-Dauer und Audit-Log-Pfad im Output stehen, niemals ausgelassen werden
- **MUSS [MUST]** auf `simulation` die nicht-anwendbaren Klassen (A, C) explizit als `[skipped on simulation]` markieren
- **SOLLTE [SHOULD]** "Notable patterns" maximal 5 Einträge enthalten — bei mehr Findings auf das Audit-Log verweisen

### Hard Rules

1. **Read-only.** Der Agent ruft **nur** `GET`-Endpunkte auf dem Pollen-Daemon auf. Keine `POST`, keine `PUT`, keine `DELETE`. Auch nicht im Fehlerfall.
2. **Keine Bewegung.** Der Agent triggert nie ein Pose-Target, einen Move oder einen Mode-Wechsel.
3. **Keine Recovery.** Eine detektierte Klasse-A-Anomalie wird gemeldet, **nicht** durch `stop-current-app` + Restart "behoben". Recovery gehört zu einem dedizierten Skill.
4. **Bounded Lifecycle.** Maximum `duration_seconds=900` (15 Min). Eine Endlos-Schleife ist nicht möglich.
5. **Drei-Wege-Cross-Check für Klasse A.** Ein einzelner `backend_status.ready=false`-Read reicht nicht — die Regel aus [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md) Schicht 5 ist verbindlich.
6. **Verifikations-Datum.** Jedes Event-Record trägt `verification_basis` mit Plattform + Firmware + Datum.
7. **JSON-Lines-Audit.** Das Audit-Log ist JSON-Lines, nicht JSON-Array, damit ein abrupter Abbruch das File nicht korrumpiert.

## Akzeptanzkriterien

- [ ] Der Agent existiert unter `agents/motion-monitor.md` mit gültiger Frontmatter (`name: motion-monitor`, `description`, `distribution: plugin`, `tools`, optionale Tags)
- [ ] Die `description` benennt die drei Logfile-Quellen (Daemon-Journal, App-Log, eigenes Audit-Log) und alle vier Anomalie-Klassen
- [ ] Pre-flight prüft fünf Gates in der spezifizierten Reihenfolge mit konkreter Fehler-Aktion pro Gate
- [ ] Polling-Cadence-Default ist 10 Hz (`poll_interval_ms=100`), Maximum 20 Hz, Minimum 1 Hz
- [ ] Bounded session: `duration_seconds` Default 300, Maximum 900, Werte außerhalb [60, 900] werden abgelehnt
- [ ] Drei-Wege-Cross-Check für Klasse A ist implementiert: `backend.ready` + `head_joints` + Pose-Mikro-Drift müssen alle drei "no liveness" anzeigen, bevor Klasse A gemeldet wird
- [ ] Klasse C wird auf `simulation` automatisch übersprungen und im Report als `[skipped on simulation]` markiert
- [ ] Audit-Log wird im JSON-Lines-Format unter `audit_log_dir/<session-id>.jsonl` geschrieben, Session-Header und Session-Footer rahmen den Stream
- [ ] Pro detektierter Anomalie wird das verbindliche Event-Record-Schema aus [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md) verwendet, alle fünf Pflichtfelder vorhanden
- [ ] Der Agent setzt **keine** mutierenden Daemon-Calls ab — auch nicht im Fehlerfall (Hard Rule 1+3)
- [ ] Das `result`-Feld im Report ist genau einer der Werte `pass` / `warn` / `fail` / `aborted`
- [ ] Cross-Refs auf [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md), [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/de.md), [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md), [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md) sind sichtbar
- [ ] `pre-commit run --all-files` läuft auf der Agent-Datei grün

## Quellen

- Anomalie-Klassifikation, Detect-Signale, Event-Record-Schema: [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md)
- REST-Endpunkt-Inventar: [`reachy-mini/daemon-rest-api`](../../reachy-mini/daemon-rest-api/de.md)
- URDF-Limits, Drei-Schichten-Validität, `_status.ready`-Bug: [`reachy-mini/motor-positions`](../../reachy-mini/motor-positions/de.md)
- Triage-Klassen für Log-Pattern-Matches: [`reachy-mini/app-logging`](../../reachy-mini/app-logging/de.md)
- Plattform-Profile (Wireless / Lite / Simulation) Vorbild: [`claude/reachy-mini-on-device`](../reachy-mini-on-device/de.md)
- Pollen-SDK-Quelltext (Daemon-Status-Loop, Joint-Reads): <https://github.com/pollen-robotics/reachy_mini>
- systemd `journalctl --user --follow` Referenz: <https://www.freedesktop.org/software/systemd/man/latest/journalctl.html>

## Offene Fragen

- Soll der Agent auf `lite` SSH zum Host-PC benutzen, oder ist lokales `journalctl` ausreichend (wenn der Host der MCP-Client ist)? Empfehlung: lokales `journalctl`, weil der typische Lite-Setup den Host als Entwickler-Maschine hat; SSH-Variante als optional in einer späteren Spec-Revision
- Welcher exakte Klasse-A-Latency-Wert ist die Obergrenze zwischen Pose-Befehl und `backend.ready=false`-Flip im Fehler-Fall? Die `motion-anomaly-detection` Open Question ist hier load-bearing; ohne gemessenen Wert kann der Agent nur "lange genug warten" sagen, was eine ergonomische Schwäche ist
- Soll der Agent eine optionale Heatmap-Ausgabe pro Klasse über die Session-Dauer produzieren (z. B. Klasse-B-Events alle 10 s aggregiert)? Vorschlag: nein für v1, später optional
- Wie verhält sich der Agent, wenn der Daemon während der Session neu startet? Der `journalctl --follow` sollte das überleben; die REST-Polls bekommen kurz `connection refused`. Soll der Agent den Restart als `info`-Event ins Audit-Log schreiben oder als `warn`?
- Mehrfach-Reachy-Setup: ein Entwickler hat zwei Geräte (Wireless + Lite); soll der Agent eine Session-Gruppe verwalten oder bleibt es bei ein Reachy pro Session? Vorschlag: ein Reachy pro Session, Multi-Reachy ist Caller-Sache (zwei Agent-Aufrufe)
- Audit-Log-Rotation: aktuell schreibt der Agent ein File pro Session unter `~/.cache/reachy-mini-monitor/`. Soll es eine maximale Anzahl behaltener Files geben, oder bleibt Aufräumen Caller-Sache? Empfehlung: Aufräumen ist Caller-Sache (auch dokumentiert in der "Nicht-Ziele"-Sektion)
- Lite-Plattform: alle vier Klassen sind nur auf Wireless hardware-verifiziert (`motion-anomaly-detection` Open Question). Sobald Lite vor Ort ist, muss der Agent gegen die tatsächlichen Daemon-Response-Shapes und Servo-Hardware-Eigenheiten validiert werden
