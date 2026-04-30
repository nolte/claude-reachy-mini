# App-Architektur: Reachy-Mini-Show

Status: draft

## Kontext
Dieses Repository (`claude-reachy-mini`) liefert Skills, Agents und Specs als Toolbox für die Entwicklung. Die konkrete Anwendung, die mit dieser Toolbox entsteht, ist eine **Pollen-Reachy-Mini-App** — ein Python-Paket, das der Reachy-Daemon als Subprozess auf dem Roboter startet und das die 29 Motion-Specs aus diesem Plugin in lebende Behaviors überführt. Diese Spezifikation legt das App-Layout, den Lifecycle, die Befehls-Schnittstelle und den Distributionspfad fest. Sie ist die Quelle der Wahrheit, gegen die die Skills `behavior-scaffold` und `reachy-mini-sdk` und der Agent `reachy-mini-on-device` ihre Vorschläge ausrichten. Die App lebt in einem **separaten App-Repository** (Vorschlag: `nolte/reachy-mini-show`) — dieses Plugin-Repository selbst enthält keinen App-Code.

## Ziele
- Eine einzige App, die alle 29 Motion-Slugs als `Move`-Subklassen implementiert
- Volle Konformität mit Pollens App-System: Daemon-Subprozess, eine App pro Zeit, Hugging-Face-Spaces als Distribution
- Live-Befehl-Annahme über lokalen WebSocket — späterer Konsum durch externe Integrationen (HA, Wyoming-Bridge) im jeweiligen Konsumenten-Repository
- Lokal entwickelbar via `ReachyMini(use_sim=True)` — keine Hardware nötig zum Starten
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
- **MUSS [MUST]** semantische Versionierung nutzen
- **MUSS [MUST]** den `reachy_mini`-SDK-Pin auf eine konkrete Minor-Version setzen (z. B. `^1.7.0`); SDK-Major-Update ist immer eine bewusste Re-Validierung

### Repository-Layout
Pollen-CLI-konformes Layout mit Provenienz-Marker (`CLAUDE.md`, Plugin-URL):

```
reachy-mini-show/
├── pyproject.toml              # Pollen-konformes Format, SDK-Pin, Provenienz-URLs
├── README.md                   # HF-Frontmatter `reachy_mini_python_app`, Provenienz-Notiz
├── CLAUDE.md                   # Verweis auf claude-reachy-mini Plugin (Authoring-Quelle)
├── reachy_mini_show/
│   ├── __init__.py
│   ├── main.py                 # Pollen-Entry: main(reachy, stop_event)
│   ├── server.py               # WebSocket-Server :8765
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
│   └── config.py               # Defaults und Plattform-Profile
└── tests/                      # Unit-Tests gegen ReachyMini(use_sim=True)
```

### Provenienz-Marker (Pflicht)

- **MUSS [MUST]** in `README.md` unmittelbar nach dem HF-Frontmatter einen Provenienz-Block tragen mit (1) einem Verweis auf das Claude-Code-Plugin `claude-reachy-mini` (`https://github.com/nolte/claude-reachy-mini`), (2) einem Verweis auf den Motion-Catalog (`spec/reachy-mini/motions/`), (3) einem Verweis auf diese Architektur-Spec
- **MUSS [MUST]** eine `CLAUDE.md` im App-Repo-Root tragen, die die empfohlenen Plugin-Skills (`reachy-mini-sdk`, `behavior-scaffold`, Agent `reachy-mini-on-device`) namentlich auflistet und auf das Plugin-Repo verlinkt
- **MUSS [MUST]** im `pyproject.toml` unter `[project.urls]` mindestens diese Einträge tragen: `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
- **SOLLTE [SHOULD]** ein Code-Header in `main.py` einen einzeiligen Verweis tragen: `# Behaviors derived from spec/reachy-mini/motions/ in nolte/claude-reachy-mini`

### Lifecycle
- **MUSS [MUST]** die Pollen-Konvention `main(reachy: ReachyMini, stop_event: threading.Event)` implementieren
- **MUSS [MUST]** drei parallele Tasks unter `asyncio.run(...)` starten: WebSocket-Server, Behavior-Worker (liest Queue, ruft `mini.async_play_move(...)`), Idle-Loop (wenn Queue leer und kein Behavior aktiv → läuft `waiting-idle` oder konfigurierter Idle-Mode)
- **MUSS [MUST]** auf `stop_event` alle drei Tasks sauber beenden, laufendes Behavior via `mini.cancel_move()` abbrechen und Reachy in `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS` fahren
- **MUSS [MUST]** bei einer Task-Exception alle anderen Tasks sauber beenden und in eine Sicherheitspose fahren — keine hängenden Verbindungen, keine eingefrorene Pose
- **DARF NICHT [MUST NOT]** Hardware-Reconnect in der App selbst implementieren — Pollens Daemon übergibt eine bereits verbundene Instanz; Verbindungs-Lifecycle gehört dem Daemon

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

### Plattform-Profile
- **MUSS [MUST]** zwischen Wireless / Lite / Simulation unterscheiden, basierend auf SDK-Capability-Discovery
- Wireless: vollständig (IMU-Reads aktiv, Battery-Polling aktiv, alle Behaviors)
- Lite: keine IMU-Reads, kein Battery-Polling; sonst voll
- Simulation: keine Audio-Wiedergabe, keine Sensor-Events außer Pose-Read

### Distributionspfad
- **MUSS [MUST]** lokal-entwickelbar sein über `with ReachyMini(use_sim=True) as mini:`
- **MUSS [MUST]** auf echte Hardware deploybar sein über Pollens `local`-Source-Slot (Daemon-REST-API oder Reachy-Dashboard)
- **MUSS [MUST]** als Hugging-Face-Space publizierbar sein über `git push <hf-remote>` — Tag `reachy_mini_python_app` macht die App im Reachy-Dashboard installierbar

### Logging und Observability
- **MUSS [MUST]** strukturiertes Python-`logging` mit `INFO`-Default und `DEBUG` per ENV-Var nutzen
- **MUSS [MUST]** wichtige Lifecycle-Events (Behavior gestartet/beendet, Idle-Mode-Wechsel, Verbindungs-Probleme) sowohl ins Log als auch als WebSocket-Event ausgeben
- **DARF NICHT [MUST NOT]** Tokens, Credentials oder rohe Audio-Bytes ins Log ausgeben

### Versionierung
- **MUSS [MUST]** semantische Versionen in `pyproject.toml` führen
- **MUSS [MUST]** Changelogs maschinenlesbar (Conventional Commits + Release-Drafter analog zum Plugin-Repo)
- **SOLLTE [SHOULD]** für Tanz-Bausteine die `bpm`-Range pro Release dokumentieren (Hardware-Performance kann sich mit Firmware-Versionen verändern)

## Akzeptanzkriterien
- [ ] App-Repo folgt Pollen-CLI-Layout, mit `reachy_mini_python_app`-Tag im HF-Frontmatter
- [ ] `main(reachy, stop_event)` startet drei parallele Tasks (WebSocket, Behavior-Worker, Idle-Loop)
- [ ] Lokaler WebSocket auf `127.0.0.1:8765` nimmt JSON-Commands an und broadcastet JSON-Events
- [ ] Jedes Command und jedes Event trägt ein `protocol_version`-Feld; `get_status` liefert `supported_protocol_versions`
- [ ] Commands mit unbekannter Major-Version werden mit `error code: "unsupported_protocol_version"` abgelehnt
- [ ] Alle 29 Motion-Slugs sind als `Move`-Subklassen implementiert und in der Registry eingetragen
- [ ] BPM-Tanz-Bausteine akzeptieren konstruktor-parametrisierte BPM und Beat-Anzahl
- [ ] Lokaler Test mit `ReachyMini(use_sim=True)` läuft ohne Hardware durch
- [ ] App-Provenienz ist sichtbar: README, CLAUDE.md und `pyproject.toml [project.urls]` verweisen auf das `claude-reachy-mini`-Plugin
- [ ] Push an HF-Remote installiert die App im Reachy-Dashboard ohne manuellen Eingriff
- [ ] `stop_event` führt zur Ruhepose ohne Aktuator-Klemmen oder hängende Verbindungen
- [ ] Eine Task-Exception bricht alle anderen Tasks sauber ab und fährt in Sicherheitspose
- [ ] Plattform-Profile blenden nicht-vorhandene Sensor-Reads korrekt aus

## Offene Fragen
- ~~Heißt der Slug `reachy-mini-show`?~~ **Beantwortet**: ja, durchgehend.
- ~~Audio-Files aus Plugin-Repo gespiegelt oder eigen?~~ **Beantwortet**: das App-Repo hält seine eigenen Audio-Files; keine Spiegelung aus dem Plugin-Repo.
- ~~Beispiel-App-Skelett im Plugin-Repo unter `examples/`?~~ **Beantwortet**: erstmal kein Example. Wenn `behavior-scaffold` ein konkretes Layout-Vorbild braucht, kann es per Pollen-CLI zur Laufzeit erzeugt werden.
- ~~WebSocket-Protokoll-Versionierung?~~ **Beantwortet**: `protocol_version`-Feld in jedem Command und Event ist jetzt Anforderung; `get_status` liefert `supported_protocol_versions`.
- Wie wird die Pollen-CLI exakt aufgerufen? Vorschlag: über den `behavior-scaffold`-Skill kapseln, sodass der Entwickler nur die Hülle füttert.
- Welcher GitHub-Owner für das App-Repo — `nolte` direkt oder eine Org? Vorschlag: `nolte/reachy-mini-show`.
- Soll der WebSocket optional auch UNIX-Sockets sprechen (für VM- oder Container-isolierte Konsumenten)? Default bleibt TCP.
- Wie wird ein Behavior abgebrochen, das in der WebSocket-Queue noch wartet (nicht das aktive)? Vorschlag: `cancel` leert die Queue und stoppt das aktive Behavior; ein zukünftiger `cancel_pending` könnte das später trennen.
