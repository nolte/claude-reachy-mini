# REST-API des Reachy-Mini-Daemons

Status: draft

## Kontext
Der Pollen-Daemon (Paket `reachy_mini`, FastAPI-Anwendung) ist der zentrale Server, mit dem alle Reachy-Mini-Geräte (Wireless, Lite) lokal gesteuert werden. Er stellt unter `http://<host>:8000/` eine HTTP/REST-Surface bereit, deren einzige autoritative Beschreibung das live ausgelieferte `openapi.json` ist. Bestehende Specs in diesem Repo dokumentieren nur die Endpoint-Teilmengen, die für ihr jeweiliges Thema unmittelbar relevant sind — `app-architecture` deckt `/api/apps/*` und Teile von `/api/daemon/*` ab, `mcp-server` listet einzelne Read-Endpoints als MCP-Tool-Mapping, `host-provisioning` kennt den Lock-Status. Eine zentrale Übersicht aller Endpoints, gegen die Drift gemessen werden kann, fehlt bisher. Diese Spec konsolidiert das vollständige Endpoint-Inventar (Stand: live verifiziert gegen `http://reachy-mini.local:8000/openapi.json`, OpenAPI 3.1.0, FastAPI-Titel `FastAPI`, version `0.1.0`) gruppiert nach Namensraum und mit kurzer Verwendungs-Notiz pro Gruppe.

## Ziele
- Jeder im Daemon erreichbare HTTP-Endpoint ist hier mit Methode, Pfad und Summary aufgeführt
- Pro Namensraum existiert eine Verwendungs-Notiz, die klarstellt, *wozu* die Endpoint-Familie gut ist und welche bestehenden Skills/Agents sie konsumieren
- Andere Specs dürfen auf diese Liste verweisen, statt eigene Endpoint-Tabellen zu führen
- Drift gegen das live `openapi.json` wird durch eine automatisierte Prüfung erkennbar (Existenz-Check, kein Schema-Diff)
- Die Tabellen sind als Existenz-Inventar gedacht: Request/Response-Schemata bleiben die Domäne der live OpenAPI

## Nicht-Ziele
- Vollständige Request/Response-Schemata pro Endpoint — das `openapi.json` bleibt die einzige autoritative Quelle dafür
- Implementierungs-Details des Daemons (Router-Aufbau, Dependency-Injection)
- WebSocket-Surfaces des Daemons — OpenAPI 3.x beschreibt keine WebSockets; falls vorhanden, gehören sie in eine eigene Spec
- Endpoints von Drittsystemen (Home Assistant Core, Hugging Face Hub) — auch wenn sie syntaktisch `/api/...` heißen, sind sie hier nicht gemeint
- Migrations- oder Versionierungs-Policy der REST-API

## Anforderungen

### Autoritative Quelle und Pflege

- Die in dieser Spec gelisteten Endpoints **MÜSSEN** mit dem `openapi.json` eines laufenden Daemons übereinstimmen, das unter `http://<daemon-host>:8000/openapi.json` abrufbar ist
- Jede Änderung an dieser Spec **MUSS** durch einen Abruf der live OpenAPI motiviert sein und im Pull-Request-Body den verwendeten Daemon-Host plus Datum nennen
- Wer einen Endpoint in einer anderen Spec, einem Skill oder einem Agent referenziert, **DARF** auf diese Spec verweisen und **DARF NICHT** zusätzlich eine eigene Existenz-Tabelle führen
- Diese Spec **MUSS NICHT** Request- oder Response-Schemata duplizieren; sie führt nur (Methode, Pfad, Summary)

### Endpoint-Inventar

Die folgenden Tabellen geben jeden vom Daemon ausgelieferten HTTP-Endpoint wieder. Methode und Pfad sind als Stringliterale kopierbar; Pfad-Parameter (`{name}`) folgen FastAPI-Konventionen.

#### Apps — App-Lifecycle (`/api/apps/*`)

Lifecycle-Surface für vom Daemon verwaltete Apps: installieren, listen, starten, stoppen, updaten, entfernen. Konsumiert von `app-architecture`, `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`, `app-log-triage`.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/apps/check-updates` | Check App Updates |
| `GET` | `/api/apps/current-app-status` | Current App Status |
| `POST` | `/api/apps/install` | Install App |
| `POST` | `/api/apps/install-private-space` | Install Private Space |
| `GET` | `/api/apps/job-status/{job_id}` | Job Status |
| `GET` | `/api/apps/list-available` | List All Available Apps |
| `GET` | `/api/apps/list-available/{source_kind}` | List Available Apps |
| `POST` | `/api/apps/remove/{app_name}` | Remove App |
| `POST` | `/api/apps/restart-current-app` | Restart App |
| `POST` | `/api/apps/start-app/{app_name}` | Start App |
| `POST` | `/api/apps/stop-current-app` | Stop App |
| `POST` | `/api/apps/update/{app_name}` | Update App |

#### Daemon — Daemon-Lifecycle und App-Lock (`/api/daemon/*`)

Steuert den Daemon selbst (Start/Stop/Restart) und liest den App-Lock, der zur Zeit nur eine App gleichzeitig zulässt. Konsumiert vom Robot-Busy-Check in `reachy-mini-deploy`, `reachy-mini-on-device`, `host-provisioning`, `mcp-server-bootstrap`.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/api/daemon/restart` | Restart Daemon |
| `GET` | `/api/daemon/robot-app-lock-status` | Get Robot App Lock Status |
| `POST` | `/api/daemon/start` | Start Daemon |
| `GET` | `/api/daemon/status` | Get Daemon Status |
| `POST` | `/api/daemon/stop` | Stop Daemon |

#### Move — Bewegung und Move-Playback (`/api/move/*`)

Primäre Bewegungs-Surface: direkter Goto, kontinuierliches Set-Target-Streaming, Stop, Wake-Up / Goto-Sleep und Playback aufgenommener Move-Datasets. Konsumiert vom `reachy-mini-sdk`, `dance-choreography`, `home-assistant-bridge` (über den MCP-Layer) und allen Behaviors, die nicht direkt das `ReachyMini`-Python-Objekt nutzen.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/api/move/goto` | Goto |
| `POST` | `/api/move/play/goto_sleep` | Play Goto Sleep |
| `POST` | `/api/move/play/recorded-move-dataset/{dataset_name}/{move_name}` | Play Recorded Move Dataset |
| `POST` | `/api/move/play/wake_up` | Play Wake Up |
| `GET` | `/api/move/recorded-move-datasets/list/{dataset_name}` | List Recorded Move Dataset |
| `GET` | `/api/move/running` | Get Running Moves |
| `POST` | `/api/move/set_target` | Set Target |
| `POST` | `/api/move/stop` | Stop Move |

#### State — Sensor- und Pose-Lesezugriff (`/api/state/*`)

Reine Read-Surface für aktuelle Pose, Body-Yaw, Antennen-Joint-Positionen, Direction-of-Arrival (DoA) und den gesammelten Vollzustand. Konsumiert vom `reachy-mini-sdk` (`mini.imu`-Äquivalente für REST-Clients), `mcp-server-bootstrap`, `home-assistant-bridge` (für Sensor-Entities).

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/state/doa` | Get Doa |
| `GET` | `/api/state/full` | Get Full State |
| `GET` | `/api/state/present_antenna_joint_positions` | Get Antenna Joint Positions |
| `GET` | `/api/state/present_body_yaw` | Get Body Yaw |
| `GET` | `/api/state/present_head_pose` | Get Head Pose |

#### Motors — Motor-Modus und Diagnostik (`/api/motors/*`)

Wechsel zwischen Motor-Modi (z. B. Compliance / Stiffness) und Auslesen des Motorstatus für Diagnose. Berührt unmittelbar Hardware-Sicherheit — Konsumenten **SOLLTEN** Mode-Wechsel als seltene, bewusst gesetzte Operation behandeln, nicht als Loop-Schritt.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/api/motors/set_mode/{mode}` | Set Motor Mode |
| `GET` | `/api/motors/status` | Get Motor Status |

#### Media — Audio-Ausgabe, Sounds und Mikrofon-Lock (`/api/media/*`)

Acquire/Release auf das Media-Subsystem (exklusiver Lock für Audio-Pfad), Sound-Verwaltung (Upload, Liste, Löschen), Playback (start, stop). Konsumiert von `audio-beat-tracking`, `dance-choreography`, `app-scaffold` (Sound-Asset-Konventionen).

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/api/media/acquire` | Acquire Media |
| `POST` | `/api/media/play_sound` | Play Sound |
| `POST` | `/api/media/release` | Release Media |
| `GET` | `/api/media/sounds` | List Sounds |
| `POST` | `/api/media/sounds/upload` | Upload Sound |
| `DELETE` | `/api/media/sounds/{filename}` | Delete Sound |
| `GET` | `/api/media/status` | Media Status |
| `POST` | `/api/media/stop_sound` | Stop Sound |

#### Volume — Lautsprecher- und Mikrofon-Lautstärke (`/api/volume/*`)

Read/Write von Speaker- und Mikrofon-Lautstärke, plus Test-Sound. Konsumiert von `home-assistant-bridge` (Volume-Entities) und Bring-Up-Routinen.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/volume/current` | Get Volume |
| `GET` | `/api/volume/microphone/current` | Get Microphone Volume |
| `POST` | `/api/volume/microphone/set` | Set Microphone Volume |
| `POST` | `/api/volume/set` | Set Volume |
| `POST` | `/api/volume/test-sound` | Play Test Sound |

#### Camera — Kamera-Introspektion (`/api/camera/*`)

Stand v0.1.0 nur eine Specs-Abfrage; tatsächliche Frame-Streams sind kein REST-Inhalt. Konsumenten, die Bilder brauchen, sprechen den entsprechenden Streaming-Kanal an (außerhalb dieser Spec).

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/camera/specs` | Get Camera Specs |

#### Kinematics — URDF und STL (`/api/kinematics/*`)

Kinematik-Metadaten, URDF-Datei und STL-Meshes. Konsumiert von Sim/Visualisierung und vom `reachy-mini-sdk` zur Kollisions-/Limit-Validierung.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/kinematics/info` | Get Kinematics Info |
| `GET` | `/api/kinematics/stl/{filename}` | Get Stl File |
| `GET` | `/api/kinematics/urdf` | Get Urdf |

#### HF-Auth — Hugging-Face-OAuth und Tokens (`/api/hf-auth/*`)

OAuth-Flow gegen Hugging Face, Token-Speicherung, Status-Abfragen für den „Central Robot" und das Relay. Konsumiert von Spaces-/Privat-App-Installs (`/api/apps/install-private-space`) und allen Konsumenten, die HF-authentifizierte Aktionen anstoßen.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/api/hf-auth/central-robot-status` | Get Central Robot Status |
| `GET` | `/api/hf-auth/oauth/callback` | Oauth Callback |
| `GET` | `/api/hf-auth/oauth/configured` | Is Oauth Configured |
| `DELETE` | `/api/hf-auth/oauth/session/{session_id}` | Cancel Oauth Session |
| `GET` | `/api/hf-auth/oauth/start` | Start Oauth |
| `GET` | `/api/hf-auth/oauth/status/{session_id}` | Get Oauth Status |
| `GET` | `/api/hf-auth/relay-status` | Get Relay Status |
| `POST` | `/api/hf-auth/save-token` | Save Token |
| `GET` | `/api/hf-auth/status` | Get Auth Status |
| `DELETE` | `/api/hf-auth/token` | Delete Token |

#### Cache — System-Caches (`/cache/*`)

Hugging-Face-Cache und App-Cache zurücksetzen. Operationen sind destruktiv (löschen lokale Dateien) — Konsumenten **SOLLTEN** sie nur auf expliziten Wunsch des Nutzers aufrufen, nie als Auto-Recovery-Pfad.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/cache/clear-hf` | Clear Huggingface Cache |
| `POST` | `/cache/reset-apps` | Reset Apps |

#### Wifi — Netzwerk-Konfiguration auf Wireless (`/wifi/*`)

Scan, Connect, Forget, Setup-Hotspot, Status, Error-Handling. Praktisch nur auf der Wireless-Variante relevant; auf Lite ist das WLAN Sache des Host-Rechners.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/wifi/connect` | Connect To Wifi Network |
| `GET` | `/wifi/error` | Get Last Wifi Error |
| `POST` | `/wifi/forget` | Forget Wifi Network |
| `POST` | `/wifi/forget_all` | Forget All Wifi Networks |
| `POST` | `/wifi/reset_error` | Reset Last Wifi Error |
| `POST` | `/wifi/scan_and_list` | Scan Wifi |
| `POST` | `/wifi/setup_hotspot` | Setup Hotspot |
| `GET` | `/wifi/status` | Get Wifi Status |

#### Update — System-Updates (`/update/*`)

Read-Endpoints (`available`, `info`, `install-source`, `validate-ref`) und zwei Mutations-Endpoints (`start`, `start-from-ref`). Konsumiert vom `host-provisioning`-Pull-Service-Spec, das den Update-Pfad gegen den App-Lock koordiniert.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/update/available` | Available |
| `GET` | `/update/info` | Get Update Info |
| `GET` | `/update/install-source` | Install Source |
| `POST` | `/update/start` | Start Update |
| `POST` | `/update/start-from-ref` | Start Update From Ref |
| `GET` | `/update/validate-ref` | Validate Ref |

#### Health-Check — Liveness-Probe (`/health-check`)

Einziger Endpoint; per `POST` aufgerufen. Tauglich für Wartezyklen in `mcp-server-bootstrap` und externe Monitoring-Sonden.

| Methode | Pfad | Summary |
|---|---|---|
| `POST` | `/health-check` | Health Check |

#### Dashboard / HTML-Seiten (`/`, `/logs`, `/settings`)

Ausgelieferte HTML-Seiten der Daemon-UI; kein JSON, **SOLLTEN NICHT** als Programmatik-Endpoints konsumiert werden. Hier nur zur Existenz aufgeführt, damit Doku-Drift keine fehlerhaften „Endpoint fehlt"-Warnungen erzeugt.

| Methode | Pfad | Summary |
|---|---|---|
| `GET` | `/` | Dashboard |
| `GET` | `/logs` | Logs Page |
| `GET` | `/settings` | Settings |

### Drift-Erkennung

- Eine programmatische Prüfung **MUSS** möglich sein, die jeden (METHOD, PATH) aus dem live `openapi.json` gegen die Tabellen dieser Spec abgleicht
- Die Prüfung **SOLLTE** als kleines Skript unter `scripts/` (z. B. `scripts/check-daemon-rest-api-spec.py`) hinterlegt sein, sobald die Spec den `draft`-Status verlässt
- Bei Abweichung **SOLLTE** die Prüfung mit Exit-Code ≠ 0 und einer Liste der betroffenen Endpoints abbrechen; sie **DARF** in `task lint` integriert werden

## Akzeptanzkriterien

- [ ] Spec existiert unter `spec/reachy-mini/daemon-rest-api/de.md` (kanonisch) und `spec/reachy-mini/daemon-rest-api/en.md` (Übersetzung)
- [ ] Jeder Endpoint aus `http://reachy-mini.local:8000/openapi.json` (Stand 2026-05-12, OpenAPI 3.1.0, version `0.1.0`, 79 Operations) ist mindestens einmal in der DE-Datei aufgeführt
- [ ] Jede Endpoint-Tabelle nennt Methode, Pfad und Summary in genau dieser Reihenfolge
- [ ] Jede Namensraum-Sektion hat eine Verwendungs-Notiz, die ohne Schemawissen lesbar ist
- [ ] Spec verweist explizit auf das live `openapi.json` als autoritative Quelle und untersagt Schema-Duplikation
- [ ] Pfade enthalten FastAPI-Parameter-Notation `{name}` statt `<name>` oder freier Beschreibung
- [ ] DE- und EN-Version listen denselben Endpoint-Satz; Diff über die `|`-Tabellenzeilen ergibt nur Sprach-Unterschiede in Notiz/Summary, keine fehlenden oder zusätzlichen Pfade
- [ ] Bestehende Specs, die einzelne Endpoint-Familien duplizieren (`app-architecture` für `/api/apps/*`), behalten ihre Inhalte; neue Drift wird stattdessen hier behoben

## Referenzen

- Autoritative Live-Quelle: `http://<daemon-host>:8000/openapi.json` — typische Hosts: `http://reachy-mini.local:8000` (Wireless), `http://127.0.0.1:8000` (Lite oder Sim)
- Daemon-Quellcode (Router-Aufbau, Implementierungs-Wahrheit): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollens gehostete REST-API-Doku: <https://huggingface.co/docs/reachy_mini/API/rest-api> — beachten, dass diese Doku der live OpenAPI hinterherhinken kann
- Querverweise innerhalb dieses Repos: `spec/reachy-mini/app-architecture/` (App-Lifecycle), `spec/reachy-mini/mcp-server/` (MCP-Wrapper), `spec/reachy-mini/ha-integration/` (HA-Bridge), `spec/reachy-mini/host-provisioning/` (Update- und Pull-Service-Pfad)

## Offene Fragen

- Welche Endpoints sind plattform-spezifisch (nur Wireless: `/wifi/*`; nur Lite: ?) und sollten in einer eigenen Spalte markiert werden? Verifizieren beim ersten Lite-Hardware-Kontakt
- Stellt der Daemon eine WebSocket-Surface bereit (z. B. für Streaming-Pose-Targets, Event-Bus)? Falls ja, gehört das in eine eigene Spec — diese hier bleibt rein REST
- Soll die Drift-Prüfung als Pre-Commit-Hook oder als CI-Schritt verankert werden? Pre-Commit wäre schnell, scheitert aber ohne Netzwerk-Zugriff zum Gerät — CI mit gemocktem `openapi.json` ist wahrscheinlich robuster
- Ist `version: 0.1.0` der FastAPI-Info-Block tatsächlich die Daemon-Release-Version, oder bleibt sie unabhängig davon konstant? Bei der ersten beobachteten Bumps konsumierende Specs verlinken
