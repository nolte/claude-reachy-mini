# MCP Server for Reachy Mini

Status: draft

## Context

Reachy Mini is operated from two worlds today: **Pollen-owned apps** (`reachy_mini_apps` entry point, daemon-managed, app-lock slot) and **direct SDK calls** from a developer's script or REPL. Both paths assume the operator knows Python and the `reachy_mini` SDK; both run locally and are not directly LLM-consumable. Whenever an LLM (Claude, GPT, local models) is supposed to read a pose or trigger a move during a chat, it has to generate code, have it executed, and parse the result back from stdout — a three-layer-of-friction pipeline.

The **MCP server for Reachy Mini** closes that gap. It is a local server process that exposes a clearly bounded tool inventory (read sensors, move motors, query daemon lifecycle) over the [Model Context Protocol](https://modelcontextprotocol.io) and internally talks to the **Pollen daemon's REST API** — not the Python SDK directly. With it, any MCP-capable frontend (Claude Desktop, Claude Code, Cursor, custom clients) can drive Reachy Mini without writing or running its own Python script.

This spec defines the canonical shape of that server: which tools it ships, which safety gates it enforces, on which platforms it runs, and how it relates to the existing plugin skills (`reachy-mini-sdk`, `reachy-mini-start`, `reachy-mini-deploy`) and agents (`reachy-mini-on-device`). It is a knowledge spec — the operations spec for bringing the server up lives in [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/en.md), and the server implementation lives outside this plugin.

Term clarification: "MCP server" = the local process specified here, exposing tools per the MCP protocol; "MCP client" = the LLM frontend consuming the tools; "daemon" = Pollen's own `reachy-mini-daemon` with REST / WebSocket surface on port 8000 — not to be confused.

## Goals

- A direct LLM-↔-robot bridge without a script-generation detour, with a clear three-tier tool inventory (Read / Write / Lifecycle), so that an MCP client can read a pose or trigger a move with one call
- **REST wrapper** as the only data path — the MCP server holds no Python SDK instance, no app-lock, no WebSocket streaming connection; it consumes the daemon
- Safety gates built in, not bolted on: pose-range validation against `control-surface`, app-lock pre-check before write operations, safe-torque wrapping around motor toggles, audit trail on every tool invocation
- **localhost-only** as the default; remote use cases are opt-in and out of v1 scope
- Platform profiles cleanly demarcated: what works on Wireless / Lite / Simulation
- Clear relationship to the existing skills / agents: the server is a **different distribution** of the plugin's knowledge, not a replacement for the skill / agent surface

## Non-Goals

- Replace Pollen's daemon — the daemon stays the only direct hardware driver; the MCP server is a consumer, not a driver
- Replace Pollen's app framework — apps with the `reachy_mini_apps` entry point stay Pollen's surface; the MCP server **runs alongside** them and never holds an app-lock
- A choreography / sequence / move composition surface of its own — motion composition belongs to apps and to the [`dance-choreography`](../../claude/dance-choreography/en.md) skill
- 50–100 Hz tight loops over the MCP protocol — MCP is request-response, not streaming; tight loops stay inside the SDK / apps framework (`set_target` over the Python API)
- Hugging Face publishing, deployment, on-device test — owned by other skills / agents (`reachy-app-publish-hf`, `reachy-mini-deploy`, `reachy-mini-on-device`)
- Real-time audio stream capture and playback — that also lives in Pollen's audio pipeline (`reachy_mini.media.*`); the MCP server can at most ship triggers ("play file X")
- Multi-user / tenant separation — the server is single-user-localhost-only; every LLM session that uses the server has full access
- Plugin releases, CI / CD integration, cloud deployment — the server is a local tool, not a service

## Requirements

### Architecture — REST wrapper

- **MUST** the MCP server perform every robot operation through Pollen's daemon REST API — never through the Python SDK directly; source: [`src/reachy_mini/daemon/app/routers/`](https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers)
- **MUST** the daemon endpoint be configurable (default `http://127.0.0.1:8000`, Wireless mDNS override `http://reachy-mini.local:8000`, Lite localhost) — but **never** default to a non-localhost address
- **MUST NOT** the MCP server hold a `ReachyMini` SDK instance or a direct hardware connection — that would collide with the app-lock and break parallel-running apps
- **SHOULD** the MCP server use a short-lived HTTP connection per tool call (no long-lived SSE / WebSocket stream), so a daemon restart doesn't silently tear down the LLM session
- **MUST** the MCP server return an **MCP-conformant error response** to the MCP client on a daemon connection refusal (no Python traceback, no silent ignore)

### Tool inventory — Tier 1: Read (safe, no lock check)

| Tool | Daemon path (source) | Response |
|---|---|---|
| `get_head_pose` | `state.py` router → joint / pose state | 4×4 transform matrix |
| `get_motor_status` | `motors.py` router → status read | list `{id, enabled, position, effort}` |
| `get_imu` (Wireless only) | `state.py` router → IMU submessage | `{accel, gyro, quat, temp_C}`; error on Lite / sim |
| `get_battery` (Wireless only) | `daemon.py` router → system status | `{soc_pct, voltage_v}`; error on Lite / sim |
| `get_current_app` | `apps.py` router → `current-app-status` | `{app_name?, lock_holder?}` |
| `list_apps` | `apps.py` router → installed apps list | list `{name, version, entry_point}` |

- **MUST** every Tier 1 tool run without an app-lock pre-check — reads are read-only, they do not block another app
- **SHOULD** Tier 1 tools accept an optional `cache_ms` parameter (default 0), so an MCP client can cache repeated reads inside the same turn — daemon load reduction

### Tool inventory — Tier 2: Write (with safety gates)

| Tool | Daemon path | Safety gates |
|---|---|---|
| `goto_pose(head, antennas, body_yaw, duration, method)` | `move.py` router → `goto_target` | pose range; app-lock pre-check |
| `set_pose(head, antennas, body_yaw)` | `move.py` router → `set_target` (single tick) | pose range; app-lock pre-check; **no 50 Hz loop** |
| `set_automatic_body_yaw(enabled)` | `move.py` router | app-lock pre-check |
| `look_at_world(x, y, z)` | `kinematics.py` router | pose range after IK; app-lock pre-check |
| `look_at_image(u, v)` | `kinematics.py` router | pose range after IK; app-lock pre-check |
| `wake_up()` | `move.py` router → matching endpoint | safe-torque (internal in `wake_up`); app-lock pre-check |
| `goto_sleep()` | `move.py` router → matching endpoint | safe-torque (internal); app-lock pre-check |
| `enable_motors()` | `motors.py` router | **safe-torque wrapping** (short `goto_target` to current pose, then enable); app-lock pre-check |
| `disable_motors()` | `motors.py` router | **safe-torque wrapping** (`goto_target` to `SLEEP_HEAD_POSE` first, then disable); app-lock pre-check |

- **MUST** before every Tier 2 tool, query the **app-lock status** (`apps.py` router → `robot-app-lock-status`); if another app holds the lock, abort with a clear hint — never force-stop
- **MUST** every tool that takes a pose validate the values against the limits from [`reachy-mini/control-surface`](../control-surface/en.md) (pitch / roll ±40°, head yaw ±60°, yaw relative ±65°, body yaw ±155°, antennas ±180°); out-of-range → error to the MCP client, never silent clipping
- **MUST** `enable_motors` / `disable_motors` always execute the safe-torque pattern from [`reachy-mini/app-logging`](../app-logging/en.md) and Pollen's [`safe-torque.md`](https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md) — the server takes that responsibility off the LLM
- **MUST NOT** the MCP server let `set_pose` calls run in a loop — that is the tight-loop pattern and belongs in an SDK app; the server rejects a second `set_pose` call within 200 ms and points at `goto_pose`

### Tool inventory — Tier 3: Lifecycle (operational)

| Tool | Daemon path | Note |
|---|---|---|
| `stop_current_app()` | `apps.py` router → `stop-current-app` | sets `stop_event`; releases app-lock |
| `emergency_stop()` | several | emergency-stop escalation: `stop_event` → 2 s → SIGTERM → 1 s → SIGKILL → pose reset → `disable_motors` |
| `start_app(name)` (optional v2) | `apps.py` router → `start-app` | only when no app-lock is held; otherwise error |

- **MUST** `emergency_stop` be the only emergency-stop path; analogous to the escalation sequence in [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md), but adapted to the MCP context
- **SHOULD** `stop_current_app` offer an optional `force=true` variant that skips the cleanup timeout — user-confirmed, never default

### Security model

- **MUST** the server bind on `127.0.0.1:<port>` by default, never on `0.0.0.0` without an explicit opt-in
- **MUST** the server write an **audit log** — one line per tool invocation with `{timestamp, tool, args (truncated), result, latency_ms}` under `~/.cache/reachy-mini-mcp/<YYYY-MM-DD>.log`
- **MUST** the audit log not contain PII / tokens / WiFi credentials / sensor streams — only tool metadata (PII clause analogous to [`reachy-mini/app-logging`](../app-logging/en.md))
- **MUST** the server validate the pose range as a server-internal gate **before** the daemon call goes out — the daemon also rejects, but we want to avoid pointless round-trips
- **SHOULD** the server activate a rate limiter on repeated out-of-range attempts from the same MCP client (e.g. block for 10 s after 5 rejected calls) — defence against LLM hallucinations
- **MUST NOT** the server expose tools that write directly into `~/.ssh/`, git configs, Hugging Face tokens, or other local secrets — the scope is robot operation, not system administration

### Platform profiles

| Platform | Daemon address (default) | Tier 1 | Tier 2 | Tier 3 | Note |
|---|---|---|---|---|---|
| Reachy Mini Wireless | `http://reachy-mini.local:8000` (mDNS) or configured IP | ✓ incl. IMU + battery | ✓ | ✓ | Server can run on the RPi 4 CM4 itself or remotely over an SSH tunnel |
| Reachy Mini Lite | `http://127.0.0.1:8000` | ✓ without IMU + battery | ✓ | ✓ | Daemon runs on the host PC, server runs alongside it |
| Simulation (`use_sim=True`) | `http://127.0.0.1:8000` | ✓ without IMU + battery (sim publishes no real values) | ✓ (sim accepts pose targets) | ✓ partially | The emergency-stop SIGKILL step does not apply (no app subprocess) |

- **MUST** the server perform a platform detect before the first tool call (`daemon.py` router → system status); the detected platform is returned per tool response as a metadata field
- **MUST** tools that are unavailable on the detected platform (e.g. `get_imu` on Lite) reject with a clear error rather than answer with a stub value

### MCP protocol conformance

- **MUST** the server implement the official MCP protocol against the active spec version (see references)
- **MUST** every tool ship a **JSON Schema description** of its parameters and response — MCP clients rely on it for argument validation
- **MUST** the server support the `tools/list` and `tools/call` protocol pattern; resources / prompts are out of v1 scope but can be added later
- **SHOULD** tool descriptions make explicit which tier a tool belongs to and which safety gates apply, so MCP clients (or the LLM behind them) understand what a tool does before invoking it

### Relationship to existing skills / agents

- **MUST** the server **not** be embedded inside a `claude-reachy-mini` skill or agent; skills run inside the Claude Code session, the MCP server is a **standalone process**
- **MUST** the server implementation live in a **separate repository** (e.g. `reachy-mini-mcp-server`, analogous to `reachy-mini-app`); the plugin ships the spec and a bootstrap skill, not the server code
- **SHOULD** the server cite [`reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) as the canonical knowledge base for SDK idioms, so an operator knows where the background concepts originate
- **SHOULD** the server name [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) as "the right surface for bulk triage / live trial / telemetry sampling" — the MCP server is single-shot interaction, not a test lifecycle

### Consumers and boundary

- **SHOULD** the server not ship dedicated tools for `dance-choreography` — choreographies are authoring artefacts, not tools
- **MUST NOT** the server generate a spec / choreography / app code — it operates the robot, it does not produce plugin content
- **SHOULD** the server expose a `health-check` tool variant that probes daemon reachability, platform detection, and audit-log writability in one call — useful for the first-contact handshake of an MCP client

## Acceptance Criteria

- [ ] The spec lives at `spec/reachy-mini/mcp-server/de.md` (DE canonical) and `en.md` (EN translation), structurally identical
- [ ] Tool inventory covers three tiers (Read / Write / Lifecycle), each tool annotated with the daemon-path source and the safety gates
- [ ] Tier 1 reads run without app-lock pre-check; Tier 2 writes run with app-lock pre-check; Tier 3 lifecycle has its own emergency-stop escalation
- [ ] Pose-range validation against `control-surface` is specified as a server-internal gate, not delegated
- [ ] Safe-torque wrapping around `enable_motors` / `disable_motors` is mandatory
- [ ] localhost-only binding is the default; remote opt-in is explicitly marked "v1 out of scope"
- [ ] Audit-log path and format are fixed; PII clause inherited from `app-logging`
- [ ] Platform table covers Wireless / Lite / Simulation and names per platform which Tier 1 tools are available
- [ ] Relationship to existing skills / agents is explicit (no embedding, separate repository, separate distribution)
- [ ] MCP protocol conformance is formulated as a MUST, with a reference to the official spec
- [ ] Cross-refs to [`reachy-mini/control-surface`](../control-surface/en.md), [`reachy-mini/app-logging`](../app-logging/en.md), [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md), [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md), [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/en.md) are visible
- [ ] `pre-commit run --all-files` passes on the spec files

## References

> Source references to Pollen code files point at the matching directory inside the daemon tree; concrete endpoint paths are to be verified against the router files when the implementation starts. Markdown sources are cited at file level.

- Pollen daemon routers (source of all REST endpoints): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers>
- Pollen daemon main module (FastAPI app bootstrap): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/main.py>
- Apps router (app-lock status, current-app status, start / stop app): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/apps.py>
- State router (pose, joint positions, IMU): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/state.py>
- Move router (goto_target, set_target): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/move.py>
- Motors router (enable / disable, status): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/motors.py>
- Kinematics router (look_at_world, look_at_image): <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/kinematics.py>
- Pollen skill `safe-torque` (anti-jerk pattern, source for Tier 2 wrapping): <https://github.com/pollen-robotics/reachy_mini/blob/main/skills/safe-torque.md>
- Model Context Protocol — specification: <https://modelcontextprotocol.io>
- Model Context Protocol — Python server SDK: <https://github.com/modelcontextprotocol/python-sdk>
- Internal cross-refs:
  - [`reachy-mini/control-surface`](../control-surface/en.md) — pose-range limits source
  - [`reachy-mini/app-logging`](../app-logging/en.md) — PII clause role model
  - [`reachy-mini/app-architecture`](../app-architecture/en.md) — daemon lifecycle context
  - [`claude/reachy-mini-sdk`](../../claude/reachy-mini-sdk/en.md) — SDK idioms, method choice, safe-torque
  - [`claude/reachy-mini-on-device`](../../claude/reachy-mini-on-device/en.md) — test lifecycle as a sister surface, with the emergency-stop escalation
  - [`claude/mcp-server-bootstrap`](../../claude/mcp-server-bootstrap/en.md) — operations skill for bringing up the server

## Open Questions

- Implementation language: Python (with the `mcp` SDK)? Rust (with community MCP crates)? Proposal: Python for v1 — same platform requirements as the Pollen daemon, easy packaging via `uv`.
- Server distribution: PyPI package `reachy-mini-mcp-server`? Or a Hugging Face Space with a `pyproject.toml` entry point? PyPI is the clean variant for tooling.
- Cache strategy: should the server cache Tier 1 reads itself (e.g. pose read every 100 ms) or leave it to the MCP client? `cache_ms` per tool is a good middle ground.
- Multi-tool choreographies: does the MCP client need a `goto_pose_sequence` tool that runs several poses as one transaction, or is that an app concern? Argument for a single-tool approach: simpler; argument for a sequence tool: fewer round-trips.
- WebSocket tools: a few operations (e.g. move cancellation during a `goto_target`) need a long-lived stream. Currently flagged out of scope, but consumer use cases may change that.
- Authentication for remote mode: when someone does open the server on `0.0.0.0`, what is the auth layer? OAuth, bearer token, mTLS? Out of v1 scope; wait for consumer pressure.
- Server on Wireless hardware: does the server run **on** Reachy Mini itself (local daemon, local MCP server, SSH tunnel to the LLM host) or **on the developer host** (remote daemon, local MCP server)? The latter feels simpler; the former reduces latency. Consumer test in v0.
- Tool versioning: an MCP client caches tool definitions — what happens on schema changes? Server version reported in the capability handshake, MCP client invalidates its cache.
- Relationship with Pollen's own tooling: does Pollen plan an official MCP server? If yes, we should coordinate — otherwise we end up with two competing servers.
