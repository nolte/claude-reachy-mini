# MCP Server Bootstrap Skill

Status: draft

## Context

The knowledge spec [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/en.md) defines the canonical shape of an MCP server that exposes Reachy Mini through the daemon REST API to an MCP client (Claude Desktop, Claude Code, Cursor, custom client). The implementation lives in a separate repository (`reachy-mini-mcp-server`), distributed as a PyPI package. What is missing is the **operational shell** that takes a developer from "package not installed" to "MCP client talking to Reachy" in five minutes — install, configure, bring up, health-check against the local daemon, and emit a ready-to-paste config snippet for the MCP client frontend.

This `mcp-server-bootstrap` skill is exactly that operational shell. It is **narrow**: it does not install a package that is already installed, it spins up the server only as a deliberately short-lived health-check subprocess, and it never writes tools itself. The server implementation, the tool inventory, the safety gates — those are the knowledge spec's and the server repo's business. This skill is operations: verify preconditions, bring the server up or register it as a daemon unit, emit the MCP client configuration, verify the audit-log path is writable.

Term clarification: "bootstrap" here = first-time bring-up or restart of an already-installed server, plus generation of the MCP client configuration; **not** writing server code, **not** implementing tools, **not** building a distribution.

## Goals

- A Reachy Mini MCP server session runs with a single skill invocation, given the Pollen daemon and the server package are available
- The pre-flight checks from the knowledge spec ([`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/en.md)) are run consistently in the same order — daemon reachability, platform detect, audit-log writability, MCP client configuration visibility
- Three run modes cleanly demarcated: **`stdio`** (server as a subprocess of the MCP client, short-lived), **`http`** (standalone local daemon process for several clients), **`systemd`** (system unit for a server on a provisioned host)
- MCP client configuration snippets are emitted for the common frontends (Claude Desktop, Claude Code, Cursor) — copy-paste ready, not generic hand-waving
- The skill stays narrow: no server code edits, no daemon configuration, no plugin / app lifecycle interventions; everything delegated to neighbours

## Non-Goals

- The server implementation itself — the code lives in `reachy-mini-mcp-server` (separate repo); this skill does not scaffold a server repo
- Tool inventory definitions or tool code — spec / implementation business, not operations
- Daemon restart, app-lock force release, hardware recovery — none of the Tier 3 operations from the knowledge spec are mirrored here
- Auth configuration for remote mode — the knowledge spec says v1 is localhost-only; bootstrap therefore stays localhost-only too
- Plugin / app distribution (`app-scaffold`, `reachy-app-publish-hf`)
- Live trial against hardware (`reachy-mini-on-device` agent)
- CI / CD integration, GitHub Actions workflows, cloud deployment
- Multi-user / tenant separation — the knowledge spec says single-user-localhost; bootstrap respects that

## Requirements

### Trigger and activation

- **MUST** carry a `description` that activates Claude Code on phrasings like "start the MCP server for Reachy Mini", "bring up MCP server", "start the Reachy MCP server", "bootstrap the reachy-mini MCP", "configure Claude Desktop for Reachy Mini"
- **MUST** include the keywords in the `description`: MCP, server, Reachy Mini, bootstrap, start, configure
- **SHOULD** explicitly call out when _not_ to activate: server-code edits (separate repo), tool implementation (separate repo), daemon restart (hardware concern, not bootstrap), live-trial / on-device tests ([`reachy-mini-on-device`](../reachy-mini-on-device/en.md) agent), plugin releases (`nolte-shared:release-publish-trigger`)

### Input parameters

- **MUST** accept the run mode (`mode`: `stdio` | `http` | `systemd`); default is `stdio` (lowest latency, simplest setup, no lifecycle daemon needed)
- **MUST** accept the desired MCP client frontend (`frontend`: `claude-desktop` | `claude-code` | `cursor` | `generic`); the configuration snippet is generated from this choice
- **SHOULD** accept the daemon address as an optional override parameter (`daemon_url`, default `http://127.0.0.1:8000`) — Wireless with mDNS default `http://reachy-mini.local:8000` supported separately
- **SHOULD** accept the bind port for `http` mode (`mcp_port`, default `47600`) and check for conflicts before start
- **SHOULD** accept an optional `log_level` (`info` | `debug`, default `info`)
- **MUST NOT** the skill default to `0.0.0.0` as the bind address — localhost-only is binding, inherited from the knowledge spec

### Pre-flight (every run, before any start action)

The skill **MUST** check the following gates in order before starting the server, and abort on the first failure with a concrete remediation hint:

1. **Server package installed** — `reachy-mini-mcp-server --version` resolves in the active Python environment; if missing, abort with `uv tool install reachy-mini-mcp-server` (or `uv pip install reachy-mini-mcp-server` in the active venv) as the recommendation
2. **Daemon reachable** — HTTP probe against `<daemon_url>/api/daemon/status` (or the equivalent status endpoint from [`apps.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers) / [`daemon.py`](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers)); on Wireless via mDNS lookup on `reachy-mini.local`; on failure, recommend the daemon start (Lite: `reachy-mini-daemon`; Wireless: `systemctl status reachy-mini-daemon.service` over SSH)
3. **Platform detect** — the server package runs a short platform-detect routine (`wireless` / `lite` / `simulation`); the detected profile is reported, so the operator knows which Tier 1 tools (IMU, battery) are available
4. **Audit-log path writable** — `~/.cache/reachy-mini-mcp/<YYYY-MM-DD>.log` must be creatable; missing permission aborts, never silently fall back to `/tmp/`
5. **Port free (`http` mode)** — when `mode=http`, verify `mcp_port` is free; on conflict abort with a clear recommendation, do not pick a random different port
6. **No existing MCP server instance** — for `stdio` / `http`, check whether an instance is already running (pid file / socket); on conflict abort, do not start two parallel servers

- **MUST NOT** the skill enter the start step without a green pre-flight
- **SHOULD** the skill name the next concrete action on every failed gate (e.g. "daemon unreachable → start it with `reachy-mini-daemon`")

### Run modes

| Mode | Run shape | Lifecycle | Use case |
|---|---|---|---|
| **`stdio`** (default) | Server as a subprocess of the MCP client, stdin / stdout pipe | Lives with the MCP client session | Claude Desktop / Claude Code / Cursor — most frontends expect exactly this |
| **`http`** | Server as a standalone local process, HTTP listener on `127.0.0.1:<port>` | Own process, started / stopped by the user | Several MCP clients at once (e.g. Claude Code + a local web UI), or sessions across multiple backends |
| **`systemd`** | Server as a systemd user unit, automatic restart | Daemonised, optionally running on a provisioned host | Wireless Reachy as an "always-on endpoint" or Lite host as a persistent MCP service |

- **MUST** the skill document the stop path for every mode (Ctrl-C / `pkill` / `systemctl --user stop`); no mode without a clear stop instruction
- **SHOULD** the skill render the user-unit file (`~/.config/systemd/user/reachy-mini-mcp.service`) idempotently in `systemd` mode, not overwrite it on every run — the user keeps manual customisations
- **MUST NOT** the skill create systemd system units (`/etc/systemd/system/`) — the MCP server is single-user, not a system-wide service

### Workflow

1. **Pre-flight** as above; abort on the first failure with a concrete action
2. **Server start** in the chosen `mode`:
   - `stdio`: the skill reports the start command and leaves the actual spawning to the MCP client (config snippet — see next step)
   - `http`: the skill starts the server as a background process, waits 2 s, then runs the health check
   - `systemd`: the skill renders / updates the user unit, reloads systemd (`systemctl --user daemon-reload`), starts the unit (`systemctl --user start reachy-mini-mcp.service`)
3. **Health check** — `tools/list` probe against the server, expect at least the Tier 1 inventory plus `health-check` (see knowledge spec)
4. **MCP client configuration snippet** — frontend-specific, copy-paste ready, populated with the active `mode` and `daemon_url`
5. **Report** — pre-flight status, server run status, config snippet path or content, stop instructions

- **MUST** the skill execute the workflow strictly sequentially — no parallelism, no background wait without a health check
- **MUST NOT** the skill modify server code, write a `pyproject.toml`, or add tools — that is the server repo's business

### MCP client configuration snippets

One **canonical** snippet per frontend:

- **`claude-desktop`**: JSON entry in `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%/Claude/claude_desktop_config.json` (Windows); `mcpServers` block with `command`, `args`, `env` mapping
- **`claude-code`**: entry in `~/.claude/settings.json`'s `mcpServers` block (same schema)
- **`cursor`**: `mcpServers` entry in Cursor's MCP config (path is version-dependent — the skill must pull it from the Cursor docs)
- **`generic`**: generic JSON, `mode=stdio`, with a note that any MCP-capable client can consume it the same way

- **MUST** the skill name the path to the config file and the schema of the entry for every frontend value — never a snippet without an explanation of where it goes
- **SHOULD** the skill offer to either **only print** the snippet (default) or write it to the file (opt-in, with confirmation) — never without confirmation
- **MUST NOT** the skill overwrite existing `mcpServers` entries from the user without a diff display and confirmation

### Out-of-scope clarification

- **MUST NOT** the skill write server code, tool code, or server configuration beyond the run mode — that belongs in the `reachy-mini-mcp-server` repo
- **MUST NOT** the skill start, stop, or reconfigure the Pollen daemon — that is hardware operation, not MCP bootstrap
- **MUST NOT** the skill move the Reachy, trigger pose tools, or read audit logs of other server instances — the skill is operations, not tool invocation
- **SHOULD** the skill point at the [`reachy-mini-on-device`](../reachy-mini-on-device/en.md) agent for a dedicated live trial — the MCP server is not meant for bulk test lifecycles
- **SHOULD** the skill point at [`reachy-mini-sdk`](../reachy-mini-sdk/en.md) for SDK idioms when an MCP client session has follow-up questions about method choice or safe-torque

## Acceptance Criteria

- [ ] The skill lives at `skills/mcp-server-bootstrap/SKILL.md` with valid frontmatter (`name: mcp-server-bootstrap`, `description`, optional tags) and is accepted by the catalog generator
- [ ] The `description` carries the keywords (MCP, server, Reachy Mini, bootstrap, start, configure) and explicitly calls out at least three anti-triggers
- [ ] Three run modes (`stdio`, `http`, `systemd`) are documented, each with a clear lifecycle and stop instruction
- [ ] Six pre-flight gates run in the specified order, each with a concrete remediation hint
- [ ] The server is **never** started without a green pre-flight
- [ ] The health check happens after the server start and before the config snippet is emitted
- [ ] MCP client configuration snippets are spelled out for `claude-desktop`, `claude-code`, `cursor`, `generic`, with the correct config-file path per frontend
- [ ] Writing the snippet into the config file is opt-in, with a diff display when the existing `mcpServers` entry would be overwritten
- [ ] The systemd mode renders user units only (`~/.config/systemd/user/`), never system units
- [ ] The skill **does not** modify server code; the skill **does not** write tool code
- [ ] Cross-refs to [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/en.md), [`reachy-mini-sdk`](../reachy-mini-sdk/en.md), [`reachy-mini-on-device`](../reachy-mini-on-device/en.md), [`reachy-mini-start`](../reachy-mini-start/en.md), [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md) are visible
- [ ] On a daemon connection refusal the skill emits a concrete daemon-bring-up instruction, never a silent failure
- [ ] `pre-commit run --all-files` passes on the skill file

## References

> Source references to Pollen code files point at the matching directory in the daemon tree; Markdown sources are cited at file level. Configuration paths per MCP client follow the respective frontend-doc state.

- Knowledge spec (canonical source for tool inventory and security model): [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/en.md)
- Pollen daemon status endpoint (pre-flight source): <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon/app/routers>
- Pollen daemon reachability pattern (Wireless mDNS, Lite localhost): [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md) and [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) §78
- Model Context Protocol — specification: <https://modelcontextprotocol.io>
- Model Context Protocol — Python server SDK: <https://github.com/modelcontextprotocol/python-sdk>
- Claude Desktop MCP configuration: <https://docs.anthropic.com/en/docs/claude-code/mcp>
- systemd user units (for `systemd` mode): <https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html>
- Internal cross-refs:
  - [`reachy-mini/mcp-server`](../../reachy-mini/mcp-server/en.md) — knowledge of tool inventory and security
  - [`claude/reachy-mini-sdk`](../reachy-mini-sdk/en.md) — SDK idioms (method choice, safe-torque) for follow-up questions of the server operator
  - [`claude/reachy-mini-on-device`](../reachy-mini-on-device/en.md) — test lifecycle as a sister surface
  - [`claude/reachy-mini-start`](../reachy-mini-start/en.md) — app-start pattern as a role model for clean pre-flight sequences
  - [`reachy-mini/host-provisioning`](../../reachy-mini/host-provisioning/en.md) — systemd-unit conventions and mDNS default

## Open Questions

- Server package name: is `reachy-mini-mcp-server` the right name on PyPI? Coordination with Pollen would be worthwhile, in case they plan to publish something similar.
- Health-check tool: is `health-check` available as a dedicated MCP tool in the server, or is `tools/list` enough as an implicit reachability probe? Proposal: an explicit `health-check` tool, because an empty tool inventory would be a silent bug.
- `mcp_port` default: `47600` is a plausibly unallocated range, but is there a Pollen / MCP convention to align with?
- Per-frontend config-file paths: Cursor's MCP path is version-dependent — should the skill maintain it, or should the user look it up?
- Auto-restart backoff in `systemd` mode: what are sensible defaults (`Restart=on-failure`, `RestartSec=5s`)? Consumer consensus needed.
- Multi-Reachy bootstrap: a developer has two devices (e.g. Wireless + Lite) — should the skill bring up several server instances in parallel, or is that an app concern?
- Server update path: how does a version bump of the `reachy-mini-mcp-server` package make itself felt in the bootstrap skill? Should the skill check whether the installed package is stale?
- MCP client config-file backup: take a backup before writing (`<file>.bak.<timestamp>`)? Proposal: yes, low cost, high protection.
