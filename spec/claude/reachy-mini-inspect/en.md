# Read-only inspect skill for the Reachy Mini daemon

Status: draft

## Context

Anyone operating a Reachy Mini daemon day-to-day often just wants a quick read on the current state: Is the daemon up? Does an app hold the app-lock? Are the motors stiff or compliant? Where is the head pointing? How loud is the speaker? Today that read carries noticeable friction. Three paths exist — and none fit a fast "show me":

1. **Manual `curl` against individual `/api/...` endpoints** — the user needs to know the exact paths, parse JSON, and assemble a picture from multiple responses.
2. **Python REPL with the `reachy_mini` SDK** — works, but pulls in a library, may hold an SDK connection, and is oversized for a single question.
3. **The planned MCP server** (`spec/reachy-mini/mcp-server/`) — serves LLM frontends outside Claude Code (Claude Desktop, Cursor) via the MCP protocol; it is designed as a separate server process and requires setup. The MCP-server code lives outside this plugin.

What's missing is a **skill distribution inside Claude Code** for the same read access: directly in the main context, no server setup, no SDK instantiation. `reachy-mini-inspect` closes that gap. It is the complementary skill distribution of what the planned MCP server covers as Tier-1 reads — both may coexist and consume the identical REST endpoints from [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/en.md).

## Goals

- Three operation modes with clearly bounded output volume: `quick` (default, minimal status), `full` (all read endpoints), `raw <endpoint>` (a single GET passed through)
- Response as a compact Markdown table / structured report in the main context — suitable as a conversational interjection, not a full document
- Read-only guarantee: the skill calls **only** GET endpoints, never a mutating endpoint (POST / PUT / DELETE)
- Clear separation from the MCP-server distribution: the skill links the mcp-server spec but duplicates no tools — both distributions reference the same REST API as the source of truth
- Low barrier: no installation of an additional process; the skill is immediately invocable from Claude Code as soon as the plugin is loaded
- Readable error output on an unreachable daemon — a single clear line, no Python traceback, no silent ignore

## Non-Goals

- Write access to the daemon (set operations stay with `reachy-mini-sdk` for SDK idioms or with the MCP server for LLM consumers)
- A separate server process — that's the mcp-server spec
- App lifecycle (install / start / stop / remove / update) — belongs to `reachy-mini-start`, `reachy-mini-deploy`, `reachy-mini-on-device`
- Behavior or motion development — `reachy-mini-sdk`, `app-scaffold`
- Hardware diagnostics below the REST surface (USB detection, firmware versions, driver logs) — that's the domain of Pollen's hardware-troubleshooting docs
- Streaming / high-frequency polling — `quick` and `full` are snapshots; anyone who needs continuous pose reads belongs in an SDK app with a `set_target` loop
- Logging or crash triage — that's `app-log-triage`
- A custom caching layer — the skill is stateless; whoever wants caching does it on the caller side

## Requirements

### Configuration and daemon reachability

- The daemon host **MUST** be configurable (default `http://127.0.0.1:8000`, Wireless mDNS override `http://reachy-mini.local:8000`, Lite localhost) — never default to a non-localhost address
- The skill **MUST** run a reachability check as the first operation in each mode (`GET /api/daemon/status`) and abort on connection refused with a single clear error message — no stacktrace, no silent fallback
- The skill **MUST** validate the HTTP status on every response; HTTP `5xx` and timeouts are marked **explicitly** in the output, never emitted as an empty cell
- The skill **SHOULD** use a default request timeout of 5 s per GET and **SHOULD NOT** exceed a total of 15 s in the `quick` and `full` modes

### Operation modes

- The skill **MUST** support three modes — `quick` (default), `full`, `raw <endpoint>`
- The `quick` mode **MUST** issue exactly these three GETs and return them as a Markdown table: `GET /api/daemon/status`, `GET /api/daemon/robot-app-lock-status`, `GET /api/motors/status`
- The `full` mode **MUST** cover, in addition to the three `quick` GETs, the aggregated state and the audio/volume picture — sources: `GET /api/state/full` (pose, body yaw, antennas, DoA), `GET /api/media/status`, `GET /api/volume/current`, `GET /api/volume/microphone/current`, `GET /api/apps/current-app-status`
- The `raw` mode **MUST** validate the path against the endpoint inventory in [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/en.md) — a path not listed there is rejected with a clear hint
- The `raw` mode **MUST** be restricted to GET endpoints — even when the daemon inventory documents an endpoint under another method, this skill rejects any method other than `GET`
- The `full` mode **SHOULD** parallelise the GETs to keep snapshot latency low

### Read-only guarantee

- The skill **MUST NOT** ever call a POST, PUT, DELETE, or PATCH endpoint — not even as a "harmless" probe (e.g. `POST /health-check`); health liveness is covered via the GET status endpoint
- The skill **MUST NOT** touch, change, or force-stop the app-lock — the skill reads the lock state, no more
- The skill **MUST NOT** loop response reads in `quick` or `full` — one snapshot per call, no polling logic
- The skill **MUST NOT** hold a custom caching layer — every call hits the daemon fresh; that keeps the skill stateless and avoids stale-data bugs

### Output format

- The output **MUST** contain a Markdown table per section (for example "Daemon", "App lock", "Motors", "Pose", "Audio", "Current app") — column names in the language of the running conversation
- The output **MUST** mark a missing response explicitly as `unreachable`, `timeout`, or `http <code>` — never leave it blank
- The skill **MUST** print the authoritative daemon host and the snapshot timestamp (ISO-8601 UTC) on its final line, so a second inspection can be compared
- The skill **MUST NOT** spill raw JSON bodies into the main context — the `raw` mode returns the response as a formatted code block, everything else is reduced to the fields relevant to the snapshot

### Relation to other skills / specs

- The skill body **MUST** explicitly reference [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/en.md) as the authoritative endpoint source
- The skill body **MUST**, in its scoping section, reference [`spec/reachy-mini/mcp-server/`](../../reachy-mini/mcp-server/en.md) and briefly explain the complementary distribution — why both exist, what sets them apart
- The skill **SHOULD** redirect inspect requests that go beyond reads to the appropriate skill — move operations → `reachy-mini-sdk`, app start → `reachy-mini-start`, live trial → `reachy-mini-on-device`, log analysis → `app-log-triage`
- The skill **MUST NOT** duplicate tools or operations from the mcp-server spec — the mcp-server spec describes its own distribution; this skill is the plugin-skill distribution of the same read surface

## Acceptance Criteria

- [ ] The skill exists at `skills/reachy-mini-inspect/SKILL.md` with valid frontmatter (`name: reachy-mini-inspect`, `description`, `distribution: plugin`)
- [ ] The `description` activates on phrasings like "show me the state", "check the daemon", "read the motor position", "where is the head pointing", "show robot state", "check daemon health", "wie steht der Reachy gerade"
- [ ] Quick mode is the default — a call without arguments returns exactly the three specified GETs
- [ ] Full mode covers the endpoints named in the requirements and is parallelised
- [ ] Raw mode validates the path against `spec/reachy-mini/daemon-rest-api/` and rejects non-GET requests
- [ ] The skill never calls a POST, PUT, DELETE, or PATCH endpoint anywhere in its code path — verifiable via code review or via a dedicated test snippet in the skill body
- [ ] Connection refused yields a single clear error message naming the daemon host, with no stacktrace
- [ ] The output is a Markdown table in the language of the running conversation; every missing response is explicitly marked
- [ ] Cross-references to `spec/reachy-mini/daemon-rest-api/` and `spec/reachy-mini/mcp-server/` are visible in the skill body
- [ ] DE and EN specs are structurally in sync; the order of requirements and acceptance criteria is identical

## References

- Authoritative endpoint inventory: [`spec/reachy-mini/daemon-rest-api/`](../../reachy-mini/daemon-rest-api/en.md) (lands on develop with PR #23)
- Complementary MCP distribution: [`spec/reachy-mini/mcp-server/`](../../reachy-mini/mcp-server/en.md)
- MCP-server bootstrap skill (the other distribution): [`spec/claude/mcp-server-bootstrap/`](../mcp-server-bootstrap/en.md)
- Live source of endpoint truth: `http://<daemon-host>:8000/openapi.json` — typical hosts: `http://reachy-mini.local:8000` (Wireless), `http://127.0.0.1:8000` (Lite or sim)
- Skill-vs-agent heuristic: [`spec/claude/skill-vs-agent/`](https://github.com/nolte/claude-shared/blob/develop/spec/claude/skill-vs-agent/) (in the claude-shared plugin)
- Platform limits and hardware inventory as reading context: [`spec/reachy-mini/control-surface/`](../../reachy-mini/control-surface/en.md)
- Anomaly classes and the binding event-record schema (this skill is the **live consumer** for Class A via the three-way liveness cross-check in `full` mode): [`spec/reachy-mini/motion-anomaly-detection/`](../../reachy-mini/motion-anomaly-detection/en.md)
- Sibling agent for bounded sessions with audit-log output (same Tier-1 reads, different output format): [`spec/claude/motion-monitor/`](../motion-monitor/en.md)

## Open Questions

- Default timeout per GET is suggested at 5 s — verify on first hardware contact; on Wireless with cold mDNS resolution the first GET can take noticeably longer
- Should `full` mode use the aggregated `GET /api/state/full` OR fetch the individual endpoints (`present_head_pose`, `present_body_yaw`, `present_antenna_joint_positions`, `doa`) and combine? Trade-off: `/api/state/full` is a single round trip but potentially with less detail; the individual endpoints in parallel are more requests but more granular
- How does the skill handle the edge case "daemon up but response 200 with empty body"? Suggestion: mark as `unknown` with a clear hint in the output, do not treat as success
- Should `raw` mode emit the JSON response in a canonical format (sorted keys, 2-space indent) or pass the body through verbatim? Tentative: canonical — comparable between calls
- On `quick` or `full` with an app running concurrently: should the snapshot mark that the values were observed under app load? Suggestion: yes, with an extra line "observed while app `<name>` is running"
- Should there be a `--json` switch that replaces the table with a JSON object for machine consumers? Deliberately left open — the skill is primarily for conversational consumption; JSON consumers can hit the REST API directly
