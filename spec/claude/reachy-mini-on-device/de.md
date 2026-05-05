# On-Device-Test-Agent für Reachy Mini

Status: draft

## Kontext
Sobald die Hardware da ist, müssen Behaviors auf dem echten Reachy Mini überprüft werden, bevor sie in eine App oder eine Hugging-Face-Veröffentlichung wandern. Manuell heißt das: SSH oder USB anbinden, Code synchronisieren, Abhängigkeiten installieren, Behavior starten, Logs und Telemetrie sammeln, bei Fehlverhalten stoppen, ein Protokoll schreiben. Diese Schritte sind sequenziell, latenz-lastig, fehleranfällig und produzieren viel Rohausgabe — genau die Art von Aufgabe, die den Hauptthread eines Claude-Code-Gesprächs zumüllt, wenn sie inline läuft. Der Agent `reachy-mini-on-device` kapselt diesen Lifecycle in einer eigenen Tool-Session und liefert dem Hauptthread nur eine knappe, strukturierte Zusammenfassung zurück. Er testet, er entwickelt nicht — Code-Anpassungen bleiben Aufgabe des Hauptthreads, der die Skills `reachy-mini-sdk`, `app-scaffold` und `home-assistant-bridge` nutzt.

## Ziele
- Ein Behavior wird mit einem einzigen Aufruf auf das echte Gerät gebracht und live ausgeführt
- Während des Laufs werden Telemetrie und Logs strukturiert eingesammelt, ohne den Hauptkontext zu fluten
- Fehlverhalten führt zu kontrolliertem Notstopp und sauberer Trennung vom Gerät, nicht zu hängenden Verbindungen
- Das Ergebnis kommt als knappe PASS/FAIL-Zusammenfassung mit Verweis auf einen Volltext-Log-Artefakt
- Der Agent bleibt narrow: Test- und Beobachtungs-Aufgabe, keine Bewegungs-Logik, keine Code-Änderung am Behavior

## Nicht-Ziele
- Hardware-Bringup (eigener Skill geplant)
- Firmware-Flash (eigener Skill geplant)
- Entwicklung des Behaviors oder der App (eigene Repos, eigene Skills)
- Veröffentlichung des Behaviors auf Hugging Face (`behavior-publish-hf`, geplant)
- Audio-/Beat-Tracking (`audio-beat-tracking`, geplant)
- Dauerhafter Betrieb / Watchdog im Produktiv-Setup — der Agent ist ein Test-Lifecycle, kein Daemon

## Skill-vs-Agent-Begründung
Diese Aufgabe wird als **Agent** und nicht als Skill modelliert, weil mehrere Begründungen aus `nolte-shared/spec/claude/skill-vs-agent/` simultan zutreffen:

- **Lange, latenz-lastige Tool-Session** — SSH/USB-Connect, scp/rsync-Deploy, abwartendes Behavior-Step-Watching mit Sekunden- bis Minuten-Latenzen pro Phase. Skills sind für interaktive Inline-Workflows optimiert; ein langer, sequenzieller Lifecycle gehört in eine eigene Tool-Session.
- **Kontext-Volumen** — Rohe Behavior-Logs und Sensor-Telemetrie können tausende Zeilen pro Lauf erzeugen. Im Hauptthread würden sie den Kontext verschlingen, ohne der nachgelagerten Entscheidung zu nützen. Der Agent reduziert das auf eine Zusammenfassung und einen Datei-Artefakt-Pfad.
- **Mehrstufige Orchestrierung mit Fehler-Recovery** — Connect → Deploy → Install Deps → Start → Watch → Stop → Disconnect. Jede Stufe hat eigene Failure-Modes (Auth-Fehler, Disk-Full, Behavior-Crash, USB-Disconnect), die eigene Recovery brauchen, ohne den Hauptthread zu unterbrechen.
- **Eigenständiges Tool-Set** — Der Agent braucht Bash für `ssh`/`scp`/`rsync` plus ggf. ein gerätespezifisches CLI. Diese Tools gehören nicht in den Hauptthread, der Code-Editing dominiert.
- **Spezialisiertes Verhalten** — Sicherheits-Notstopp, gracefuler Cleanup bei Disconnect und striktes Output-Format sind Anforderungen, die ein dedizierter Agent kanonisiert.
- **Distribution: `plugin`** — der Agent gehört zum Plugin und wird mit ihm verteilt.

## Anforderungen

### Eingaben
- **MUSS [MUST]** den Ziel-Behavior-Pfad annehmen (lokales Verzeichnis im konsumierenden App-Repo)
- **MUSS [MUST]** die Plattform annehmen — `wireless` / `lite` / `simulation` —, weil sich Deploy-Pfad, Telemetrie-Verfügbarkeit und Notstopp-Mechanik plattform-spezifisch unterscheiden (siehe Plattform-Profile-Sektion)
- **MUSS [MUST]** plattform-passend die Verbindungs-Adresse annehmen: für `wireless` einen SSH-Host (`user@host` plus optionaler Identitäts-Pfad) zur Roboter-IP; für `lite` einen SSH-Host zum **Host-PC**, der den Reachy via USB-C hält; für `simulation` keinen Host (das Behavior läuft im selben Python-Prozess)
- **MUSS [MUST]** ein hartes Timeout pro Lauf annehmen; nach Timeout wird der Notstopp-Pfad ausgelöst
- **SOLLTE [SHOULD]** einen Trigger-Modus annehmen: `autonomous` (Behavior läuft ohne externe Trigger) vs. `interactive` (HA-Event löst Step aus, optional über `home-assistant-bridge`-Patterns)
- **KANN [MAY]** zusätzliche Optionen annehmen: Trockenlauf (Deploy ohne Run), nur-Watch (kein Deploy), Verbosity der Zusammenfassung

### Plattform-Profile

Der Agent unterscheidet zwischen den drei Reachy-Mini-Plattformen, weil sich Deploy-Pfad, Telemetrie-Verfügbarkeit und Sicherheits-Schwellen materiell unterscheiden:

- **Reachy Mini** (Wireless) — autark mit RPi 4 CM4 + LiFePO4-Akku. Deploy direkt zur Roboter-IP via SSH oder über die Daemon-REST-API. Volle Telemetrie: IMU (Accelerometer, Gyroscope, Quaternion, Temperatur), Battery-Polling, daemon-publizierte Joint-Positions/Head-Pose mit 50 Hz. Notstopp-Trigger können Strom-Spike, IMU-Temperatur-Schwelle und Battery-Brown-out einschließen.
- **Reachy Mini Lite** — am Host-PC via USB-C, externe 6,8–7,6 V-Versorgung. Deploy primär über den Host-PC: SSH zum Host, dort den Pollen-Daemon ansprechen — **nicht** SSH zum Reachy direkt. Aktuator-Set ist identisch zu Wireless, aber **keine IMU-Telemetrie** und **kein Battery-Sensor** — Sicherheits-Schwellen kommen aus Stewart-Joint-Limits (URDF) und vom Daemon publizierten Effort/Strom-Daten, falls verfügbar. IMU-Daten sind kein FAIL-Kriterium.
- **Simulation** — `ReachyMini(spawn_daemon=True, use_sim=True)` im selben Python-Prozess. **Kein Deploy nötig**, kein SSH, keine USB-Verbindung. Voll deterministisch. Keine reale Sensor-Telemetrie (außer Pose-Read), keine Audio-Wiedergabe, keine LED-Effekte. PASS/FAIL-Kriterien fokussieren sich auf Pose-Erreichbarkeit, Move-Lifecycle und Logik — nicht auf physische Welt-Effekte.

Anforderungen:

- **MUSS [MUST]** beim Behavior-Start die Plattform aus Eingabe und (zur Verifikation) aus dem `DaemonStatus` lesen — Mismatch zwischen Eingabe und tatsächlicher Plattform ist ein FAIL
- **MUSS [MUST]** plattform-spezifische PASS/FAIL-Kriterien anwenden: „IMU-Telemetrie fehlt" ist auf Lite und Simulation **kein** FAIL; auf Wireless ist es eines
- **MUSS [MUST]** den Deploy-Pfad in der Output-Zusammenfassung explizit ausweisen: `via_ssh_direct` (Wireless), `via_host_usb` (Lite), `in_process` (Simulation)
- **MUSS [MUST]** auf Simulation explizit kennzeichnen, welche Akzeptanzkriterien _nicht_ geprüft werden konnten (z. B. echte Pose-Erreichung, Audio-Sync, Servo-Wärme) und das im Output-Protokoll als `not_applicable_in_simulation`-Liste ausgeben
- **DARF NICHT [MUST NOT]** Lite-Telemetrie-Lücken (fehlende IMU-/Battery-Daten) als „Sensoren offline" werten — das ist die Norm, kein Defekt

### Lifecycle
- **MUSS [MUST]** den Lifecycle in dieser Reihenfolge ausführen: connect → **robot-busy check** → sync code → install deps → start behavior → watch & sample → stop → disconnect
- **MUSS [MUST]** den `connect`-Schritt plattform-spezifisch ausführen: für `wireless` SSH zur Roboter-IP, für `lite` SSH zum Host-PC + Daemon-API-Probe, für `simulation` ein No-Op (`spawn_daemon=True, use_sim=True`-Konstruktor liefert die Verbindung im selben Prozess; reines `use_sim=True` ohne `spawn_daemon=True` würde mit `ConnectionError` scheitern)
- **MUSS [MUST]** vor `start behavior` einen **Robot-Busy-Check** durchführen: `GET http://<host>:8000/api/apps/current-app-status` und `GET /api/daemon/robot-app-lock-status`. Hält bereits eine andere App das Lock (`state != "free"` oder `current-app-status != null`), mit klarer Fehlermeldung abbrechen — `holder_name`/`current_app` benennen, plus den expliziten Hinweis, dass Pollen nur eine App zur Zeit erlaubt. Niemals eine zweite Session erzwingen.
- **MUSS [MUST]** den `sync code`- und `install deps`-Schritt auf Simulation überspringen (kein Deploy nötig)
- **MUSS [MUST]** in jeder Phase das beobachtete Ergebnis strukturiert protokollieren (Phase, Status, Dauer, Fehler-Klasse falls vorhanden)
- **MUSS [MUST]** bei Disconnect oder unerwartetem Behavior-Exit kontrolliert enden — keine hängenden SSH-Sessions, keine offen gelassenen Behavior-Prozesse
- **SOLLTE [SHOULD]** zwischen den Phasen einen Health-Check einschieben — auf Wireless mit IMU-Temperatur und Battery-Stand; auf Lite mit Daemon-Effort-/Strom-Daten falls verfügbar; in Simulation entfällt der Check

### Notstopp
- **MUSS [MUST]** den Notstopp **primär über `stop_event`** signalisieren (Pollens App-Lifecycle-Vertrag, `src/reachy_mini/apps/manager.py`): `stop_event.set()` + Wartezeit für graceful Cleanup; auf REST-Ebene `POST /api/apps/stop-current-app`. Dem Behavior wird ein konfigurierbares **Cleanup-Timeout** (Default 2 s) eingeräumt, in dem es seinen `run()` selbst zu Ende fährt
- **MUSS [MUST]** nach Ablauf des Cleanup-Timeouts hart eskalieren: `SIGTERM` → 1 s warten → `SIGKILL` falls noch lebend; danach den Pose-Reset auf `INIT_HEAD_POSE` (4×4-Identitäts-Matrix, Kopf zentriert) plus `INIT_ANTENNAS_JOINT_POSITIONS` selbst auslösen — Pose-Konstanten verifiziert in `src/reachy_mini/reachy_mini.py`
- **MUSS [MUST]** plattform-spezifische Trigger-Quellen für den Notstopp zulassen:
  - **Wireless**: IMU-Temperatur-Schwelle (`mini.imu["temperature"]`), Battery-Brown-out, Strom-Spike (vom Daemon publiziert), Bewegungs-Limit-Verstoß
  - **Lite**: Daemon-publizierte Effort-/Strom-Daten falls verfügbar, Bewegungs-Limit-Verstoß; **keine** IMU- oder Battery-Trigger
  - **Simulation**: nur Logik-Trigger (Pose außerhalb URDF-Grenze, Hook-Exception, Timeout); kein physischer Notstopp nötig, aber Pose-Reset trotzdem ausführen für Konsistenz
- **MUSS [MUST]** den Notstopp ohne Nutzer-Bestätigung auslösen, wenn ein plattform-passender Sicherheits-Threshold reißt
- **MUSS [MUST]** den Notstopp im Output-Protokoll als gesondertes Ereignis ausweisen, mit Trigger-Quelle, Plattform, und ob Graceful-Cleanup vor Eskalation gegriffen hat
- **DARF NICHT [MUST NOT]** den Notstopp auf Wireless oder Lite auf eine reine Log-Notiz reduzieren — die physische Konsequenz hat Vorrang
- **DARF NICHT [MUST NOT]** ohne Graceful-Phase direkt SIGKILL eskalieren — das `stop_event` ist der vertragliche Notausstieg; nur wenn es nicht greift, wird hart abgebrochen

### Ausgabe / Output-Format
- **MUSS [MUST]** in den Hauptthread nur eine strukturierte Zusammenfassung zurückgeben: Gesamt-Status (`PASS` / `FAIL` / `ABORTED`), Hooks-Statistik (welche Hooks aufgerufen, wie oft, mit welcher mittleren Latenz), Anomalien-Liste, Dauer
- **MUSS [MUST]** den Volltext-Log als Datei-Artefakt unter `.audits/on-device/<ISO-timestamp>-<behavior-name>.log` ablegen und den Pfad in der Zusammenfassung nennen
- **DARF NICHT [MUST NOT]** rohe Logs oder Sensor-Streams in den Hauptthread zurückgeben
- **SOLLTE [SHOULD]** ein Maschinen-lesbares Beiseite-Artefakt mitliefern (z. B. JSON mit denselben Daten), wenn Folge-Schritte automatisiert werden

### Sicherheit und Geheimnisse
- **MUSS [MUST]** SSH-/Geräte-Credentials aus der Umgebung oder aus einer ssh-config lesen, niemals als Argument im Klartext
- **DARF NICHT [MUST NOT]** Credentials in den Output-Log oder in die Zusammenfassung schreiben — Maskierung Pflicht, falls eine Identifier-Notation nötig ist
- **SOLLTE [SHOULD]** SSH-Host-Key-Verifikation aktivieren; bei einer Erst-Verbindung den Fingerprint im Output ausweisen, statt automatisch zu akzeptieren
- **MUSS [MUST]** plattform-bewusste Auth-Modelle anwenden:
  - **Wireless**: SSH zum Reachy direkt (Pi-OS-User), Daemon-Token im selben Pfad
  - **Lite**: SSH zum Host-PC (User des Operators), zusätzlich der Pollen-Daemon-Token, der vom Host an den Daemon weitergereicht wird
  - **Simulation**: keine SSH-Auth, keine Daemon-Token — alle Operationen laufen im selben Python-Prozess

### Boundaries
- **SOLLTE [SHOULD]** auf `reachy-mini-sdk` verweisen, sobald der Hauptthread Bewegungs-Idiomatik nach dem Test anpassen soll
- **SOLLTE [SHOULD]** auf `app-scaffold` verweisen, wenn der Test zeigt, dass das Behavior strukturell unvollständig ist
- **SOLLTE [SHOULD]** auf `home-assistant-bridge` verweisen, wenn der `interactive`-Modus mit echten HA-Events laufen soll
- **DARF NICHT [MUST NOT]** Inhalte aus diesen Skills duplizieren — der Agent ist Beobachter und Orchestrator, nicht Wissensbasis

## Akzeptanzkriterien
- [ ] Der Agent erkennt die Plattform (`wireless` / `lite` / `simulation`) aus Eingabe und verifiziert sie gegen `DaemonStatus`
- [ ] Der Output-Bericht weist den Deploy-Pfad explizit aus (`via_ssh_direct` / `via_host_usb` / `in_process`)
- [ ] Auf Simulation-Läufen erscheint eine `not_applicable_in_simulation`-Liste mit übersprungenen Kriterien
- [ ] Notstopp-Trigger sind plattform-spezifisch konfiguriert (IMU/Battery nur Wireless, Effort-Daten Lite, Logik-Trigger überall)
- [ ] Lite-Telemetrie-Lücken (IMU/Battery fehlen) erzeugen kein FAIL
- [ ] Der Agent ist unter `agents/reachy-mini-on-device.md` mit gültiger Frontmatter angelegt — `name: reachy-mini-on-device`, `description`, `distribution: plugin`, optional Tags
- [ ] Die `description` aktiviert auf Phrasings wie „test the behavior on the device", „deploy and run X on Reachy Mini", „live-trial behavior <name>"
- [ ] Eine Skill-vs-Agent-Begründung ist im Agent-Body sichtbar (mindestens Tool-Session-Länge, Kontext-Volumen, Orchestrierung)
- [ ] Lifecycle-Phasen (connect / deploy / install / start / watch / stop / disconnect) sind im Body dokumentiert
- [ ] Notstopp-Verhalten und Default-Sicherheits-Thresholds sind dokumentiert (mit TBD-Markern, wo Hardware-verifiziert werden muss)
- [ ] Eingabe-Parameter (Behavior-Pfad, Geräte-Adresse, Timeout, Trigger-Modus) sind dokumentiert
- [ ] Ausgabe-Format ist als striktes Schema dokumentiert; rohe Logs landen in `.audits/on-device/<timestamp>-<name>.log`
- [ ] `.audits/` ist in `.gitignore` enthalten, sodass Logs niemals committet werden
- [ ] Verweise auf `reachy-mini-sdk`, `app-scaffold`, `home-assistant-bridge` sind im Body sichtbar
- [ ] Der Agent wird vom Skill-Agent-Katalog-Generator akzeptiert (Frontmatter validiert, `name` matcht Dateinamen, `distribution` ist gesetzt)
- [ ] Aussagen ohne Hardware-Verifikation tragen einen `⚠ TBD: validate against real hardware`-Hinweis

## Quellen
- Upstream-SDK-Repo (`ReachyMini`-Klasse, `use_sim`-Konstruktor, Pose-Konstanten `INIT_HEAD_POSE` / `INIT_ANTENNAS_JOINT_POSITIONS`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/reachy_mini.py>
- Daemon-Implementierung (REST-API, App-Lock, Lifecycle, Status — entscheidet über `connect`/`watch`-Pfade): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- IO-Protokoll (Telemetrie-Messages `JointPositionsMsg`, `HeadPoseMsg`, `ImuDataMsg`, Befehle wie `SetMicrophoneVolumeCmd`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- IMU- und Audio-Beispiele (Messmuster für Sample-Streams im `watch`-Schritt): <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/imu_example.py>, <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/sound_record.py>
- Upstream-Claude-Skills `debugging` und `safe-torque` (parallele Sicherheits-/Diagnose-Heuristiken, gegen die die Notstopp-Logik abgeglichen wird): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>, <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Troubleshooting-Doku (Failure-Modes je Plattform): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/troubleshooting>

## Offene Fragen
- Welches Deploy-Protokoll ist kanonisch — `rsync` über SSH, `scp`, ein gerätespezifisches Tool, oder unterstützt das SDK Remote-Run direkt?
- Welches Telemetrie-Format liefert das SDK (Events, Sample-Streams, Log-Lines)? Davon hängt das Output-Schema ab.
- Welche minimale Ruhepose ist für den Notstopp sicher? Vorschlag: alle Joints in Mittellage, Antennen neutral. Endgültig vor erster Hardware-Inbetriebnahme bestätigen.
- Welche Sicherheits-Thresholds (Strom, Temperatur, Bewegungs-Limits) sind ab Werk verfügbar, welche müssen wir im Agent selbst messen?
- Soll der Agent Multi-Run-Vergleiche unterstützen (zwei Läufe vergleichen, um Regressionen zu erkennen), oder strikt einen Lauf pro Aufruf?
- Wie integriert sich der Agent mit CI? Vorschlag: Auf Hardware-Lauf nur lokal/auf einem Hardware-Runner, in der CI nur Trockenlauf-Modus.
- Wie verhält sich der Agent, wenn das SDK selbst eine Test-/Mock-Schicht bietet — fällt er darauf zurück, statt echte Hardware zu erwarten? Tendenz: nein, Mock ist Sache eines separaten Skills.
- Welche Maximal-Größe darf das Log-Artefakt haben, bevor rotation oder Trimm-Strategien greifen?
