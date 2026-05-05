# App-Architektur: Reachy-Mini-Show

Status: draft

## Kontext
Dieses Repository (`claude-reachy-mini`) liefert Skills, Agents und Specs als Toolbox für die Entwicklung. Die konkrete Anwendung, die mit dieser Toolbox entsteht, ist eine **Pollen-Reachy-Mini-App** — ein Python-Paket, das der Reachy-Daemon als Subprozess auf dem Roboter startet und das die 29 Motion-Specs aus diesem Plugin in lebende Behaviors überführt. Diese Spezifikation legt das App-Layout, den Lifecycle, die Befehls-Schnittstelle und den Distributionspfad fest. Sie ist die Quelle der Wahrheit, gegen die die Skills `app-scaffold` und `reachy-mini-sdk` und der Agent `reachy-mini-on-device` ihre Vorschläge ausrichten. Die App lebt in einem **separaten App-Repository** (Vorschlag: `nolte/reachy-mini-show`) — dieses Plugin-Repository selbst enthält keinen App-Code.

## Ziele
- Eine einzige App, die alle 29 Motion-Slugs als `Move`-Subklassen implementiert
- Volle Konformität mit Pollens App-System: Daemon-Subprozess, eine App pro Zeit, Hugging-Face-Spaces als Distribution
- Live-Befehl-Annahme über lokalen WebSocket — späterer Konsum durch externe Integrationen (HA, Wyoming-Bridge) im jeweiligen Konsumenten-Repository
- Lokal entwickelbar via `ReachyMini(spawn_daemon=True, use_sim=True)` — keine Hardware nötig zum Starten
- **Sichtbare Provenienz**: jedes Repository, das die App liest, sieht den Verweis auf Claude Code und auf `claude-reachy-mini` als Quelle der Behavior-Specs und Authoring-Skills

## Nicht-Ziele
- HA-Custom-Integration (separates Repository, mit eigenem Plugin/Skill-Set)
- Wyoming-Voice-Stack (separater systemd-Service auf Reachy, nicht Bestandteil dieser App)
- Cloud-AI-Aufrufe aus dem App-Prozess heraus
- Mehrere parallele Apps (Pollens Limit ist eine App pro Zeit)
- Sandboxing oder Privilege Separation (Pollen liefert das nicht)
- Authentifizierung auf dem WebSocket — er ist `localhost`-only

## Anforderungen

### App-Identität
- **MUSS [MUST]** den Slug `reachy-mini-show` durchgehend für Repo-Name, Python-Package-Name und HF-Space-Name tragen
- **MUSS [MUST]** als Hugging-Face-Space gepackt sein, mit dem Tag `reachy_mini_python_app` im README-Frontmatter (sonst keine Discovery im Reachy-Dashboard)
- **MUSS [MUST]** im `pyproject.toml` einen Entry-Point in der Group `reachy_mini_apps` deklarieren, der die App-Klasse benennt — der Daemon entdeckt Apps ausschließlich über diese Group:

  ```toml
  [project.entry-points."reachy_mini_apps"]
  reachy-mini-show = "reachy_mini_show.main:ReachyMiniShowApp"
  ```

- **MUSS [MUST]** semantische Versionierung nutzen
- **MUSS [MUST]** den `reachy_mini`-SDK-Pin auf eine konkrete Minor-Version setzen (z. B. `^1.7.0`); SDK-Major-Update ist immer eine bewusste Re-Validierung

### Repository-Layout
Layout konform zur Pollen-Robotics-CLI `reachy-mini-app-assistant` (Default-Template), erweitert um Provenienz-Marker (`CLAUDE.md`, Plugin-URL):

```
reachy-mini-show/
├── pyproject.toml              # Pollen-konformes Format, SDK-Pin, Provenienz-URLs,
│                               #   [project.entry-points."reachy_mini_apps"]
├── README.md                   # HF-Frontmatter `reachy_mini_python_app`, Provenienz-Notiz
├── CLAUDE.md                   # Verweis auf claude-reachy-mini Plugin (Authoring-Quelle)
├── index.html                  # Hugging-Face-Space-Landing-Page (Pollen-CLI-Default)
├── style.css                   # Landing-Page-Style (Pollen-CLI-Default)
├── reachy_mini_show/
│   ├── __init__.py
│   ├── main.py                 # ReachyMiniApp-Subklasse + __main__ → wrapped_run()
│   ├── server.py               # WebSocket-Server :8765 (show-spezifische Erweiterung)
│   ├── behaviors/
│   │   ├── __init__.py         # Slug → Klasse Registry
│   │   ├── base.py             # gemeinsame Move-Subklasse
│   │   ├── emotions/           # happy, sad, angry, surprised, excited, sleepy,
│   │   │                       #   confused, curious, disappointed, proud, shy, disgust
│   │   ├── social/             # greeting-wave, farewell-wave, bow, peek,
│   │   │                       #   agreeing-nod, disagreeing-shake, recognition
│   │   ├── state/              # waiting-idle, alert-listening, thinking
│   │   ├── dance/              # groove-bob, sway-side, headbang-soft, spin-look-around
│   │   └── defensive/          # flinch, alarm, scanning
│   ├── audio/                  # WAV-Samples (F32LE, 48 kHz, 2 ch — Pollen-konform)
│   ├── static/                 # optional: Web-UI-Assets, falls custom_app_url gesetzt
│   └── config.py               # Defaults und Plattform-Profile
└── tests/                      # Unit-Tests gegen ReachyMini(spawn_daemon=True, use_sim=True)
```

Das Skelett wird **nicht von Hand** angelegt, sondern über das offizielle CLI:

```bash
uv pip install reachy-mini
reachy-mini-app-assistant create reachy-mini-show /pfad/zum/ziel        # ohne HF-Push
reachy-mini-app-assistant create reachy-mini-show /pfad/zum/ziel --publish   # legt HF-Space + Git-Remote an
```

Manuelle Skelette weichen subtil von Pollens Erwartungen ab und brechen beim ersten Daemon-Lauf.

### Provenienz-Marker (Pflicht)

- **MUSS [MUST]** in `README.md` unmittelbar nach dem HF-Frontmatter einen Provenienz-Block tragen mit (1) einem Verweis auf das Claude-Code-Plugin `claude-reachy-mini` (`https://github.com/nolte/claude-reachy-mini`), (2) einem Verweis auf den Motion-Catalog (`spec/reachy-mini/motions/`), (3) einem Verweis auf diese Architektur-Spec
- **MUSS [MUST]** eine `CLAUDE.md` im App-Repo-Root tragen, die die empfohlenen Plugin-Skills (`reachy-mini-sdk`, `app-scaffold`, Agent `reachy-mini-on-device`) namentlich auflistet und auf das Plugin-Repo verlinkt
- **MUSS [MUST]** im `pyproject.toml` unter `[project.urls]` mindestens diese Einträge tragen: `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
- **SOLLTE [SHOULD]** ein Code-Header in `main.py` einen einzeiligen Verweis tragen: `# Behaviors derived from spec/reachy-mini/motions/ in nolte/claude-reachy-mini`

### Lifecycle
- **MUSS [MUST]** Pollens App-Vertrag implementieren: eine Klasse `ReachyMiniShowApp(ReachyMiniApp)` aus `reachy_mini` mit der Pflicht-Methode `run(self, reachy_mini: ReachyMini, stop_event: threading.Event)`. Der Daemon ruft `run()` mit einer bereits verbundenen Instanz auf und sendet `SIGINT`, der `stop_event.set()` triggert.
- **MUSS [MUST]** in `main.py` einen `if __name__ == "__main__":`-Block tragen, der `ReachyMiniShowApp().wrapped_run()` aufruft (für Direkt-Lauf via `python -m reachy_mini_show.main`); `wrapped_run()` übernimmt Connect, optionale Services, und ruft dann `run()`.
- **MUSS [MUST]** innerhalb von `run()` drei parallele Tasks unter `asyncio.run(...)` starten: WebSocket-Server, Behavior-Worker (liest Queue, ruft `reachy_mini.async_play_move(...)`), Idle-Loop (wenn Queue leer und kein Behavior aktiv → läuft `waiting-idle` oder konfigurierter Idle-Mode)
- **MUSS [MUST]** auf `stop_event` alle drei Tasks sauber beenden, laufendes Behavior via `reachy_mini.cancel_move()` abbrechen und Reachy in `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS` fahren — danach kehrt `run()` zurück, der Daemon setzt das Robot in seine Default-Pose
- **MUSS [MUST]** bei einer Task-Exception alle anderen Tasks sauber beenden und in eine Sicherheitspose fahren — keine hängenden Verbindungen, keine eingefrorene Pose
- **DARF NICHT [MUST NOT]** Hardware-Reconnect in der App selbst implementieren — Pollens Daemon übergibt eine bereits verbundene Instanz; Verbindungs-Lifecycle gehört dem Daemon
- **DARF NICHT [MUST NOT]** den Lifecycle als freie `main(reachy, stop_event)`-Funktion modellieren — der Daemon erwartet die `ReachyMiniApp`-Subklasse, sonst greift weder Discovery noch der `__main__`-Direktlauf-Pfad

### Behavior-Implementierung
- **MUSS [MUST]** pro Motion-Slug genau eine `Move`-Subklasse haben, organisiert in den Kategorie-Unterordnern (`emotions/`, `social/`, `state/`, `dance/`, `defensive/`)
- **MUSS [MUST]** jede Klasse eine `duration: float`-Property und eine `evaluate(t: float)`-Methode implementieren, gemäß Pollen-`Move`-ABC
- **MUSS [MUST]** eine Slug-Registry in `behaviors/__init__.py` führen, die Slug-String auf Klasse mappt — das ist die Lookup-Quelle für Befehle
- **MUSS [MUST]** Phasen-Werte aus den Motion-Specs (Pose-Δ, Antennen, Body-Yaw, Easing, Dauer pro Phase) 1:1 übernehmen — keine willkürlichen Anpassungen
- **MUSS [MUST]** BPM-parametrisierte Tanz-Bausteine (`groove-bob`, `sway-side`, `headbang-soft`) konstruktor-parametrisiert anbieten: `GrooveBob(bpm: float, beats: int, lead_time_s: float = 0.0)`
- **SOLLTE [SHOULD]** für loopfähige Behaviors (`waiting-idle`, `alert-listening`, `thinking`) einen `loop_count: int | None`-Parameter akzeptieren — `None` heißt unbegrenzt loopen, bis externes Stop-Signal

### Befehls-Schnittstelle (lokaler WebSocket)
- **MUSS [MUST]** einen WebSocket-Server auf `127.0.0.1:8765` (Port konfigurierbar via ENV) bereitstellen
- **MUSS [MUST]** ein **Protokoll-Versions-Feld** `protocol_version` in jedem Command und jedem Event tragen, im Format `<major>.<minor>` (z. B. `"1.0"`); Major-Wechsel kennzeichnen Breaking Changes
- **MUSS [MUST]** Commands mit nicht-unterstützter **Major**-Version mit einem `error`-Event mit `code: "unsupported_protocol_version"` ablehnen, ohne den Behavior-Worker zu beeinflussen
- **SOLLTE [SHOULD]** Minor-Version-Differenzen tolerieren (forward-compatible) — neue optionale Felder ignorieren, fehlende neue Felder durch Default ersetzen
- **MUSS [MUST]** JSON-Messages annehmen mit den folgenden Command-Typen:

  ```jsonc
  {"type": "play_behavior", "protocol_version": "1.0", "slug": "happy", "speed": 1.0}
  {"type": "cancel", "protocol_version": "1.0"}
  {"type": "set_idle_mode", "protocol_version": "1.0", "mode": "waiting-idle"}
  {"type": "set_dance", "protocol_version": "1.0", "block": "groove-bob", "bpm": 110, "beats": 16}
  {"type": "speak", "protocol_version": "1.0", "text": "...", "behavior_during": "thinking"}
  {"type": "get_status", "protocol_version": "1.0"}
  ```

- **MUSS [MUST]** JSON-Events broadcasten, jeweils mit `protocol_version`-Feld:

  ```jsonc
  {"type": "behavior_started", "protocol_version": "1.0", "slug": "happy", "started_at": "<iso>"}
  {"type": "behavior_finished", "protocol_version": "1.0", "slug": "happy", "status": "PASS|FAIL|ABORTED", "duration_s": 2.4}
  {"type": "low_battery", "protocol_version": "1.0", "percentage": 18}
  {"type": "error", "protocol_version": "1.0", "code": "unsupported_protocol_version|invalid_command|behavior_not_found|...", "message": "..."}
  ```

- **MUSS [MUST]** auf `get_status` eine Antwort liefern, die `supported_protocol_versions: ["1.0", ...]` als Array trägt, sodass Konsumenten ihren eigenen Versions-Match feststellen können
- **MUSS [MUST]** eine offene Queue verwalten: ein neues `play_behavior` während eines laufenden Behaviors bricht das laufende ab (per Pollen-Konvention läuft nur ein Move zur Zeit)
- **MUSS [MUST]** mehrere parallele Clients erlauben — alle Clients erhalten alle Events; Commands-Sender ist beliebig
- **DARF NICHT [MUST NOT]** TLS oder Auth verlangen — `localhost`-only; bei externer Erreichbarkeit ist das Aufgabe einer separaten Reverse-Proxy-Schicht im Konsumenten-Setup

### Audio-Asset-Management
- **MUSS [MUST]** Audio-Files unter `audio/` ablegen, Format WAV mit `F32LE`, 48 kHz, 2 Channels (Pollen-Pipeline-konform)
- **MUSS [MUST]** Audio-Pfade über `importlib.resources` referenzieren — keine harten Pfade
- **SOLLTE [SHOULD]** Lautstärke-Default je Behavior in der Move-Klasse als Konstante dokumentieren

### Konfiguration
- **MUSS [MUST]** Defaults in `config.py` als Dataclass halten
- **MUSS [MUST]** ENV-Var-Overrides unterstützen: `REACHY_SHOW_PORT`, `REACHY_SHOW_IDLE_MODE`, `REACHY_SHOW_LOG_LEVEL`
- **DARF NICHT [MUST NOT]** HA-spezifische Config-Werte (HA-URL, HA-Token) führen — diese leben im Konsumenten-Repo
- **KANN [MAY]** ein Settings-Web-UI exponieren, indem `custom_app_url` auf der `ReachyMiniApp`-Subklasse gesetzt wird (z. B. `"http://0.0.0.0:8042"`); Pollen startet dann automatisch einen FastAPI-Server, der `static/` aus dem Package serviert. Das Dashboard zeigt das Settings-Icon und öffnet die UI unter `http://localhost:8042` (Lite/Sim) bzw. `http://reachy-mini.local:8042` (Wireless). Wenn nicht benötigt, `custom_app_url = None` setzen.

### Plattform-Profile
- **MUSS [MUST]** zwischen Wireless / Lite / Simulation unterscheiden, basierend auf SDK-Capability-Discovery
- Wireless: vollständig (IMU-Reads aktiv, Battery-Polling aktiv, alle Behaviors)
- Lite: keine IMU-Reads, kein Battery-Polling; sonst voll
- Simulation: keine Audio-Wiedergabe, keine Sensor-Events außer Pose-Read

### Entwicklungs- und Distributionspfade

Entwicklung findet auf dem **Notebook** des Entwicklers statt, nicht auf dem Roboter selbst (auch wenn der Wireless einen RPi 4 CM4 trägt — das ist Lauf-, keine Dev-Hardware). Es gibt zwei Sim-Pfade — wähle einen:

**A) Externer Daemon, App-Prozess separat** (Standard für die Show-App, weil App-Logs sauber von Daemon-Logs getrennt sind):

```bash
reachy-mini-app-assistant create reachy-mini-show .
uv venv && source .venv/bin/activate
uv pip install -e .

# Terminal 1 — Daemon
reachy-mini-daemon --sim                      # voller Sim-Pfad mit MuJoCo-Viewer
# oder, wenn GStreamer/MuJoCo nicht installiert sind:
reachy-mini-daemon --mockup-sim --no-media --headless

# Terminal 2 — App (verbindet sich gegen localhost:8000)
python -m reachy_mini_show.main
```

**B) In-Process-Daemon im Test-/Smoke-Pfad** (für Unit-Tests, kein zweites Terminal nötig):

```python
with ReachyMini(spawn_daemon=True, use_sim=True) as mini:
    ...
```

`spawn_daemon=True` bootet einen Daemon-Subprozess innerhalb des Python-Prozesses; `use_sim=True` allein reicht **nicht** — ohne `spawn_daemon=True` versucht das SDK eine externe Daemon-Verbindung und scheitert mit `ConnectionError`. Das ist eine wichtige API-Falle, die die offizielle Doku verschweigt.

**System-Dependencies für den vollen `--sim`-Pfad** (MuJoCo-Viewer + GStreamer-WebRTC):

- `gir1.2-gst-plugins-base-1.0`, `gir1.2-gstreamer-1.0`, `python3-gi` (Debian/Ubuntu) — sonst `ValueError: Namespace GstApp not available` beim Daemon-Start
- MuJoCo (Python-Wheel kommt automatisch)

Wer ohne GStreamer entwickelt, nutzt `--mockup-sim --no-media --headless` plus `request_media_backend = "no_media"` auf der App-Klasse — funktioniert vollständig für Pose-, Antennen- und Body-Yaw-Tests, nur Audio/Video bleiben aus.

Drei kanonische Deploy-Pfade nach Wireless:

1. **Hugging-Face-Space (Standard, mit Internet)** — `git push <hf-remote>` aus dem App-Repo. Sobald der Tag `reachy_mini_python_app` im README-Frontmatter steht, erscheint die App im Reachy-Dashboard und ist mit einem Klick installierbar.
2. **Daemon-REST-API direkt** — funktioniert gegen jeden erreichbaren Daemon (Wireless via `reachy-mini.local:8000`, Lite via Host-PC, Sim via `localhost:8000`). Die in Pollens `docs/source/SDK/apps.md` genannten Endpoint-Namen sind teilweise veraltet — das **autoritative** Schema ist immer `http://<daemon-host>:8000/openapi.json` (live abrufbar). Stand v1.7.1, verifiziert gegen einen laufenden Wireless:

   ```bash
   # aus HF installieren (öffentlicher Space)
   curl -X POST http://reachy-mini.local:8000/api/apps/install \
     -H "Content-Type: application/json" \
     -d '{"url": "https://huggingface.co/spaces/<user>/reachy-mini-show"}'
   # für private Spaces: POST /api/apps/install-private-space (HF-Token im Body)

   # Verzeichnis: HF-Spaces + lokal installierte Apps
   curl       http://reachy-mini.local:8000/api/apps/list-available
   curl       http://reachy-mini.local:8000/api/apps/list-available/installed   # nur installierte

   # Lifecycle
   curl       http://reachy-mini.local:8000/api/apps/current-app-status
   curl -X POST http://reachy-mini.local:8000/api/apps/start-app/reachy_mini_show
   curl -X POST http://reachy-mini.local:8000/api/apps/restart-current-app
   curl -X POST http://reachy-mini.local:8000/api/apps/stop-current-app

   # Wartung
   curl       http://reachy-mini.local:8000/api/apps/check-updates
   curl -X POST http://reachy-mini.local:8000/api/apps/update/reachy_mini_show
   curl -X POST http://reachy-mini.local:8000/api/apps/remove/reachy_mini_show

   # Async-Jobs (Install/Update geben job_id zurück)
   curl       http://reachy-mini.local:8000/api/apps/job-status/<job_id>

   # Robot-Lock (welche App hält Hardware-Zugriff)
   curl       http://reachy-mini.local:8000/api/daemon/robot-app-lock-status
   curl       http://reachy-mini.local:8000/api/daemon/status
   ```

   Wichtige Drifts gegenüber Pollens `apps.md`:
   - `/api/apps/list` aus der Doku **existiert nicht** — korrekter Endpoint ist `/api/apps/list-available`
   - Der App-Name im Pfad ist der **Python-Package-Name** (snake_case `reachy_mini_show`), nicht der Repo-/HF-Slug (`reachy-mini-show`)
   - Async-Operationen (Install, Update) liefern eine `job_id`; den Fortschritt holt man mit `GET /api/apps/job-status/<job_id>`

3. **Offline / manuell (kein Internet, z. B. Konferenz)** — direkt ins Shared-venv des Wireless installieren:

   ```bash
   scp -r /pfad/zur/app pollen@reachy-mini.local:/tmp/reachy-mini-show
   ssh pollen@reachy-mini.local \
     "/venvs/apps_venv/bin/pip install /tmp/reachy-mini-show"
   # nach Code-Änderungen: Daemon oder App über REST-API neu starten
   ```

Anforderungen:

- **MUSS [MUST]** lokal-entwickelbar sein über mindestens einen der beiden Sim-Pfade oben (externer Daemon **oder** `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:`); reines `use_sim=True` ohne `spawn_daemon=True` ist **kein** gültiger Sim-Pfad und scheitert mit `ConnectionError`
- **MUSS [MUST]** alle drei Deploy-Pfade unterstützen — die HF-Space-Route ist Default, die REST- und SSH-Pfade sind Fallback ohne Dashboard bzw. ohne Internet
- **MUSS [MUST]** wissen, dass auf Wireless alle Apps in das Shared-venv `/venvs/apps_venv/` installiert werden (kein per-App-venv); Abhängigkeits-Konflikte mit anderen installierten Apps sind ein realer Failure-Mode
- **DARF NICHT [MUST NOT]** auf dem Wireless eigenen Code im Daemon-Service (`reachy-mini-daemon.service`) modifizieren — der Service ist Pollen-Eigentum

### Logging und Observability
- **MUSS [MUST]** strukturiertes Python-`logging` mit `INFO`-Default und `DEBUG` per ENV-Var nutzen
- **MUSS [MUST]** wichtige Lifecycle-Events (Behavior gestartet/beendet, Idle-Mode-Wechsel, Verbindungs-Probleme) sowohl ins Log als auch als WebSocket-Event ausgeben
- **MUSS [MUST]** wissen, dass auf Wireless `stdout`/`stderr` der App vom Daemon eingefangen werden und über `sudo journalctl -u reachy-mini-daemon` lesbar sind — auf Lite/Simulation erscheinen Logs direkt im Daemon-Terminal. Diagnostik-Beispiele:

  ```bash
  ssh pollen@reachy-mini.local
  sudo journalctl -u reachy-mini-daemon -f                              # live
  sudo journalctl -u reachy-mini-daemon --since '5 min ago' \
    | grep -v "uvicorn\|GET \|POST "                                    # gefiltert
  ```

- **DARF NICHT [MUST NOT]** Tokens, Credentials oder rohe Audio-Bytes ins Log ausgeben

### Versionierung
- **MUSS [MUST]** semantische Versionen in `pyproject.toml` führen
- **MUSS [MUST]** Changelogs maschinenlesbar (Conventional Commits + Release-Drafter analog zum Plugin-Repo)
- **SOLLTE [SHOULD]** für Tanz-Bausteine die `bpm`-Range pro Release dokumentieren (Hardware-Performance kann sich mit Firmware-Versionen verändern)

## Akzeptanzkriterien
- [ ] App-Repo wurde mit `reachy-mini-app-assistant create` angelegt und passiert `reachy-mini-app-assistant check` ohne Findings
- [ ] `pyproject.toml` deklariert genau einen Entry-Point in der Group `reachy_mini_apps`, der auf die `ReachyMiniShowApp`-Klasse zeigt
- [ ] README trägt den Tag `reachy_mini_python_app` im YAML-Frontmatter
- [ ] `ReachyMiniShowApp.run(reachy_mini, stop_event)` startet drei parallele Tasks (WebSocket, Behavior-Worker, Idle-Loop) und kehrt nach `stop_event.set()` sauber zurück
- [ ] `python -m reachy_mini_show.main` läuft via `wrapped_run()` direkt gegen einen lokalen `reachy-mini-daemon --sim`
- [ ] Lokaler WebSocket auf `127.0.0.1:8765` nimmt JSON-Commands an und broadcastet JSON-Events
- [ ] Jedes Command und jedes Event trägt ein `protocol_version`-Feld; `get_status` liefert `supported_protocol_versions`
- [ ] Commands mit unbekannter Major-Version werden mit `error code: "unsupported_protocol_version"` abgelehnt
- [ ] Alle 29 Motion-Slugs sind als `Move`-Subklassen implementiert und in der Registry eingetragen
- [ ] BPM-Tanz-Bausteine akzeptieren konstruktor-parametrisierte BPM und Beat-Anzahl
- [ ] Lokaler Test mit `ReachyMini(spawn_daemon=True, use_sim=True)` läuft ohne Hardware durch
- [ ] App-Provenienz ist sichtbar: README, CLAUDE.md und `pyproject.toml [project.urls]` verweisen auf das `claude-reachy-mini`-Plugin
- [ ] Push an HF-Remote installiert die App im Reachy-Dashboard ohne manuellen Eingriff
- [ ] Die drei Deploy-Pfade (HF-Push, REST `POST /api/apps/install`, Offline `scp` + `pip install` ins `/venvs/apps_venv/`) sind in der Doku des App-Repos dokumentiert
- [ ] Die in der Doku verwendeten REST-Endpoints sind verifiziert gegen `http://<daemon-host>:8000/openapi.json`, nicht aus Pollens `apps.md` abgeschrieben
- [ ] Auf Wireless sind App-Logs über `sudo journalctl -u reachy-mini-daemon` sichtbar
- [ ] `stop_event` führt zur Ruhepose ohne Aktuator-Klemmen oder hängende Verbindungen
- [ ] Eine Task-Exception bricht alle anderen Tasks sauber ab und fährt in Sicherheitspose
- [ ] Plattform-Profile blenden nicht-vorhandene Sensor-Reads korrekt aus

## Quellen
- Upstream-SDK-Repo (Quelle der Wahrheit für `Move`, `ReachyMini`, `ReachyMiniApp`, App-Lifecycle): <https://github.com/pollen-robotics/reachy_mini>
- Offizielle Apps-Doku (Build, Publish, REST-Install, Logs, Web-UI): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- App-Subsystem (`ReachyMiniApp`-ABC, `wrapped_run`, App-Lock, App-Manager): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps>
- App-Templates (kanonische Vorlage für `pyproject.toml`, `main.py`, `README.md`, `index.html`, `style.css` mit `reachy_mini_python_app`-Tag): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates>
- Daemon (REST-API, App-Lock, Lifecycle, Status — der Subprozess, der diese App startet): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- IO-Protokoll (Befehls- und Telemetrie-Messages, Referenz für unser WebSocket-Protokoll): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- SDK-Konzept-Doku (Apps, Quickstart, Core-Concept): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Pollens `AGENTS.md` (Einsprungspunkt, an dem AI-Agents die App-Authoring-Skills finden): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- HF-Blog-Tutorial (Schritt-für-Schritt mit Screenshots): <https://huggingface.co/blog/pollen-robotics/make-and-publish-your-reachy-mini-apps>
- Lauffähiges Minimal-App-Beispiel: <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>
- Conversation-App als komplettes Referenz-Repo (Audio-Pipeline, LLM-Tools, FastAPI-UI): <https://github.com/pollen-robotics/reachy_mini_conversation_app>

## Offene Fragen
- ~~Heißt der Slug `reachy-mini-show`?~~ **Beantwortet**: ja, durchgehend.
- ~~Audio-Files aus Plugin-Repo gespiegelt oder eigen?~~ **Beantwortet**: das App-Repo hält seine eigenen Audio-Files; keine Spiegelung aus dem Plugin-Repo.
- ~~Beispiel-App-Skelett im Plugin-Repo unter `examples/`?~~ **Beantwortet**: erstmal kein Example. Wenn `app-scaffold` ein konkretes Layout-Vorbild braucht, kann es per Pollen-CLI zur Laufzeit erzeugt werden.
- ~~WebSocket-Protokoll-Versionierung?~~ **Beantwortet**: `protocol_version`-Feld in jedem Command und Event ist jetzt Anforderung; `get_status` liefert `supported_protocol_versions`.
- ~~Wie wird die Pollen-CLI exakt aufgerufen?~~ **Beantwortet**: das offizielle Tool heißt `reachy-mini-app-assistant` (`uv pip install reachy-mini`); Sub-Commands `create <name> <dest> [--publish] [--template default|conversation]`, `check <path>`, `publish <path>`. Der Skill `app-scaffold` kapselt diesen CLI-Aufruf.
- ~~`main(reachy, stop_event)` als freie Funktion vs. `ReachyMiniApp`-Subklasse mit `run()`?~~ **Beantwortet**: Pollen erwartet die Subklasse mit `run(self, reachy_mini, stop_event)`; ein `__main__`-Block ruft `wrapped_run()`. Ein freier `main()` würde weder vom Daemon-Discovery-Pfad (Entry-Point-Group `reachy_mini_apps`) noch vom Direkt-Lauf-Pfad korrekt eingebunden.
- Welcher GitHub-Owner für das App-Repo — `nolte` direkt oder eine Org? Vorschlag: `nolte/reachy-mini-show`.
- Soll der WebSocket optional auch UNIX-Sockets sprechen (für VM- oder Container-isolierte Konsumenten)? Default bleibt TCP.
- Wie wird ein Behavior abgebrochen, das in der WebSocket-Queue noch wartet (nicht das aktive)? Vorschlag: `cancel` leert die Queue und stoppt das aktive Behavior; ein zukünftiger `cancel_pending` könnte das später trennen.
