# Home-Assistant-Integration: Architektur

Status: draft

## Kontext
Der Reachy Mini soll als nativer Smart-Home-Bewohner in Home Assistant (HA) eingebunden sein — bidirektional, ohne Brückencode in Drittsystemen, mit allen für Smart-Home relevanten HA-Features: Voice Assist Pipeline, Camera Stream, Media Player, Behavior-Trigger via Service-Calls, Telemetrie als Sensoren, Behaviors als Reaktion auf Automations. Diese Spezifikation legt fest, wie die Integration verteilt, entdeckt und kommuniziert wird, welche Schichten sie hat und wie HA und Reachy entitätenseitig aufeinander treffen. Sie ist die Architektur-Quelle, gegen die der `home-assistant-bridge`-Skill, der `app-scaffold`-Skill und der Agent `reachy-mini-on-device` ihre Vorschläge ausrichten.

## Ziele
- Reachy Mini erscheint in HA als nativer Custom-Component-Eintrag, installierbar via HACS
- Maximaler HA-Funktionsumfang: Entities, Services, Events, Voice Assist Pipeline, Camera Stream, Media Player
- Voice Assist als Hauptanwendungsfall: Reachy ist ein Wyoming Voice Satellite mit lokaler Wake-Word-Detection, eingebunden in die HA Voice Assist Pipeline
- Discovery und Setup ohne YAML — Config Flow plus mDNS/Zeroconf
- Sicher per Default: TLS, Token-Auth, Spam-Schutz auf Service-Aufrufen
- Klar getrennt: Voice-Layer (Wyoming) und Robotik-Layer (REST/WebSocket über Reachy-Daemon) sind orthogonale Schichten, die im HA-Glue-Layer zusammengeführt werden

## Nicht-Ziele
- Eigene Wake-Word-Engine, eigene STT, eigene TTS — HA Voice Assist Pipeline ist Quelle der Wahrheit
- Cloud-Abhängigkeit als Pflicht: alles Lokale muss ohne Internetzugang laufen können
- HA-Add-on-Verpackung — die Integration ist ein Custom Component, kein Add-on
- Andere Smart-Home-Hubs (Hubitat, openHAB, SmartThings) — `home-assistant-bridge` deckt nur HA ab
- Aufnahme-Stream-Editor / Behavior-Authoring-UI in HA — das gehört in eine separate App, nicht in die Integration

## Anforderungen

### Verteilungs-Form
- **MUSS [MUST]** als HA Custom Integration in eigenem Repository ausgeliefert werden (`nolte/reachy-mini-hass` als Vorschlag), Layout `custom_components/reachy_mini/` mit `manifest.json` und `hacs.json`
- **MUSS [MUST]** über HACS installierbar sein
- **DARF NICHT [MUST NOT]** im selben Repository wie das `claude-reachy-mini`-Plugin liegen — die Plugin-Skills sind die Toolbox; die Integration ist die App, die mit dieser Toolbox gebaut wird
- **SOLLTE [SHOULD]** semantisches Versioning nutzen, gepinnt an die unterstützte `reachy_mini`-SDK-Version

### Architektur-Schichten
Drei klar getrennte Schichten:

1. **Voice-Layer (Wyoming)** — Reachy Mini als Wyoming Voice Satellite. Audio-In (4× PDM MEMS Mic-Array, 16 kHz) und Audio-Out (5 W @ 4 Ω Speaker) sind über das Wyoming-Protokoll an HA angebunden. Die HA Voice Assist Pipeline (Wake Word → STT → Intent → TTS) übernimmt die gesamte Sprachverarbeitung.
2. **Robotik-Layer (REST/WebSocket)** — Pose, Joint-Steuerung, Behavior-Wiedergabe, Telemetrie und Kamera laufen über die offizielle Reachy-Daemon-API. Konventionen für Auth, Reconnect, Backpressure folgen dem `home-assistant-bridge`-Skill.
3. **HA-Glue-Layer (Custom Integration)** — vereint beide Schichten, exponiert HA-Entities/Services/Events, koordiniert Voice-getriggerte Behaviors mit der Roboter-Bewegung.

Diese Trennung ist Pflicht: Voice-Pipeline-Änderungen (z. B. neuer Wake-Word-Engine in HA) dürfen Robotik-Code nicht beeinflussen, und Robotik-Updates (z. B. neue Behaviors) dürfen den Voice-Stack nicht brechen.

### Discovery und Setup
- **MUSS [MUST]** Zeroconf/mDNS-Service `_reachy_mini._tcp.local.` ankündigen, damit HA den Roboter automatisch erkennt
- **MUSS [MUST]** einen Config Flow in HA bereitstellen — kein YAML-Setup
- **MUSS [MUST]** im Config Flow folgende Felder erfassen: Hostname/IP, Port, Long-Lived Access Token (Reachy-Daemon), erkannte Plattform (Wireless / Lite / Simulation), bevorzugte Voice Pipeline (HA-Pipeline-Selektor), Default-Wake-Word
- **SOLLTE [SHOULD]** beim Erst-Setup eine Test-Bewegung (`wake_up`) auslösen, damit der Nutzer die Hardware-Verbindung sofort verifizieren kann
- **MUSS [MUST]** mehrere parallele Config Entries für mehrere Reachy-Mini-Geräte unterstützen (jedes als eigenes Device)

### Verbindung zum Reachy-Daemon
- **MUSS [MUST]** REST für synchrone Aktionen nutzen (Pose-Befehl, Behavior-Trigger, Status-Abfragen)
- **MUSS [MUST]** WebSocket für Telemetrie-Streams nutzen (`JointPositionsMsg`, `HeadPoseMsg`, `ImuDataMsg` bei 50 Hz)
- **MUSS [MUST]** Wiederverbindung mit Exponential Backoff implementieren — verlorene Verbindung darf den HA-Startup nicht blockieren; nach Reconnect: State-Resync via `get_states`
- **MUSS [MUST]** den Authorization-Header mit Long-Lived Access Token an jeden Request anhängen
- **DARF NICHT [MUST NOT]** den Token in HA-Logs ausgeben — Maskierung Pflicht (`token[:4] + "…"`)

### HA Entity Surface

| Entity | Typ | Beschreibung |
|---|---|---|
| `camera.reachy_mini_head` | Camera | Stream der Kopf-Kamera (Sony IMX708, 12 MP, autofocus); WebRTC-Backend |
| `media_player.reachy_mini_speaker` | Media Player | 5 W @ 4 Ω Speaker, Volume 0–100, TTS-Entgegennahme |
| `light.reachy_mini_led_ring` | Light | LED-Ring am Mic-Modul (`LED_EFFECT`, `LED_BRIGHTNESS`, `LED_GAMMIFY`, `LED_SPEED`) |
| `sensor.reachy_mini_imu_*` | Sensor | Accelerometer, Gyroscope, Quaternion, Temperatur — **nur Wireless** |
| `sensor.reachy_mini_battery` | Sensor | Akku-Stand in Prozent — **nur Wireless** |
| `sensor.reachy_mini_head_pose` | Sensor | aktuelle Head-Pose (4×4-Matrix als Attribut, eulerwinkel als State) |
| `select.reachy_mini_idle_mode` | Select | Optionen: `waiting-idle` / `alert-listening` / `thinking` / `off` |
| `select.reachy_mini_behavior` | Select | Auswahl aus den 29 Motion-Slugs als One-Shot-Trigger |
| `switch.reachy_mini_motors_enabled` | Switch | Motoren ein/aus (`enable_motors` / `disable_motors`) |
| `switch.reachy_mini_gravity_compensation` | Switch | Gravity-Compensation an Kopf-Motoren ein/aus |
| `switch.reachy_mini_automatic_body_yaw` | Switch | `set_automatic_body_yaw` ein/aus |
| `binary_sensor.reachy_mini_person_detected` | Binary Sensor | Person im Kamerabild erkannt |
| `binary_sensor.reachy_mini_sound_detected` | Binary Sensor | Sound im Mic-Array erkannt (DOA-fähig) |
| `binary_sensor.reachy_mini_motion_active` | Binary Sensor | aktuell läuft ein Behavior |
| `update.reachy_mini_firmware` | Update | aktuelle Daemon-Version vs. neueste verfügbare |
| `assist_satellite.reachy_mini` | Assist Satellite | Wyoming-Voice-Satellite-Integration (HA `assist_satellite`-Domain ab `2024.10`) |

- **MUSS [MUST]** alle Entities mit korrektem `device_class` und `state_class` anlegen, damit Long-Term-Statistics in HA funktionieren
- **SOLLTE [SHOULD]** Plattform-Profile berücksichtigen: IMU- und Akku-Sensoren erscheinen nicht auf Lite oder Simulation

### Number-Entity-Setpoint-Semantik

HA-Slider (NumberEntity) folgen einem **Setpoint-Modell**, nicht einem Hardware-Position-Modell: der angezeigte Slider-Wert spiegelt den **User-Intent** wider, nicht die zur Lese-Zeit aktuelle Servo-Position. Wer das missachtet, baut sich einen sichtbaren Off-by-one-Echo-Bug in den Slider, weil HA nach jedem Push den State zur Validierung abfragt und den zurückgemeldeten Wert dem Slider zuweist.

- **MUSS [MUST]** der Setter einer Number-Entity (Antennen, Head-X/Y/Z, Head-Roll/Pitch/Yaw, Body-Yaw und alle weiteren Pose-Achsen) den App-internen State-Setpoint **synchron im selben Tick** schreiben, in dem die Setter-Funktion läuft. Wenn die App-Architektur den eigentlichen Hardware-Befehl asynchron über eine Command-Queue an einen Control-Loop delegiert, **MUSS** der State zusätzlich synchron geschrieben werden, bevor der Setter zurückkehrt — nicht erst beim nächsten Queue-Drain
- **MUSS [MUST]** der Getter derselben Number-Entity den **App-State-Setpoint** zurückgeben, **nicht** die live aus dem Daemon gelesene Hardware-Joint-Position; die Hardware lagt durch Servo-Kinematik typischerweise um Hunderte Millisekunden hinter dem Setpoint, der State-Read ist immer schneller als die physische Bewegung. Der Slider zeigt den User-Intent, nicht den Servo-Lag
- **MUSS [MUST]** für die identische Achse Setter-Schreib-Feld und Getter-Lese-Feld dieselbe State-Variable referenzieren — Asymmetrie zwischen den beiden (Setter schreibt App-State, Getter liest Hardware) erzeugt das Off-by-one-Echo-Symptom
- **DARF NICHT [MUST NOT]** in der NumberEntity-Implementierung im selben handle-Block eines `NumberCommandRequest` der State-Echo-Response vor der State-Aktualisierung yieldet werden; die Reihenfolge muss zuerst-schreiben-dann-yielden sein
- **SOLLTE [SHOULD]** das Pattern in der App-Code-Basis als Helper / Mixin gekapselt werden, sodass alle Pose-Achsen (Antennen, Head-Achsen, Body-Yaw) demselben synchronen Setter/Getter-Schema folgen. Sonst ist das Pattern für jede neue Pose-Achse erneut latent zu reparieren

### Custom Services
- **MUSS [MUST]** folgende Services exponieren:
  - `reachy_mini.play_behavior(behavior: str, speed: float = 1.0)` — startet eines der 29 Motion-Specs
  - `reachy_mini.cancel_behavior()` — bricht das aktuell laufende Behavior ab
  - `reachy_mini.goto_pose(x_mm, y_mm, z_mm, roll_deg, pitch_deg, yaw_deg, duration: float = 1.0, method: str = "min_jerk")` — direkter Pose-Befehl mit Interpolations-Modus aus dem `InterpolationTechnique`-Enum
  - `reachy_mini.set_antennas(left_deg: float, right_deg: float, duration: float = 0.5)`
  - `reachy_mini.set_body_yaw(value_deg: float, duration: float = 0.5)`
  - `reachy_mini.look_at_world(x: float, y: float, z: float, duration: float = 0.5)` — World-Frame-Tracking
  - `reachy_mini.look_at_image(u: int, v: int, duration: float = 0.5)` — Pixel-Tracking auf der Kamera
  - `reachy_mini.speak(message: str, voice: str = None, behavior_during: str = None)` — TTS via HA-Pipeline auf den Speaker, optional gleichzeitig ein Behavior
  - `reachy_mini.set_idle_mode(mode: str)` — wechselt den Idle-State
  - `reachy_mini.wake_up()` / `reachy_mini.goto_sleep()` — High-Level-SDK-Methoden
  - `reachy_mini.start_recording()` / `reachy_mini.stop_recording()` — Trajektorien-Capture, Rückgabe als HA-Event
- **MUSS [MUST]** alle Services mit HA-Service-Schemas (`vol.Schema`) validieren — kein ungeprüfter Input
- **SOLLTE [SHOULD]** ein Rate-Limit gegen Service-Spam haben: max. 10 Behaviors / Minute, max. 1 `goto_pose` / 0,5 s

### Custom Events
- **MUSS [MUST]** folgende HA-Events feuern:
  - `reachy_mini_behavior_started(behavior: str, source: str)` — `source` aus `service_call` / `voice` / `automation`
  - `reachy_mini_behavior_finished(behavior: str, status: str)` — `status` ∈ {`PASS`, `FAIL`, `ABORTED`} analog zum Agent `reachy-mini-on-device`
  - `reachy_mini_wake_word_detected(wake_word: str)`
  - `reachy_mini_voice_command_received(intent: str, slots: dict)` — abgeleitet aus HA-Intent-Pipeline
  - `reachy_mini_person_detected(confidence: float, bounding_box: dict)`
  - `reachy_mini_sound_direction_detected(angle_deg: float, confidence: float)` — DOA aus Mic-Array
  - `reachy_mini_low_battery(percentage: float)` — nur Wireless
  - `reachy_mini_safety_threshold_exceeded(metric: str, value: float)` — z. B. Temperatur, Strom, Joint-Limit
- **DARF NICHT [MUST NOT]** Events öfter als 5 Hz pro Event-Typ feuern — HA-State-Bus würde sonst überlaufen

### Voice Assist (Wyoming)
- **MUSS [MUST]** Reachy als HA Wyoming Satellite registrieren, mit Audio-Eingang (Mic-Array) und Audio-Ausgang (Speaker)
- **MUSS [MUST]** lokale Wake-Word-Detection laufen lassen — Engine `microWakeWord` (lokal auf RPi 4 CM4) oder `openWakeWord` (auf HA-Host); konfigurierbares Wake-Word, Default-Vorschlag `> ⚠ TBD: pick portfolio default`
- **MUSS [MUST]** Voice Activity Detection (VAD) auf dem Audio-Stream laufen lassen, um Aufzeichnung beim Stillstand zu beenden
- **MUSS [MUST]** Audio in 16 kHz, mono, 16-bit PCM an HA streamen (Wyoming-Standard)
- **MUSS [MUST]** TTS-Audio von HA empfangen und über den Reachy-Speaker abspielen
- **SOLLTE [SHOULD]** während der Voice-Interaktion entsprechende Behaviors triggern: `alert-listening` während Wake-Word + Listening, `thinking` während STT/Intent, `recognition` oder `agreeing-nod` nach erfolgreichem Intent, `confused` nach fehlgeschlagenem Intent
- **DARF NICHT [MUST NOT]** Audio in die Cloud schicken, wenn die HA-Pipeline lokal konfiguriert ist
- Wyoming-Protokoll-Doku: <https://github.com/rhasspy/wyoming>

### Audio-Routing
- **MUSS [MUST]** Speaker-Output für Wyoming-TTS denselben Audio-Pfad nutzen wie `reachy_mini.speak`-Service-Calls — kein separater Pfad
- **SOLLTE [SHOULD]** Eingabe-Audio während aktiver Bewegung (Servo-Surren) optional dämpfen: Echo-Cancellation oder Mic-Mute während großer Pose-Änderungen — `> ⚠ TBD: empirisch testen`
- **SOLLTE [SHOULD]** Lautstärke kontextabhängig steuern (z. B. nachts leiser via HA-Sensor)

### Kamera-Stream
- **MUSS [MUST]** den Kamera-Stream als WebRTC-Stream über die Reachy-Daemon-WebRTC-API bereitstellen (`media_server.py`, `webrtc_client_gstreamer.py` aus dem SDK)
- **SOLLTE [SHOULD]** zusätzlich einen RTSP-Endpoint anbieten, damit HA-seitige Object-Detection-Integrationen (Frigate, DeepStack, Doods) den Stream konsumieren können
- **KANN [MAY]** einen Snapshot-Service (`reachy_mini.take_snapshot`) als Convenience exponieren

### Behavior-Mapping zur Motion-Catalogue
- **MUSS [MUST]** alle 29 Motion-Slugs aus `spec/reachy-mini/motions/` als gültige Werte des `select.reachy_mini_behavior`-Entities und des `reachy_mini.play_behavior`-Services akzeptieren
- **MUSS [MUST]** den `speed`-Parameter (0,5–2,0) auf alle Phasen-Dauern proportional anwenden — `speed=2.0` halbiert alle Dauern, `speed=0.5` verdoppelt sie
- **DARF NICHT [MUST NOT]** zwei Behaviors gleichzeitig wiedergeben — neuer `play_behavior`-Aufruf bricht das laufende Behavior via `cancel_move` ab
- **SOLLTE [SHOULD]** für loopfähige Behaviors (`thinking`, `alert-listening`, `waiting-idle`, BPM-Tanz-Bausteine) automatisch Loop-Anzahl vom HA-Caller akzeptieren oder bis zum nächsten Service-Call durchlaufen lassen

### Reachy → HA Patterns
- **MUSS [MUST]** Telemetrie-Events (Person erkannt, Sound erkannt, niedriger Akku) in HA-Events übersetzen, sodass Automations darauf reagieren können
- **MUSS [MUST]** Reachy-getriggerte HA-Service-Aufrufe über die HA Long-Lived Access Token Auth abwickeln — siehe `home-assistant-bridge`-Skill
- **SOLLTE [SHOULD]** Telemetrie-Frequenz an HA-State-Update-Praxis anpassen — IMU publiziert mit 50 Hz vom Daemon, HA-Sensor sollte aber max. mit 1–5 Hz aktualisiert werden, sonst werden HA-Datenbanken überlastet (Recorder-Schutz)

### Sicherheit
- **MUSS [MUST]** TLS-Verifikation für die Verbindung zwischen HA und Reachy-Daemon erzwingen (`verify=True`); für lokale Setups mit Self-Signed-Cert ein dediziertes CA-Bundle nutzen, kein `verify=False`
- **MUSS [MUST]** Long-Lived Access Token sowohl HA → Reachy als auch Reachy → HA durchgehend nutzen
- **MUSS [MUST]** Token im HA-Secrets-Storage halten und im Config Flow nicht als Plaintext anzeigen
- **MUSS [MUST]** alle Service-Aufrufe durch HA-User-Permissions filtern — nicht jeder HA-User darf z. B. `reachy_mini.disable_motors` aufrufen
- **SOLLTE [SHOULD]** Rate-Limits gegen Service-Spam implementieren (max. 10 Behaviors / Minute, max. 1 `goto_pose` / 0,5 s)
- **DARF NICHT [MUST NOT]** Audio-Streams unverschlüsselt über öffentliche Netzwerke schicken — Reachy und HA müssen im selben LAN oder via VPN sein

### Plattform-Profile
- **MUSS [MUST]** zwischen den drei Plattformen unterscheiden:
  - **Wireless** — voller Feature-Satz, IMU verfügbar, Akku-Sensor verfügbar
  - **Lite** — gleiches Aktuator-Set wie Wireless, aber keine IMU-Telemetrie und kein Akku-Sensor; entsprechende Sensor-Entities werden nicht erzeugt
  - **Simulation** — alle Aktuator-Befehle akzeptiert, keine Sensor-Daten (außer Pose-Read), kein Audio
- **SOLLTE [SHOULD]** beim Erst-Setup die Plattform automatisch erkennen (`DaemonStatus.no_media` als Indikator für Sim, `DaemonStatus.camera_specs_name` als Indikator für Hardware-Variante)

### Versionspolitik und Kompatibilität
- **MUSS [MUST]** die unterstützte HA-Version im `manifest.json` deklarieren — Vorschlag `>=2024.10` (`assist_satellite`-Domain stabil seit `2024.10`); `> ⚠ TBD: validate against current HA Voice Assist features`
- **MUSS [MUST]** die kompatible `reachy_mini`-SDK-Version pinnen — bei SDK-Major-Update wird die Integration neu validiert
- **SOLLTE [SHOULD]** Migration-Pfade für Config-Flow-Änderungen via HA `async_migrate_entry` bereitstellen
- **SOLLTE [SHOULD]** Renovate-Konfiguration im Custom-Component-Repo aktivieren, damit SDK-Pin und HA-Mindestversion automatisiert nachgeführt werden

## Akzeptanzkriterien
- [ ] HA findet Reachy via mDNS automatisch und startet den Config Flow
- [ ] Nach Setup erscheint Reachy mit allen relevanten Entities im HA-Dashboard
- [ ] Wyoming Voice Satellite ist registriert und antwortet auf Wake-Word mit `alert-listening`
- [ ] STT/Intent/TTS-Pipeline läuft Ende-zu-Ende ohne Cloud
- [ ] Alle 29 Motion-Slugs sind als Service-Parameter aufrufbar
- [ ] Camera-Stream ist im HA-Dashboard sichtbar (WebRTC)
- [ ] LED-Ring reagiert auf `light.reachy_mini_led_ring`-Befehle
- [ ] IMU-Sensoren publizieren auf Wireless mit ~1 Hz HA-Update-Rate
- [ ] Reachy → HA Service-Calls funktionieren (z. B. `light.turn_on`)
- [ ] `reachy_mini_low_battery` Event wird bei < 20 % Akku gefeuert
- [ ] TLS-Verifikation ist standardmäßig aktiv und kann nur mit explizitem CA-Bundle relaxiert werden
- [ ] Token werden nicht in Logs ausgegeben (Maskierung in jedem Code-Pfad)
- [ ] Plattform-Profile blenden inkompatible Entities korrekt aus
- [ ] HACS-Installation funktioniert ohne manuellen Eingriff
- [ ] Mehrere parallele Reachy-Mini-Geräte als eigene Devices konfigurierbar
- [ ] Migration von einer früheren Config-Flow-Version läuft ohne User-Eingriff durch

## Quellen
- Upstream-SDK-Repo (Quelle der Wahrheit für Daemon-API, IO-Protokoll, Media-Stack): <https://github.com/pollen-robotics/reachy_mini>
- Daemon (REST-API, App-Lock, Lifecycle — das, was die HA-Integration als Reachy-Endpoint anspricht): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- IO-Protokoll (`JointPositionsMsg`, `HeadPoseMsg`, `ImuDataMsg`, LED-/Mic-Befehle wie `SetMicrophoneVolumeCmd`): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- Media-Stack (Kamera, WebRTC via GStreamer, Audio-DOA, Speaker — Grundlage für `camera.*`- und `media_player.*`-Entities): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/media>
- REST-API-Doku (HA-Integration spricht primär gegen diese): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx> mit OpenAPI-Schema unter <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/openapi.json>
- SDK-Integrations-Doku (Beispiele für externe Konsumenten — analog zu HA-Bridge): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/integration.md>
- Wyoming-Protokoll (Voice-Layer, externer Standard): <https://github.com/rhasspy/wyoming>

## Offene Fragen
- Welche Mindest-HA-Version setzen wir genau? Vorschlag `>=2024.10` wegen `assist_satellite`-Domain.
- Welches Default-Wake-Word? „Hey Reachy"? Lokales Modell muss trainiert werden — `microWakeWord` ist trainierbar, Aufwand ~30 min.
- Wie wird Audio-Latenz zwischen Wyoming-TTS und Behavior-Trigger synchronisiert? Pattern aus `control-surface` (~50 ms Audio-Buffer) berücksichtigen — möglicherweise gemeinsame Phase-Lock-Logik.
- Soll die Integration optional eine Cloud-AI-Anbindung (OpenAI, Anthropic) ausliefern, oder strikt lokal? Tendenz: strikt lokal in der Integration; HA Cloud / Nabu Casa kann das auf der Pipeline-Seite ergänzen.
- Welche WebRTC-Auflösung als Default? 720p mit Konfigurierbarkeit auf 480p / 1080p.
- Soll der HACS-Repo-Slug `reachy-mini-hass` oder `home-assistant-reachy-mini` heißen? HACS-Konvention bevorzugt das `<integration-name>-hass`-Pattern.
- Gibt es eine Plugin-Service-API für Tanz-Apps (`reachy_mini.start_dance(bpm: float, duration: float)`), oder muss App-Code die BPM-Tanz-Bausteine direkt orchestrieren?
- Welche LED-Effekt-Vokabular-Werte werden in `light.reachy_mini_led_ring` exponiert (`solid`, `breathing`, `chasing`, `pulse`)? Hängt von ReSpeaker-Firmware ab — `> ⚠ TBD`.
- Wie verhält sich die Integration bei einem Daemon-Reboot mitten in einem Behavior? Vorschlag: aktuelle Behavior-State persistieren, nach Reconnect zur Neutralpose fahren.
- Soll der Custom Component eine Diagnostics-API (`config/diagnostics`) für HA Bug-Reports liefern? HA-Konvention seit `2022.x` — sollte mit aufgenommen werden.
- Wie integriert sich die Integration mit HA-Energie-Dashboard? Vorschlag: Sensor `sensor.reachy_mini_power_consumption` (estimiert oder vom Daemon, falls verfügbar).
- Soll der Wyoming-Satellite eigenständig auf Reachy laufen oder als Bridge in der HA Custom Integration? Tendenz: eigenständiger Wyoming-Server auf Reachy (RPi 4 CM4 ist stark genug), HA spricht ihn nur als Client an.
