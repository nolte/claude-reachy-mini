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
- **Short, low-latency operation** — a handful of REST calls (`GET /api/daemon/status`, `GET /api/apps/list-available/installed`, `GET /api/daemon/robot-app-lock-status`, `POST /api/apps/start-app/<name>`, `GET /api/apps/current-app-status`); no large-volume reads or installer output.
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

1. Resolve the daemon endpoint from `(device, platform)`. The Pollen daemon listens on `:8000` by default. Confirm reachability with a single low-cost call to `GET /api/daemon/robot-app-lock-status`. On 5xx / connection refused / timeout, abort with the diagnostic.
2. Check the **hardware-daemon layer** with `GET /api/daemon/status`. The response includes `state` (`running` / `stopped` / …). If `state != "running"`, abort with: "The Pollen hardware daemon on `<device>` reports `state=<state>`. Apps cannot connect to `localhost` until it is `running`. Bring it up via `POST /api/daemon/start` (outside this skill) and re-invoke." Do **not** start it from inside this skill — Hard rule 3.
3. List installed apps via `GET /api/apps/list-available/installed` (the OpenAPI route is `/api/apps/list-available/{source_kind}`; the `installed` filter is the local entry-point view). Confirm `app_name` is present. If not, abort with: "App `<name>` is not in the daemon's entry-point catalog. Run `reachy-mini-deploy` first, or refresh the daemon's view." Do not attempt to install.
4. Read the app-lock state: `GET /api/daemon/robot-app-lock-status`. Branch on result:
   - `state: free` → safe to start, proceed.
   - Lock held by `app_name` already → tell the user the app is already running, exit cleanly (no-op success).
   - Lock held by a different app → branch on `if_busy`.

## Workflow

```text
1. Pre-flight (above).
2. Sticky-error check (BEFORE any start-app call):
     If GET /api/apps/current-app-status returns state == "error" from a prior run:
       - surface the prior error to the user (traceback snippet + identified root cause hint)
       - ask: "Found a stale error memorial for <prior_app>. Run `stop-current-app` to clear it
                and proceed with starting <app_name>? [yes/no]"
       - yes  → POST /api/apps/stop-current-app, verify current-app-status == null, then continue
       - no   → exit; the daemon will keep rejecting start-app with HTTP 400 until cleared
3. If lock held by another app and if_busy=prompt:
     ask the user: "App <holder_name> is currently running. Stop it and start <app_name>? [yes/no]"
     - yes  → POST /api/apps/stop-current-app, wait for state==free (poll every 1s, up to 10s),
             then POST /api/apps/start-app/<app_name>
     - no   → exit with no state change
4. If if_busy=replace and the user has explicitly confirmed at the call site:
     POST /api/apps/stop-current-app, wait, POST /api/apps/start-app/<app_name>
5. If if_busy=abort and lock is held by another app:
     exit with a clear message naming the holder; no state change
6. Issue POST /api/apps/start-app/<app_name>.
   If the response is HTTP 400 {"detail":"An app is already running"}:
     - this means the sticky-error check (step 2) missed a memorial, or a concurrent change occurred
     - offer the user a single recovery: POST /api/apps/stop-current-app, then re-issue start
     - if the second start also fails with HTTP 400, exit and surface both response bodies
7. After start, poll GET /api/apps/current-app-status for up to 15 s at 1-s intervals.
   Three terminal branches:
     - state == "running" and info.name == <app_name>  → SUCCESS, continue to (8)
     - state == "error"                                → FAIL: surface the error field verbatim
                                                          (include the traceback snippet so the user
                                                           sees the actual root cause). If the error
                                                           contains "Could not connect to daemon on
                                                           localhost", point at Pre-flight step 2
                                                           (hardware daemon down). DO NOT auto-retry —
                                                           one recovery per invocation is the limit.
     - timeout (state still "starting" after 15 s)     → surface the timeout and exit; the user
                                                          can decide whether to wait or investigate.
8. Surface a one-line confirmation to the user: "Started <app_name> on <device>; FastAPI settings UI on http://<device>:<custom_app_url_port> if the app exposes one."
```

> **REST contract — verified against Wireless firmware 1.7.1 on 2026-05-12.** The endpoint paths above are observed and confirmed (`/api/daemon/status`, `/api/daemon/robot-app-lock-status`, `/api/apps/list-available/installed`, `/api/apps/current-app-status`, `/api/apps/start-app/{name}`, `/api/apps/stop-current-app`). For full response shapes see `spec/claude/reachy-mini-start/de.md` → "Verifizierter REST-Vertrag". The Lite contract is **not yet** verified.

## Gotchas

- **The Pollen daemon is two layers, not one.** The HTTP-API layer on `:8000` answers REST calls and spawns app processes; a separate hardware-subprocess layer is what `ReachyMini()` connects to via `localhost` IPC. Both must be `running` for an app to come up. If only the HTTP layer is up, `start-app` returns `HTTP 200 state=starting`, the spawned app crashes inside `_initialize_client` with `ConnectionError: Could not connect to daemon on localhost`, and `current-app-status.state` transitions to `error` while `robot-app-lock-status` auto-releases to `free`. Pre-flight step 2 catches this proactively. (Observed on Wireless firmware 1.7.1, 2026-05-12.)
- **`current-app-status` is `null` when nothing is running and an object otherwise.** Do not assume the response is always an object — empty-state responses are bare JSON `null`. Parse defensively (`d = response or {}`).
- **Lock-state and current-app-state can disagree after a crash.** `robot-app-lock-status=free` does NOT imply "no recent app activity" — `current-app-status` may still report `state=error` from the last attempt until a `stop-current-app` call clears it. Read both endpoints before making decisions.
- **The `start-app` endpoint reads `current-app-status`, not `lock-status`, when checking for conflicts.** A sticky `state="error"` memorial therefore makes the daemon reject new starts with `HTTP 400 "An app is already running"` even while the lock is `free`. The fix is always `POST /api/apps/stop-current-app` first. (Observed on Wireless firmware 1.7.1, 2026-05-12 — recovered the very same session.)
- **Lock-status values observed so far**: `free`, `local_app` (the entry-point name appears in `holder_name`). Marketplace apps likely produce a different state value (`hf_space`?) — not yet observed.
- **`backend_status.ready=false` is not a pre-flight blocker.** Only `daemon/status.state == "running"` is required for `start-app` to succeed. The `ready` flag appears to indicate whether the hardware backend has emitted its first heartbeat; an app started successfully in our verification while `ready=false`.
- **The Pollen daemon enforces a single-app invariant.** Two apps cannot run concurrently. The `if_busy` branching is therefore not a convenience — it is the only way to switch apps cleanly.
- **The daemon's entry-point view can be stale right after a deploy.** If `reachy-mini-deploy` just finished and `app_name` is not yet visible in `/api/apps/list-available/installed`, that is usually a fresh-install caching delay. Tell the user; do not retry silently more than once at a 2-second interval.
- **`custom_app_url` is owned by the app, not by the daemon.** The skill cannot guarantee the FastAPI settings UI is reachable just because the daemon reports the app as running. Surface the URL hint based on what the app declares in its `ReachyMiniApp` subclass, but do not block on its reachability.
- **`reachy-mini-start` is not a replacement for the on-device agent.** A short start + confirmation is enough for ad-hoc use; for any test that wants to assert pose, sensor data, or shutdown semantics, the user should reach for `reachy-mini-on-device` instead.

## Boundaries to neighbouring artifacts

- Code deployment / install → `reachy-mini-deploy` (agent)
- Live trial run with telemetry → `reachy-mini-on-device` (agent)
- App skeleton / scaffold → `app-scaffold` (skill)
- SDK idioms inside the app → `reachy-mini-sdk` (skill)
- HA integration of the running app → `home-assistant-bridge` (skill)
