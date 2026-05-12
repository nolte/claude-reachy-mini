---
name: reachy-mini-inspect
description: Snapshot the current state of a running Reachy Mini daemon as a Markdown table directly in the conversation — daemon liveness, app-lock, motor mode, head/body/antenna pose, audio system, speaker and microphone volume, currently running app. Read-only: only GET endpoints, never POST/PUT/DELETE/PATCH, never touches the app-lock. Activate when the user says "zeig mir den Zustand", "wie steht der Reachy gerade", "Daemon-Status abfragen", "Motorposition lesen", "show robot state", "check daemon health", "where is the head pointing", or invokes the modes `quick` / `full` / `raw <endpoint>` directly. Do NOT use to start, stop, install, update, or remove an app (that's `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`), to write motion or behavior code (`reachy-mini-sdk`, `app-scaffold`), to analyze app logs (`app-log-triage`), or to stand up an MCP server for LLM frontends (`mcp-server-bootstrap`).
tags: [reachy-mini, inspect]
---

# Reachy Mini Inspect

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-inspect/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-inspect/en.md).

Authoritative endpoint inventory: [`spec/reachy-mini/daemon-rest-api/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md). Complementary distribution for LLM frontends outside Claude Code: [`spec/reachy-mini/mcp-server/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/mcp-server/de.md).

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

- `GET /api/state/full` — head pose, body yaw, antenna joint positions, DoA
- `GET /api/media/status` — audio subsystem status
- `GET /api/volume/current` — speaker volume
- `GET /api/volume/microphone/current` — microphone volume
- `GET /api/apps/current-app-status` — currently running app (if any)

### `raw <endpoint>` (single GET, validated)

Issue exactly one `GET` against `<daemon_host><endpoint>` and emit the response as a fenced code block. Before issuing, the path **MUST** be matched against the endpoint inventory in [`spec/reachy-mini/daemon-rest-api/`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/daemon-rest-api/de.md); a path not listed there or listed under a non-`GET` method is rejected with a clear message.

## Pre-flight (every mode, in order — abort on first failure)

1. Resolve the daemon endpoint from `daemon_host` (default localhost). On Wireless with mDNS, the first call can be noticeably slower than steady-state — accept that.
2. Issue `GET /api/daemon/status` as the reachability check. On connection refused / DNS failure / timeout, abort with a single line: `daemon unreachable at <daemon_host>: <connection_error>`. No stacktrace.
3. For `raw` mode only, validate `endpoint` against the daemon-rest-api inventory before issuing the call. Reject silently is forbidden; report `endpoint <path> is not in spec/reachy-mini/daemon-rest-api/ — refusing to call`.

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

## Boundaries to neighbouring artifacts

- App lifecycle (install / start / stop / update / remove) → `reachy-mini-deploy` (agent), `reachy-mini-start` (skill)
- Live trial with telemetry → `reachy-mini-on-device` (agent)
- Behavior / motion authoring → `reachy-mini-sdk` (skill), `app-scaffold` (skill)
- Log analysis after a failure → `app-log-triage` (skill)
- MCP-server distribution for LLM frontends → `mcp-server-bootstrap` (skill); reachy-mini-inspect is the parallel plugin-skill distribution of the same Tier-1 reads, never a duplicate of mcp-server tools

> ⚠ TBD: validate against real hardware — every concrete REST endpoint and aggregated-response field above is a best-effort design until verified on a physical Reachy. The endpoint paths are confirmed present in the live `openapi.json` (see `spec/reachy-mini/daemon-rest-api/`), but response shapes and edge-case behaviour (empty body, timeout under load, mDNS cold-start) need first-contact verification on both Wireless and Lite.
