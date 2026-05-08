# MCP-Server für Reachy Mini

Status: draft

## Kontext

Der Reachy Mini wird heute aus zwei Welten heraus gesteuert: **Pollen-eigene Apps** (`reachy_mini_apps`-Entry-Point, vom Daemon gemanagt, App-Lock-Slot) und **direkte SDK-Calls** aus einem Skript oder REPL des Entwicklers. Beide Pfade setzen voraus, dass der Bediener Python und das `reachy_mini`-SDK kennt; beide laufen lokal und sind nicht direkt LLM-konsumierbar. Wenn ein LLM (Claude, GPT, lokale Modelle) im Gespräch eine Pose lesen oder einen Move triggern soll, muss er heute Code generieren, das als Skript ausführen lassen und das Ergebnis aus stdout zurücklesen — eine Pipeline mit drei Schichten Reibung.

Der **MCP-Server für Reachy Mini** schließt diese Lücke. Er ist ein lokaler Server-Prozess, der über das [Model Context Protocol](https://modelcontextprotocol.io) ein klar abgegrenztes Tool-Inventar exponiert (Sensoren lesen, Motoren bewegen, Daemon-Lifecycle abfragen) und intern gegen die **REST-API des Pollen-Daemons** arbeitet — nicht gegen das Python-SDK direkt. Damit kann jedes MCP-fähige Frontend (Claude Desktop, Claude Code, Cursor, eigene Clients) den Reachy Mini direkt bedienen, ohne ein eigenes Python-Skript schreiben oder ausführen zu müssen.

Diese Spec definiert die kanonische Form dieses Servers: welche Tools er anbietet, welche Sicherheits-Gates er einhält, auf welchen Plattformen er läuft, und in welchem Verhältnis er zu den schon existierenden Plugin-Skills (`reachy-mini-sdk`, `reachy-mini-start`, `reachy-mini-deploy`) und Agents (`reachy-mini-on-device`) steht. Sie ist Wissens-Spec — die Operations-Spec zum Hochfahren des Servers liegt in [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/de.md), die Server-Implementation lebt außerhalb dieses Plugins.

Begriffsklärung: „MCP-Server" = der hier spezifizierte lokale Prozess, der Tools nach MCP-Protokoll bereitstellt; „MCP-Client" = das LLM-Frontend, das die Tools konsumiert; „Daemon" = der Pollen-eigene `reachy-mini-daemon` mit REST-/WebSocket-Surface auf Port 8000 — nicht zu verwechseln.

## Ziele

- Eine direkte LLM-↔-Roboter-Brücke ohne Skript-Generierungs-Umweg, mit klarem Tool-Inventar in drei Tiers (Read / Write / Lifecycle), so dass ein MCP-Client mit einem Aufruf eine Pose lesen oder einen Move triggern kann
- **REST-Wrapper** als einziger Daten-Pfad — der MCP-Server hält keine Python-SDK-Instanz, kein App-Lock, keine WebSocket-Streaming-Verbindung; er konsumiert nur den Daemon
- Sicherheits-Gates eingebaut, nicht aufgeschraubt: Pose-Range-Validation gegen `control-surface`, App-Lock-Vorprüfung vor Schreib-Operationen, Safe-Torque-Wrapping bei Motor-Toggle, Audit-Trail jeder Tool-Invocation
- **localhost-only** als Default; Remote-Use-Cases sind opt-in und außerhalb von v1
- Plattform-Profile sauber abgegrenzt: was geht auf Wireless / Lite / Simulation
- Klares Verhältnis zu bestehenden Skills/Agents: der Server ist eine **andere Distribution** des Plugin-Wissens, nicht ein Ersatz für die Skill-/Agent-Surface

## Nicht-Ziele

- Pollens Daemon ersetzen — der Daemon bleibt der einzige direkte Hardware-Treiber; der MCP-Server ist Konsument, nicht Treiber
- Pollens App-Framework ersetzen — Apps mit `reachy_mini_apps`-Entry-Point bleiben Pollens Surface; der MCP-Server **läuft parallel**, hält selbst kein App-Lock
- Eine eigene Choreographie-/Sequenz-/Move-Composition-Surface — Bewegungs-Composition gehört zu Apps und zum [`dance-choreography`](../../claude/dance-choreography/de.md)-Skill
- 50–100 Hz tight loops über das MCP-Protokoll — MCP ist request-response, nicht streaming-tauglich; tight loops bleiben im SDK-Apps-Framework (`set_target` über die Python-API)
- Hugging-Face-Publish, Deploy, On-Device-Test — eigene Skills / Agents (`reachy-app-publish-hf`, `reachy-mini-deploy`, `reachy-mini-on-device`)
- Audio-Stream-Capture und -Playback in Real-Time — auch das gehört in Pollens Audio-Pipeline (`reachy_mini.media.*`); der MCP-Server kann höchstens Trigger („spiele Datei X ab") liefern
- Multi-User / Mandantentrennung — der Server ist Single-User-localhost-only; jede LLM-Session, die den Server nutzt, hat vollen Zugriff
- Plugin-Releases, CI/CD-Integration, Cloud-Deployment — der Server ist lokales Werkzeug, nicht Service

## Anforderungen

### Architektur — REST-Wrapper

- **MUSS [MUST]** der MCP-Server alle Roboter-Operationen über Pollens Daemon-REST-API ausführen — nicht über das Python-SDK direkt; Quelle: [`src/reachy_mini/daemon/app/routers/`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers)
- **MUSS [MUST]** der Daemon-Endpunkt konfigurierbar sein (Default `http://127.0.0.1:8000`, Wireless mDNS-Override `http://reachy-mini.local:8000`, Lite localhost) — aber **niemals** als Default auf eine Nicht-localhost-Adresse zeigen
- **DARF NICHT [MUST NOT]** der MCP-Server eine `ReachyMini`-SDK-Instanz oder direkte Hardware-Verbindung halten — das würde mit dem App-Lock kollidieren und parallel laufende Apps brechen
- **SOLLTE [SHOULD]** der MCP-Server pro Tool-Aufruf eine kurzlebige HTTP-Verbindung benutzen (kein langlebiger SSE-/WebSocket-Stream), damit ein Daemon-Restart die LLM-Session nicht stillschweigend abreißt
- **MUSS [MUST]** der MCP-Server bei Daemon-Connection-Refused dem MCP-Client eine **MCP-konforme Fehlerantwort** liefern (kein Python-Traceback, kein silent ignore)

### Tool-Inventar — Tier 1: Read (sicher, kein Lock-Check nötig)

| Tool | Daemon-Pfad (Quelle) | Antwort |
|---|---|---|
| `get_head_pose` | `state.py` Router → Joint/Pose-State | 4×4-Transform-Matrix |
| `get_motor_status` | `motors.py` Router → Status-Read | Liste `{id, enabled, position, effort}` |
| `get_imu` (Wireless only) | `state.py` Router → IMU-Submessage | `{accel, gyro, quat, temp_C}`; auf Lite/Sim Fehler |
| `get_battery` (Wireless only) | `daemon.py` Router → System-Status | `{soc_pct, voltage_v}`; auf Lite/Sim Fehler |
| `get_current_app` | `apps.py` Router → `current-app-status` | `{app_name?, lock_holder?}` |
| `list_apps` | `apps.py` Router → Liste der installierten Apps | Liste `{name, version, entry_point}` |

- **MUSS [MUST]** jedes Tier-1-Tool ohne App-Lock-Vorprüfung laufen — Reads sind read-only, blockieren keine andere App
- **SOLLTE [SHOULD]** Tier-1-Tools ein optionales `cache_ms`-Parameter annehmen (Default 0), damit ein MCP-Client wiederholte Reads im selben Turn lokal cachen lässt — Daemon-Last reduzieren

### Tool-Inventar — Tier 2: Write (mit Sicherheits-Gates)

| Tool | Daemon-Pfad | Sicherheits-Gates |
|---|---|---|
| `goto_pose(head, antennas, body_yaw, duration, method)` | `move.py` Router → `goto_target` | Pose-Range; App-Lock-Vorprüfung |
| `set_pose(head, antennas, body_yaw)` | `move.py` Router → `set_target` (single tick) | Pose-Range; App-Lock-Vorprüfung; **kein 50-Hz-Loop** |
| `set_automatic_body_yaw(enabled)` | `move.py` Router | App-Lock-Vorprüfung |
| `look_at_world(x, y, z)` | `kinematics.py` Router | Pose-Range nach IK; App-Lock-Vorprüfung |
| `look_at_image(u, v)` | `kinematics.py` Router | Pose-Range nach IK; App-Lock-Vorprüfung |
| `wake_up()` | `move.py` Router → entsprechender Endpunkt | Safe-Torque (intern in `wake_up`); App-Lock-Vorprüfung |
| `goto_sleep()` | `move.py` Router → entsprechender Endpunkt | Safe-Torque (intern); App-Lock-Vorprüfung |
| `enable_motors()` | `motors.py` Router | **Safe-Torque-Wrapping** (kurzes `goto_target` zur aktuellen Pose, dann enable); App-Lock-Vorprüfung |
| `disable_motors()` | `motors.py` Router | **Safe-Torque-Wrapping** (`goto_target` zu `SLEEP_HEAD_POSE` zuerst, dann disable); App-Lock-Vorprüfung |

- **MUSS [MUST]** vor jedem Tier-2-Tool **App-Lock-Status** abfragen (`apps.py` Router → `robot-app-lock-status`); hält eine andere App das Lock, mit klarem Hinweis abbrechen — niemals force-stoppen
- **MUSS [MUST]** für jedes Tool, das eine Pose annimmt, die Werte gegen die Limits aus [`reachy-mini/control-surface`](../control-surface/de.md) validieren (Pitch/Roll ±40°, Head-Yaw ±60°, Yaw-relativ ±65°, Body-Yaw ±155°, Antennen ±180°); out-of-range → Fehler an den MCP-Client, niemals stille Clipping-Korrektur
- **MUSS [MUST]** `enable_motors`/`disable_motors` das Safe-Torque-Pattern aus [`reachy-mini/app-logging`](../app-logging/de.md) und Pollens [`safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md) **immer** ausführen — der Server nimmt dem LLM die Verantwortung dafür ab
- **DARF NICHT [MUST NOT]** der MCP-Server `set_pose`-Aufrufe in einer Schleife durchstoßen lassen — das ist tight-loop-Pattern und gehört in eine SDK-App; der Server lehnt ab dem zweiten `set_pose`-Aufruf innerhalb von 200 ms ab und verweist auf `goto_pose`

### Tool-Inventar — Tier 3: Lifecycle (operativ)

| Tool | Daemon-Pfad | Anmerkung |
|---|---|---|
| `stop_current_app()` | `apps.py` Router → `stop-current-app` | Setzt `stop_event`; releaset App-Lock |
| `emergency_stop()` | mehrere | Notstopp-Eskalation: `stop_event` → 2 s → SIGTERM → 1 s → SIGKILL → Pose-Reset → `disable_motors` |
| `start_app(name)` (optional v2) | `apps.py` Router → `start-app` | Nur wenn kein App-Lock; sonst Fehler |

- **MUSS [MUST]** `emergency_stop` der einzige Notstopp-Pfad sein; analog zur Eskalations-Sequenz aus [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md), aber für den MCP-Kontext angepasst
- **SOLLTE [SHOULD]** `stop_current_app` eine optionale `force=true`-Variante anbieten, die einen Cleanup-Timeout überspringt — User-konfirmiert, niemals Default

### Sicherheits-Modell

- **MUSS [MUST]** der Server per Default an `127.0.0.1:<port>` binden, niemals an `0.0.0.0` ohne explizites Opt-in
- **MUSS [MUST]** der Server ein **Audit-Log** schreiben — pro Tool-Invocation eine Zeile mit `{timestamp, tool, args (gekürzt), result, latency_ms}` unter `~/.cache/reachy-mini-mcp/<YYYY-MM-DD>.log`
- **MUSS [MUST]** das Audit-Log keine PII / Tokens / WiFi-Credentials / Sensor-Streams enthalten — nur Tool-Metadaten (PII-Klausel analog zu [`reachy-mini/app-logging`](../app-logging/de.md))
- **MUSS [MUST]** der Server Pose-Range-Validation als Server-internen Gate haben, **bevor** der Daemon-Aufruf rausgeht — der Daemon lehnt zwar auch ab, aber wir wollen keine sinnlosen Round-Trips
- **SOLLTE [SHOULD]** der Server bei wiederholten out-of-range-Versuchen aus dem gleichen MCP-Client einen Rate-Limiter aktivieren (z. B. nach 5 abgelehnten Aufrufen für 10 s sperren) — gegen LLM-Hallucinations
- **DARF NICHT [MUST NOT]** der Server Tools liefern, die direkt in `~/.ssh/`, Git-Configs, Hugging-Face-Tokens oder andere lokale Secrets schreiben — der Scope ist Roboter-Bedienung, nicht System-Administration

### Plattform-Profile

| Plattform | Daemon-Adresse (Default) | Tier 1 | Tier 2 | Tier 3 | Bemerkung |
|---|---|---|---|---|---|
| Reachy Mini Wireless | `http://reachy-mini.local:8000` (mDNS) oder konfigurierte IP | ✓ inkl. IMU + Battery | ✓ | ✓ | Server kann auf RPi 4 CM4 selbst laufen oder remote per SSH-Tunnel |
| Reachy Mini Lite | `http://127.0.0.1:8000` | ✓ ohne IMU + Battery | ✓ | ✓ | Daemon läuft auf Host-PC, Server daneben |
| Simulation (`use_sim=True`) | `http://127.0.0.1:8000` | ✓ ohne IMU + Battery (Sim publisht keine echten Werte) | ✓ (Sim akzeptiert Pose-Targets) | ✓ teilweise | Notstopp-SIGKILL-Schritt entfällt (kein App-Subprozess) |

- **MUSS [MUST]** der Server vor dem ersten Tool-Aufruf einen Plattform-Detect ausführen (`daemon.py` Router → System-Status); die erkannte Plattform wird pro Tool-Antwort als Metadaten-Feld zurückgegeben
- **MUSS [MUST]** Tools, die auf der erkannten Plattform nicht verfügbar sind (z. B. `get_imu` auf Lite), mit einem klaren Fehler abweisen, nicht mit einem Stub-Wert antworten

### MCP-Protokoll-Konformität

- **MUSS [MUST]** der Server das offizielle MCP-Protokoll implementieren, gegen die jeweilige aktuelle Spec-Version (siehe Quelle)
- **MUSS [MUST]** alle Tools eine **JSON-Schema-Beschreibung** ihrer Parameter und Antwort liefern — MCP-Clients verlassen sich darauf für Argument-Validation
- **MUSS [MUST]** der Server das `tools/list`- und `tools/call`-Protokoll-Pattern unterstützen; Resources / Prompts sind v1-out-of-scope, können aber später ergänzt werden
- **SOLLTE [SHOULD]** Tool-Beschreibungen klar machen, welche Tier-Stufe ein Tool hat und welche Sicherheits-Gates es betrifft, damit MCP-Clients (oder das LLM dahinter) verstehen, was ein Tool tut, bevor es es ruft

### Verhältnis zu existierenden Skills/Agents

- **MUSS [MUST]** der Server **nicht** in einem `claude-reachy-mini`-Skill oder -Agent eingebettet sein; Skills laufen innerhalb der Claude-Code-Session, der MCP-Server ist ein **eigenständiger Prozess**
- **MUSS [MUST]** die Server-Implementation in einem **separaten Repo** liegen (z. B. `reachy-mini-mcp-server`, analog zu `reachy-mini-app`); das Plugin liefert die Spec und einen Bootstrap-Skill, nicht den Server-Code
- **SOLLTE [SHOULD]** der Server [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md) als kanonische Wissensbasis für SDK-Idiome zitieren, damit ein Operator weiß, wo die Hintergrund-Konzepte herkommen
- **SOLLTE [SHOULD]** der Server [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md) als „die richtige Surface für Bulk-Triage / Live-Trial / Telemetrie-Sampling" benennen — der MCP-Server ist Single-Shot-Interaktion, nicht Test-Lifecycle

### Konsumenten und Boundary

- **SOLLTE [SHOULD]** der Server für `dance-choreography` keine eigenen Tools anbieten — Choreographien sind Authoring-Artefakte, keine Tools
- **DARF NICHT [MUST NOT]** der Server eine Spec / Choreographie / App-Code generieren — er bedient den Roboter, er erzeugt keine Plugin-Inhalte
- **SOLLTE [SHOULD]** der Server eine `health-check`-Tool-Variante anbieten, die Daemon-Erreichbarkeit, Plattform-Detection und Audit-Log-Schreibbarkeit in einem Aufruf prüft — nützlich beim Erstkontakt eines MCP-Clients

## Akzeptanzkriterien

- [ ] Die Spec-Datei lebt unter `spec/reachy-mini/mcp-server/de.md` (DE kanonisch) und `en.md` (EN-Übersetzung), strukturell identisch
- [ ] Tool-Inventar deckt drei Tiers ab (Read / Write / Lifecycle), pro Tool mit Daemon-Pfad-Verweis und Sicherheits-Gate-Annotation
- [ ] Tier-1-Reads laufen ohne App-Lock-Vorprüfung; Tier-2-Writes laufen mit App-Lock-Vorprüfung; Tier-3-Lifecycle hat eigene Notstopp-Eskalation
- [ ] Pose-Range-Validation gegen `control-surface` ist als Server-internes Gate spezifiziert, nicht delegiert
- [ ] Safe-Torque-Wrapping um `enable_motors` / `disable_motors` ist verbindlich
- [ ] localhost-only-Bindung ist Default; Remote-Opt-in ist explizit als „v1 out of scope" markiert
- [ ] Audit-Log-Pfad und -Format sind festgelegt; PII-Klausel inheritet aus `app-logging`
- [ ] Plattform-Tabelle deckt Wireless / Lite / Simulation ab und nennt pro Plattform, welche Tier-1-Tools verfügbar sind
- [ ] Verhältnis zu bestehenden Skills/Agents ist explizit benannt (kein Embedding, eigene Repo, eigene Distribution)
- [ ] MCP-Protokoll-Konformität ist als MUST formuliert, mit Verweis auf die offizielle Spec
- [ ] Cross-Refs auf [`reachy-mini/control-surface`](../control-surface/de.md), [`reachy-mini/app-logging`](../app-logging/de.md), [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md), [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md), [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/de.md) sind sichtbar
- [ ] `pre-commit run --all-files` läuft auf den Spec-Dateien grün

## Quellen

> Quell-Verweise auf Pollen-Code-Dateien zeigen auf das jeweilige Verzeichnis im Daemon-Tree; konkrete Endpunkt-Pfade sind in den Router-Dateien zu verifizieren, sobald die Implementation startet. Markdown-Quellen sind Datei-Level zitiert.

- Pollen-Daemon-Router (Quelle aller REST-Endpunkte): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers>
- Pollen-Daemon-Hauptmodul (FastAPI-App-Bootstrap): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/main.py>
- Apps-Router (App-Lock-Status, current-app-Status, start/stop-app): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/apps.py>
- State-Router (Pose, Joint-Positions, IMU): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/state.py>
- Move-Router (goto_target, set_target): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/move.py>
- Motors-Router (enable/disable, Status): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/motors.py>
- Kinematics-Router (look_at_world, look_at_image): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/kinematics.py>
- Pollen-Skill `safe-torque` (Anti-Jerk-Pattern, Quelle für Tier-2-Wrapping): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Model Context Protocol — Spezifikation: <https://modelcontextprotocol.io>
- Model Context Protocol — Python-Server-SDK: <https://github.com/modelcontextprotocol/python-sdk>
- Interne Cross-Refs:
  - [`reachy-mini/control-surface`](../control-surface/de.md) — Pose-Range-Limits-Quelle
  - [`reachy-mini/app-logging`](../app-logging/de.md) — PII-Klausel-Vorbild
  - [`reachy-mini/app-architecture`](../app-architecture/de.md) — Daemon-Lifecycle-Kontext
  - [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/de.md) — SDK-Idiome, Method-Choice, Safe-Torque
  - [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/de.md) — Test-Lifecycle als Schwester-Surface, mit Notstopp-Eskalation
  - [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/de.md) — Operations-Skill für Server-Hochfahren

## Offene Fragen

- Implementations-Sprache: Python (mit `mcp`-SDK)? Rust (mit Community-MCP-Crates)? Vorschlag: Python für v1 — gleiche Plattform-Voraussetzungen wie der Pollen-Daemon, einfaches Packaging über `uv`.
- Server-Distribution: PyPI-Paket `reachy-mini-mcp-server`? Oder Hugging-Face-Space mit `pyproject.toml`-Eintry-Point? PyPI ist die saubere Variante für Tooling.
- Cache-Strategy: soll der Server Tier-1-Reads selbst cachen (z. B. Pose-Read pro 100 ms) oder dem MCP-Client überlassen? `cache_ms`-Parameter pro Tool ist ein guter Mittelweg.
- Multi-Tool-Choreographien: braucht der MCP-Client ein `goto_pose_sequence`-Tool, das mehrere Posen als eine Transaktion ausführt, oder ist das App-Sache? Argument für Single-Tool-Approach: einfacher; Argument für Sequence-Tool: weniger Round-Trips.
- WebSocket-Tools: einige Operationen (z. B. Move-Cancellation während eines `goto_target`) brauchen einen langlebigen Stream. Aktuell als out-of-scope markiert, aber Konsumenten-Use-Cases könnten das ändern.
- Authentifizierung für Remote-Modus: wenn jemand den Server doch auf `0.0.0.0` öffnet, was ist die Auth-Schicht? OAuth, Bearer Token, mTLS? Out-of-scope v1; Konsumenten-Druck abwarten.
- Server auf Wireless-Hardware: läuft der Server **auf** dem Reachy Mini selbst (lokaler Daemon, lokaler MCP-Server, SSH-Tunnel zum LLM-Host) oder **auf dem Entwickler-Host** (Daemon remote, MCP-Server lokal)? Letzteres scheint einfacher; ersteres reduziert Latenz. Konsumenten-Test in v0.
- Tool-Versionierung: ein MCP-Client speichert Tool-Definitionen ab — was passiert bei Schema-Änderungen? Server-Version im Capability-Handshake melden, MCP-Client invalidiert seinen Cache.
- Verhältnis zu Pollens eigenem Tooling: hat Pollen Pläne für einen offiziellen MCP-Server? Falls ja, sollten wir uns abstimmen — sonst gibt es zwei konkurrierende Server.
