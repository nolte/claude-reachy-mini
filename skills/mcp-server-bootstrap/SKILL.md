---
name: mcp-server-bootstrap
description: >-
  Bootstrap a Reachy Mini MCP server session — install-check the
  `reachy-mini-mcp-server` PyPI package, run the six-gate pre-flight
  (daemon reachable, platform detected, audit-log writable, port free,
  no duplicate instance), start the server in one of three modes
  (`stdio` / `http` / `systemd` user-unit), health-check via `tools/list`,
  and emit a copy-paste-ready MCP-client configuration snippet for the
  chosen frontend (`claude-desktop`, `claude-code`, `cursor`, `generic`).
  Activate on phrasings like "MCP-Server für Reachy Mini starten",
  "MCP server hochfahren", "start the Reachy MCP server", "bootstrap the
  reachy-mini MCP", "configure Claude Desktop for Reachy Mini". Do NOT
  activate to write MCP server code or tool implementations (lives in the
  `reachy-mini-mcp-server` repo, not in this plugin), to restart the
  Pollen hardware daemon (out of scope), to run live-trial behaviors
  (`reachy-mini-on-device` agent), to deploy or publish apps
  (`reachy-mini-deploy`, `reachy-app-publish-hf`), or for plugin releases
  (`nolte-shared:release-publish-trigger`).
tags: [reachy-mini, mcp, bootstrap, server]
---

# MCP Server Bootstrap

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/mcp-server-bootstrap/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/mcp-server-bootstrap/en.md).

The knowledge spec [`reachy-mini/mcp-server`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/mcp-server/de.md) defines the canonical shape of a Reachy-Mini MCP server (tool inventory, security model, localhost-only constraint). The server implementation itself lives in a separate repo (`reachy-mini-mcp-server`, distributed as a PyPI package). This skill is the **operational shell** around that package — it does not write server code, it does not implement tools, it does not configure the Pollen hardware daemon. It runs pre-flight, starts the server in the chosen mode, health-checks it, and emits an MCP-client config snippet.

## Skill-vs-Agent rationale

This is a **skill** rather than an agent — every load-bearing dimension points the same way:

- **Mid-flow user gating is the contract** — six pre-flight gates abort with a concrete remediation, `systemd` mode renders a user-unit only after confirmation, snippet-write-into-config is opt-in with diff display; an agent's fire-and-forget shape would lose those gates.
- **Short, sequential operation** — install-check → six gates → start → health-check → snippet; a handful of subprocess calls, not a long-running supervisory loop.
- **Output flows naturally inline** — pre-flight status, server-run status, config snippet, stop instructions — all read inline in the conversation; isolating them behind an agent boundary would obscure the copy-paste-ready snippet.
- **Counter-dimension considered** — none of "context-window protection", "fire-and-forget background job", or "structured PASS/FAIL report" apply here; skill wins.

## When this skill activates

Use this skill when the developer wants to:

- **start an MCP server session** for an already-installed `reachy-mini-mcp-server` package
- **generate an MCP-client config snippet** for Claude Desktop, Claude Code, Cursor, or a generic MCP client
- **re-bootstrap** an existing setup (Pollen-daemon address changed, frontend switched, mode changed)
- **register a `systemd` user-unit** so the MCP server runs on the local host as an always-on endpoint

Trigger phrasings — DE: "MCP-Server für Reachy Mini starten", "MCP server hochfahren", "Claude Desktop für Reachy Mini konfigurieren", "reachy-mini MCP bootstrappen". EN: "start the Reachy MCP server", "bootstrap the reachy-mini MCP", "configure Claude Desktop for Reachy Mini", "spin up the MCP server".

## When NOT to activate

- writing MCP server code or new tools → `reachy-mini-mcp-server` repo (separate)
- restarting / starting / stopping the Pollen hardware daemon → out of scope (hardware lifecycle, not MCP bootstrap)
- live-trial validation of a behavior on the device → agent [`reachy-mini-on-device`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-on-device.md)
- deploying or installing an app → agent [`reachy-mini-deploy`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-deploy.md)
- publishing an app to Hugging Face Spaces → [`reachy-app-publish-hf`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-app-publish-hf/SKILL.md)
- plugin releases of `claude-reachy-mini` itself → `nolte-shared:release-publish-trigger`
- bulk on-device test lifecycles → on-device agent, not MCP server
- multi-user / remote-network exposure → spec mandates localhost-only single-user; not this skill's scope

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `mode` | yes | `stdio` | `stdio` (server is a subprocess of the MCP client, stdin/stdout pipe), `http` (own local process, listener on `127.0.0.1:<port>`), `systemd` (user-unit at `~/.config/systemd/user/`) |
| `frontend` | yes | — | `claude-desktop`, `claude-code`, `cursor`, or `generic`; drives the config-snippet shape and target file path |
| `daemon_url` | no | `http://127.0.0.1:8000` | Pollen daemon address; Wireless via mDNS typically `http://reachy-mini.local:8000` |
| `mcp_port` | no | `47600` | `http`-mode bind port on `127.0.0.1`; checked for conflict during pre-flight |
| `log_level` | no | `info` | `info` or `debug` |
| `write_config` | no | `false` | If `true`, after presenting the snippet, ask for confirmation and then write into the frontend's config file (with diff display on existing `mcpServers` entries) |

## Hard rules

1. **Localhost-only.** The skill **MUST NOT** bind to `0.0.0.0` or any non-loopback interface. The spec `reachy-mini/mcp-server` mandates single-user localhost; remote exposure is out of scope for this bootstrap path.
2. **No server-code edits.** The skill **MUST NOT** modify `reachy-mini-mcp-server` source, `pyproject.toml`, or tool inventories. That belongs to the server repo, not to this operational shell.
3. **No daemon mutations.** The skill **MUST NOT** restart, reload, or reconfigure the Pollen hardware daemon. The pre-flight only *probes* it via `GET /api/daemon/status`.
4. **No silent fallback.** A failed pre-flight gate **MUST** abort with a concrete next-action message. The skill **MUST NOT** silently fall back to `/tmp/` for the audit log, pick a random port, or kill a sibling server instance.
5. **No system-wide systemd units.** The `systemd` mode **MUST** only render user-units under `~/.config/systemd/user/`. Writing to `/etc/systemd/system/` is forbidden.
6. **Pre-flight always.** The skill **MUST** run all six pre-flight gates in order on every invocation, even when called back-to-back. Skipping pre-flight on a "fast re-run" is forbidden.
7. **Snippet-write is opt-in.** Writing the config snippet into the frontend's config file requires `write_config=true` plus an explicit user confirmation. A pre-existing `mcpServers` entry **MUST** trigger a diff display before any overwrite.

## Pre-flight gates (run in this order, abort on first failure)

The skill **MUST** run these six gates in sequence. Each gate aborts with a concrete remediation; never proceeds with a yellow signal.

1. **Server package installed** — `reachy-mini-mcp-server --version` resolves in the active Python environment. If missing: abort with `uv tool install reachy-mini-mcp-server` (preferred) or `uv pip install reachy-mini-mcp-server` (inside the active venv) as the recommended fix.
2. **Daemon reachable** — `GET <daemon_url>/api/daemon/status` returns HTTP 200 within 3 s. On `connection refused` / DNS failure / timeout: abort with the diagnostic. Wireless: also try the mDNS default `http://reachy-mini.local:8000` if the explicit `daemon_url` failed. Remediation: Lite → `reachy-mini-daemon`; Wireless → SSH to the host and check `systemctl status reachy-mini-daemon.service`.
3. **Platform detect** — read `wireless_version`, `version`, and the `backend_status` shape from `/api/daemon/status` to classify the runtime as `wireless`, `lite`, or `simulation`. Surface the detected profile in the report so the operator knows which Tier-1 tools (IMU, battery, audio) will be reachable.
4. **Audit-log path writable** — verify `~/.cache/reachy-mini-mcp/<YYYY-MM-DD>.log` is creatable. If permission denied: abort with the path and `mkdir -p` remediation. **Never** silently substitute `/tmp/`.
5. **Port free** (`http` mode only) — verify `mcp_port` is free on `127.0.0.1` via a TCP-bind probe. On conflict: abort and name the holder via `lsof -i :<port>` (when available); never pick a different port silently.
6. **No duplicate instance** — for `stdio` and `http` modes, check whether a `reachy-mini-mcp-server` process is already running (pid-file at `~/.cache/reachy-mini-mcp/server.pid` or an active socket on `mcp_port`). On conflict: abort and reference the existing instance; never start two parallel servers.

After all six gates pass, the skill is allowed to start the server.

## Workflow

```text
1. Pre-flight (six gates above). Abort on first failure with the gate name and a concrete remediation.
2. Start the server in the chosen mode:
   - stdio: the skill does NOT spawn the server itself. It records the start command
            (e.g., `reachy-mini-mcp-server --daemon-url <url> --log-level <level>`)
            and lets the MCP client spawn it via the config snippet (step 4).
   - http : spawn `reachy-mini-mcp-server --transport http --bind 127.0.0.1
                                          --port <mcp_port>
                                          --daemon-url <daemon_url>
                                          --log-level <log_level>`
            as a background process; wait 2 s for warm-up.
   - systemd: render `~/.config/systemd/user/reachy-mini-mcp.service` idempotently
              (preserve user edits if the user-unit file already exists; only update
              the lines this skill owns: ExecStart, Environment, Restart settings).
              Run `systemctl --user daemon-reload`, then
              `systemctl --user start reachy-mini-mcp.service`.
3. Health-check (skip for stdio — handled by the MCP client):
   - http : poll `GET http://127.0.0.1:<mcp_port>/health` (or equivalent), expect 200
   - systemd: poll the same endpoint after `systemctl --user is-active` returns active
   The server must respond within 5 s of the start command. If not, abort and surface
   the server's stderr log path.
4. Probe `tools/list` against the server. The response MUST list at least:
   - `health-check` (always present per the knowledge spec)
   - Tier-1 read tools: `daemon-status`, `state-full`, `app-lock-status`
   Anything less means the server installation is broken; abort and recommend
   reinstalling `reachy-mini-mcp-server`.
5. Generate the MCP-client config snippet for the chosen frontend (see next section).
6. If `write_config=true`: present the diff against any existing `mcpServers` entry
   in the frontend's config file; ask for explicit confirmation; then write. If
   `write_config=false` (default): output the snippet to the conversation only.
7. Surface the final report:
   - pre-flight summary (six lines, one per gate, ✓/✗)
   - run mode and how to stop the server
   - tool inventory delta (`got N tools, expected baseline ≥ 4`)
   - config snippet (or written-to path)
```

The skill **MUST** execute these steps strictly sequentially. No parallel work, no background wait without health-check.

## MCP-client configuration snippets

The skill emits a frontend-specific snippet that drops the new `mcpServers` entry into the right config file. The skill **MUST** always state the target file path before showing the snippet.

### `claude-desktop`

Target config file:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%/Claude/claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

Snippet (mode=stdio, the most common case):

```json
{
  "mcpServers": {
    "reachy-mini": {
      "command": "reachy-mini-mcp-server",
      "args": ["--daemon-url", "<daemon_url>", "--log-level", "<log_level>"],
      "env": {}
    }
  }
}
```

For `http` mode the entry uses `"url": "http://127.0.0.1:<mcp_port>"` instead of `command` + `args`.

### `claude-code`

Target config file: `~/.claude/settings.json` — `mcpServers` block, same schema as Claude Desktop.

Note: Claude Code can also pick up `.claude/mcp.json` per-project; for a project-local registration, the skill **MAY** offer that path as an alternative, but **MUST** ask before writing into a project-local file.

### `cursor`

Target config file: Cursor's MCP settings file — path is version-dependent (typically reachable via Cursor → Settings → MCP). The skill **MUST** point the user at the in-app settings rather than guess the file path.

Snippet schema mirrors Claude Desktop (`command` + `args` for stdio, `url` for http).

### `generic`

For any other MCP-capable client (`stdio`-mode snippet, with a one-line comment explaining the schema):

```json
{
  "command": "reachy-mini-mcp-server",
  "args": ["--daemon-url", "<daemon_url>", "--log-level", "<log_level>"]
}
```

Plus a pointer to the MCP spec (<https://modelcontextprotocol.io>) so the operator can adapt the schema to a less common frontend.

## Run-mode reference

| Mode | Run shape | Lifecycle | Stop |
|---|---|---|---|
| **`stdio`** (default) | server is spawned as a subprocess of the MCP client (stdin/stdout pipe) | lives with the MCP-client session | quit the MCP client; the OS reaps the subprocess |
| **`http`** | server runs as an own local process; HTTP listener on `127.0.0.1:<mcp_port>` | own process, user-controlled | `pkill -f reachy-mini-mcp-server` or `kill $(cat ~/.cache/reachy-mini-mcp/server.pid)` |
| **`systemd`** | server runs as a systemd user-unit at `~/.config/systemd/user/reachy-mini-mcp.service`; auto-restart on failure | daemonized, optionally always-on | `systemctl --user stop reachy-mini-mcp.service`; disable with `systemctl --user disable reachy-mini-mcp.service` |

The skill **MUST** name the stop command in the report for the chosen mode — no run-mode without a clear stop path.

## Report format

After every successful run the skill emits a five-section report:

1. **Pre-flight summary** — six lines, ✓/✗ per gate; first ✗ aborted the run, the skill never produces a report after a green path with a hidden gate failure
2. **Run mode and status** — `stdio` (config-only) / `http <port> PID <pid>` / `systemd active since <timestamp>`, plus the stop command
3. **Tool inventory** — `expected ≥ 4 baseline tools, got <N>`; a mismatch is a hard fail and surfaces the missing tool names
4. **Config snippet** — frontend-specific, with the target file path stated above the snippet; either inline or written-to path (depending on `write_config`)
5. **Next steps** — pointer to `reachy-mini-sdk` for SDK-idiom follow-ups, to `reachy-mini-on-device` for hardware-side validation when the operator wants to actually exercise the server's tools

## Gotchas

- **`reachy-mini-mcp-server` runs in its own venv when installed via `uv tool install`.** A `pip list` in your project venv won't show it; pre-flight gate 1 uses `reachy-mini-mcp-server --version` rather than a Python-import check for that reason.
- **Wireless mDNS lookups are slow on the first call.** Pre-flight gate 2's 3-second budget is enough for steady-state but might miss the first cold-start probe after a daemon boot; if gate 2 fails on a freshly-booted Wireless, retry once before declaring the daemon unreachable.
- **`backend_status.ready=false` is not a pre-flight blocker.** Per [`reachy-mini/motor-positions`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/motor-positions/de.md) Layer 5 the daemon's `_status.ready` flag is desync'd from the actual loop in `reachy_mini==1.7.1`; the skill checks only `daemon/status.state == "running"` for pre-flight liveness.
- **Cursor's MCP config file path moves between versions.** The skill points at in-app settings rather than guessing; this is intentional. A future Cursor-Settings-Spec could pin the path, but not from this skill.
- **systemd user-units are *not* the same as session-D-Bus services.** A user-unit started via `systemctl --user start` survives logout only if `loginctl enable-linger <user>` is set. The skill mentions this in the systemd-mode report when relevant; it does **not** flip the linger flag itself.
- **`stdio` mode does not actually spawn anything during this skill's run.** The config snippet tells the MCP client how to spawn the server when the client itself starts; the skill's job is to verify the *recipe*, not to run the server. The health-check in step 3 of the workflow is skipped for `stdio` for that reason.

## Boundaries to neighbouring artifacts

- knowledge of MCP-server tool inventory and security model → [`reachy-mini/mcp-server`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/mcp-server/de.md)
- SDK idioms (method choice, safe-torque) for MCP-client follow-up questions → [`reachy-mini-sdk`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-mini-sdk/SKILL.md)
- read-only state snapshot of the running daemon → [`reachy-mini-inspect`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-mini-inspect/SKILL.md) (parallel plugin-skill distribution of the same Tier-1 reads)
- live-trial of a behavior against the actual hardware → agent [`reachy-mini-on-device`](https://github.com/nolte/claude-reachy-mini/blob/develop/agents/reachy-mini-on-device.md)
- starting an installed Reachy app on the daemon → [`reachy-mini-start`](https://github.com/nolte/claude-reachy-mini/blob/develop/skills/reachy-mini-start/SKILL.md) (different surface; runs an app, not the MCP server)
- host provisioning for an always-on systemd MCP service → [`reachy-mini/host-provisioning`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/reachy-mini/host-provisioning/de.md)

External canonical sources:

- Model Context Protocol spec — <https://modelcontextprotocol.io>
- Model Context Protocol Python SDK — <https://github.com/modelcontextprotocol/python-sdk>
- Claude Desktop / Claude Code MCP docs — <https://docs.anthropic.com/en/docs/claude-code/mcp>
- systemd user-units reference — <https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html>
