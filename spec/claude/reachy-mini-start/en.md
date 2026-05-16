# Start Skill for Reachy Mini Apps

Status: draft

## Context
A Reachy Mini app that has been deployed onto the Pollen daemon's environment (typically via the `reachy-mini-deploy` agent) is registered as an entry point but is not yet running. Bringing the app online — i.e. instructing the Pollen daemon to actually invoke its `ReachyMiniApp.run()` — is a short, latency-light operation that is dominated by an interactive concern: the daemon enforces a single-app invariant, so any app already running has to be stopped first, and that stop is a state-changing decision the user must approve. The `reachy-mini-start` skill encapsulates this short start lifecycle in the main thread, with explicit user prompts at the only ambiguity (lock contention). It starts, it does not deploy and it does not run a full trial — those are the `reachy-mini-deploy` agent and the `reachy-mini-on-device` agent.

## Goals
- Bring a deployed app online on a real Reachy in a single skill invocation
- Make the single-app-invariant trade-off explicit at the user surface — never silently stop a running third-party app
- Confirm the daemon actually transitioned the app to `running` after the start call, not just that the start request was accepted
- Stay narrow: start and verify-running, no deploy, no trial, no daemon lifecycle work
- Compose cleanly with `reachy-mini-deploy`: a typical end-to-end is "deploy → start" or "deploy → on-device trial"; the skill is one half of that pair

## Non-Goals
- Deploying / installing code on the device — owned by `reachy-mini-deploy` (agent)
- Running a full live trial with telemetry — owned by `reachy-mini-on-device` (agent)
- Behavior / motion code editing — `reachy-mini-sdk`, `app-scaffold`
- Pollen daemon restart, reload, or reconfiguration — out of scope, never performed by this skill
- Hugging Face publishing — `reachy-mini-app-assistant publish`, or a future `reachy-app-publish-hf`
- Persistent supervisor / auto-restart — the skill is single-shot

## Skill-vs-Agent rationale
This concern is modelled as a **skill** rather than an agent because the skill-bias dimensions from `nolte-shared/spec/claude/skill-vs-agent/` apply directly:

- **Mid-flow user approval is required** — when another app holds the daemon lock, the only safe path is to ask the user. An agent (fire-and-forget) cannot route this question back to the parent thread.
- **Output flows naturally back to the conversation** — a one-line "started `<app>` on `<device>`" plus a state confirmation; no need for the structured-report boundary that an agent provides.
- **Short, low-volume operation** — a handful of REST calls; no installer logs, no rsync transfers, no telemetry samples. Context-window protection is not a load-bearing concern here.
- **Persists across the conversation** — the user may invoke `reachy-mini-start` more than once in a session (start app A, swap to app B, swap back), each invocation is a fresh decision point.
- **Counter-dimension** — tool restriction (the agent-side argument) does not apply meaningfully; the skill needs Bash for `curl` / `ssh`, which is the same surface the main thread already has.

## Requirements

### Inputs
- **MUST** accept `app_name` — the installed name as registered in `entry_points(group='reachy_mini_apps')`
- **MUST** accept `device` — an SSH host / daemon endpoint. For `wireless` this is the robot itself; for `lite` it is the host PC
- **MUST** accept `platform` — exactly one of `wireless` or `lite`. `simulation` **MUST** be rejected with a clear error
- **MAY** accept `if_busy` — `prompt` (default), `abort`, or `replace`. `prompt` asks the user; `abort` refuses silently when busy; `replace` stops the held app and starts the requested one **only after** the caller has explicitly confirmed at the invocation site
- **MUST NOT** accept plaintext credentials in any input — credentials come exclusively from `ssh_config` / environment

### Lifecycle
- **MUST** run pre-flight checks before any state change: HTTP-API reachability, hardware-daemon state, installed-apps listing, current app-lock state
- **MUST** treat the Pollen daemon as two independent layers: the **HTTP-API layer** (FastAPI on `:8000`, answering `/api/apps/*` and `/api/daemon/*`) and a **hardware subprocess** the app talks to via IPC once it instantiates `ReachyMini()`. Apps need both running simultaneously
- **MUST** read `GET /api/daemon/status` before the start call and abort cleanly when `state != "running"`. Surface a clear diagnostic (reported state, suggested daemon bring-up via `POST /api/daemon/start` — **outside** this skill's scope) and stop; **MUST NOT** start the hardware daemon itself — that belongs to the daemon-lifecycle scope
- **MUST** abort cleanly when `app_name` is not visible in the daemon's entry-point catalog and surface `reachy-mini-deploy` as the natural follow-up; **MUST NOT** attempt to install
- **MUST** branch on the lock state explicitly:
  - `free` → start
  - already held by `app_name` → no-op success (the app is already running)
  - held by a different app → branch on `if_busy`
- **MUST**, when `if_busy=prompt`, ask the user once with the holder name and the proposed action, and wait for an explicit confirmation; **MUST NOT** carry a "yes" from a previous invocation forward
- **MUST**, when `if_busy=replace`, accept that the caller has already confirmed at the invocation site; the skill does not double-prompt
- **MUST**, after starting, verify via `GET /api/apps/current-app-status` that the daemon reports the requested app as `running`; a 200 OK on the start call alone is **not** sufficient confirmation. Poll budget is 5–15 s with 1-s intervals
- **MUST** treat a post-start `state=error` as a first-class failure branch: the daemon may successfully spawn the app process, the process may then fail inside `ReachyMini()` initialization, and the daemon reports `state=error` with a traceback in `current-app-status.error`. The most common root cause is a hardware-daemon layer that was not `running`. The skill **MUST** pass the `error` field through to the user, point at the daemon lifecycle as the next step, and **MUST NOT** silently re-issue the start
- **MUST** recognise that `current-app-status.state="error"` is **sticky**: the daemon does not clean up the app-slot memorial on its own. As long as the memorial stands, `POST /api/apps/start-app/<name>` rejects every new start with **`HTTP 400 "An app is already running"`** — even when `robot-app-lock-status.state == "free"`. The daemon therefore reads `current-app-status` as the conflict source, not `lock-status`. The skill **MUST** interpret both endpoints together and **MUST NOT** conclude "no conflict" from `lock-status` alone
- **MUST** offer a recovery sequence when a sticky `state=error` memorial is found: ask the user for confirmation → call `POST /api/apps/stop-current-app` → verify `GET /api/apps/current-app-status` now returns `null` → re-issue `POST /api/apps/start-app/<name>`. The recovery is allowed **exactly once** per skill invocation; a second `state=error` exits the skill and surfaces the root cause rather than entering a retry loop
- **MUST** surface a one-line confirmation to the user with the started app name and the device; if the app declares a `custom_app_url`, mention it as a hint without blocking on its reachability
- **MUST NOT** restart, reload, or reconfigure the Pollen daemon under any circumstance — neither the HTTP-API layer nor the hardware subprocess via `POST /api/daemon/start` / `POST /api/daemon/stop`
- **MUST NOT** force-stop a running third-party app without explicit user confirmation
- **MUST NOT** install, modify, or copy code as part of this lifecycle

### Output
- **MUST** return a one-line user-visible confirmation in the natural conversation flow
- **SHOULD** include the daemon-reported app state and any actionable hint (e.g. "FastAPI settings UI at `http://<device>:<port>`")
- **MUST NOT** return raw daemon logs, dependency traces, or credentials

### Security and secrets
- **MUST** read SSH / device credentials from environment or `ssh_config`
- **MUST NOT** write credentials into any user-visible response
- **SHOULD** keep SSH host-key verification on; surface the fingerprint rather than auto-accepting on first connect
- **MUST** apply platform-aware endpoint resolution: on Wireless the daemon endpoint is the robot, on Lite it is the host PC

### Boundaries
- **SHOULD** point at `reachy-mini-deploy` whenever the requested app is not yet in the daemon's entry-point catalog
- **SHOULD** point at `reachy-mini-on-device` whenever the user wants to test, observe, or assert behavior — not just bring the app online
- **SHOULD** point at `reachy-mini-sdk` and `app-scaffold` when the issue is upstream of the start call (broken behavior code, broken app skeleton)
- **MUST NOT** duplicate content from those artifacts — this skill is a daemon-side start/verify flow, not a knowledge base
- **MUST NOT** invoke other skills as a chain inside this skill's logic; it is a leaf operation

## Acceptance Criteria
- [ ] The skill rejects `platform=simulation` with a clear error pointing at the on-device agent
- [ ] The skill checks `GET /api/daemon/status` during pre-flight and aborts when `state != "running"`, with a diagnostic pointing at the daemon lifecycle (`POST /api/daemon/start`, **not** executed inside this skill)
- [ ] The skill aborts when `app_name` is not in `entry_points(group='reachy_mini_apps')` and surfaces `reachy-mini-deploy` as the next step
- [ ] The skill never auto-stops a running third-party app; `if_busy=prompt` always asks first, `if_busy=replace` requires upstream confirmation, `if_busy=abort` exits without state change
- [ ] The skill never restarts the Pollen daemon — neither the HTTP-API layer nor the hardware subprocess
- [ ] The skill never installs, modifies, or copies code
- [ ] After a successful start, the skill verifies via the daemon's status endpoint that the requested app is reported as `running` — a 200 OK on `start-app` alone is not sufficient
- [ ] When `current-app-status.state == "error"` after the start, the skill passes the `error` field including a traceback snippet through to the user, identifies the `ConnectionError: Could not connect to daemon on localhost` pattern as a hardware-layer diagnostic, and does **not** silently re-issue the start
- [ ] When pre-flight finds a sticky `current-app-status.state == "error"` from a prior run and the `start-app` call is therefore rejected with `HTTP 400 "An app is already running"`, the skill offers a one-shot recovery sequence `stop-current-app → start-app` after user confirmation; a second failure exits the skill without further retries
- [ ] The skill interprets `robot-app-lock-status` and `current-app-status` together and does not infer "no conflict" from `lock-status.state == "free"` alone
- [ ] The skill surfaces a one-line confirmation that includes the started app, the device, and any FastAPI hint the app declares
- [ ] The skill exists as `skills/reachy-mini-start/SKILL.md` with valid frontmatter — `name: reachy-mini-start`, `description`, optional `tags`
- [ ] The `description` activates on phrasings like "start the app on the reachy", "bring this app online", "let reachy run the app now", "auf dem Reachy starten", "App auf dem Roboter starten"
- [ ] A skill-vs-agent rationale is visible in the skill body (at least mid-flow user approval, output flows naturally, short low-volume operation)
- [ ] References to `reachy-mini-deploy`, `reachy-mini-on-device`, `app-scaffold`, `reachy-mini-sdk` are visible in the body
- [ ] Statements without hardware verification carry a `⚠ TBD: validate against real hardware` marker; the verified REST contract (section below) does **not** carry this marker

## Verified REST contract

Verification basis: **Reachy Mini Wireless, firmware 1.7.1, `reachy-mini.local`, 2026-05-12.** The endpoint paths and response shapes below are observed and confirmed and replace the earlier TBD assumptions. The contract is **not yet** verified for Lite (see Open Questions).

| Method + Path | Response (example) | Use in the skill |
|---|---|---|
| `GET /api/daemon/status` | `{"type":"daemon_status","state":"running"\|"stopped","wireless_version":true,"backend_status":{"ready":true\|false,"motor_control_mode":"enabled","control_loop_stats":{...},"error":null},"wlan_ip":"...","version":"1.7.1",...}` | Pre-flight hardware-layer check (MUST requirement). Only `state` is pre-flight relevant; `backend_status.ready=false` is **not** a show-stopper (observed: start succeeded at `state="running" + backend_status.ready=false`) |
| `GET /api/daemon/robot-app-lock-status` | `{"state":"free","holder_name":null}` or `{"state":"local_app","holder_name":"<app>"}` | Pre-flight lock branching. Observed states: `free`, `local_app`. Further values (presumably `hf_space` or similar for marketplace apps) not yet observed |
| `GET /api/apps/list-available/installed` | Array of `{"name":"<entry_point>","source_kind":"installed","description":"","url":null,...}` | Pre-flight entry-point lookup |
| `GET /api/apps/current-app-status` | `null` (nothing started) **or** `{"info":{"name":"...","source_kind":"installed",...},"state":"starting"\|"running"\|"error","error":null\|"<traceback>"}`. **State `error` is sticky** — persists until a `stop-current-app` call | Post-start verification and failure diagnostic; primary conflict source for `start-app` |
| `POST /api/apps/start-app/{app_name}` | Success: `{"info":{...},"state":"starting","error":null}`. Conflict: `HTTP 400 {"detail":"An app is already running"}` (see failure signature below) | Start trigger |
| `POST /api/apps/stop-current-app` | `null` (HTTP 200). Also clears `current-app-status` back to `null` | Soft-stop of a holding app **and** cleanup of a sticky `state=error` memorial |

### Observed failure signatures

**Signature 1 — hardware daemon `stopped` at start attempt.** App is spawned, fails inside `ReachyMini()._initialize_client`:

```text
current-app-status.state == "error"
current-app-status.error  == "Process exited with code 1\n
                                ...
                                File \".../reachy_mini/reachy_mini.py\", line 441, in _initialize_client
                                  raise ConnectionError(
                              ConnectionError: Could not connect to daemon on localhost.
                              Is the Reachy Mini daemon running?"
```

After this crash, `robot-app-lock-status` is `free` again (the daemon releases the lock) but `current-app-status` retains the error state until the next `stop-current-app` call.

**Signature 2 — sticky `current-app-status.state="error"` blocks subsequent start attempts.** Even when `robot-app-lock-status.state == "free"`, the `start-app` endpoint replies:

```text
POST /api/apps/start-app/<name>
→ HTTP 400 {"detail":"An app is already running"}
```

The daemon therefore reads `current-app-status` as the conflict source, not `lock-status`. Recovery: call `POST /api/apps/stop-current-app` (which clears the memorial), then start again. This sequence is specified in the Lifecycle as a one-shot auto-recovery path.

The skill **MUST** therefore interpret the lock state and the current-app state together, not each in isolation.

## References
- App lifecycle contract: <https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/apps/manager.py>
- Daemon REST surface: <https://github.com/pollen-robotics/reachy_mini/tree/main/src/reachy_mini/daemon>
- Pollen `AGENTS.md`: <https://github.com/pollen-robotics/reachy_mini/blob/main/AGENTS.md>
- Sibling agent for code deployment: `agents/reachy-mini-deploy.md`
- Sibling agent for live trial runs: `agents/reachy-mini-on-device.md`
- Hardware verification run (first live trial of the skill): Reachy Mini Wireless, firmware 1.7.1, `reachy-mini.local`, 2026-05-12 — endpoint paths and failure signatures from this run flowed into "Verified REST contract"

## Open Questions
- For Lite: is the daemon's REST surface identical to Wireless's, or does the host-PC daemon expose a different path prefix? Confirm on first Lite hardware contact — the Wireless contract is now verified (see "Verified REST contract")
- Should the skill expose a "start-and-wait-for-app-ready" flag that polls the app's own `custom_app_url` until it returns 200, or is the daemon-side `running` confirmation enough? Leaning: enough; app-side reachability is the consumer's concern, not the daemon's contract
- Should `if_busy=replace` carry through to a future composite skill ("deploy then start") so the user only confirms once for both halves? Defer until the composite skill exists
- Should the skill be allowed to invoke a separate `reachy-mini-bring-up` skill when the hardware daemon is `state="stopped"`, or does it remain a pure diagnostic? For now: pure diagnostic; escalate once the bring-up skill exists
- Are there `current-app-status.state` values beyond `null` / `starting` / `running` / `error`? Read the Pollen source at `src/reachy_mini/apps/manager.py` for the enumeration and pull it into the verified REST contract
- Are there `robot-app-lock-status.state` values beyond `free` / `local_app`? In particular, a dedicated value for marketplace apps (e.g. `hf_space`) is suspected. Verify on the first HF-Space-app start and add to the verified REST contract
- Which conditions flip `backend_status.ready` from `false` to `true`? In our verification an app started successfully at `state="running" + backend_status.ready=false`. If `ready=true` is a precondition for specific sensor-API access (camera, IMU), this should be offered as an optional "extended pre-flight" check in the skill
