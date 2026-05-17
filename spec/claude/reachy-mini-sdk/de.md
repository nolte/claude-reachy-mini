# Reachy-Mini-SDK-Skill

Status: draft

## Kontext
Das `reachy_mini`-Python-SDK von Pollen Robotics / Hugging Face ist die primäre Schnittstelle, um den Reachy-Mini-Roboter programmatisch zu steuern: Kopf-Bewegungen (Pan/Tilt/Roll), Antennen, Behaviors, optional Audio- und Vision-Streams. Claude Code soll bei jeder Berührung dieses SDKs idiomatischen, lauffähigen Code produzieren — dafür braucht es eine treffsichere Wissensbasis, die genau dann aktiviert wird, wenn die Aufgabe das SDK berührt, und sich beim Drift offenbart, statt veraltete Patterns zu wiederholen. Diese Spezifikation regelt, was der Skill `reachy-mini-sdk` liefert und welche Themen explizit anderen Skills überlassen bleiben.

## Ziele
- Claude Code erkennt zuverlässig, wenn eine Aufgabe das `reachy_mini`-SDK berührt, und aktiviert genau dann diesen Skill
- Claude Code erzeugt idiomatischen, gegen die offizielle API verifizierten Code
- Jeder Code-Vorschlag bezieht sich auf eine namentlich genannte SDK-Version
- Das Plugin macht Versions-Drift sichtbar, statt ihn zu kaschieren
- Nicht-Anliegen werden klar an spezialisierte Skills delegiert, statt diesen Skill aufzublähen

## Nicht-Ziele
- Hardware-Bringup, Kalibrierung, Firmware-Flash (eigener Skill geplant)
- Simulation / MuJoCo / URDF des Reachy-Modells (separater Skill möglich)
- Veröffentlichung von Behaviors auf Hugging Face Spaces / Hub (eigener Skill `reachy-app-publish-hf` geplant)
- Beat- und Tempo-Erkennung für Tanz-Anwendungen (eigener Skill `audio-beat-tracking` geplant)
- Home-Assistant-Integration (eigener Skill `home-assistant-bridge`)
- Scaffolding eines neuen Behaviors (eigener Skill `app-scaffold`)
- Live-Deployment / Test auf dem Gerät (eigener Agent `reachy-mini-on-device`)

## Anforderungen

### Trigger und Aktivierung
- **MUSS [MUST]** eine treffsichere Skill-Description liefern, die Claude Code bei jeder Aufgabe aktiviert, die das `reachy_mini`-SDK berührt — erkennbar an Imports von `reachy_mini`, der Klasse `ReachyMini`, Behavior-Definitionen oder API-Aufrufen für Kopf-/Antennen-Bewegung
- **MUSS [MUST]** Schlüssel-Trigger-Begriffe in der Description enthalten: `reachy_mini`, `ReachyMini`, Behavior, Antennen, Pan/Tilt/Roll, Move
- **SOLLTE [SHOULD]** explizit benennen, _wann der Skill nicht_ aktiviert werden soll (z. B. reine Hardware-Bringup-Aufgaben oder reine Simulation ohne SDK-Kontakt)

### Wissensbasis-Inhalt
- **MUSS [MUST]** die folgenden API-Bausteine des SDKs dokumentieren:
  - Konstruktion und Connection-Management der `ReachyMini`-Instanz inklusive Lifecycle (Open/Close, Context-Manager, Sync- vs. Async-Variante, `spawn_daemon=True, use_sim=True` für Sim)
  - **Wake-/Sleep-Lifecycle**: `wake_up()` vor jeder Bewegung — sonst werden Pose-Befehle stillschweigend ignoriert; `goto_sleep()` für Idle/Shutdown; Pose-Konstanten `SLEEP_HEAD_POSE`, `INIT_HEAD_POSE`, `INIT_ANTENNAS_JOINT_POSITIONS` aus `src/reachy_mini/reachy_mini.py`
  - **Bewegungs-API mit Methoden-Wahl**: explizit beide Pfade dokumentieren —
    - `goto_target(head=<4×4>, antennas=[r, l] in rad, body_yaw, duration, method)` als **Default für choreographierte Bewegungen ≥ 0,5 s**; `method` aus `InterpolationTechnique`-Enum: `MIN_JERK` (Default), `LINEAR`, `EASE_IN_OUT`, `CARTOON`
    - `set_target(head, antennas, body_yaw)` als **Real-Time-Pfad für High-Frequency-Loops** (50–100 Hz Tick-Frequenz, **Single-Owner-Loop**); nicht mit `goto_target` mischen, sonst überschreiben sie sich gegenseitig
    - Quelle: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/motion-philosophy.md>
  - Kopf-Bewegung: 6 DoF Stewart-Plattform, 4×4-Pose-Matrix, Builder `create_head_pose(x, y, z, roll, pitch, yaw, degrees=True)`; Wertebereiche siehe [`reachy-mini/control-surface`](../../reachy-mini/control-surface/de.md) (Head pitch/roll ±40°, Head yaw ±60°, Body yaw ±155°, Yaw-Delta ≤ 65°)
  - Antennen-Steuerung: 2× XL330-M077-T, **Reihenfolge `[right, left]` in Radiant** (nicht in Grad!), Winkel, Geschwindigkeit, Synchronisation mit Kopf-Moves
  - **`Move`-ABC und `play_move()`/`async_play_move()`**: für wiederverwendbare Bewegungen eigene `Move`-Subklasse mit `duration`-Property und `evaluate(t) → (head, antennas, body_yaw)`-Methode (Quelle: `src/reachy_mini/motion/move.py`)
  - **App-Lifecycle**: `ReachyMiniApp`-Subklasse mit `run(self, reachy_mini, stop_event)`; `wrapped_run()` im `__main__`; Tick-Frequenz, sauberes Stoppen via `stop_event`, Exception-Handling
  - **Safe-Torque-Anti-Jerk-Pattern** (Pflicht beim Motor-Toggle, sonst springt der Kopf):
    1. Vor `disable_motors()`: zu `SLEEP_HEAD_POSE` fahren
    2. Vor `enable_motors()`: das Goal auf die aktuelle Pose setzen (kurzes `goto_target` mit `duration ≈ 0.05`), erst dann enablen
    3. Bei Mixed-Motor-Zustand (manche an, manche aus): erst voll `disable_motors()` aller IDs, dann sequenziell wieder enablen
    Quelle: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- **MUSS [MUST]** mindestens ein lauffähiges, minimales Code-Beispiel pro dokumentiertem Bereich enthalten
- **MUSS [MUST]** für jedes Beispiel die SDK-Version benennen, gegen die es verifiziert wurde, und auf die offizielle Quelle verweisen (Pollen-Robotics-Doku oder offizielles GitHub-Repo)
- **SOLLTE [SHOULD]** Async-Patterns abdecken (Tasks, Cancellation, Cleanup bei Exceptions), wenn das SDK eine Async-Oberfläche bietet
- **SOLLTE [SHOULD]** typische Fehlerbedingungen benennen (Hardware nicht angeschlossen, USB-/Serial-Fehler, Protokoll-Mismatch zwischen SDK und Firmware, fehlender Daemon)
- **KANN [MAY]** Hinweise zu Update-Frequenz, Latenz und Performance von Behaviors aufnehmen

### Code-Beispiel-Konventionen
- **MUSS [MUST]** alle Beispiele auf eine Python-Untergrenze zielen, die der offiziellen SDK-Anforderung entspricht; bei Drift wird die Untergrenze per Spec-Update angepasst
- **MUSS [MUST]** Beispiele im Stil zeigen, den das offizielle SDK selbst vorgibt (z. B. `with`-Statement, falls das SDK ein Context-Manager-Modell vorlebt)
- **MUSS [MUST]** jeden Snippet mit einem Quell-Verweis auf die offizielle Pollen-Robotics-Doku oder das offizielle GitHub-Repo versehen
- **MUSS [MUST]** vor jeder verwendeten SDK-Funktion deren Existenz in `src/reachy_mini/reachy_mini.py` (oder dem zuständigen Submodul) belegen — gibt es die Funktion nicht oder hat sie eine andere Signatur, wird sie nicht halluziniert, sondern als Open Question mit Verweis auf den Source-Tree markiert. Quelle für die Regel: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/deep-dive-docs.md> („Before using any SDK function: 1. Verify it exists … 2. Check the signature … 3. Read the docstring.")
- **DARF NICHT [MUST NOT]** Code-Beispiele enthalten, die ungeprüft aus älteren Reachy-SDKs (Reachy 2, Reachy Pro) übernommen wurden — Wiederverwendungen müssen markiert werden, wenn die API für Reachy Mini abweicht

### Plattform-Profile (Wireless / Lite / Simulation)
Wirklich kanonisch ist [`reachy-mini/control-surface`](../../reachy-mini/control-surface/de.md). Für Schnell-Orientierung im SDK-Kontext:

| Plattform | Compute / Connect | Aktuator-Set | Sensoren / Audio | Sim-Konstruktor |
|---|---|---|---|---|
| Reachy Mini (Wireless) | RPi 4 CM4 + LiFePO4-Akku, autark, mDNS `reachy-mini.local:8000` | Stewart-Kopf, 2 Antennen, Body-Yaw | IMU (`mini.imu`), Battery, Mic-Array, Kamera, LED-Ring | n/a |
| Reachy Mini Lite | tethered an Host-PC via USB-C, externe Spannung | identisch zu Wireless | **keine IMU**, **kein Battery-Sensor**, sonst voll | n/a |
| Simulation | softwareseitig im Python-Prozess oder via `reachy-mini-daemon --sim` | identisch (logisch) | keine echten Sensoren (außer Pose-Read), **keine Audio-/Kamera-Wiedergabe**; voller `--sim` braucht GStreamer + MuJoCo, sonst `--mockup-sim --no-media --headless` | `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:` |

- **MUSS [MUST]** im Code-Beispiel und in jedem Wissens-Snippet die Plattform benennen, gegen die er gilt; Plattform-spezifische Annahmen (IMU-Read auf Wireless) als solche markieren statt sie als universell zu zeigen

### Sicherheits-Limits (kanonisch im control-surface)
Hard-coded Werte aus dem SDK, die in jedem Code-Snippet einzuhalten sind:

| Achse | Min | Max | Quelle |
|---|---|---|---|
| Head pitch / roll | −40° | +40° | `analytical_kinematics.py` plus offizielle Hardware-Datasheet-Tabelle |
| Head yaw | −60° | +60° | dito |
| Head yaw relativ zum Body | — | ±65° | `max_relative_yaw` |
| Body yaw | −155° | +155° | `max_body_yaw=np.deg2rad(160)` |
| Antenne (je) | −180° | +180° | URDF |

- **MUSS [MUST]** Wertebereiche in jedem Code-Snippet einhalten und gegen Überschreitung per Assertion sichern; Quelle und vollständige Tabelle: [`reachy-mini/control-surface`](../../reachy-mini/control-surface/de.md)

### Versions-Pinning und Drift-Erkennung
- **MUSS [MUST]** im Skill-Body die SDK-Version benennen, gegen die der Skill aktuell verifiziert ist (z. B. `reachy_mini==0.x.y`)
- **SOLLTE [SHOULD]** einen wiederholbaren Drift-Check vorsehen: bei jedem neuen `reachy_mini`-Release wird der Skill gegen die aktuelle API geprüft, entweder manuell beim nächsten Touchpoint oder über einen geplanten Audit-Skill
- **MUSS [MUST]** Aussagen, die mangels Hardware oder mangels Verifikation nicht belegt sind, durch eine sichtbare Markierung kennzeichnen (z. B. `> ⚠ TBD: zu validieren mit echter Hardware`)

### Schnittstellen zu benachbarten Skills
- **SOLLTE [SHOULD]** auf den Skill `app-scaffold` verweisen, sobald die Aufgabe ein _neues_ Behavior anlegt — statt Scaffolding-Logik zu duplizieren
- **SOLLTE [SHOULD]** auf den Skill `home-assistant-bridge` verweisen, sobald die Aufgabe Reachy mit Home Assistant verbindet
- **SOLLTE [SHOULD]** auf den geplanten Skill `audio-beat-tracking` verweisen, sobald die Aufgabe Audio analysiert (z. B. für Tanz-Synchronisation)
- **SOLLTE [SHOULD]** auf den geplanten Agent `reachy-mini-on-device` verweisen, sobald die Aufgabe ein Behavior live auf dem Gerät testen will

## Akzeptanzkriterien
- [ ] Der Skill ist unter `skills/reachy-mini-sdk/SKILL.md` mit gültiger Frontmatter (`name: reachy-mini-sdk`, `description`, optional `tags`) angelegt und wird vom Katalog-Generator akzeptiert
- [ ] Die `description`-Frontmatter ist so geschrieben, dass Claude Code den Skill bei einer Test-Aufgabe aktiviert, die `from reachy_mini import ReachyMini` enthält
- [ ] Die Wissensbasis dokumentiert mindestens: Konstruktion / Lifecycle, Kopf-Bewegung, Antennen, Behavior-Loop, Cleanup
- [ ] Mindestens ein lauffähiges Code-Beispiel pro dokumentiertem Bereich existiert
- [ ] Jedes Beispiel trägt eine Quell-Referenz und benennt die SDK-Version
- [ ] Die geprüfte SDK-Version ist im Skill-Body explizit ausgewiesen
- [ ] Out-of-Scope-Themen (Bringup, Simulation, HF-Publishing, Beat-Tracking, HA-Bridge, App-Scaffolding, On-Device-Testing) sind als „dafür gibt es Skill / Agent X" markiert
- [ ] Aussagen ohne Hardware-Verifikation tragen einen sichtbaren TBD-Hinweis
- [ ] Der Skill wird im MkDocs-Katalog korrekt gerendert (Build läuft `task docs --strict` ohne Fehler)

## Quellen
- Upstream-SDK-Repo (kanonische Quelle für API, Versionierung, Lizenz): <https://github.com/pollen-robotics/reachy_mini>
- SDK-Source-Tree (`ReachyMini`, IO, Media, Motion, Daemon, Apps, Tools): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini>
- Motion-Modul (`Move`-ABC, Easing-Modi `MIN_JERK`/`CARTOON`, `goto`, `recorded_move`): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/motion>
- API-Doku (MDX-Quellen für `reachymini`, `media`, `motion`, `daemon`, `apps`, `tools`, `utils`, REST-API, OpenAPI-Schema): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/API>
- SDK-Konzept-Doku (Quickstart, Core-Concept, Apps, Python-/JavaScript-SDK, Media-Architektur, Installation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Lauffähige Beispiele (kanonische Vorbilder für Code-Snippets): <https://github.com/pollen-robotics/reachy_mini/tree/main/examples>
- Plattform-Profile-Doku (Wireless / Lite / Simulation): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/platforms>
- Pollens `AGENTS.md` (Einstiegspunkt für AI-Agents in den Pollen-Workflow): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>

Plugin-interne Wissens-Specs (kanonische Quelle für Cross-Phase-Konventionen):

- Anomalie-Klassen und verbindliches Event-Record-Schema (dieser Skill ist **Pre-Flight-Konsument** für Klasse A Pose-Range, Klasse B Pose-Delta/dt, Klasse D lokaler IK-Check): [`reachy-mini/motion-anomaly-detection`](../../reachy-mini/motion-anomaly-detection/de.md)

Pollens parallele Authoring-Skills (jeder ist Quelle für einen bestimmten Aspekt der Wissensbasis und wird im Drift-Check abgeglichen):

- `motion-philosophy.md` — `goto_target` vs. `set_target`, Methoden-Wahl: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/motion-philosophy.md>
- `control-loops.md` — 50–100 Hz Single-Owner-Loop-Konvention: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/control-loops.md>
- `safe-torque.md` — Anti-Jerk-Pattern beim Motor-Toggle: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- `deep-dive-docs.md` — „Never invent functions"-Regel: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/deep-dive-docs.md>
- `setup-environment.md` — Voraussetzungen vor jedem App-Lauf: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/setup-environment.md>
- `interaction-patterns.md` — Antennen-als-Buttons, Head-as-Joystick, No-GUI-Pattern: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/interaction-patterns.md>
- `symbolic-motion.md` — `t_beats`-Konvention für BPM-synchronisierte Moves: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/symbolic-motion.md>
- `rest-api.md` — REST-Surface (alternativer Transport zur Python-API): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/rest-api.md>
- `debugging.md` — Debugging-by-Dichotomy, Daemon-Health-Checks: <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/debugging.md>
- `ai-integration.md` — LLM-Tools, Move-Queue (für AI-Apps): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/ai-integration.md>

## Offene Fragen
- Welche genaue `reachy_mini`-Version pinnen wir initial? Vorschlag: die letzte stabile vor Hardware-Eintreffen, dokumentiert im Skill-Body.
- Hat das SDK eine offizielle Compatibility-Matrix mit Python-Versionen, die wir verlinken sollten?
- Sollen Code-Beispiele die Async- oder die synchrone Variante des SDKs favorisieren? Hängt davon ab, was das SDK tatsächlich primär anbietet.
- Wie tief sollen Behaviors-Konventionen für die Hugging-Face-Veröffentlichung in diesem Skill auftauchen, _bevor_ ein eigener `reachy-app-publish-hf`-Skill existiert?
- Welches Tag-Set ist sinnvoll? Vorschlag: `[reachy-mini, sdk, python, robotics]`. Endgültig im Frontmatter klären.
- Soll der Skill auch auf reine Simulations-Aufgaben (MuJoCo / URDF ohne echte Hardware) reagieren? Tendenz: nein — das gehört in einen separaten Simulation-Skill.
- Wie häufig wird der Drift-Check ausgeführt? Vorschlag: vierteljährlich oder bei jedem `reachy_mini`-Major-Release.
- Wer ist die autoritative Quelle bei Konflikten zwischen Pollen-Robotics-Doku und SDK-Source-Code? Vorschlag: Source wins, Doku als Sekundärquelle.
