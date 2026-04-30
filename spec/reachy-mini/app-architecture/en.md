# App Architecture: Reachy Mini Show

Status: draft

## Context
This repository (`claude-reachy-mini`) ships skills, agents, and specs as a toolbox for development. The concrete application built with that toolbox is a **Pollen Reachy Mini app** — a Python package that the Reachy daemon launches as a subprocess on the robot, turning the 29 motion specs from this plugin into live behaviors. This specification defines the app layout, lifecycle, command interface, and distribution path. It is the source of truth against which the `behavior-scaffold` and `reachy-mini-sdk` skills and the `reachy-mini-on-device` agent calibrate their proposals. The app lives in a **separate app repository** (proposed: `nolte/reachy-mini-show`) — this plugin repository itself contains no app code.

## Goals
- A single app that implements all 29 motion slugs as `Move` subclasses
- Full conformance with Pollen's app system: daemon subprocess, one app at a time, Hugging Face Spaces as distribution
- Live command intake via a local WebSocket — later consumption by external integrations (HA, Wyoming bridge) lives in their own consumer repos
- Locally developable via `ReachyMini(use_sim=True)` — no hardware needed to start
- **Visible provenance**: any repository reading the app sees the reference to Claude Code and to `claude-reachy-mini` as the source of behavior specs and authoring skills

## Non-Goals
- HA custom integration (separate repository, with its own plugin / skill set)
- Wyoming voice stack (separate systemd service on Reachy, not part of this app)
- Cloud AI calls from the app process
- Multiple concurrent apps (Pollen's limit is one app at a time)
- Sandboxing or privilege separation (Pollen does not provide it)
- Authentication on the WebSocket — it is `localhost`-only

## Requirements

### App identity
- **MUST** carry the slug `reachy-mini-show` (or an owner-chosen slug) consistently for repo name, Python package name, and HF Space name
- **MUST** be packaged as a Hugging Face Space, with the tag `reachy_mini_python_app` in the README frontmatter (otherwise no discovery in the Reachy dashboard)
- **MUST** use semantic versioning
- **MUST** pin the `reachy_mini` SDK to a specific minor version (e.g. `^1.7.0`); an SDK major upgrade is always a deliberate re-validation

### Repository layout
Pollen-CLI-conformant layout with provenance markers (`CLAUDE.md`, plugin URL):

```
reachy-mini-show/
├── pyproject.toml              # Pollen format, SDK pin, provenance URLs
├── README.md                   # HF frontmatter `reachy_mini_python_app`, provenance note
├── CLAUDE.md                   # reference to the claude-reachy-mini plugin (authoring source)
├── reachy_mini_show/
│   ├── __init__.py
│   ├── main.py                 # Pollen entry: main(reachy, stop_event)
│   ├── server.py               # WebSocket server :8765
│   ├── behaviors/
│   │   ├── __init__.py         # slug → class registry
│   │   ├── base.py             # shared Move subclass
│   │   ├── emotions/           # happy, sad, angry, surprised, excited, sleepy,
│   │   │                       #   confused, curious, disappointed, proud, shy, disgust
│   │   ├── social/             # greeting-wave, farewell-wave, bow, peek,
│   │   │                       #   agreeing-nod, disagreeing-shake, recognition
│   │   ├── state/              # waiting-idle, alert-listening, thinking
│   │   ├── dance/              # groove-bob, sway-side, headbang-soft, spin-look-around
│   │   └── defensive/          # flinch, alarm, scanning
│   ├── audio/                  # WAV samples (F32LE, 48 kHz, 2 ch — Pollen-conformant)
│   └── config.py               # defaults and platform profiles
└── tests/                      # unit tests against ReachyMini(use_sim=True)
```

### Provenance markers (mandatory)

- **MUST** carry, in `README.md` immediately after the HF frontmatter, a provenance block with (1) a reference to the Claude Code plugin `claude-reachy-mini` (`https://github.com/nolte/claude-reachy-mini`), (2) a reference to the motion catalog (`spec/reachy-mini/motions/`), (3) a reference to this architecture spec
- **MUST** carry a `CLAUDE.md` at the app repo root that names the recommended plugin skills (`reachy-mini-sdk`, `behavior-scaffold`, agent `reachy-mini-on-device`) and links the plugin repo
- **MUST** carry, in `pyproject.toml` under `[project.urls]`, at least: `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
- **SHOULD** carry a one-line code header in `main.py`: `# Behaviors derived from spec/reachy-mini/motions/ in nolte/claude-reachy-mini`

### Lifecycle
- **MUST** implement Pollen's convention `main(reachy: ReachyMini, stop_event: threading.Event)`
- **MUST** start three parallel tasks under `asyncio.run(...)`: WebSocket server, behavior worker (reads queue, calls `mini.async_play_move(...)`), idle loop (when queue empty and no behavior active → runs `waiting-idle` or the configured idle mode)
- **MUST** end all three tasks cleanly on `stop_event`, abort the running behavior via `mini.cancel_move()`, and drive Reachy to `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS`
- **MUST** end every other task cleanly on a task exception and drive to a safe pose — no hanging connections, no frozen pose
- **MUST NOT** implement hardware reconnect inside the app — Pollen's daemon hands over a connected instance; the connection lifecycle belongs to the daemon

### Behavior implementation
- **MUST** carry exactly one `Move` subclass per motion slug, organised by the category subfolders (`emotions/`, `social/`, `state/`, `dance/`, `defensive/`)
- **MUST** implement each class with a `duration: float` property and an `evaluate(t: float)` method, per the Pollen `Move` ABC
- **MUST** maintain a slug registry in `behaviors/__init__.py` that maps the slug string to a class — that is the lookup source for commands
- **MUST** carry the phase values from the motion specs (pose Δ, antennas, body yaw, easing, per-phase duration) 1:1 — no arbitrary adjustments
- **MUST** offer the BPM-parametrised dance blocks (`groove-bob`, `sway-side`, `headbang-soft`) constructor-parameterised: `GrooveBob(bpm: float, beats: int, lead_time_s: float = 0.0)`
- **SHOULD** accept a `loop_count: int | None` parameter for loopable behaviors (`waiting-idle`, `alert-listening`, `thinking`) — `None` means loop unbounded until an external stop signal

### Command interface (local WebSocket)
- **MUST** expose a WebSocket server on `127.0.0.1:8765` (port configurable via ENV)
- **MUST** accept JSON messages with the following command types:

  ```jsonc
  {"type": "play_behavior", "slug": "happy", "speed": 1.0}
  {"type": "cancel"}
  {"type": "set_idle_mode", "mode": "waiting-idle"}
  {"type": "set_dance", "block": "groove-bob", "bpm": 110, "beats": 16}
  {"type": "speak", "text": "...", "behavior_during": "thinking"}
  {"type": "get_status"}
  ```

- **MUST** broadcast JSON events:

  ```jsonc
  {"type": "behavior_started", "slug": "happy", "started_at": "<iso>"}
  {"type": "behavior_finished", "slug": "happy", "status": "PASS|FAIL|ABORTED", "duration_s": 2.4}
  {"type": "low_battery", "percentage": 18}
  {"type": "error", "message": "..."}
  ```

- **MUST** maintain an open queue: a new `play_behavior` while a behavior is running aborts the running one (per Pollen convention only one move runs at a time)
- **MUST** allow multiple concurrent clients — every client receives every event; any client may send commands
- **MUST NOT** require TLS or authentication — `localhost`-only; external reach is the job of a separate reverse-proxy layer in the consumer setup

### Audio asset management
- **MUST** keep audio files under `audio/`, format WAV with `F32LE`, 48 kHz, 2 channels (Pollen pipeline conformant)
- **MUST** reference audio paths via `importlib.resources` — no hard paths
- **SHOULD** document a default volume per behavior as a constant in the move class

### Configuration
- **MUST** keep defaults in `config.py` as a dataclass
- **MUST** support ENV-var overrides: `REACHY_SHOW_PORT`, `REACHY_SHOW_IDLE_MODE`, `REACHY_SHOW_LOG_LEVEL`
- **MUST NOT** carry HA-specific config (HA URL, HA token) — those live in the consumer repo

### Platform profiles
- **MUST** distinguish Wireless / Lite / Simulation, based on SDK capability discovery
- Wireless: full (IMU reads active, battery polling active, all behaviors)
- Lite: no IMU reads, no battery polling; otherwise full
- Simulation: no audio playback, no sensor events apart from pose read

### Distribution path
- **MUST** be locally developable via `with ReachyMini(use_sim=True) as mini:`
- **MUST** be deployable to real hardware via Pollen's `local` source slot (daemon REST API or the Reachy dashboard)
- **MUST** be publishable as a Hugging Face Space via `git push <hf-remote>` — the `reachy_mini_python_app` tag makes the app installable from the Reachy dashboard

### Logging and observability
- **MUST** use structured Python `logging` with `INFO` as default and `DEBUG` via ENV var
- **MUST** emit important lifecycle events (behavior started / finished, idle-mode change, connection issues) both into the log and as WebSocket events
- **MUST NOT** log tokens, credentials, or raw audio bytes

### Versioning
- **MUST** keep semantic versions in `pyproject.toml`
- **MUST** keep machine-readable changelogs (Conventional Commits + release-drafter analogous to the plugin repo)
- **SHOULD** document the `bpm` range per release for the dance blocks (hardware performance can shift between firmware versions)

## Acceptance Criteria
- [ ] App repo follows the Pollen CLI layout, with the `reachy_mini_python_app` tag in the HF frontmatter
- [ ] `main(reachy, stop_event)` starts three parallel tasks (WebSocket, behavior worker, idle loop)
- [ ] The local WebSocket on `127.0.0.1:8765` accepts JSON commands and broadcasts JSON events
- [ ] All 29 motion slugs are implemented as `Move` subclasses and registered
- [ ] BPM dance blocks accept constructor-parametrised BPM and beat count
- [ ] A local test with `ReachyMini(use_sim=True)` runs without hardware
- [ ] App provenance is visible: README, CLAUDE.md, and `pyproject.toml [project.urls]` reference the `claude-reachy-mini` plugin
- [ ] A push to the HF remote installs the app in the Reachy dashboard without manual intervention
- [ ] `stop_event` drives to the rest pose without actuator clamping or dangling connections
- [ ] A task exception aborts every other task cleanly and drives to a safe pose
- [ ] Platform profiles correctly hide unavailable sensor reads

## Open Questions
- Is the slug `reachy-mini-show`, or do you have a different name in mind?
- Should `audio/` content be mirrored from the plugin repo, or does the app repo carry its own sound files?
- How exactly is the Pollen CLI invoked? Proposal: wrap it via the `behavior-scaffold` skill so the developer only feeds the shell.
- Which GitHub owner for the app repo — `nolte` directly or an org? Proposal: `nolte/reachy-mini-show`.
- Should the plugin repo carry an example app skeleton (e.g. under `examples/`) as a non-shipped reference? Pro: legible lifecycle for `behavior-scaffold`. Con: two sources of truth for the layout.
- How is the WebSocket protocol versioned? Proposal: a `protocol_version` field on every command and event, plus a `get_status` that names the supported protocol versions.
- Should the WebSocket optionally also speak UNIX sockets (for VM- or container-isolated consumers)? TCP is the default.
- How is a behavior cancelled that is still pending in the WebSocket queue (not the active one)? Proposal: `cancel` clears the queue and stops the active behavior; a future `cancel_pending` could split that later.
