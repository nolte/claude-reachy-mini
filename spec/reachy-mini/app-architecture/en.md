# App Architecture: Reachy Mini Show

Status: draft

## Context
This repository (`claude-reachy-mini`) ships skills, agents, and specs as a toolbox for development. The concrete application built with that toolbox is a **Pollen Reachy Mini app** — a Python package that the Reachy daemon launches as a subprocess on the robot, turning the 29 motion specs from this plugin into live behaviors. This specification defines the app layout, lifecycle, command interface, and distribution path. It is the source of truth against which the `behavior-scaffold` and `reachy-mini-sdk` skills and the `reachy-mini-on-device` agent calibrate their proposals. The app lives in a **separate app repository** (proposed: `nolte/reachy-mini-show`) — this plugin repository itself contains no app code.

## Goals
- A single app that implements all 29 motion slugs as `Move` subclasses
- Full conformance with Pollen's app system: daemon subprocess, one app at a time, Hugging Face Spaces as distribution
- Live command intake via a local WebSocket — later consumption by external integrations (HA, Wyoming bridge) lives in their own consumer repos
- Locally developable via `ReachyMini(spawn_daemon=True, use_sim=True)` — no hardware needed to start
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
- **MUST** carry the slug `reachy-mini-show` consistently for repo name, Python package name, and HF Space name
- **MUST** be packaged as a Hugging Face Space, with the tag `reachy_mini_python_app` in the README frontmatter (otherwise no discovery in the Reachy dashboard)
- **MUST** declare an entry point in the `reachy_mini_apps` group inside `pyproject.toml` that names the app class — the daemon discovers apps exclusively through this group:

  ```toml
  [project.entry-points."reachy_mini_apps"]
  reachy-mini-show = "reachy_mini_show.main:ReachyMiniShowApp"
  ```

- **MUST** use semantic versioning
- **MUST** pin the `reachy_mini` SDK to a specific minor version (e.g. `^1.7.0`); an SDK major upgrade is always a deliberate re-validation

### Repository layout
Layout per the official Pollen Robotics CLI `reachy-mini-app-assistant` (default template), extended with provenance markers (`CLAUDE.md`, plugin URL):

```
reachy-mini-show/
├── pyproject.toml              # Pollen format, SDK pin, provenance URLs,
│                               #   [project.entry-points."reachy_mini_apps"]
├── README.md                   # HF frontmatter `reachy_mini_python_app`, provenance note
├── CLAUDE.md                   # reference to the claude-reachy-mini plugin (authoring source)
├── index.html                  # Hugging Face Space landing page (Pollen CLI default)
├── style.css                   # landing page styles (Pollen CLI default)
├── reachy_mini_show/
│   ├── __init__.py
│   ├── main.py                 # ReachyMiniApp subclass + __main__ → wrapped_run()
│   ├── server.py               # WebSocket server :8765 (show-specific extension)
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
│   ├── static/                 # optional: web-UI assets when custom_app_url is set
│   └── config.py               # defaults and platform profiles
└── tests/                      # unit tests against ReachyMini(spawn_daemon=True, use_sim=True)
```

The skeleton is **not hand-written**; it is created via the official CLI:

```bash
uv pip install reachy-mini
reachy-mini-app-assistant create reachy-mini-show /path/to/dest             # without HF push
reachy-mini-app-assistant create reachy-mini-show /path/to/dest --publish   # creates HF Space + git remote
```

Hand-rolled skeletons drift subtly from Pollen's expectations and break on the first daemon run.

### Provenance markers (mandatory)

- **MUST** carry, in `README.md` immediately after the HF frontmatter, a provenance block with (1) a reference to the Claude Code plugin `claude-reachy-mini` (`https://github.com/nolte/claude-reachy-mini`), (2) a reference to the motion catalog (`spec/reachy-mini/motions/`), (3) a reference to this architecture spec
- **MUST** carry a `CLAUDE.md` at the app repo root that names the recommended plugin skills (`reachy-mini-sdk`, `behavior-scaffold`, agent `reachy-mini-on-device`) and links the plugin repo
- **MUST** carry, in `pyproject.toml` under `[project.urls]`, at least: `Plugin = "https://github.com/nolte/claude-reachy-mini"`, `SDK = "https://github.com/pollen-robotics/reachy_mini"`, `Specs = "https://github.com/nolte/claude-reachy-mini/tree/develop/spec/reachy-mini/"`
- **SHOULD** carry a one-line code header in `main.py`: `# Behaviors derived from spec/reachy-mini/motions/ in nolte/claude-reachy-mini`

### Lifecycle
- **MUST** implement Pollen's app contract: a class `ReachyMiniShowApp(ReachyMiniApp)` from `reachy_mini` with the required method `run(self, reachy_mini: ReachyMini, stop_event: threading.Event)`. The daemon calls `run()` with an already-connected instance and sends `SIGINT`, which triggers `stop_event.set()`.
- **MUST** carry an `if __name__ == "__main__":` block in `main.py` that calls `ReachyMiniShowApp().wrapped_run()` (for direct execution via `python -m reachy_mini_show.main`); `wrapped_run()` handles connect, optional services, and then invokes `run()`.
- **MUST** start three parallel tasks inside `run()` under `asyncio.run(...)`: WebSocket server, behavior worker (reads queue, calls `reachy_mini.async_play_move(...)`), idle loop (when queue empty and no behavior active → runs `waiting-idle` or the configured idle mode)
- **MUST** end all three tasks cleanly on `stop_event`, abort the running behavior via `reachy_mini.cancel_move()`, and drive Reachy to `INIT_HEAD_POSE` + `INIT_ANTENNAS_JOINT_POSITIONS` — `run()` then returns and the daemon resets the robot to its default pose
- **MUST** end every other task cleanly on a task exception and drive to a safe pose — no hanging connections, no frozen pose
- **MUST NOT** implement hardware reconnect inside the app — Pollen's daemon hands over a connected instance; the connection lifecycle belongs to the daemon
- **MUST NOT** model the lifecycle as a free `main(reachy, stop_event)` function — the daemon expects the `ReachyMiniApp` subclass; without it neither the discovery path nor the `__main__` direct-run path works

### Behavior implementation
- **MUST** carry exactly one `Move` subclass per motion slug, organised by the category subfolders (`emotions/`, `social/`, `state/`, `dance/`, `defensive/`)
- **MUST** implement each class with a `duration: float` property and an `evaluate(t: float)` method, per the Pollen `Move` ABC
- **MUST** maintain a slug registry in `behaviors/__init__.py` that maps the slug string to a class — that is the lookup source for commands
- **MUST** carry the phase values from the motion specs (pose Δ, antennas, body yaw, easing, per-phase duration) 1:1 — no arbitrary adjustments
- **MUST** offer the BPM-parametrised dance blocks (`groove-bob`, `sway-side`, `headbang-soft`) constructor-parameterised: `GrooveBob(bpm: float, beats: int, lead_time_s: float = 0.0)`
- **SHOULD** accept a `loop_count: int | None` parameter for loopable behaviors (`waiting-idle`, `alert-listening`, `thinking`) — `None` means loop unbounded until an external stop signal

### Command interface (local WebSocket)
- **MUST** expose a WebSocket server on `127.0.0.1:8765` (port configurable via ENV)
- **MUST** carry a **protocol-version field** `protocol_version` in every command and every event, formatted as `<major>.<minor>` (e.g. `"1.0"`); a major change marks a breaking change
- **MUST** reject commands with an unsupported **major** version with an `error` event carrying `code: "unsupported_protocol_version"`, without affecting the behavior worker
- **SHOULD** tolerate minor-version differences (forward-compatible) — ignore new optional fields, default missing new fields
- **MUST** accept JSON messages with the following command types:

  ```jsonc
  {"type": "play_behavior", "protocol_version": "1.0", "slug": "happy", "speed": 1.0}
  {"type": "cancel", "protocol_version": "1.0"}
  {"type": "set_idle_mode", "protocol_version": "1.0", "mode": "waiting-idle"}
  {"type": "set_dance", "protocol_version": "1.0", "block": "groove-bob", "bpm": 110, "beats": 16}
  {"type": "speak", "protocol_version": "1.0", "text": "...", "behavior_during": "thinking"}
  {"type": "get_status", "protocol_version": "1.0"}
  ```

- **MUST** broadcast JSON events, each with a `protocol_version` field:

  ```jsonc
  {"type": "behavior_started", "protocol_version": "1.0", "slug": "happy", "started_at": "<iso>"}
  {"type": "behavior_finished", "protocol_version": "1.0", "slug": "happy", "status": "PASS|FAIL|ABORTED", "duration_s": 2.4}
  {"type": "low_battery", "protocol_version": "1.0", "percentage": 18}
  {"type": "error", "protocol_version": "1.0", "code": "unsupported_protocol_version|invalid_command|behavior_not_found|...", "message": "..."}
  ```

- **MUST** answer `get_status` with a response carrying `supported_protocol_versions: ["1.0", ...]` as an array, so consumers can perform their own version match
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
- **MAY** expose a settings web UI by setting `custom_app_url` on the `ReachyMiniApp` subclass (e.g. `"http://0.0.0.0:8042"`); Pollen then auto-starts a FastAPI server that serves `static/` from the package. The dashboard shows a settings icon and opens the UI at `http://localhost:8042` (Lite / Sim) or `http://reachy-mini.local:8042` (Wireless). When unused, set `custom_app_url = None`.

### Platform profiles
- **MUST** distinguish Wireless / Lite / Simulation, based on SDK capability discovery
- Wireless: full (IMU reads active, battery polling active, all behaviors)
- Lite: no IMU reads, no battery polling; otherwise full
- Simulation: no audio playback, no sensor events apart from pose read

### Development and distribution paths

Development happens on the developer's **laptop**, not on the robot itself (even though the Wireless ships a RPi 4 CM4 — that is run-, not dev-hardware). There are two sim paths — pick one:

**A) External daemon, separate app process** (default for the Show app, since app logs stay cleanly separated from daemon logs):

```bash
reachy-mini-app-assistant create reachy-mini-show .
uv venv && source .venv/bin/activate
uv pip install -e .

# Terminal 1 — daemon
reachy-mini-daemon --sim                      # full sim path with the MuJoCo viewer
# or, when GStreamer / MuJoCo are not installed:
reachy-mini-daemon --mockup-sim --no-media --headless

# Terminal 2 — app (connects to localhost:8000)
python -m reachy_mini_show.main
```

**B) In-process daemon for tests / smoke runs** (no second terminal needed):

```python
with ReachyMini(spawn_daemon=True, use_sim=True) as mini:
    ...
```

`spawn_daemon=True` boots a daemon subprocess inside the Python process; `use_sim=True` alone is **not** enough — without `spawn_daemon=True` the SDK tries to connect to an external daemon and fails with `ConnectionError`. This is an important API trap the official docs gloss over.

**System dependencies for the full `--sim` path** (MuJoCo viewer + GStreamer WebRTC):

- `gir1.2-gst-plugins-base-1.0`, `gir1.2-gstreamer-1.0`, `python3-gi` (Debian / Ubuntu) — otherwise `ValueError: Namespace GstApp not available` on daemon start
- MuJoCo (Python wheel arrives automatically)

When developing without GStreamer, use `--mockup-sim --no-media --headless` plus `request_media_backend = "no_media"` on the app class — works fully for pose, antenna, and body-yaw tests; only audio / video are off.

Three canonical deploy paths to Wireless:

1. **Hugging Face Space (default, with internet)** — `git push <hf-remote>` from the app repo. As soon as the `reachy_mini_python_app` tag is in the README frontmatter, the app appears in the Reachy dashboard and is installable in one click.
2. **Daemon REST API direct** — works against any reachable daemon (Wireless via `reachy-mini.local:8000`, Lite via the host PC, Sim via `localhost:8000`). The endpoint names in Pollen's `docs/source/SDK/apps.md` are partly stale — the **authoritative** schema is always `http://<daemon-host>:8000/openapi.json` (live). As of v1.7.1, verified against a running Wireless:

   ```bash
   # install from HF (public space)
   curl -X POST http://reachy-mini.local:8000/api/apps/install \
     -H "Content-Type: application/json" \
     -d '{"url": "https://huggingface.co/spaces/<user>/reachy-mini-show"}'
   # for private spaces: POST /api/apps/install-private-space (HF token in body)

   # directory: HF spaces + locally installed apps
   curl       http://reachy-mini.local:8000/api/apps/list-available
   curl       http://reachy-mini.local:8000/api/apps/list-available/installed   # installed only

   # lifecycle
   curl       http://reachy-mini.local:8000/api/apps/current-app-status
   curl -X POST http://reachy-mini.local:8000/api/apps/start-app/reachy_mini_show
   curl -X POST http://reachy-mini.local:8000/api/apps/restart-current-app
   curl -X POST http://reachy-mini.local:8000/api/apps/stop-current-app

   # maintenance
   curl       http://reachy-mini.local:8000/api/apps/check-updates
   curl -X POST http://reachy-mini.local:8000/api/apps/update/reachy_mini_show
   curl -X POST http://reachy-mini.local:8000/api/apps/remove/reachy_mini_show

   # async jobs (install / update return a job_id)
   curl       http://reachy-mini.local:8000/api/apps/job-status/<job_id>

   # robot lock (which app holds hardware access)
   curl       http://reachy-mini.local:8000/api/daemon/robot-app-lock-status
   curl       http://reachy-mini.local:8000/api/daemon/status
   ```

   Important drifts vs. Pollen's `apps.md`:
   - `/api/apps/list` from the doc **does not exist** — the correct endpoint is `/api/apps/list-available`
   - The app name in the path is the **Python package name** (snake_case `reachy_mini_show`), not the repo / HF slug (`reachy-mini-show`)
   - Async operations (install, update) return a `job_id`; poll progress via `GET /api/apps/job-status/<job_id>`

3. **Offline / manual (no internet, e.g. at a conference)** — install directly into the Wireless shared venv:

   ```bash
   scp -r /path/to/app pollen@reachy-mini.local:/tmp/reachy-mini-show
   ssh pollen@reachy-mini.local \
     "/venvs/apps_venv/bin/pip install /tmp/reachy-mini-show"
   # after code changes: restart the daemon or the app via the REST API
   ```

Requirements:

- **MUST** be locally developable via at least one of the two sim paths above (external daemon **or** `with ReachyMini(spawn_daemon=True, use_sim=True) as mini:`); plain `use_sim=True` without `spawn_daemon=True` is **not** a valid sim path and fails with `ConnectionError`
- **MUST** support all three deploy paths — the HF Space route is the default; REST and SSH are fallbacks without dashboard or without internet
- **MUST** know that on Wireless every app installs into the shared venv `/venvs/apps_venv/` (no per-app venv); dependency conflicts with other installed apps are a real failure mode
- **MUST NOT** modify code in the daemon service (`reachy-mini-daemon.service`) on the Wireless — that service is Pollen-owned

### Logging and observability
- **MUST** use structured Python `logging` with `INFO` as default and `DEBUG` via ENV var
- **MUST** emit important lifecycle events (behavior started / finished, idle-mode change, connection issues) both into the log and as WebSocket events
- **MUST** know that on Wireless the app's `stdout` / `stderr` is captured by the daemon and read via `sudo journalctl -u reachy-mini-daemon` — on Lite / Simulation logs print directly in the daemon terminal. Diagnostic examples:

  ```bash
  ssh pollen@reachy-mini.local
  sudo journalctl -u reachy-mini-daemon -f                              # live
  sudo journalctl -u reachy-mini-daemon --since '5 min ago' \
    | grep -v "uvicorn\|GET \|POST "                                    # filtered
  ```

- **MUST NOT** log tokens, credentials, or raw audio bytes

### Versioning
- **MUST** keep semantic versions in `pyproject.toml`
- **MUST** keep machine-readable changelogs (Conventional Commits + release-drafter analogous to the plugin repo)
- **SHOULD** document the `bpm` range per release for the dance blocks (hardware performance can shift between firmware versions)

## Acceptance Criteria
- [ ] App repo was created via `reachy-mini-app-assistant create` and passes `reachy-mini-app-assistant check` without findings
- [ ] `pyproject.toml` declares exactly one entry point in the `reachy_mini_apps` group, pointing at the `ReachyMiniShowApp` class
- [ ] README carries the `reachy_mini_python_app` tag in the YAML frontmatter
- [ ] `ReachyMiniShowApp.run(reachy_mini, stop_event)` starts three parallel tasks (WebSocket, behavior worker, idle loop) and returns cleanly after `stop_event.set()`
- [ ] `python -m reachy_mini_show.main` runs via `wrapped_run()` directly against a local `reachy-mini-daemon --sim`
- [ ] The local WebSocket on `127.0.0.1:8765` accepts JSON commands and broadcasts JSON events
- [ ] Every command and every event carries a `protocol_version` field; `get_status` returns `supported_protocol_versions`
- [ ] Commands with an unknown major version are rejected with `error code: "unsupported_protocol_version"`
- [ ] All 29 motion slugs are implemented as `Move` subclasses and registered
- [ ] BPM dance blocks accept constructor-parametrised BPM and beat count
- [ ] A local test with `ReachyMini(spawn_daemon=True, use_sim=True)` runs without hardware
- [ ] App provenance is visible: README, CLAUDE.md, and `pyproject.toml [project.urls]` reference the `claude-reachy-mini` plugin
- [ ] A push to the HF remote installs the app in the Reachy dashboard without manual intervention
- [ ] The three deploy paths (HF push, REST `POST /api/apps/install`, offline `scp` + `pip install` into `/venvs/apps_venv/`) are documented in the app repo
- [ ] Every REST endpoint cited in the docs is verified against `http://<daemon-host>:8000/openapi.json`, not copied from Pollen's `apps.md`
- [ ] On Wireless, app logs are visible via `sudo journalctl -u reachy-mini-daemon`
- [ ] `stop_event` drives to the rest pose without actuator clamping or dangling connections
- [ ] A task exception aborts every other task cleanly and drives to a safe pose
- [ ] Platform profiles correctly hide unavailable sensor reads

## References
- Upstream SDK repo (source of truth for `Move`, `ReachyMini`, `ReachyMiniApp`, app lifecycle): <https://github.com/pollen-robotics/reachy_mini>
- Official Apps doc (build, publish, REST install, logs, web UI): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- Apps subsystem (`ReachyMiniApp` ABC, `wrapped_run`, app lock, app manager): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps>
- App templates (canonical scaffold for `pyproject.toml`, `main.py`, `README.md`, `index.html`, `style.css` with the `reachy_mini_python_app` tag): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/apps/templates>
- Daemon (REST API, app lock, lifecycle, status — the subprocess that launches this app): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- IO protocol (command and telemetry messages, reference for our WebSocket protocol): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/io/protocol.py>
- SDK concept docs (Apps, Quickstart, Core Concept): <https://github.com/pollen-robotics/reachy_mini/tree/main/docs/source/SDK>
- Pollen's `AGENTS.md` (entry point that AI agents follow to find the app authoring skills): <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- HF blog tutorial (step-by-step with screenshots): <https://huggingface.co/blog/pollen-robotics/make-and-publish-your-reachy-mini-apps>
- Runnable minimal app example: <https://github.com/pollen-robotics/reachy_mini/blob/main/examples/minimal_demo.py>
- Conversation app as a complete reference repo (audio pipeline, LLM tools, FastAPI UI): <https://github.com/pollen-robotics/reachy_mini_conversation_app>

## Open Questions
- ~~Is the slug `reachy-mini-show`?~~ **Answered**: yes, end-to-end.
- ~~Audio mirrored from the plugin repo or owned?~~ **Answered**: the app repo carries its own audio; no mirroring from the plugin repo.
- ~~Example app skeleton in the plugin repo under `examples/`?~~ **Answered**: no example for now. If `behavior-scaffold` needs a concrete layout reference, it can be generated at runtime via the Pollen CLI.
- ~~WebSocket protocol versioning?~~ **Answered**: a `protocol_version` field on every command and event is now a requirement; `get_status` returns `supported_protocol_versions`.
- ~~How exactly is the Pollen CLI invoked?~~ **Answered**: the official tool is `reachy-mini-app-assistant` (`uv pip install reachy-mini`); sub-commands `create <name> <dest> [--publish] [--template default|conversation]`, `check <path>`, `publish <path>`. The `behavior-scaffold` skill wraps this CLI invocation.
- ~~Free `main(reachy, stop_event)` function vs. `ReachyMiniApp` subclass with `run()`?~~ **Answered**: Pollen expects the subclass with `run(self, reachy_mini, stop_event)`; an `__main__` block calls `wrapped_run()`. A free `main()` would not be picked up by either the daemon discovery path (entry-point group `reachy_mini_apps`) or the direct-run path.
- Which GitHub owner for the app repo — `nolte` directly or an org? Proposal: `nolte/reachy-mini-show`.
- Should the WebSocket optionally also speak UNIX sockets (for VM- or container-isolated consumers)? TCP is the default.
- How is a behavior cancelled that is still pending in the WebSocket queue (not the active one)? Proposal: `cancel` clears the queue and stops the active behavior; a future `cancel_pending` could split that later.
