---
name: reachy-mini-start
description: Start an already-installed Reachy Mini app via the Pollen daemon's REST surface, with explicit user prompts when another app currently holds the daemon's app-lock or when the requested app is not yet visible in the daemon's entry-point catalog. Use when the user says "start the app on the reachy", "bring this app online", "let reachy run the app now", "auf dem Reachy starten", "App auf dem Roboter starten", or "App jetzt online bringen". Do NOT use to deploy / install code (that's the `reachy-mini-deploy` agent), to run a full live trial with telemetry (that's the `reachy-mini-on-device` agent), to write motion or behavior code (that's `reachy-mini-sdk` and `app-scaffold`), or to publish to Hugging Face. Confirms intent with the user before performing any takeover or daemon-side state change.
tags: [reachy-mini, deploy]
---

# Reachy Mini Start

Spec: <https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-start/de.md> (DE canonical) / [`en.md`](https://github.com/nolte/claude-reachy-mini/blob/develop/spec/claude/reachy-mini-start/en.md).

## Skill-vs-Agent rationale

This is a **skill** rather than an agent because the operation is short, mid-flow user confirmation is the whole point, and the output flows back into the main conversation naturally:

- **Mid-flow user approval is required** — when another app currently holds the Pollen daemon lock, the only safe path is to ask the user before stopping it. An agent (fire-and-forget) cannot surface such a question back to the parent thread.
- **Short, low-latency operation** — a handful of REST calls (`GET /api/apps/installed`, `GET /api/daemon/robot-app-lock-status`, `POST /api/apps/start-app/<name>`); no large-volume reads or installer output.
- **Output flows naturally** — the user sees "App `X` started, daemon confirmed running" inline; no need to isolate a structured PASS/FAIL report.
- **Counter-dimension** — context-window protection (the agent-side argument) does not apply; the operation is a few lines of stdout, not hundreds.

## When this skill activates

Use this skill when the user wants to:

- start an already-installed Reachy Mini app on a real device
- "bring online" an app that has just been deployed via the `reachy-mini-deploy` agent
- swap from one running Pollen-daemon app to another (with the user's explicit confirmation)

## When NOT to activate

- deploying / installing code on the device → `reachy-mini-deploy` (agent)
- live-trial run with telemetry / structured PASS-FAIL → `reachy-mini-on-device` (agent)
- developing the behavior or app skeleton → `reachy-mini-sdk` / `app-scaffold` (skills)
- publishing to Hugging Face → `reachy-mini-app-assistant publish` directly
- restarting / reconfiguring the Pollen daemon itself — out of scope

## Hard rules

1. **Never auto-stop a running third-party app.** When the daemon's app-lock is held by an app the user did not name explicitly, ask once with the holder's name and the proposed action, and wait for confirmation. A "yes" to the previous run does not carry over.
2. **Never install or modify code from inside this skill.** Starting only — if the requested app is not visible in `entry_points(group='reachy_mini_apps')`, surface that fact and point at `reachy-mini-deploy`. Don't try to "auto-fix" by deploying.
3. **Never restart the Pollen daemon.** If the daemon is unresponsive or its entry-point view is stale, surface that as a diagnostic and stop. The user (or a future bring-up skill) handles the daemon lifecycle.
4. **Read SSH credentials from environment / `ssh_config`, never from a plaintext argument.** Don't echo them in any response or trace.
5. **Confirm the platform before starting.** On Lite the daemon runs on the host PC, not on the Reachy itself; on Wireless it runs on the robot. The endpoint and SSH host follow from this — no platform default that hides the difference.

## Inputs

| Field | Required | Default | Notes |
|---|---|---|---|
| `app_name` | yes | — | The app's installed name. Must match an entry-point name in `reachy_mini_apps`. |
| `device` | yes | — | SSH host / Pollen-daemon endpoint. Wireless: the robot. Lite: the host PC. |
| `platform` | yes | — | `wireless` or `lite`. Same constraint as `reachy-mini-deploy`: `simulation` is rejected. |
| `if_busy` | no | `prompt` | `prompt` (default): ask the user; `abort`: refuse silently when busy; `replace`: stop the held app and start the requested one *only after* confirming with the user. The skill never auto-replaces. |

## Pre-flight (every run, in order — abort on first failure)

1. Resolve the daemon endpoint from `(device, platform)`. The Pollen daemon listens on `:8000` by default. Confirm reachability with a single low-cost call to `GET /api/daemon/robot-app-lock-status` (or the equivalent — verify on first hardware contact). On 5xx / connection refused / timeout, abort with the diagnostic.
2. List installed apps: `GET /api/apps/installed` (verify endpoint name on first hardware contact). Confirm `app_name` is in the list. If not, abort with: "App `<name>` is not in the daemon's entry-point catalog. Run `reachy-mini-deploy` first, or refresh the daemon's view." Do not attempt to install.
3. Read the app-lock state: `GET /api/daemon/robot-app-lock-status`. Branch on result:
   - `state: free` and no `current-app-status` → safe to start, proceed.
   - Lock held by `app_name` already → tell the user the app is already running, exit cleanly (no-op success).
   - Lock held by a different app → branch on `if_busy`.

## Workflow

```text
1. Pre-flight (above).
2. If lock held by another app and if_busy=prompt:
     ask the user: "App <holder_name> is currently running. Stop it and start <app_name>? [yes/no]"
     - yes  → POST /api/apps/stop-current-app, wait for state==free (poll every 1s, up to 10s),
             then POST /api/apps/start-app/<app_name>
     - no   → exit with no state change
3. If if_busy=replace and the user has explicitly confirmed at the call site:
     POST /api/apps/stop-current-app, wait, POST /api/apps/start-app/<app_name>
4. If if_busy=abort and lock is held by another app:
     exit with a clear message naming the holder; no state change
5. After start, verify: GET /api/apps/current-app-status returns the requested app and state==running
6. Surface a one-line confirmation to the user: "Started <app_name> on <device>; FastAPI settings UI on http://<device>:<custom_app_url_port> if the app exposes one."
```

> ⚠ TBD: validate against real hardware — every concrete REST endpoint and lock-state field above is a best-effort design until verified on a physical Reachy. Confirm against the live SDK and device on first contact, and update this skill (and the spec) accordingly.

## Gotchas

- **The Pollen daemon enforces a single-app invariant.** Two apps cannot run concurrently. The `if_busy` branching is therefore not a convenience — it is the only way to switch apps cleanly.
- **The daemon's entry-point view can be stale right after a deploy.** If `reachy-mini-deploy` just finished and `app_name` is not yet visible in `/api/apps/installed`, that is usually a fresh-install caching delay. Tell the user; do not retry silently more than once at a 2-second interval.
- **`custom_app_url` is owned by the app, not by the daemon.** The skill cannot guarantee the FastAPI settings UI is reachable just because the daemon reports the app as running. Surface the URL hint based on what the app declares in its `ReachyMiniApp` subclass, but do not block on its reachability.
- **`reachy-mini-start` is not a replacement for the on-device agent.** A short start + confirmation is enough for ad-hoc use; for any test that wants to assert pose, sensor data, or shutdown semantics, the user should reach for `reachy-mini-on-device` instead.

## Boundaries to neighbouring artifacts

- Code deployment / install → `reachy-mini-deploy` (agent)
- Live trial run with telemetry → `reachy-mini-on-device` (agent)
- App skeleton / scaffold → `app-scaffold` (skill)
- SDK idioms inside the app → `reachy-mini-sdk` (skill)
- HA integration of the running app → `home-assistant-bridge` (skill)
