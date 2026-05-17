---
name: reachy-mini-inspect
description: >-
  Snapshot the current state of a running Reachy Mini daemon as a Markdown
  table directly in the conversation — daemon liveness, app-lock, motor mode,
  head/body/antenna pose, audio system, speaker and microphone volume,
  currently running app. Read-only — only GET endpoints, never
  POST/PUT/DELETE/PATCH, never touches the app-lock. Activate when the user
  says "zeig mir den Zustand", "wie steht der Reachy gerade", "Daemon-Status
  abfragen", "Motorposition lesen", "show robot state", "check daemon
  health", "where is the head pointing", or invokes the modes `quick` /
  `full` / `raw <endpoint>` directly. Do NOT use to start, stop, install,
  update, or remove an app (that's `reachy-mini-start`, `reachy-mini-deploy`,
  `reachy-mini-on-device`), to write motion or behavior code
  (`reachy-mini-sdk`, `app-scaffold`), to analyze app logs
  (`app-log-triage`), or to stand up an MCP server for LLM frontends
  (`mcp-server-bootstrap`).
tags: [reachy-mini, inspect]
---

# Reachy Mini Inspect

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-inspect/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-inspect/en.md).

Authoritative endpoint inventory: [`spec/reachy-mini/daemon-rest-api/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md). Joint-limit and recovery details (incl. the daemon `_status.ready` bug, the IMU/I²C `silent dead` pattern, and Stage-3 recovery paths) live in [`spec/reachy-mini/motor-positions/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md). Complementary distribution for LLM frontends outside Claude Code: [`spec/reachy-mini/mcp-server/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/mcp-server/de.md).

## Skill-vs-Agent rationale

This is a **skill** rather than an agent — the load-bearing dimensions all point the same way:

- **Single-turn read** — three to nine GETs at most; no multi-stage orchestration that would benefit from agent isolation.
- **No externally-visible mutation** — read-only by contract; no draft → ready flip, no remote write, no user-confirmation gate to defer past the first call.
- **Output naturally inline** — a compact Markdown table that the user reads in flow; isolating it behind an agent boundary would obscure the immediate answer.
- **Context-window protection N/A** — the response is tens of lines, not hundreds; no installer-style log spill to siphon off.

## When this skill activates

Use this skill when the user wants a read-only snapshot:

- "show robot state" / "zeig mir den Zustand" / "wie steht der Reachy gerade"
- "check daemon health" / "Daemon-Status abfragen"
- "read the motor position" / "Motorposition lesen"
- "where is the head pointing" / "wo schaut der Reachy gerade hin"
- a direct mode mention: `quick`, `full`, or `raw <endpoint>`

## When NOT to activate

- start / stop / install / update / remove an app → `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`
- write motion or behavior code → `reachy-mini-sdk`, `app-scaffold`
- analyze a crashed app's logs → `app-log-triage`
- bring up an MCP server for an external LLM frontend → `mcp-server-bootstrap`
- behavior live-trial with telemetry → `reachy-mini-on-device` (agent)
- continuous / high-frequency pose reads (50–100 Hz) — that belongs in an SDK-app `set_target` loop, not here

## Hard rules

1. **Read-only HTTP only.** The skill calls **only** GET endpoints listed in [`spec/reachy-mini/daemon-rest-api/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md). **Never** POST, PUT, DELETE, or PATCH — even `POST /health-check` is forbidden; daemon liveness is read via `GET /api/daemon/status`.
2. **Never touch the app-lock.** The skill reads `GET /api/daemon/robot-app-lock-status`; it does not stop, start, replace, or force-release a held lock. If the user wants to act on the lock, point at `reachy-mini-start`.
3. **Localhost-first defaults.** The daemon host defaults to `http://127.0.0.1:8000`. The Wireless mDNS override `http://reachy-mini.local:8000` is opt-in via input, never auto-selected. Never default to a non-localhost address.
4. **No polling, no loop.** One snapshot per call. The skill does not retry a failed GET, does not wait for state to change, does not run in a loop.
5. **No custom caching.** Every call hits the daemon fresh. Aggregating cache state across invocations is the caller's job, not this skill's.
6. **No raw-JSON spill into the main context.** Only the `raw` mode emits a response body, and even there it is formatted as a single fenced code block, not a multi-screen dump.

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `mode` | no | `quick` | One of `quick`, `full`, `raw`. |
| `daemon_host` | no | `http://127.0.0.1:8000` | Override examples: `http://reachy-mini.local:8000` (Wireless mDNS), `http://<lite-host>:8000` (Lite). |
| `endpoint` | yes if `mode=raw` | — | A `GET` path from `spec/reachy-mini/daemon-rest-api/`. The skill rejects any path not listed there and any path declared under another HTTP method. |
| `timeout_per_get_s` | no | `5` | Per-GET request timeout in seconds. The total time budget for `quick` and `full` is capped at 15 s. |

## Modes

### `quick` (default — 3 GETs)

Issue these reads and aggregate them into a Markdown table:

- `GET /api/daemon/status` — daemon liveness and version
- `GET /api/daemon/robot-app-lock-status` — `state` plus optional `holder_name`
- `GET /api/motors/status` — motor mode per axis

### `full` (parallel — adds 5 GETs)

In addition to the three `quick` GETs, fan out these reads in parallel:

- `GET /api/state/full?with_head_joints=true` — head pose, **Stewart joint vector `[body, s1..s6]`**, body yaw, antenna joint positions, DoA. **The `with_head_joints=true` query parameter is mandatory:** without it the daemon returns `head_joints: null` even in a healthy state. Verified live 2026-05-13.
- `GET /api/media/status` — audio subsystem status
- `GET /api/volume/current` — speaker volume
- `GET /api/volume/microphone/current` — microphone volume
- `GET /api/apps/current-app-status` — currently running app (if any)

### `raw <endpoint>` (single GET, validated)

Issue exactly one `GET` against `<daemon_host><endpoint>` and emit the response as a fenced code block. Before issuing, the path **MUST** be matched against the endpoint inventory in [`spec/reachy-mini/daemon-rest-api/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md); a path not listed there or listed under a non-`GET` method is rejected with a clear message.

## Pre-flight (every mode, in order — abort on first failure)

1. Resolve the daemon endpoint from `daemon_host` (default localhost). On Wireless with mDNS, the first call can be noticeably slower than steady-state — accept that.
2. Issue `GET /api/daemon/status` as the reachability check. On connection refused / DNS failure / timeout, abort with a single line: `daemon unreachable at <daemon_host>: <connection_error>`. No stacktrace.
3. **Backend liveness — read with care.** `backend_status.ready` and `backend_status.last_alive` in the status response are **not authoritative on their own**: the daemon (`reachy_mini==1.7.1`, source `backend/robot/backend.py`) sets a `self.ready` `threading.Event` but never propagates it into `self._status.ready`, and `_status.last_alive` is similarly de-synced from the loop's `self.last_alive`. Treat them as "best signal" but cross-check with one of: (a) `mean_control_loop_frequency > 40 Hz` plus `nb_error == 0`, (b) `head_joints` is a list (after appending `?with_head_joints=true`), (c) two consecutive `state/full` reads show micro-drift in pose components. If all three confirm liveness, the loop is alive even when `backend_status.ready=false`. If none confirm liveness, the daemon is in the "silent dead" pattern — see [`spec/reachy-mini/motor-positions/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) §"Stage-3 recovery paths" for the diagnosis flow.
4. For `raw` mode only, validate `endpoint` against the daemon-rest-api inventory before issuing the call. Reject silently is forbidden; report `endpoint <path> is not in spec/reachy-mini/daemon-rest-api/ — refusing to call`.

## Output format

For `quick` and `full`: one Markdown table per section, with column names in the language of the running conversation. Section names: **Daemon**, **App lock**, **Motors**, **Pose**, **Audio**, **Current app** (the last three only in `full`).

Missing or failed responses are marked explicitly, never blank:

- `unreachable` — connection refused / DNS failure
- `timeout` — request exceeded `timeout_per_get_s`
- `http <code>` — non-2xx response (e.g. `http 500`, `http 404`)
- `unknown` — 2xx with an empty or unrecognised body

Final line of every output: `daemon: <daemon_host> · snapshot: <ISO-8601-UTC>` so a second inspection is directly comparable.

For `raw <endpoint>`: one fenced code block with the JSON response, canonicalised (sorted keys, 2-space indent) so successive snapshots diff cleanly.

## Concurrency note

When a Pollen app is running, the daemon's state endpoints still answer — they're independent of the app-lock. The snapshot is taken while that app is live, which can shift pose / motor values relative to an idle robot. The output **MUST** add a one-line note `observed while app '<current_app>' is running` when `current-app-status` reports a non-null app; in `quick` mode the same note appears even though `current-app-status` isn't part of the three GETs (the lock-status response already names the holder).

## Gotchas

- **`/api/state/full` is a single round-trip** that bundles pose, body yaw, antenna positions, and DoA. The skill prefers this aggregate in `full` mode over fanning out to the four individual `/api/state/present_*` endpoints. If the daemon's aggregated response lacks a field that the user explicitly asked about, fall back to the matching individual endpoint for that one field — do not loop over all four.
- **First Wireless mDNS resolution is slow.** A cold `reachy-mini.local` lookup can spend several seconds in DNS before any HTTP traffic. Don't tighten `timeout_per_get_s` below 5 s for Wireless without explicit user opt-in.
- **HTTP 200 with an empty body is not success.** The daemon occasionally returns a 200 with `{}` while it warms up. The skill maps that to `unknown` in the table, with a hint to retry once.
- **The `raw` mode is not a backdoor to mutation.** Even when the daemon-rest-api inventory documents an endpoint that supports POST or DELETE, this skill only invokes the GET variant. A path that exists exclusively under a non-GET method is rejected at validation time.
- **Output is for humans first.** The skill is primarily a conversational tool. A `--json` machine-readable shape is intentionally out of scope; programmatic consumers should hit the REST API directly using the inventory in `spec/reachy-mini/daemon-rest-api/`.
- **`backend_status.ready=false` is not a definitive stop.** Verified live 2026-05-13: the daemon-side `_status.ready` flag is never set to `true` in the polling loop — it stays `false` even when the bus is healthy and the loop is running at ~50 Hz. The skill therefore does **not** route `backend.ready=false` to a Stage-3 alert by itself; cross-checks per Pre-flight §3 (mean_freq, `head_joints` populated, pose micro-drift) decide.
- **"Silent dead" pattern — escalate, do not retry.** When all three Pre-flight §3 cross-checks fail (no joints, no mean-freq, no drift), the backend bus is genuinely hung. Surface that fact to the user with the suspected wait-point (e.g. `wchan=bcm2835_i2c_xfer` → BMI088 IMU on I²C bus 4, see [`spec/reachy-mini/motor-positions/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) §"Stage-3 recovery paths"). **Do not** auto-issue `POST /api/daemon/restart` or `POST /api/motors/set_mode/*` from this skill — those are mutations and belong to recovery skills, not to a read-only inspector.

## Boundaries to neighbouring artifacts

- App lifecycle (install / start / stop / update / remove) → `reachy-mini-deploy` (agent), `reachy-mini-start` (skill)
- Live trial with telemetry → `reachy-mini-on-device` (agent)
- Behavior / motion authoring → `reachy-mini-sdk` (skill), `app-scaffold` (skill)
- Log analysis after a failure → `app-log-triage` (skill)
- Bounded background monitoring of a behavior session with anomaly classification → `motion-monitor` (agent); this skill is the one-shot snapshot, `motion-monitor` is the 5–15 min audit-log producer. They consume the same Tier-1 reads but report differently.
- Motion-anomaly classification (Class A head-against-body, B jerky motion, C antenna wobble, D Stewart-limit) — this skill's `full`-mode three-way liveness cross-check (mean_freq + head_joints + pose micro-drift) is the **Class A live-detection signal** from [`reachy-mini/motion-anomaly-detection`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motion-anomaly-detection/de.md). Single-shot inspect surfaces the liveness state; ongoing classification of all four classes belongs to `motion-monitor`.
- MCP-server distribution for LLM frontends → `mcp-server-bootstrap` (skill); reachy-mini-inspect is the parallel plugin-skill distribution of the same Tier-1 reads, never a duplicate of mcp-server tools

Most concrete REST behaviour was verified live on a Reachy Wireless v1.7.1 (2026-05-13): the eight T1–T8 motion targets from [`spec/reachy-mini/motor-positions/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) §"T1–T8 live verification" ran with pose-diff norms ≤ 0.08 rad, the `with_head_joints=true` parameter is mandatory for live joints, the `_status.ready`/`last_alive` desync is recorded as a daemon bug, and the IMU/I²C `silent dead` pattern plus Stage-3 recovery paths are documented in motor-positions Layer 5. Edge cases still requiring verification: empty-body 200s under load, mDNS cold-start latency on a freshly booted Wireless, and Lite (USB-host-driven) behaviour — those remain ⚠ TBD until first contact with those configurations.
