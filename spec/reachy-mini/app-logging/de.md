# Logging und Fehleranalyse während der App-Entwicklung

Status: draft

## Kontext

Während der Entwicklung einer Reachy-Mini-App verteilen sich Log-Einträge auf **mindestens drei separate Prozess- und Logger-Bäume**: den App-eigenen Code (Python-`logging`), das SDK-Paket `reachy_mini` (Logger `reachy_mini.*`) und den Pollen-Daemon (eigener Prozess, eigener Logger-Baum). Wer diese Topologie nicht kennt, sucht häufig stundenlang am falschen Ort, weil eine App-Aktion fast immer Log-Spuren in mehreren dieser Bäume gleichzeitig hinterlässt — und je nach Plattform (Wireless / Lite / Simulation) und je nach Hosting-Modus (Pollen-Daemon vs. direkter `pytest`-Lauf) landen die Spuren in unterschiedlichen Senken.

Diese Spec konsolidiert die Logging-Topologie, die in den Pollen-Quellen verstreut ist — `skills/debugging.md`, `src/reachy_mini/reachy_mini.py`, `src/reachy_mini/daemon/daemon.py`, `src/reachy_mini/apps/manager.py`, `src/reachy_mini/daemon/robot_app_lock.py` — und macht sie als kanonische Wissensbasis für die App-Entwicklung verfügbar. Sie ist die Grundlage, auf die der `reachy-mini-sdk`-Skill (Wissensbasis), der `app-scaffold`-Skill (Test-Stub) und der `reachy-mini-on-device`-Agent (Log-Tailing-Schritt) aufsetzen, ohne ihr Wissen jedes Mal selbst zusammensuchen zu müssen.

Begriffsklärung: „Logs" meint in dieser Spec strukturell aufgenommene Diagnose-Ausgaben des Standard-Python-`logging`-Frameworks plus stdout-/stderr-Streams, die der Daemon vom App-Subprozess capturet. „Fehleranalyse" umfasst sowohl das Lesen dieser Logs während der Entwicklung (lokales `pytest`, lokaler Daemon, Ad-hoc-Lauf) als auch die ersten Triage-Schritte gegen Pollens kanonisches Common-Issues-Inventar.

## Ziele

- Eine vollständige Karte aller Log-Quellen, gruppiert nach Logger-Baum und nach Plattform, sodass jeder Eintrag auf seine Senke und sein Herkunfts-Modul zurückgeführt werden kann
- Eine eindeutige Konvention für App-eigenes Logging (`logging.getLogger(__name__)`, RFC-2119-mäßig formuliert), die mit Pollens SDK-Konvention kompatibel bleibt
- Ein klar dokumentiertes Hosting-Verhalten: was passiert mit `print(...)` und Exceptions in einer App, die unter Pollen-Daemon-Hosting läuft, vs. in einer App, die direkt per `pytest` oder `python -m` ausgeführt wird
- Ein Triage-Katalog für die häufigsten Failure-Modes mit Zuordnung zu konkreten Log-Patterns, abgeglichen mit Pollens `skills/debugging.md`
- Die Pollen-Heuristik „verify basics first" (`examples/minimal_demo.py` als Sanity-Check vor App-spezifischer Diagnose) als verbindliche Erstmaßnahme bei jeder neuen Failure-Klasse

## Nicht-Ziele

- Production-Logging auf provisionierten Hosts (`reachy-app@<slug>.service`, journald-Vacuum, App-Pull-Service-Logs) — abgedeckt in [`reachy-mini/host-provisioning`](../host-provisioning/de.md)
- Log-Tailing-Skript-Logik des `reachy-mini-on-device`-Agents (welche Filter, welcher Zeit-Anker) — abgedeckt in [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md)
- Hardware-Failure-Recovery (Mic-FPC-Cable, Motors-Diagnosis, Spherical-Joints) — abgedeckt unter [`pollen-robotics/reachy_mini/docs/source/troubleshooting/`](https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting)
- WebRTC-/JS-Side-Logging einer Browser-App (eigene Surface, eigene Tooling-Welt) — Pollens `AGENTS.md` § JS apps deckt das ab
- Eine generische Python-`logging`-Tutorial-Sammlung — diese Spec setzt Vertrautheit mit `logging.getLogger`, `setLevel`, `StreamHandler` voraus
- Strukturiertes Logging im Sinne von JSON-Lines / OpenTelemetry — nicht aktueller Pollen-Standard, eigene Spec, falls überhaupt benötigt
- Performance-Profiling, Tracing, Flame-Graphs — andere Tooling-Welt

## Anforderungen

### Log-Quellen-Inventar

Jede Reachy-Mini-App-Sitzung erzeugt Log-Einträge in **drei orthogonalen Logger-Bäumen** plus zwei stdout-/stderr-Streams. Die folgende Tabelle ist die kanonische Karte:

| Quelle | Logger-Name (Python) | Default-Level | Erzeugt von | Quelle der Konfiguration |
|---|---|---|---|---|
| App-Code | beliebig (Konvention: `logging.getLogger(__name__)`) | von der App selbst gesetzt; ohne Setzung schlägt Python auf `WARNING` zurück | jede `logger.info(...)`-Stelle in `reachy_mini_app/main.py` und Sub-Modulen | `logging.basicConfig(...)` oder explizite Handler im App-Code |
| SDK-Hauptklasse | `reachy_mini` (per `getLogger(__name__)` in [`reachy_mini.py:133`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py)) | `INFO` | `ReachyMini`-Methoden (Connection-Mode-Wahl, Media-Re-Acquire, Move-Cancellation) | Konstruktor-Parameter `ReachyMini(log_level: str = "INFO")` |
| Daemon-Hauptloop | `reachy_mini.daemon.daemon` (per `getLogger(__name__)` in [`daemon.py:50`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py)) | `INFO` | Daemon-Lifecycle, Media-Server-Bring-up, Central-Signaling-Relay | CLI-Flag `reachy-mini-daemon --verbose` (setzt `DEBUG`); programmatisch `log_level`-Parameter |
| App-Lock | `reachy_mini.daemon.robot_app_lock` ([`robot_app_lock.py:42`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/robot_app_lock.py)) | erbt vom Daemon | Lock-Acquire / Lock-Release / Konflikt-Pfade (genau die Stellen, an denen eine zweite App abgewiesen wird) | erbt; nicht separat konfigurierbar |
| App-Manager | `reachy_mini.apps.manager` plus Child `reachy_mini.apps.manager.runner` ([`manager.py:68`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py)) | erbt vom Daemon | App-Subprozess-Spawn, Lifecycle-Übergänge, Aufnahme von App-`stdout`/`stderr` | erbt |
| App-`stdout` (im Daemon-Hosting) | gepiped → `reachy_mini.apps.manager.runner.info` | erbt | jeder `print(...)` und alles, was das App-Subprocess auf stdout schreibt; Subprozess wird mit `python -u` (unbuffered) gespawnt | nicht direkt konfigurierbar; muss über App-Logger laufen, um saubere Levels zu haben |
| App-`stderr` (im Daemon-Hosting) | gepiped → `reachy_mini.apps.manager.runner.error` (mit Heuristik) oder `.warning` | erbt | Tracebacks, ungefangene Exceptions, alles auf stderr; die Heuristik in `manager.py:206–209` klassifiziert Zeilen, die wie Errors aussehen, als `error`, sonst als `warning` | nicht direkt konfigurierbar |

- **MUSS [MUST]** jede neue App in der eigenen Code-Basis `logging.getLogger(__name__)` als Logger-Quelle nutzen, niemals `logging.getLogger("root")` oder `print()` für Diagnose; das ergibt einen Logger-Namen, der mit dem Python-Modul-Namen übereinstimmt und damit granulare Modul-Filter und Level-Switches erlaubt
- **DARF NICHT [MUST NOT]** der App-Code den Wurzel-Logger global re-konfigurieren (kein `logging.basicConfig(level=DEBUG)` ohne Bedingung); damit würde der SDK-Logger-Baum unkontrolliert mit-laut

### Plattform-Profile — wo Logs landen

Wo die obigen Logger-Bäume tatsächlich sichtbar werden, hängt von der Ziel-Plattform und vom Lauf-Modus ab:

| Plattform / Modus | App-Logger-Senke | SDK-Logger-Senke | Daemon-Logger-Senke | Werkzeug zum Lesen |
|---|---|---|---|---|
| Wireless, App im Daemon-Hosting | über App-Manager-Runner ins Daemon-Log | im Daemon-Prozess | systemd-journald | `ssh pollen@reachy-mini.local "sudo journalctl -u reachy-mini-daemon.service -f"` |
| Wireless, App via SSH manuell gestartet | App-`stdout` direkt im SSH-Terminal | im selben Prozess wie die App | separater Daemon-Service, weiterhin journald | App-Output im SSH-Terminal; Daemon parallel via `journalctl` |
| Lite, App im Daemon-Hosting | über App-Manager-Runner ins Daemon-Log | im Daemon-Prozess | Daemon-stdout im Terminal des `reachy-mini-daemon`-Aufrufs | Terminal des Daemon-Prozesses; ggf. mit `--verbose` |
| Lite, App via `python -m` oder Editor | App-`stdout` im Terminal des App-Aufrufs | im selben Prozess wie die App | separater Daemon-Prozess, eigenes Terminal | zwei Terminals nebeneinander |
| Lite, `pytest` lokal | pytest-Capture (`-s` deaktiviert es) | im pytest-Prozess | nur falls Test mit `ReachyMini(spawn_daemon=True)` einen Daemon hochzieht — dann im Test-Subprozess | `pytest -s` zeigt Live-Output; ohne `-s` ist es im Test-Failure-Output sichtbar |
| Simulation (`use_sim=True`) | App-`stdout` im Terminal | im selben Prozess wie die App | falls Daemon-Sim mit `spawn_daemon=True` startet, im selben Prozess; bei `reachy-mini-daemon --sim` als externer Prozess im eigenen Terminal | wie bei Lite |

- **MUSS [MUST]** jeder neu geschriebene Log-Pfad einer App auf seine Senke pro Plattform überprüft werden, bevor er als „funktioniert" markiert wird; ein Eintrag, der lokal in `pytest` sichtbar ist, ist nicht automatisch in journald sichtbar
- **SOLLTE [SHOULD]** im Default-Setup die App während der Entwicklung im **direkten Lauf-Modus** (`pytest` oder `python -m reachy_mini_app.main`) entwickelt werden, damit App-Logger und SDK-Logger beide direkt im Terminal landen; der Daemon-Hosting-Pfad ist für die Validierung gegen das echte Lifecycle-Modell, nicht für die tägliche Iteration
- **SOLLTE [SHOULD]** auf Wireless im Daemon-Hosting der Filter `journalctl -u reachy-mini-daemon.service -f --since '<lauf-start>'` mit zusätzlichem `grep -v "uvicorn\|GET \|POST "` verwendet werden, um HTTP-Rauschen aus dem REST-Surface auszublenden — Quelle: [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md) Z. 67–71

### Log-Einträge im App-Code erzeugen

Pollens SDK setzt vollständig auf das Standard-Python-`logging`-Framework — keine eigenen Wrapper, kein Loguru, kein structlog. Apps folgen dem.

- **MUSS [MUST]** der App-Code in jedem Modul oben einen Logger als Modul-globale Konstante anlegen:

  ```python
  import logging

  logger = logging.getLogger(__name__)
  ```

- **MUSS [MUST]** Diagnose-Ausgaben über diesen Logger laufen, niemals über `print(...)`. Begründung: nur Logger-Aufrufe respektieren Levels, Handler-Konfigurationen und Multi-Senken-Routing; `print(...)` umgeht alles davon und wird im Daemon-Hosting zu `info`-Rauschen.
- **SOLLTE [SHOULD]** der App-Code für strukturelle Diagnose-Punkte (Lifecycle-Übergänge, Choreographie-Sektion-Wechsel, Audio-Trigger) `logger.info(...)` verwenden; für detaillierte Trace-Ausgaben `logger.debug(...)`; für eigene Fehler-Pfade `logger.warning(...)` oder `logger.error(...)` mit `exc_info=True`, wenn ein Exception-Kontext vorliegt
- **SOLLTE [SHOULD]** für nicht-fatale Exceptions, die der App-Code selbst behandelt, `logger.exception("...")` (oder `logger.error(..., exc_info=True)`) genutzt werden — das fügt den Traceback in die Log-Zeile ein
- **DARF NICHT [MUST NOT]** der App-Code Tokens, Hugging-Face-Auth-Keys, WiFi-Credentials oder Sensorwerte mit personenbezogenem Bezug ins Log schreiben (Konsistenz mit [`reachy-mini/host-provisioning`](../host-provisioning/de.md) § Logging und Diagnose)

### Daemon-Hosting: was mit `print()` und Exceptions passiert

Pollens App-Manager spawnt den App-Subprozess in [`apps/manager.py:154`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py) mit dem Flag `python -u` (unbuffered stdout/stderr) und liest beide Streams asynchron mit:

- **stdout** → jede Zeile wird zu `reachy_mini.apps.manager.runner.info(line)` (`manager.py:188–192`)
- **stderr** → jede Zeile wird durch eine Heuristik klassifiziert: enthält die Zeile typische Error-Marker (Traceback, Exception, etc.), wird sie zu `runner.error(line)`, sonst zu `runner.warning(line)` (`manager.py:196–209`)

Konsequenzen für den App-Entwickler:

- **MUSS [MUST]** jeder `print(...)`-Aufruf im App-Code als „landet im Daemon-Log auf Level `info`" verstanden werden; das ist kein „verloren", aber auch nicht der gewünschte Diagnose-Pfad
- **MUSS [MUST]** ein **ungefangener Exception** in `ReachyMiniApp.run(...)` in jedem Fall im Daemon-Log auf Level `error` erscheinen — das ist der primäre Anker für Crash-Triage; falls er nicht auftaucht, läuft die App nicht im erwarteten Hosting-Modus
- **SOLLTE [SHOULD]** der App-Code Tracebacks selbst formatieren und über `logger.exception(...)` ausgeben, statt sie ungefangen propagieren zu lassen — das gibt mehr Kontext (App-Logger-Name, eigene Message) und lässt die App in Recovery-Pfade laufen, statt vom Daemon abgeschossen zu werden
- **DARF NICHT [MUST NOT]** im App-Code stdout / stderr direkt umkonfiguriert werden (`sys.stdout = ...`); der Daemon erwartet standardgemäßes Pipe-Verhalten

### Log-Level-Konfiguration

Drei Stellschrauben:

| Stellschraube | Wirkungsbereich | Aufruf |
|---|---|---|
| `ReachyMini(log_level=...)` | SDK-Logger-Baum (`reachy_mini.*`) | im App-Code beim Aufbau der `ReachyMini`-Instanz: `ReachyMini(log_level="DEBUG")` |
| `reachy-mini-daemon --verbose` | Daemon-Logger-Baum, plus alle Child-Logger (App-Manager, Robot-App-Lock) | beim Start des Daemon im Lite-Setup |
| App-eigenes `logger.setLevel(...)` oder `logging.basicConfig(level=...)` | nur App-eigener Logger-Baum | im `main()`-Eintrittspunkt; **nicht** den Wurzel-Logger global setzen |

- **SOLLTE [SHOULD]** für die tägliche Entwicklung der SDK-Level beim `INFO`-Default belassen werden; auf `DEBUG` umschalten **nur** während aktiver Triage einer konkreten Failure-Klasse
- **MUSS [MUST]** das gewählte Level in der App-Code-Basis dokumentiert sein, sobald es vom SDK-Default abweicht (z. B. als Konstante mit Begründung)
- **DARF NICHT [MUST NOT]** der App-Code den Daemon-Level zur Laufzeit fern-konfigurieren — der Daemon ist ein eigener Prozess; Level-Änderungen erfordern einen Daemon-Restart mit angepasstem Flag

### Common-Issues-Triage-Katalog

Pollens [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) listet die kanonischen Failure-Klassen auf. Die folgende Tabelle bindet jede Klasse an ihre erwartbaren Log-Patterns und an die erste Triage-Aktion:

| Symptom (laut Pollen) | Erwartetes Log-Muster | Erste Aktion |
|---|---|---|
| „Connection refused" / Timeout beim `ReachyMini()`-Aufbau | `ConnectionRefusedError` im App-Logger-Baum oder kein Daemon-Heartbeat im journald | Daemon-Status: `systemctl status reachy-mini-daemon.service` (Wireless) oder Daemon-Terminal-Output (Lite); zweite App? siehe nächste Zeile |
| Andere App hält das Lock | `RobotAppLock: rejected — held by <app_name>` im Daemon-Log (über `reachy_mini.daemon.robot_app_lock` Logger, `manager.py` / `robot_app_lock.py`) | per Pollen-REST-API laufende App stoppen oder dem Daemon Zeit für Auto-Release lassen |
| Robot bewegt sich nicht | App-Logger zeigt `set_target`-Aufrufe ohne Fehler, aber keine Bewegung; SDK-Logger keine Warning | `mini.get_motor_status()` per Diagnose-Snippet aufrufen; ggf. `mini.enable_motors()`; siehe Pollens debugging.md § "Robot doesn't move" |
| Jerky / unsaubere Bewegung | keine eindeutige Log-Spur — meistens kein Logging-Problem, sondern Code-Pfad | Pollens debugging.md § "Jerky motion": Single-Owner-Loop bei 50–100 Hz, kein Mix aus `goto_target` und `set_target` |
| Import-Error beim App-Start | `ModuleNotFoundError: reachy_mini` im App-Subprozess-stderr → erscheint als `runner.error` im Daemon-Log | im App-`pyproject.toml` ist `reachy-mini` als Dependency gepinnt? venv aktiv? `uv pip install -e .` im App-Verzeichnis |
| Audio-Wiedergabe fehlgeschlagen | `Failed to initialize media server` im Daemon-Logger; ggf. GStreamer-Warning | Plattform prüfen (Lite hat Audio-Backend, Sim oft nicht); Pollens debugging.md § "Sim vs Physical"; SDK-Param `media_backend="gstreamer_no_video"` als Fallback |
| „Motors in different states" | sporadische Effort-/Position-Out-of-Range-Warnings im Daemon-Logger | Recovery-Pattern aus Pollens [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md): goto SLEEP_HEAD_POSE → `disable_motors()` |

- **MUSS [MUST]** jede neu auftretende Failure-Klasse vor Spec-Update gegen diese Tabelle abgeglichen werden; passt nichts, ist es eine Open Question, kein stiller Eintrag
- **SOLLTE [SHOULD]** der App-Entwickler bei jeder Failure-Klasse zuerst die Spalte „Erwartetes Log-Muster" prüfen, bevor er auf neuen Code-Pfaden testet — fehlt das erwartete Muster, ist das Problem in einer anderen Klasse als angenommen

### Verify-Basics-First-Heuristik

Pollens [`skills/debugging.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md) § "First: Verify Basic Connectivity" formuliert eine konkrete Heuristik, die in dieser Spec verbindlich übernommen wird:

- **MUSS [MUST]** vor jeder Diagnose einer App-spezifischen Failure-Klasse zuerst [`examples/minimal_demo.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py) gegen den **gleichen Daemon, im gleichen Lauf-Modus** ausgeführt werden — schlägt das fehl, ist die Failure-Klasse Connectivity-/Daemon-/Hardware-bezogen, nicht App-bezogen
- **MUSS [MUST]** das Ergebnis dieses Sanity-Checks im Triage-Bericht dokumentiert werden (Fehler reproduziert? grün gelaufen?), damit nachgelagerte Reviewer die Klassifikation nachvollziehen können
- **DARF NICHT [MUST NOT]** mit App-Code-Änderungen begonnen werden, solange der Sanity-Check nicht grün ist — das wäre Symptom-Bekämpfung an der falschen Stelle

### Recovery-Aktionen

| Aktion | Plattform | Befehl | Wirkung |
|---|---|---|---|
| Daemon-Restart | Wireless | `ssh pollen@reachy-mini.local "sudo systemctl restart reachy-mini-daemon.service"` | beendet alle App-Locks, beendet aktive Apps, startet den Daemon-Prozess sauber |
| Daemon-Restart | Lite | Strg-C im Daemon-Terminal, dann `reachy-mini-daemon` (oder `--verbose` / `--sim`) erneut | wie oben, manuell |
| Motor-Recovery | Wireless / Lite | Pollens Safe-Torque-Pattern: `mini.goto_target(head=SLEEP_HEAD_POSE); mini.disable_motors()` (Quelle: [`skills/safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md)) | Anti-Jerk vor Disable, danach mechanisch sicher |
| App-Lock-Force-Release | Wireless / Lite | nicht offiziell — Daemon-Restart ist der unterstützte Pfad | siehe Daemon-Restart |

- **DARF NICHT [MUST NOT]** ein Daemon-Restart als Routine-Schritt in einem CI-/Test-Lauf eingebaut werden — er maskiert Failure-Klassen, die durch saubere Lock-/Lifecycle-Logik korrekt zu lösen sind

## Akzeptanzkriterien

- [ ] Die Spec listet alle drei Logger-Bäume (App-Code, SDK, Daemon) plus die zwei stdout-/stderr-Capture-Pfade unter Daemon-Hosting in einer Tabelle
- [ ] Pro Plattform (Wireless / Lite / Simulation) und pro Lauf-Modus (Daemon-Hosting / direkt) ist die Senke jedes Logger-Baums benannt
- [ ] Die App-Code-Konvention `logging.getLogger(__name__)` ist als MUST formuliert, mit klarer Begründung gegen `print(...)` und gegen Wurzel-Logger-Re-Konfiguration
- [ ] Das Daemon-Hosting-Verhalten von `print(...)` und stderr ist auf Quell-Code-Stellen in `apps/manager.py` belegt
- [ ] Die drei Log-Level-Stellschrauben (SDK-Konstruktor, Daemon-CLI-Flag, App-eigener Logger) sind benannt und gegeneinander abgegrenzt
- [ ] Der Common-Issues-Triage-Katalog deckt jede in Pollens `skills/debugging.md` aufgeführte Klasse ab und nennt pro Klasse das erwartete Log-Muster und die erste Aktion
- [ ] Die Verify-Basics-First-Heuristik (`examples/minimal_demo.py` vor App-Diagnose) ist als MUST verankert
- [ ] Recovery-Aktionen (Daemon-Restart pro Plattform, Motor-Recovery via Safe-Torque) sind als Tabelle dokumentiert
- [ ] Cross-Refs auf [`host-provisioning`](../host-provisioning/de.md) (Production-Logging), [`reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md) (Test-Agent-Tailing) und [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md) (idiomatic SDK-Use) sind sichtbar
- [ ] Quell-Verweise auf Pollen-**Code**-Dateien zeigen auf Datei + Zeilen-Nummer; Verweise auf Pollen-**Markdown**-Quellen (`AGENTS.md`, `skills/*.md`) sind Datei-Level zitiert
- [ ] Keine MUST-Klausel verlangt eine API-Funktion, deren Existenz nicht in `src/reachy_mini/` belegt ist (Konsistenz mit der `deep-dive-docs`-MUST aus [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md))

## Quellen

> Code-Source-Verweise sind gegen `pollen-robotics/reachy_mini@main` zum Stand 2026-05-06 verifiziert; bei Pollen-Refactors driften die Zeilen-Nummern still und werden über einen späteren Drift-Audit nachgezogen. Markdown-Quellen werden auf Datei-Ebene zitiert, da sie keine stabilen Zeilen-Anker tragen.

- Pollen-Skill `debugging` (kanonische Triage-Heuristik, Common-Issues-Inventar, Verify-Basics-First): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- SDK-Hauptklasse (`log_level`-Konstruktor-Parameter, `self.logger = logging.getLogger(__name__)`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>
- Daemon-Implementierung (Daemon-Logger-Setup, Default-Level, Media-Server-Bring-up): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py>
- App-Manager (App-Subprozess-Spawn mit `-u`, stdout/stderr-Capture, Runner-Child-Logger, Error-Heuristik): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Robot-App-Lock-Logger (Lock-Acquire / -Release / -Konflikt-Pfade): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/robot_app_lock.py>
- Pollens `AGENTS.md` (Einstiegspunkt, JS-vs-Python-Surface, Log-Konventions-Verweise): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Pollen-Skill `safe-torque` (Recovery-Pattern bei Motor-State-Mismatch): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Pollen-Beispiel `minimal_demo.py` (kanonischer Sanity-Check): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>
- Pollens Hardware-Troubleshooting-Doku (Hardware-Recovery, abgegrenzt): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting>
- Interne Cross-Refs: [`reachy-mini/host-provisioning`](../host-provisioning/de.md) (production journald), [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md) (test-agent log-tailing), [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md) (idiomatic SDK)

## Offene Fragen

- Soll der App-Stub im `app-scaffold`-Skill standardmäßig einen `logging.basicConfig(...)`-Block am `main()`-Eintrittspunkt mit Default-Format-String setzen, oder bleibt das dem App-Entwickler überlassen?
- Strukturiertes Logging (JSON-Lines / OpenTelemetry-Bridge): hat das auf Reachy Mini einen Use-Case, der die zusätzliche Komplexität rechtfertigt? Vorschlag: erst, wenn ein Konsument das tatsächlich braucht — bis dahin Standard-Format-String.
- Log-Rotation auf Wireless-Geräten: journald-Vacuum ist in [`host-provisioning`](../host-provisioning/de.md) abgedeckt — gibt es Entwicklungs-Szenarien, in denen ein Lokal-Daemon-Run zu großen Log-Mengen führt, die eigene Vorkehrungen brauchen?
- Cross-Plattform-Identifikation des Logger-Bestands: liefert `logging.Logger.manager.loggerDict` zur Laufzeit eine zuverlässige Selbstauskunft über alle aktiven `reachy_mini.*`-Logger, oder braucht es einen separaten Diagnose-Snippet?
- Pollens `apps/manager.py:206–209` Stderr-Heuristik: welche genauen Marker-Strings klassifizieren als `error` vs. `warning`? Die Spec verweist aktuell pauschal — soll die Klassifikations-Liste nachgezogen werden, sobald Pollens Code aktualisiert wird?
- Jupyter-/IPython-Sitzungen als Entwicklungs-Modus: gilt das gleiche `logging.basicConfig`-Default wie für `pytest`, oder fängt IPython die Streams anders?
